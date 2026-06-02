# Architecture — what already exists vs. what to build

> A design note, not a spec. It was written by merging two views of one system —
> an idealised "here are the pieces you'd build greenfield" lens, and a "here is
> what actually runs on the host today" lens — and then checking every claim
> against the live machine with `grep`, `ls`, and `ip`. Where the two disagreed,
> the working system won.

This document is generalised from a real build. Concrete addresses, paths, and
host-specific percentages have been removed; the *structure* of the findings is
what reproduces, not the magnitudes.

---

## The one finding that matters

> **The system already enforces correctly. It just enforces in several hardcoded
> places — and the one file that looks like a "manifest" has zero consumers.**

When you run more than one local agent, you inevitably grow enforcement: the
search gate has an allow-list, the daemon has a tier policy, the run-script picks
a model and an egress, the rate-limiter checks a timestamp, the drift-checker
hashes a prompt file. Each of these is correct. The problem is that the *scope*
they enforce is written down in each of them separately. Change an agent's scope
and you must find and edit every site.

A manifest (`agents.json`) looks like the fix. But a manifest only helps once the
enforcers actually *read* it. It is very easy to add a clean schema file that
nothing consumes — a seventh source of truth instead of one. The acceptance test
for the whole effort is therefore a negative `grep`: after the migration, search
the enforcement code for the old hardcoded scope literals and find **zero**.

Everything else in this repo is a thin extension of that one move.

---

## The pieces, by status

The components below are labelled by the state they tend to be in *before* you
unify them — useful for deciding what to build first on your own system.

| Piece | What it is | Typical starting state | Build priority |
|---|---|---|---|
| **manifest** | one declarative `agents.json` the gate + model-select + rate-limit + egress all read | the file may exist, but nothing reads it | **keystone — build first** |
| **validator** | the gate consults the manifest (scope + expiry) *before* acting, not just a path check | before-act gating often already works, but reads hardcoded scope, not the manifest; no expiry field exists | **build, after the manifest** |
| **gate-log** | one queryable allow / deny / success / failed log | one host may log; the other often logs nothing | **build — small; defer a cross-host bridge** |
| **revoke** | `agent-revoke <id>` — atomic key + gate-entry teardown, idempotent, key-name-driven | usually a global killswitch exists; per-agent revoke does not | **build — small, medium risk; after the manifest** |
| **signed-intent** | HMAC over the audit rows → tamper-evident | the log is append-only but has no tamper-evidence | **defer** (see open questions) |
| **owner-view** | a tile showing passports + live scope + stamps + a revoke button | nothing | **defer — build last, only if the rest lands** |

### Why the manifest is first

Until `agents.json` is the single thing the enforcers read:

- the **gate-log** records decisions made by scattered logic (you log *what*, not
  *against which declared scope*);
- **revoke** can't revoke by name (there's no canonical agent list);
- the **validator** has no manifest to validate against, and no place for a
  `valid_until` field;
- the **owner-view** has nothing structured to display.

Build the manifest, prove with `grep` that consumers read it and nothing reads the
old copies, and the other four become small.

---

## Dependency order

```
manifest  (keystone — everything below reads from it)
   |
   +-- validator   (gate reads manifest scope + valid_until before acting)   <- highest-value follow-on
   +-- revoke      (key-name-driven, idempotent, reads the manifest agent list)
   +-- gate-log    (can start independently; emits one row per decision)
   +-- signed-intent (HMAC over the gate-log rows; needs the manifest for a canonical key home)
   |
   +-- owner-view  (thin read layer over manifest + gate-log; build LAST)
```

---

## What to deliberately *not* build

The most valuable output of the merge was a list of things to skip:

- **Do not run a generated "deploy" script if a live, hand-verified gate already
  exists.** In the real build, the undeployed deploy artifact carried three
  confirmed regressions (a dead path from the other OS, an allow-list entry that
  contradicted the tier policy, and a non-atomic key write that would duplicate
  the live key). The live gate was strictly safer on every axis. The single
  biggest risk reduction was *not deploying.* Delete the stale artifact so it
  can't misfire.
- **Do not touch the live `authorized_keys` until the manifest-driven path is
  proven**, and keep a second SSH session open while you do — that file is
  irreversible-without-a-fallback.
- **Do not build per-agent WireGuard revoke while there is only one agent.** It's
  a stub until the manifest issues per-agent identities with dedicated egress.

---

## Honest open questions

- **Do tamper-evident audit and the owner-view tile earn their keep for a
  single user?** HMAC-signing the log defends against an attacker who already
  owns the box — a near-empty threat model for one user. The owner-view has
  nothing to show until the manifest and the log both exist. Both are
  spec-completeness, not user value, until you are more than one user.
- **Should a permissions manifest carry personality names?** A council UX can
  have flavour; an `agents.json` that gates real calls probably should not. This
  repo keeps functional role names (`researcher`, `structure`, `lifecycle`,
  `compression`, `orchestrator`) in the enforcement artifact and keeps any
  flavour out of it.
- **One manifest or two?** Keeping a clean schema file *and* a separate live
  config gives you a seventh source of truth. Pick one. The entire value
  proposition is "scope lives in one place."

---

## The network boundary (where a homelab exceeds the app-layer idea)

An application-layer passport says *who an agent is* and *what it may call*. It
says nothing about the packets. On your own infrastructure you can add the part a
public-network spec can't assume you have:

- each agent egresses through its own WireGuard tunnel with a dedicated address;
- an IDS (Suricata, pcap mode) on the tunnel captures TLS SNI + JA3 + DNS + file
  hashes **before NAT**;
- a read-only digest is pulled to a separate verdict host (a second physical machine
  or VM — a network namespace on the same host is not sufficient; host-level
  compromise crosses netns boundaries) and turned into MITRE-tagged verdicts.

The load-bearing decision: **attribution lives at the capture point, not in the
NAT topology.** If you capture before the rewrite, you can run any NAT downstream
and still answer "which agent sent this." "NAT exactly once" is only ever a proxy
for attribution; capture pre-NAT and the proxy is unnecessary. This is also why
the manifest's per-agent `egress_ip` field is meaningful — per-agent attribution
survives precisely because capture happens before address translation.

---

## Methodology notes (cheap, reusable)

- **Verify claims against the live host before writing them down.** Most of the
  apparent "gap" between an idealised design and a running system is the design
  not knowing what already exists. A few minutes of `grep`/`ls`/`ip` collapses it.
- **A negative `grep` is a real acceptance test.** "Nothing reads the old
  hardcoded scope anymore" is checkable, not aspirational.
- **An adversarial pass finds what a literal search misses.** In the real build,
  an auto-added outbound-NAT rule was stored as a macro that a literal
  `grep <subnet>` never matched — only an adversarial "what would I be missing"
  pass surfaced it.
- **Mutate appliances through their API, not their config files.** Hand-editing a
  firewall's stored config is silently ignored when a config daemon keeps its own
  model. Use the documented REST API and reconfigure.
