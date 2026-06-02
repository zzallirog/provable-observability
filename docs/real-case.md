> **The real case — the named build behind the pattern.** The [main README](../README.md)
> states the pattern in *roles* (CHOKE-POINT, WITNESS, ANALYTICS-NODE): a silhouette, no
> products. This document is the concrete instantiation it generalizes from — the actual
> stack, *named*, with the one live end-to-end run on real hardware. It is the grounding:
> proof the pattern is a working build, not a sketch. (This earlier stood alone as the
> `safe-agent-env` repo; it now lives here as the case study.)

---

<div align="center">

# provable-agent

**A passport declares an agent's identity. The network witnesses its behavior.
When declaration and observation are congruent across time — the identity is
proven.**

*From proven identity, clonability follows as a corollary — not a copy of an
agent's bits, but a reproduction of fully-specified conditions whose fingerprint
you can re-verify against the network. A checkable act, not a deep clone.*

A working pattern, not a spec. Built under real constraints on real hardware.
Every claim below has a live equivalent somewhere.

*Status, up front: the network witness is **proven on real traffic** (one
end-to-end run, attribution held). The continuous fingerprint is **designed but
episodic** today — it needs a rolling import to be a true time series (see [open
questions](#honest-open-questions)). Host-level isolation is **not built** — this
is network-level identity. The claim is exactly as strong as those three lines,
no stronger.*

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![macOS + Linux](https://img.shields.io/badge/macOS%20%2B%20Linux-supported-blue.svg)](#the-architecture)
[![bash + python](https://img.shields.io/badge/bash%20%2B%20python-POSIX-blue.svg)](#)
[![verified](https://img.shields.io/badge/verified-8_sandbox_%2B_1_live-brightgreen)](#how-it-was-verified)
[![egress](https://img.shields.io/badge/egress-attributed_%26_witnessed-orange)](#04--network--the-witness)
[![local-only](https://img.shields.io/badge/cloud_deps-zero-brightgreen)](#02--council--stays-home)

</div>

```
╭─ DECLARE · your machine ───────────────────────────────────────────────────╮
│                                                                            │
│  agents.json — one manifest, one source of truth                           │
│  ┌──────────────────────────────────────────┐                              │
│  │ id · role · allowed_calls · egress_ip    │ ◀── every enforcer reads     │
│  │ rate_limit · valid_until · status        │     this. nothing else.      │
│  └──────────────────────────────────────────┘                              │
│                                                                            │
│  passport → mandate-check → audit stamp → revoke                           │
│                                                                            │
│  researcher ─brief─▶  council · N local lenses ─▶  orchestrator            │
│  (gated RAG +          offline, never egress         1 capped cloud call   │
│   web, egress-gated)                                 per day, no more      │
│                                                                            │
╰────────────────────────────────────────────────────────────────────────────╯
         │
         │  only egress-permitted agents cross here
         │  WireGuard — one source-bound tunnel per agent
         ▼
╭─ WITNESS · OPNsense DMZ ───────────────────────────────────────────────────╮
│                                                                            │
│  wg0 ─▶  Suricata (IDS · pcap) ── SNI · JA3 · DNS · file-hash             │
│                   │                captured PRE-NAT                        │
│                   │                so attribution survives any NAT         │
│                   │                downstream                              │
│                   └──────────────────────────────────▶  internet           │
│                                                                            │
╰────────────────────────────────────────────────────────────────────────────╯
         │
         │  read-only digest pull — the observed can't reach the observer
         ▼
╭─ ATTEST · verdict host ────────────────────────────────────────────────────╮
│                                                                            │
│  RITA + verdict engine ─▶  per-agent behavioral fingerprint                 │
│  (beacon/C2 detect)      (egress signals → MITRE verdicts)                 │
│                        beacon · prevalence · SNI · first-seen              │
│                        MITRE-tagged verdicts                               │
│                                                                            │
│  declared identity  ≈?  observed fingerprint                               │
│  congruent, repeatedly: identity proven. agent attestable.                 │
│                                                                            │
╰────────────────────────────────────────────────────────────────────────────╯
```

> The three boxes aren't about containment. They're about proof.
> DECLARE says who the agent is. WITNESS records what it does.
> ATTEST answers: are they the same entity?

→ [The core claim](#the-core-claim) · [The architecture](#the-architecture) ·
[Verification](#how-it-was-verified) · [Gotchas](#hard-won-gotchas) ·
[Open questions](#honest-open-questions)

---

## The core claim

An agent that carries a passport and egresses through a witnessed channel produces
two independent streams of identity signal:

**Declared:** what the manifest says the agent is — role, scope, model, allowed
calls, egress binding.

**Observed:** what the network actually records — which destinations it reaches, on
what schedule, with what TLS fingerprint, carrying what DNS queries.

A containment system asks: *did the agent stay inside the lines?* This asks
something stronger: *are the declared identity and the observed behavior the same
entity, consistently, across time?*

When the answer is yes — when the behavioral fingerprint is stable, attributable,
and matches what the passport claims — the agent's identity is **proven**, not
asserted. "Congruent" here is statistical, not exact: a fingerprint is a
distribution of observations, so the match is *repeated agreement over many
windows*, not a single byte-for-byte equality. Proof is the accumulation, not the
instant — which means it requires the witness to run as a *time series*, not a
one-shot capture. Today's setup imports on demand; making it continuous is a known
gap, surfaced honestly in [the open questions](#honest-open-questions).

That proof is what makes clonability a corollary: **once an identity is proven
stable and its producing conditions are fully specified, reproducing it elsewhere
is a verifiable act, not a hope** — you run the same query against the copy and
check the fingerprint holds. The full argument is in [the attestation
loop](#the-attestation-loop) below. A passport you never check against real traffic
is a claim. Here it is checked.

---

## On the passport — credit where it's due

The "passport" here — an agent has a declared identity, a scope it may act within,
an audit trail of what it did, and a revoke — is **not my idea.** It's the
**[AgentPassports](https://github.com/sarvesh1327/agentpassports.eth)** concept: Passport = identity, Visa = scope, KeeperHub =
validator, Stamps = audit. Its author built the canonical, on-chain version.

This repo borrows only the *shape*. Everything else is added — the parts a spec
written for a public chain can't assume you have on a local machine:

- **borrowed (the shape):** identity + scope + stamps + revoke;
- **added (the local implementation):** a **chain-free** keeper — flat JSON is the
  registry, an SSH forced-command is the keeper, an append-only log is the stamps;
  no wallet, no chain, no token;
- **added (the actual payload):** a **network witness + attestation layer** that
  AgentPassports doesn't address at all. The passport says *who*; OPNsense +
  Suricata, capturing pre-NAT, say *what the agent actually did*; RITA correlates
  the two across time into a fingerprint. That loop — declaration checked against
  observation — is the original contribution here.

The honest finding: most enforcement you want already exists the moment you run
more than one agent — it just lives in six hardcoded places. The work is
**unifying scope into one file the enforcers read**, then proving with `grep` that
nothing reads the old hardcoded copies anymore. Everything else is a thin
extension once that one move lands.

---

## The architecture

### 01 · Identity — one manifest, every enforcer reads it

`agents.json` is the single source of truth:

```json
{
  "id": "researcher",
  "role": "gather-external",
  "provider": "ollama",
  "model": "qwen2.5:7b",
  "allowed_calls": ["search", "fetch"],
  "egress_ip": "10.x.x.3",
  "rate_limit": "30/hour",
  "valid_until": null,
  "status": "active"
}
```

A fail-closed read-helper — `kya_field` (*Know-Your-Agent field*: one function, one manifest lookup) — is the *single* thing the
gate, the model-selection, the rate-limiter, and the egress binding all call.
Unknown agent or missing field → non-zero exit, empty output. **Never allow on
uncertainty.**

The acceptance test is a `grep` that finds zero leftover scope literals anywhere
in the enforcement code. Scope declared once, read from one place, hardcoded
nowhere. That is the whole design.

### 02 · Council — stays home

The ASCII diagram above shows `researcher → council · N local lenses → orchestrator`. This section is the council box: every lens inside it stays local.

Council lenses and the researcher run on a local quantized model (Ollama). The
orchestrator is one capped cloud call per day — enforced by file mtime, not by
intention. Nothing leaves the host except the researcher's explicit, monitored web
fetches.

Split across two machines deliberately: one hosts the agents, the other renders the
verdicts. *Minimum viable separation:* a second physical machine or a VM with its
own network interface. A separate network namespace on the same host is not
sufficient — host-level compromise crosses netns boundaries. A second OS user on
the same machine weakens the guarantee to a software-only boundary. The threat
model requires the observer's storage to be inaccessible to the observed process.
The observer is not reachable by the observed. The thing watching egress
cannot be edited by the thing being watched.

### 03 · Researcher — the one that goes out

One lens reaches outward; the rest of the council never does. The gate is the
point: it returns only shareable knowledge — skills, public wiki — and **never**
private memory. That boundary is enforced on the serving host by a tier policy,
not by asking the model nicely.

The underlying pattern: a forced-command SSH key whose command is a read-only
search wrapper. The holder of that key cannot run anything except a sanitised,
prefix-checked, tier-floored query. A leaked key leaks a search box, not a shell.

### 04 · Network — the witness

Each agent egresses through a dedicated WireGuard tunnel (`agentwg`) into an
OPNsense VM acting as a choke-point. Firewall policy on `wg0`:

```
block quick from 10.20.40.0/24 to <RFC1918>   # agents cannot see LAN
pass  quick from 10.20.40.0/24 to any          # agents reach internet
nat on vtnet0 from (wg0:network) -> (vtnet0:0) # outbound NAT
```

Suricata runs on `wg0` in pcap mode. It captures TLS SNI, JA3, DNS queries, and
file hashes **pre-NAT** — before the source address is rewritten. Per-agent
attribution survives any downstream NAT topology because the record was made before
the rewrite.

The load-bearing insight: **attribution lives at the capture point, not in the NAT
design.** Capture pre-NAT and you can run whatever NAT you like downstream — "who
did this" is already permanent.

On the verdict host (separate machine, read-only digest pull) RITA — *Real
Intelligence Threat Analytics, the open-source beacon/C2-detection toolkit* —
imports the Zeek-processed capture (into a ClickHouse-backed dataset, one per agent
group). What you can query from a three-minute window of agent traffic:

```sql
-- top destinations by call count
SELECT dst, count() FROM agents.conn GROUP BY dst ORDER BY 2 DESC

-- beacon detection: consistent interval between calls → TSScore
-- prevalence: how many agents reach a given host → fingerprint baseline
-- first-seen: new destination appears → alert candidate
```

> **This is the one place the whole claim meets real traffic — so it carries the
> weight.** A live run with the research pipeline routed through the full stack
> showed exactly *one* external destination in the `agents` dataset:
> `api.anthropic.com`. Declared egress and observed egress matched, attributed to
> the right agent, with nothing else on the wire. The result was expected — the
> point is that it was *checkable against the network itself*, not asserted. One
> run is not a time series; it is the existence proof that the loop closes.

### 05 · Guardrails — barriers, not conventions

`mandate-check` runs before any agent acts. Each rule is enforced by the runtime,
not asked of the model — with one explicitly-marked exception, the **injection
scan**, which is a best-effort filter, not a fail-closed barrier:

| Guardrail | Mechanism |
|---|---|
| **killswitch** | sentinel file aborts the entire run |
| **drift detection** | SHA-256 over each agent's prompt file, taken **at startup before any agent makes a call**; mismatch aborts the whole run — the council never loads an unapproved prompt |
| **rate-limit** | call window by file mtime; window length from the manifest |
| **injection scan** *(best-effort, not fail-closed)* | a regex **first-pass** over *known* override patterns (embedded `</s>`, `[INST]`/`[SYS]` tags, `ignore previous instructions`, role-switch directives) in researcher output, stripped before council ingestion with a redaction marker so the council sees a gap, not a silence. It catches *known strings* — **not** novel or obfuscated injection, and it is blind to injection arriving inside a poisoned local RAG document (which crosses no wire and matches no pattern). A first-pass filter, not a barrier. |
| **passport check** | exists · active · within valid_until · call in allowed_calls — fail-closed on every branch |
| **audit** | every decision appended as one JSON row; optionally HMAC-signed |
| **revoke** | drops the agent's forced-command line from `authorized_keys`, flips `status` in the manifest, sends SIGHUP to active SSH sessions matching that key's `from=` address — idempotent; keep a second SSH session open while you do this |

Revoke **degrades, never bricks.** One agent pulled → that lens drops from the
council; the rest convene without it. The per-call gate still hard-denies the
revoked agent's calls. The two mechanisms are independent and meet only in the
reader's digest.

---

## The attestation loop

This is the emergent property that none of the pieces produces alone:

```
passport (declared)
    │
    ▼
mandate-check ─── allow/deny ───▶  agent acts
                                       │
                                       ▼
                               agentwg (per-agent WG)
                                       │
                                       ▼
                               OPNsense choke-point
                                       │
                               Suricata pre-NAT capture
                                       │
                                       ▼
                               Zeek → RITA
                                       │
                                       ▼
                         behavioral fingerprint (observed)
                                       │
                                       ▼
                    declared ≈ observed? ──── yes ──▶ identity proven
                                            no  ──▶ anomaly, investigate
```

**Clonability as corollary.** This is where "proven identity" pays off. If the
fingerprint is stable across runs, and the conditions that produce it are fully
specified — manifest + model + skills + egress binding — then you hold everything
needed to reproduce the agent on another host. You then run the *same RITA query*
against the new host's traffic: if the fingerprint matches, the identity held. That
is the verification, and it is what makes "clone" mean something precise here — not
a copy of bits, but a reproduction of conditions whose attestation you can check.
RITA is what tells you the reproduction is faithful; without that check, a "clone"
is just another claim.

But the corollary is only as wide as the identity it rests on, so the limit comes
in the same breath: this is **network-level identity**. The fingerprint that gets
cloned is the *behavioral* one — who the agent talks to, on what schedule, with
what TLS signature. Host-level isolation (separate OS user per agent, sandbox
profiles) is a separate axis, and it is **not built here**. The threat model this
covers is behavioral drift and unattributed egress; the threat model it does *not*
cover is a compromised host. A clone proven at the network layer is not a clone
proven at the host layer — and this repo only claims the former. See [honest open
questions](#honest-open-questions).

---

## Tooling behind the pattern

You don't need these to read the repo, but they're the machinery referenced above:

- **Verdict engine** — maps Suricata/Zeek egress signals to MITRE ATT&CK-tagged
  verdicts; reads the sliding-window digest from the firewall and appends one
  verdict row per window to an append-only log on the verdict host.
- **RITA** — *Real Intelligence Threat Analytics*, the open-source beacon/C2
  detection toolkit that builds the behavioral fingerprint from Zeek-processed
  capture.
- **CDP-driven appliance client** — drives the OPNsense REST API through an
  already-authenticated browser session (self-signed cert, auto-CSRF). Covers the
  appliance mutations described in gotcha #2. *Note the CDP exposure caveat in the
  gotchas.*

---

## How it was verified

Every guardrail was proven fail-closed in a sandbox before it was applied to a live
host. The network layer was then verified end-to-end with real agent traffic:

| Layer | Assertion |
|---|---|
| **manifest** | read-helper resolves real fields; fail-closed on unknown agent, null valid_until, missing field |
| **gate-log** | stamp emitter is shellcheck-clean; live-gate anchor present in deployed code |
| **validator** | 18-case passport harness — expired / unknown / wrong-call / revoked all DENIED; null = no-expiry, not allow-all |
| **revoke** | removes key (3→2 lines), flips status; dry-run inert; running twice is idempotent |
| **signed-intent** | HMAC-signed audit row verifies clean; detects a single-byte tamper |
| **owner-view** | passport-view schema valid; sample conforms |
| **dns-pin** | egress DNS-pin verify script is `bash -n` + shellcheck clean |
| **revoke-degrade** | one revoked agent → council degrades (failed=0), survives; not a hard abort |
| **egress end-to-end** | research pipeline routed through agentwg → OPNsense → Suricata captured SNI → Zeek processed → RITA dataset `agents` shows `api.anthropic.com` as top destination. Attribution held. |

Eight guardrail assertions proven fail-closed in the sandbox, plus one network
end-to-end run on real traffic. The discipline is the point: a guardrail you can't
write a fail-closed assertion for isn't a guardrail — it's a hope.

*The harness is not shipped — these assertions describe a sandbox run and one live
end-to-end pass against the author's private infrastructure. The table describes the
method; run equivalent assertions against your own build before you trust the
guardrails.*

---

## Hard-won gotchas

- **macOS `pf route-to` is dead post-Tahoe.** Kernel routing runs before pf; a
  `route-to` rule is silently inert. Use a scoped route: `route … -ifscope utunN
  default <gw>` plus source-bind. Verify with `curl --interface utunN`.
  ```bash
  # example: bind agent traffic to utun2 (substitute your interface and gateway)
  sudo route add -ifscope utun2 default 10.20.40.1
  # the agent process must also explicitly bind to its per-agent egress address
  curl --interface 10.20.40.3 https://api.anthropic.com
  ```
- **OPNsense `config.xml` edits don't reach `configd`.** Hand-editing XML is
  silently ignored — configd keeps its own model. Mutate only via REST API
  (`setClient` + `reconfigure`, `ids/settings/set`).
- **Assigning a WireGuard interface silently adds an outbound-NAT macro.** The
  macro doesn't appear in a literal `grep <subnet>` — you double-NAT invisibly.
  Check the NAT rules in the GUI after any WG interface assignment.
- **Containment policy can silently break agent DNS.** If the agent's resolver is
  on a now-blocked network, DNS fails closed; the symptom is "everything is slow."
  Pin `DNS = 1.1.1.1` in the WireGuard conf and verify with `dig @1.1.1.1` from
  inside the tunnel.
- **The peer you add at runtime on OPNsense doesn't survive a WG reconfiguration.**
  `wg set wg0 peer <pubkey> allowed-ips <ip>` is runtime-only. Persist via GUI →
  VPN → WireGuard → instance → Peers → add → Save → Apply, or it evaporates on the
  next GUI reconfigure.
- **CDP (Chrome DevTools Protocol) on `:9222` is unauthenticated RCE if exposed to
  the WG interface.** Keep it localhost-only. Route through an SSH tunnel if agents
  on other hosts need it. Never bind it to a WG address.
- **Local-service traffic is invisible to the network witness.** The IDS captures on
  the egress tunnel interface — traffic that never crosses that interface produces no
  records. An agent calling a locally-hosted model or a local API looks identical on
  the wire to an agent that made no calls at all. If your threat model includes
  misbehavior that never reaches the internet — data passed through a local inference
  endpoint, coordination through a shared in-process queue — the network fingerprint
  will not catch it. The witness covers egress; it is silent about what stays on the
  host.

---

## What's in this repo

A guide, not a drop-in package. Patterns described in enough detail to reimplement
against your own stack — deliberately without the author's private source, which
is coupled to one host. Read this before you go looking for code to clone: the
value here is the design and the gotchas, not a runnable bundle.

| Path | What |
|---|---|
| `docs/real-case.md` | this file — the real, named build |
| [`architecture.md`](architecture.md) | design note: what exists vs what to build, dependency order, deliberate skips, network boundary |
| [`observability.md`](observability.md) | watching your agents: pre-NAT capture, soft logging, what the logs detect, log↔agent-side correlation, prompt-hash monitoring |

---

## Honest open questions

- **Does network-level attestation compose with host-level isolation?** Right now
  agents run as the same OS user on the host machine. Network identity is proven;
  host identity is trusted. The conservative choice — separate OS user per agent,
  apparmor/sandbox profiles — is a separate axis. It's not built here. Whether it
  matters depends on your threat model.
- **Is RITA continuous or episodic?** The current setup imports a pcap on demand.
  For the fingerprint to be meaningful across time, you need a rolling import —
  hourly cron: `tcpdump -i wg0 -s0 -G 3600 -W 1 -w /tmp/agent.pcap`, then
  `zeek -C -r /tmp/agent.pcap`, then `rita import --rolling --database agents`.
  Without this, the dataset is a snapshot, not a fingerprint — and the whole
  attestation claim rests on the fingerprint being a *time series*.
- **Should tamper-evident audit earn its keep for a single-user homelab?**
  HMAC-signing the log defends against an attacker who already owns the box —
  near-empty threat model for one user. The pattern documents it; whether you build
  it depends on whether you're protecting logs from yourself or someone else.
- **One manifest or two?** The temptation is a clean schema file plus a separate
  live-config. Then you have a seventh source of truth. Pick one. The whole value
  proposition is scope in one place.
- **What can an agent do that the network will never see?** Loopback traffic — calls
  to a locally-hosted model, queries against an on-host API — crosses no monitored
  interface and produces no IDS records. The same is true for filesystem reads,
  inter-process communication, and context injected through a local RAG pipeline.
  The network witness is scoped to egress: what it proves is that the agent's
  outbound behavior matches its declared identity. It says nothing about what the
  agent did before the packet left, or whether data that stayed on-host was read,
  written, or passed to another local process. If you need that coverage, you need
  the host layer: separate OS users, syscall auditing, or process namespacing. Those
  are not built here — and the claim doesn't extend to them.

---

## License

MIT — see [LICENSE](LICENSE). Patterns, not promises. Adapt to your own threat
model. Scrub your own addresses before you publish anything derived from your
infrastructure.

---

*Safe was the wrong frame — the goal was never to lock agents down, but to know,
provably, what each agent is. That intent matured into the pattern this case now grounds:
[provable-observability](../README.md), where the witness is made continuous and its own
stability becomes the signal.*
