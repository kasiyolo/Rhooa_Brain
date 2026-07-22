# Rhooa Brain — Speed & Deployment Analysis via CAP / PACELC

Companion to [ARCHITECTURE.md](ARCHITECTURE.md). The system spans three nodes — **PC (Brain) ↔ Browser (UI) ↔ Rhooa Cloud (inference, ingest, auth, billing)** — so partitions are a *when*, not an *if*: laptops sleep, Wi-Fi drops, offices lose internet.

**Framing.** CAP says: under a network **P**artition, a distributed system chooses **C**onsistency or **A**vailability. **PACELC** completes it for speed: *if Partition → A or C; Else → Latency or Consistency*. Rhooa Brain is not one CAP system — it's six data domains, each making its own choice. The design's signature is **PA/EL almost everywhere** (stay available, stay fast, tolerate bounded staleness), with exactly two deliberately **C**-leaning domains (auth identity, billing truth).

---

## 1. CAP/PACELC classification per subsystem

| Domain | Partition behavior | PACELC | Rationale |
|---|---|---|---|
| **Brain store** (local SQLite: vectors, graph, memory) | Single-node → immune to partitions internally; always readable/writable locally | **PA/EL** | The core design bet: knowledge lives where the user is, so reads never cross a network |
| **Knowledge freshness** (ingest pipeline, watcher deltas) | Training pauses; queue accumulates; resumes on reconnect | **PA/EL** — eventual consistency | A brain that's 2 hours stale is useful; a brain that's unavailable is not |
| **Auth (JWT sessions)** | Offline signature verification keeps sessions valid ≤15 min; then expiry | **PC/EL** (bounded) | Identity is the one place we prefer "stop" over "guess". The 15-min TTL *is* the consistency/availability dial (revocation lag = availability window) |
| **Integrations** (dirty flags, agent-pull syncs) | Webhook flags accumulate cloud-side; syncs wait for the agent | **PA/EL** by construction | Designed as eventual from day one — nothing to fix |
| **Metering/billing** | Usage counted at the gateway (cloud-side) → no partition ambiguity; Local-Only usage is free by definition | **PC** trivially | The meter lives where inference lives |
| **Tenant policy files** | Agent enforces **last-known signed policy** during partition | **PA** with bounded staleness | Fail-closed alternative (no policy → no brain) available per strict-org flag |
| **Multi-device (same user, 2 PCs)** | Two independent brains; no sync | **PA/EL taken to the limit: no C at all across devices** | Honest current-design limitation — see §5 |

---

## 2. The partition matrix — what works when the internet is down

| Capability | Office internet down | Verdict |
|---|---|---|
| Local vector/graph search (data layer) | ✅ fully functional | Brain data was never remote |
| Chat with cloud Hermes | ❌ | Inference is cloud; unavoidable |
| Local-Only Mode answers | ✅ (if enabled/installed) | The AP escape hatch for the whole product |
| Training/ingest | ⏸ pauses, resumes cleanly (checkpointed queues) | By design |
| Integration syncs | ⏸ flags accumulate server-side | By design |
| Policy enforcement, audit logging, purge | ✅ all local | By design |
| **Web UI itself + JWT refresh** | ❌ after ≤15 min | ⚠️ **Finding F1 — see below** |

### ⚠️ Finding F1 — the availability hole CAP exposes
The Brain's *data* is partition-proof, but its *front door* is not: the UI is a hosted web app and JWTs refresh from the cloud. In a partition, even pure-local search goes dark within 15 minutes — **we built an AP data layer behind a CP door.**

**Mitigations (recommend both):**
1. **Offline shell:** PWA/service-worker so the UI loads without the network, or a minimal search UI in the agent's tray app.
2. **Offline grace tokens:** during a partition the agent accepts the last-known-good JWT for a policy-configurable read-only window (e.g. 8h, `search` only, no training scope changes, prominently audited). This is a *deliberate, bounded* move of the auth domain from PC toward PA — exactly the kind of dial CAP analysis exists to surface. Strict orgs set the window to 0 and keep CP.

---

## 3. Speed analysis (the "E-L" half of PACELC)

Latency budget per operation, typical EU user (cloud RTT ~30–80 ms):

| Operation | Path | Expected latency |
|---|---|---|
| Local vector + graph search (≤1M chunks, sqlite-vec) | in-process | **1–20 ms** |
| Bridge hop (browser → agent, loopback WSS) | localhost | **<1–5 ms** |
| MCP tool call via **browser relay** | cloud → browser → agent → back | **~60–170 ms** (1 cloud RTT + loopback + search) |
| MCP tool call via **outbound tunnel** | cloud → gateway → agent → back | **~35–100 ms** (1 RTT, no browser hop) |
| Chat answer (first token) | inference | **0.5–2 s** — dominates everything above |
| Silent JWT refresh | 1 call / 13 min | amortized ~0 |
| Instant-Mode in-tab search (WASM) | in-browser | 10–100 ms |

**Conclusions the numbers force:**
1. **Retrieval is never the bottleneck** — inference is. The MCP pull hop (~100 ms) is invisible inside a 1–2 s generation. The local-first bet pays: a cloud-side vector store would *not* be meaningfully faster end-to-end, but would forfeit the compliance story.
2. **The relay's browser hop costs ~40–70 ms over the tunnel** — irrelevant for chat UX; only schedulable background agents justify the tunnel, which matches its security posture (opt-in) perfectly. Deploy decision and CAP decision agree.
3. **PACELC "EL" is chosen correctly everywhere it matters:** query-time reads are local (zero network), summaries are precomputed at index time, JWT verification is offline, semantic cache short-circuits repeats. The only unavoidable EC→latency trade is inference itself.

**Ingest throughput (training speed):** bounded by upload bandwidth + gateway rate limits, not the PC. Realistic first-scan for a typical knowledge-worker corpus (~50k documents → ~1–2 M chunks after dedup): **hours on office bandwidth** — matching GRAPH-TRAINING's "useful in minutes (T0/T1 stream), smart in hours" promise. Per-seat server-side rate limits shape tenant-wide onboarding storms (500 seats × day one) — a gateway capacity-planning item, flagged for the infra plan.

---

## 4. Deployment modes on the CAP map

| Mode | Partition behavior | Consistency | Speed | Verdict |
|---|---|---|---|---|
| **Instant Web** (no agent) | Tab closed or offline → brain unreachable *and* untrained | Browser-profile silo | Good enough (WASM) | AP only while the tab lives; onboarding funnel, not a destination |
| **Agent + browser relay** (default) | F1 applies (mitigations above); data layer immune | Fresh ≤ watcher lag | Search ms, chat 1–2 s | The right default: best security posture, latency cost invisible |
| **Agent + outbound tunnel** (opt-in) | Background agents survive UI absence; still needs cloud reachable | Same | Fastest MCP path | Enable per-agent for scheduled work only — CAP gain is real but small; trust escalation is the true cost |
| **Local-Only Mode** | **Fully partition-proof** — the only true AP end-to-end deployment | Local model quality ceiling | Search ms; local inference slower (tok/s hardware-bound) | The compliance/air-gap tier and the F1 ultimate answer |

**Deployment recommendation:** default = Agent + relay, with **F1 mitigations shipped in Phase 1** (grace tokens are cheap: one policy field + one agent check); tunnel per-agent for digest-style scheduled work; Local-Only as the premium strict tier. Instant Web strictly as the funnel.

## 5. The multi-device gap (future work)

Two PCs = two brains that never converge — today that's *documented* PA-without-C. When multi-device sync becomes a requirement, the graph's design is already CRDT-friendly (append-mostly facts, `superseded_by` instead of destructive updates, trust-class conflict rule is a deterministic merge function). An E2E-encrypted sync channel (keys never at Rhooa — ENCRYPTION.md hierarchy extends naturally) would make the Brain AP *across* devices with convergent consistency. Deliberately out of scope for v1; the schema keeps the door open.

## 6. One-line summary

**Rhooa Brain is a PA/EL system with two intentional CP islands (identity, billing), one discovered availability hole (F1 — the web front door on local data), and a costed fix.** The speed story holds because every latency-critical read is local and every network hop hides inside inference time.
