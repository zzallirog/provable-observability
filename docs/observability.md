# Observability — watching your agents without being seen

> The point of this whole pattern isn't to lock agents down — it's to always be
> able to answer *"what did this one just do on the wire?"* This doc is how. It
> covers where the capture happens and why, how the logging stays invisible and
> untamperable from the agent's side, what the logs let you detect, how they line
> up with your agent-side records, and how to watch the agents' own prompt files
> for tampering.
>
> Everything below is described generically — no real addresses, hosts, or keys.
> Substitute your own.

---

## 1. Where the capture happens — and why pre-NAT

Each egress agent reaches the internet through a dedicated WireGuard tunnel into a
firewall VM, carrying a small private subnet with **one address per agent**. An IDS
(Suricata in passive **pcap** mode — observe-only, not inline) runs on the tunnel
interface and writes connection metadata to an EVE JSON log.

The load-bearing decision is *where on the path* it captures: on the tunnel
interface, on ingress, **before any NAT**. Because of that, every log record keeps
the agent's real per-agent source address — even though the packet is masqueraded
(possibly twice) before it ever leaves your network.

> This is what makes a NAT bug a non-issue. During the build, assigning the tunnel
> interface silently auto-added an outbound-NAT rule (a double-NAT). It didn't
> matter: *"NAT exactly once" was only ever a proxy for attribution*, and
> attribution is already guaranteed at the capture point. Capture pre-NAT and you
> can run any NAT you like downstream. (How egress is *routed* onto the tunnel —
> and the macOS `pf route-to` dead end that forced an interface-scoped route plus
> per-agent source-binding — is in [`architecture.md`](architecture.md).)

pcap mode (over an inline/netmap setup) was chosen because a bounded tunnel
interface makes deep capture cheap and carries no kernel-module risk — capturing on
the whole LAN switch would be noisy and expensive; a per-agent tunnel is neither.

---

## 2. Soft, untamperable logging

"Soft" here means *invisible and untamperable from the agent's side*. Two
properties working together:

**Observer ≠ observed.** The IDS runs on the firewall VM — a different machine from
the one the agents run on. The agents have no route to the capture point, can't read
the logs, can't edit them, can't even tell they exist.

**One-way pull.** A separate *verdict* host reaches the firewall over an SSH key
whose **forced command** can invoke exactly one script — emit the newest digest and
exit. The key is `restrict`ed (no PTY, no port/agent/X11 forwarding), locked to the
verdict host's address with `from=`, and parked in a key file the appliance's own UI
won't clobber. The firewall can never write to the verdict host; the verdict host can
never run anything else on the firewall. Verify it adversarially before trusting it:

```
# all three must FAIL against the forced-command key
ssh firewall 'id'              # arbitrary command  -> ignored, runs the forced cmd
ssh -tt firewall              # PTY                 -> refused
scp firewall:/etc/passwd .     # file read           -> rejected
```

The result: the thing watching egress cannot be quietly disabled or rewritten by the
thing being watched.

---

## 3. The digest → verdict pipeline

Boring on purpose. Every 15 minutes a cron job on the firewall slices the last window
of EVE events into a structured digest:

| Digest field | From | Carries |
|---|---|---|
| `top_talkers` | `flow` events | src, dst, count, bytes |
| `new_destinations` | diff vs previous digest | destinations not seen before |
| `dns_domains` | `dns` query events | domain, count |
| `tls_sni` | `tls` events | SNI, cert CN, issuer, **JA3**, **JA3S**, count |
| `http_hosts` | `http` events | host, URIs, user-agent |
| `file_hashes` | `fileinfo` events | filename, **sha256**, md5, size, magic |
| `suricata_alerts` | `alert` events | signature, category, severity, **MITRE technique/tactic** |

> **TLS note on `file_hashes`:** Suricata reconstructs file hashes from `fileinfo`
> events. For HTTPS traffic this requires TLS inspection (SSL bump with a locally-trusted
> CA installed in the agent's trust store). Without it, Suricata sees only handshake
> metadata — SNI, JA3, cert fields — for encrypted connections. If SSL inspection is
> not configured, the `file_hashes` row in the digest is populated only for plaintext
> HTTP and cleartext protocols. Name this precondition explicitly in your setup so the
> empty column isn't silently misleading.

The verdict host pulls the digest **read-only** on its own timer (offset from the
firewall's), then runs it through a small engine that maps signals to MITRE ATT&CK
tags and appends one verdict per window to an append-only log. A **sliding window**
over the last five digests (≈75 minutes) catches patterns that span windows — a
beacon that looks quiet in any single 15 minutes shows up as a repeated talker
across five.

---

## 4. What you can do with the logs

Each detection is tied to a captured field:

- **Attribute by agent, no UID tracking.** A new destination or SNI is tied to *which*
  agent role made it, because the source address is per-agent (`new_destinations`,
  `tls_sni`).
- **Catch a swapped or hijacked agent by its TLS fingerprint.** Baseline each agent's
  `ja3`. A changed JA3 from the same agent address means a *different TLS stack* is
  making the calls than the one that should be — a strong "this isn't the same
  software anymore" signal. `ja3s` and the cert fields support server-side pinning
  checks too.
- **DNS anomalies.** Unexpected domains per agent (`dns_domains`).
- **File-hash hits.** Reconstructed `sha256` of anything transferred, cross-referenced
  against threat-intel.
- **Beacons across windows.** Regular outbound to a fixed destination shows up as a
  repeated `top_talkers` entry across the 5-window arc, even if each window looks
  low-severity.

**A real triage lesson, learned from a false positive.** An early heuristic matched
`extern=\d` and so fired *"Exfiltration (TA0010)"* on **idle** windows where the value
was `extern=0` — every quiet tick paged. The fix was to make idle windows emit a
literal "no egress in window" token. The durable rule: **the tactic tag and the
severity are a pair.** A non-empty tactic at `severity=0` must not page. The tag tells
you *what kind*; the severity tells you *whether to care*.

---

## 5. Correlating with your agent side

The egress logs are only half the picture. The other half lives on the agent host: an
append-only audit log of every gate decision, the orchestrator's run start/end times,
and the manifest declaring each agent's permitted calls (`allowed_calls`). The join:

| Network side (IDS) | Agent side | What the join proves |
|---|---|---|
| flow source address | the agent's declared per-agent egress address (manifest) | the flow belongs to *this* agent identity |
| IDS event timestamp | the orchestrator run's start/end window (audit log) | the flow belongs to *this specific run* |
| TLS `sni` (the provider host) | the agent's `allowed_calls` (manifest) | an SNI **outside** `allowed_calls` is a call the gate should have blocked |

That last row is the whole point: you can now measure the difference between what an
agent was **authorised** to do and what it **actually did on the wire** — and an agent
reaching a destination it has no visa for is an alarm, not a log line.

**Honest preconditions.** The timestamp join assumes both clocks are NTP-synced — name
that, don't leave it implicit. And if one of your gates logs nothing yet, that side of
the join is reconstructed from timestamps alone until you wire it (one `logger` line on
the live gate closes it).

---

## 6. Watching the agents themselves — mtime & prompt-hash

The same "make it a fact, not a request" idea applies to the agents' own files:

- **Prompt-hash drift.** At startup the harness takes a **SHA-256 of each agent's
  prompt file** and compares it to a stored baseline. A mismatch aborts the whole run
  **before any agent makes a single call** — unless you explicitly approve the change.
  This is what catches a silently edited, injected, or swapped agent prompt: the
  council never even loads a prompt that doesn't hash to what you last approved.

- **Rate-limit as a file's mtime.** A capped call's budget (e.g. the cloud
  orchestrator's once-a-day) isn't a counter the model is asked to honour — it's the
  mtime of a sentinel file. The harness `touch`es it after a successful call and
  `stat`s it before the next. Nothing the model can say moves that timestamp; only the
  harness does.

- **Killswitch as a sentinel file.** A file's presence aborts everything. A filesystem
  fact, not an instruction.

Watching mtimes and hashes of the agents' own prompt files is cheap and high-signal:
the hash tells you *the prompt changed*; the mtime tells you *when*. Together they turn
"did someone tamper with an agent?" from a trust question into a checkable one.
