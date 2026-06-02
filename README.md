<div align="center">

# provable-observability

**A passport *declares* what an agent is. The network *witnesses* what it does.
Watch the witness long enough and a third thing appears: the *rhythm* of the
observation itself. When that rhythm holds — hour over hour, night over night — the
network is not merely observable. It is *provably* observable.**

*This document is about the third thing. It is deliberately not about where an agent
goes: destinations are public, sit behind a WAF, and were never the subject. The
subject is whether the act of observation stays continuous, attributable, and stable
over time. Stability is the signal; a broken rhythm is the alarm.*

A working pattern, not a spec — built and verified on real hardware by a single
operator. Concrete addresses, hosts, keys, and live magnitudes are deliberately
absent: the *structure* is what reproduces, and the topology belongs to the operator
alone.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![macOS + Linux](https://img.shields.io/badge/macOS%20%2B%20Linux-supported-blue.svg)](#the-architecture--four-planes-by-role)
[![continuous witness](https://img.shields.io/badge/continuous_witness-verified_e2e-brightgreen)](#proofs)
[![egress](https://img.shields.io/badge/egress-attributed_pre--NAT-orange)](#the-architecture--four-planes-by-role)

</div>

```
╭─ the claim, in one breath ──────────────────────────────────────────────────╮
│                                                                            │
│   containment asks:    did the agent stay inside the lines?                │
│   attestation asks:    is the declared identity the observed one?          │
│   THIS asks:           is the observation itself stable over time?         │
│                                                                            │
│   the first is a fence. the second is a photograph.                        │
│   the third is a pulse — and a pulse can be missed.                        │
│                                                                            │
╰────────────────────────────────────────────────────────────────────────────╯
```

---

## The finding

A single-operator homelab can run something most production fleets do not: **network-level
behavioral attestation as a continuous, self-checking signal.** Not a dashboard a human
remembers to read — an hourly, falsifiable assertion that the network is, and remains,
observable.

The inherited pattern (see [Lineage & credit](#lineage--credit)) had already proven, on
live traffic, that an agent's *declared* egress and its *observed* egress can be matched,
attributed, and checked against the network itself rather than asserted. That was an
**existence proof**: the loop can close. But it closed *once* — a photograph, not a film.
Its own honest open question read, verbatim:

> *"Is the witness continuous or episodic? The current setup imports on demand. For the
> fingerprint to be meaningful over time, you need a rolling import."*

**This work closes that question.** The witness now runs without a gap between captures;
each hour is imported into a rolling dataset; the operator receives a readable digest plus
a *soft pulse alert* when the expected rhythm of activity breaks. The fingerprint stopped
being a snapshot and became a **time series** — the only form in which *"today looks like
yesterday"* is a statement with a truth value rather than a hope.

---

## The point — why observe the observation

The instinct, looking at an autonomous agent, is to watch *where it goes*: collect its
destinations and judge them. This work deliberately does the opposite, for three reasons.

```
1 · destinations are a poor subject
    they are public, hide behind WAFs and CDNs, and rotate. Tying safety to "did it
    reach host X" is brittle and — worse — is surveillance of third parties: the wrong
    frame for one's own lab.

2 · the rhythm of metadata is a rich subject
    THAT there was egress, HOW MUCH, in WHAT RHYTHM, attributable to which declared
    identity — this is abstract, payload-free, and still carries nearly everything a
    behavioral fingerprint needs. A beacon is a rhythm; exfiltration is a volume
    anomaly; a swapped agent is a break in the fingerprint. None of these needs a
    single byte of content.

3 · stability is provable; intention is not
    you cannot prove an agent "meant well." You can prove that the network's view of it
    is continuous, attributable, and statistically the same as yesterday. "Today like
    yesterday" is a falsifiable claim — the strongest honest thing on offer.
```

So the subject is not the agent's destination. The subject is **the operator's own network,
and whether it stays continuously, attributably observable** — where the stability of that
observability stands in for "nothing changed quietly."

These are three sequential questions, each asked only after the previous is answered:

```
  CONTAINMENT ──fence──▶ ATTESTATION ──photograph──▶ CONTINUITY ⟲ pulse
  ───────────            ───────────                 ──────────
  stayed inside          declared identity =         is the observation
  the lines?             observed?                   itself stable in time?
  (boolean, per event)   (a match, once)             (a rhythm, always)
```

The asymmetry is the whole reason the fourth question needs its own machinery. A fence is
up or down — you know which. A photograph matched or it didn't — you saw it. But a pulse
can simply *stop*, and if no one is watching *for the absence*, a stopped pulse looks
exactly like a quiet one. The other three fail loudly. This one fails **silently** — so it
is the one that must be made continuous and checkable.

---

## The architecture — four planes by role

Roles, never addresses. Each node is named for what it *does*; real hosts, subnets, and
keys are absent by construction. The structure is what reproduces.

```
┌─ AGENT-HOST ─────────────────────────────────────────────────────────────────┐
│  runs    :  the agents + the manifest (agents.json) + a local gate            │
│  trust   :  THE OBSERVED. may be hostile — assume it reads and edits anything  │
│             it can reach.                                                      │
├─ CHOKE-POINT ────────────────────────────────────────────────────────────────┤
│  runs    :  firewall VM; each agent's egress tunnel terminates here; the       │
│             passive capture lives here too                                     │
│  trust   :  THE WITNESS. agents pass THROUGH it but cannot read FROM it.       │
├─ ANALYTICS-NODE ─────────────────────────────────────────────────────────────┤
│  runs    :  rolling import + fingerprint + digest builder + pulse             │
│  trust   :  THE OBSERVER. pulls read-only; the observed has no route to it.    │
├─ OPERATOR-SINK ──────────────────────────────────────────────────────────────┤
│  runs    :  a plain visible folder where digests and alerts land              │
│  trust   :  THE HUMAN SURFACE. read by a person, not a process.               │
└────────────────────────────────────────────────────────────────────────────────┘

  AGENT-HOST ══egress══▶ CHOKE-POINT ┄captures┄▶ ANALYTICS-NODE ══▶ OPERATOR-SINK
   observed    tunnel      witness     (metadata)    observer    digest   folder
      │                                   ▲                       + pulse   human
      └────── ✗ NO route ─────────────────┘   the observed cannot reach the
              ✗ cannot read                   observer or the witness
```

**The observed cannot reach the observer.** This is not a firewall rule you hope holds — it
is topology. Agents egress *outward through* the witness and have no inbound path to it; the
analytics node is unreachable to any of them. The thing watching egress cannot be quietly
disabled or rewritten by the thing being watched. The minimum that buys this is a *separate
physical machine or a VM with its own interface*; a network namespace on the same host is
not sufficient, because host-level compromise crosses netns boundaries.

The load-bearing decision sits in the WITNESS plane: **attribution lives at the capture
point, not in the NAT design.** Each agent egresses through its own tunnel carrying one
address per agent; capture happens on the tunnel interface, *before* any NAT. Because the
record is made pre-translation, the per-agent source survives any masquerade downstream —
"which agent did this" is already permanent. You can then run whatever NAT you like; "NAT
exactly once" was only ever a proxy for attribution, and the proxy is now unnecessary.

The fourth plane — **CONTINUITY** — is what turns those three from a per-window photograph
into a film: a capture that never stops, an hourly readable digest, and a soft alert when
the expected rhythm breaks. It is the plane that makes "observed once" into "observable,
continuously, and provably so."

---

## What it gave — the witness that stops blinking

Five properties turn capture-on-demand into a continuous, self-healing time series. Each was
a failure mode first; naming them is the reusable part.

**1 · Gap-free capture.** The naive loop — *capture ~55 min, then stop and process* — has a
structural hole: the minutes spent processing are minutes *without capture*, and they fall on
the same clock position every hour. An event landing in that recurring blind window is
invisible *systematically*, not by chance. The fix is to **decouple capture from processing**:
one long-lived capture rotates on a fixed interval and is *never* paused; a post-rotation hook
ships each closed window onward for processing while the next window is already being captured.
The tap stays open 100% of wall-clock time.

**2 · Pre-NAT metadata, on the tunnel only.** Capturing on a bounded per-agent tunnel interface
(not the whole LAN) keeps deep capture cheap *and* preserves the per-agent source intact,
because the record is made before masquerade. Bounded interface + pre-NAT = attribution is
permanent and the capture is cheap enough to run continuously.

**3 · Link-type conversion, or the analyzer reads nothing.** A capture taken on some tunnel
interfaces carries a BSD-loopback (NULL) link header rather than raw IP, and a modern passive
analyzer will silently emit *zero* connection logs from it. The trap: the analyzer produces
`tls`/`dns` logs but *no* `conn` log, and the importer then rejects the whole batch ("no conn
logs — skipping"). An empty `conn` log is a signal. Name the conversion as a precondition, or
spend an hour debugging the importer while the real fault sits a layer up.

**4 · The rolling-import dedup trap — import into a *unique path*.** This one is subtle and
silently breaks continuity if missed. A rolling importer keeps a per-dataset record of files
already imported, so it doesn't double-count. The trap: at least one widely-used importer keys
that dedup on the **hash of the file path**, *not* the contents. A pipeline that always writes
this hour's logs to the *same* path looks identical to the importer every hour:

```
  hour 1:  /logs/conn.log   → hash(path) = H   → imported ✔
  hour 2:  /logs/conn.log   → hash(path) = H   → "already imported" → SILENT NO-OP
  hour 3:  /logs/conn.log   → hash(path) = H   → "already imported" → SILENT NO-OP
```

Different traffic, identical path → the importer skips everything after the first hour and
*says nothing*. The dataset freezes while every component reports success — exactly the failure
that makes a "continuous" pipeline quietly episodic. The fix is to give each window a **unique
directory** while keeping the canonical filename the analyzer and importer expect
(`…/run_<timestamp>/conn.log`). Renaming the file itself does not work — the importer rejects
unrecognized log names. Uniqueness must live in the *directory*.

**5 · Digest and pulse.** From this hour's `conn` log (deterministic, one hour of data) build a
human-facing digest carrying metadata only: flow count, *external* flow count (private ranges
filtered out), unique destinations, any *first-seen* destination this hour, and TLS SNI where
present. The pulse is the cheap, load-bearing part:

```
  external flows this hour  <  threshold   →   "quiet"   →   streak += 1
                            ≥  threshold   →   "ok"      →   streak  = 0

  streak ≥ N consecutive quiet hours        →   raise a SOFT alert
  streak returns to 0                       →   clear it automatically
```

Soft on purpose: one quiet hour is a note in the digest, not a page; a *run* of them is the
alarm. Since the baseline is "activity in most hours," it is the *absence* of activity — not
its destination — that is the event.

---

## Proofs

One rule governs everything below: **a claim is only as good as the failing test you could
have written for it.** A guardrail you cannot express as "this must fail closed, and here is
the test that proves it does" is not a guardrail — it is a hope. Every result here is checked
by asserting an *outcome*, never by observing that a *step ran*.

| Property | Falsifiable as | Outcome |
|---|---|---|
| **capture is gap-free** | the tap is a single long-lived rotation with a post-rotate ship; assert wall-clock has no uncaptured interval | continuous; processing latency no longer subtracts from coverage |
| **capture self-heals** | kill the capture process; assert a *new* process appears (new PID); reboot, assert it returns | respawned by the supervisor; survives reboot via a boot hook |
| **link-type conversion is necessary** | feed the analyzer raw vs converted capture; assert a non-empty `conn` log only after conversion | confirmed — without it, zero connection records |
| **import actually grows the dataset** | controlled, *distinct* traffic each run; assert the dataset received *new, expected* rows — not that the stage "passed" | this is where the silent-skip bug surfaced, and was fixed |
| **pulse alerts on absence** | run an empty window; assert the streak climbs to threshold; restore activity, assert the alert clears | soft alert raised on streak, auto-cleared on recovery |
| **digest reaches the human surface** | assert a readable, correctly attributed digest appears in the operator's folder within the hour | end-to-end, controlled traffic appeared in the digest, attributed |

> **The result that mattered most was negative.** The assertion *"import grows the dataset"* is
> what exposed the dedup trap above: every component returned success while the dataset stood
> frozen for hours. Had the check been "did the importer exit 0," the pipeline would have
> shipped *broken and green* — a continuous witness continuously importing nothing. The fix was
> verified by the very assertion the bug had failed: distinct traffic in, distinct new rows out.

The inherited existence proof still stands beneath all of this: a controlled agent run, routed
through the full DECLARE → WITNESS → ATTEST path, produced exactly the egress its manifest
declared and no other — attributed to the right agent, checkable against the network itself.
One run is not a time series; it is the proof that the loop closes. This work makes it close
*continuously*.

*The harness is not shipped. These assertions describe a sandbox run plus a live end-to-end
pass against the operator's private infrastructure. The table describes the method; run
equivalent assertions against your own build before you trust it.*

---

## Why "provable", not "monitored"

Monitoring tells you a number. *Provable* observability makes three things checkable at once,
every hour, automatically:

```
  continuity      there is an unbroken hourly series; a gap is itself visible (capture
                  self-heals, the import is asserted to have grown — so a missing hour is
                  a fault, not a silent skip)
  attribution     every flow in the series carries its per-agent source, because capture
                  is pre-NAT
  congruence      this hour's metadata vector sits inside (or outside) its own recent
                  band, re-checked on a fixed clock
  in time
```

"Today looks like yesterday" stops being a vibe and becomes a statement with a truth value,
re-evaluated on a fixed dial. That re-evaluation — not any single measurement — is what gets
*proved*: **the network is, and stays, observable.**

**A corollary: the observability itself is reproducible.** From "proven identity" the inherited
pattern derived an elegant consequence — if a fingerprint is stable and the conditions producing
it are fully specified, a clone can be *verified*: run the same query against the copy and check
the fingerprint holds. The time series carries that consequence over to the *witness* itself.
Because observability is now continuous and falsifiable, "reproduce the observability elsewhere"
becomes a checkable act rather than a hope: stand up the same role stack (a pre-NAT witness at a
choke-point, a read-only analytics node, a pulse), run the same outcome assertions, and if the
series is continuous and attribution holds, the observability has reproduced. What clones is not
the agent — it is *the ability to provably watch it*. And, as before, the limit arrives in the
same breath: what reproduces is **network-level** observability; the host layer does not come
with it, because it was never covered.

---

## Honest limits

The value of an honest perimeter is that the reader knows *exactly* how far to trust this — a
perimeter drawn generously is worse than none, because it invites trust it cannot hold.

**What it covers — egress metadata over time:** behavioral drift (a fixed-interval beacon being
born), unattributed egress (a flow that maps to no declared agent), a swapped or hijacked agent
(a break in the TLS fingerprint from the same source), fan-out and volume anomalies, and a
stalled witness or stalled agent (a pulse miss). All from metadata; none of it reads content.

**What it is blind to, by construction:**

```
  ✔ SEES (crosses the tunnel)            ✘ BLIND (never crosses the tunnel)
  ──────────────────────────────         ──────────────────────────────────────────
  · outbound flows —                     · loopback — agent → on-host model / API
    who / how much / when                · filesystem — reading prompts, keys, logs
  · TLS SNI · JA3 · DNS ·                · IPC — sockets, in-process queues, shm
    first-seen                           · context — a poisoned RAG document
                                         · payload — even what it sees is metadata,
                                           the content is never read
```

The most important admission: **an agent calling a locally-hosted model over loopback looks, on
the wire, identical to an agent that made no calls at all.** Traffic that never crosses the
monitored tunnel produces no record — the same is true of filesystem reads, inter-process
coordination, and context injected through a local pipeline. The witness is scoped to *egress*;
about what stays on the host, it is silent.

And even within what it sees, it sees **metadata, not meaning** — a flow of such-and-such size
went to such-and-such SNI in such-and-such rhythm, not what was inside. This is a deliberate
floor, not a hole to close: the abstraction *is* the privacy property. The operator monitors the
*shape* of their network, never its content.

**Host-level isolation is a separate axis, and it is not built here.** Agents run as one OS user
on the agent host: network identity is proven, host identity is trusted. The conservative posture
— a separate OS user per agent, sandbox/syscall profiles — is real, valuable, and out of scope. A
clone proven at the network layer is not a clone proven at the host layer, and this document
claims only the former.

---

## Lineage & credit

Honest about what belongs to whom.

**The passport is not mine.** A declared identity, a scope an agent may act within, an audit
trail, and a revoke — this is the **[AgentPassports](https://github.com/sarvesh1327/agentpassports.eth)**
concept (Passport = identity, Visa = scope, KeeperHub = validator, Stamps = audit). Its author
conceived it and built the canonical, on-chain version. Only the *shape* of the passport is
borrowed here — wholesale and deliberately. There is no original contribution to the passport
idea itself.

**The network stack is mine.** [**safe-agent-env / provable-agent**](https://github.com/zzallirog/safe-agent-env)
(my own public repo, and the lineage this document cites) gives the passport a chain-free body —
flat JSON for the registry, an SSH forced-command for the keeper, an append-only log for the
stamps; no wallet, no chain, no token. But that is the small part. The one real addition is the
**network witness** a public-chain spec cannot assume you have: the passport declares *who*; the
witness, capturing pre-NAT at the choke-point, records *what the agent actually did*; the
analytics node correlates the two over time.

**This document is about the continuity of that witness.** The lineage proved the loop can close
*once*; here it closes *continuously*, and the stability of that closure becomes a quantity with a
truth value. The fourth plane is the only thing added on top of the network stack — and it is what
turns a photograph into a time series.

---

## License

MIT — see [LICENSE](LICENSE). Patterns, not promises. Adapt to your own threat model. **Scrub
your own addresses, hosts, and keys before publishing anything derived from your infrastructure.**
