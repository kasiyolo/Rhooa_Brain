# Rhooa Brain — Architecture

**A local-first knowledge layer for the whitelabeled web Hermes.**
Hermes runs on the web (cloud inference + online sources). Each user's **Brain — the knowledge graph and index built from their PC — stays local on their machine**. One click scans the PC, builds the graph, and from then on the web Hermes answers with full personal/company context.

Design priorities, in order: **1. Security & compliance → 2. Token efficiency → 3. User experience.**

---

## 1. Core decisions (locked)

| Decision | Choice | Why |
|---|---|---|
| UI & inference | **Web Hermes** (hosted), cloud LLM, online sources (web search, etc.) | Zero-install distribution, always-current model |
| Brain data at rest | **100% local** — knowledge graph, vectors, summaries, memory live only on the user's PC | Company files are never uploaded or stored server-side |
| Data in motion | **Minimal retrieved context** per query + **index-time ingest batches** (embedding + extraction, each chunk exactly once) — both ephemeral, zero-retention | The honest compliance story (see §6) |
| Learning (v1) | Frozen weights + continuously updated local knowledge graph & memory | Instant, auditable, deletable |
| Learning (v2) | Local LoRA — "Train your own AI" (applies when a local model option ships) | Roadmap premium feature |
| Billing | Server-side metering at the inference gateway | Trivial now that inference is hosted; no local ledger needed |
| Agent layer | **Rhooa Agents** provisioned per client (tenant entitlements), cloud-orchestrated, scoped Brain access enforced locally | One Brain, many agents; commercial unit = agent packs (§8) |
| Brain interface | **MCP server** hosted by the local Agent; agents pull knowledge via tool calls **only when needed** (browser relay default, opt-in outbound tunnel) | No preemptive context push — zero cost when the Brain isn't needed (§8a) |
| Onboarding | **Zero-download first use**: Instant Web Mode in the browser; Full Brain agent arrives via silent IT deployment (B2B) or optional one-click installer | Users only ever "log into a website" (§4a) |
| App integrations | Direct Microsoft Graph + Google Workspace, self-hosted Nango for the long tail; agent-pull syncs, dirty-flags-only server state | Cloud-app knowledge lands in the same local Brain — see [INTEGRATIONS.md](INTEGRATIONS.md) |
| Consistency model | **PA/EL (PACELC)**: available & fast under partition, bounded staleness; CP islands for identity & billing; offline grace tokens close the web-door gap | Full analysis in [CAP-ANALYSIS.md](CAP-ANALYSIS.md) |

**The nuance to state plainly:** "brain data stays local" means *data at rest*. Content transits to Rhooa cloud in exactly two flows, both redaction-gated, TLS, zero-retention, audited: (1) at **query time**, the few retrieved snippets needed to answer; (2) at **index time (v1)**, chunk text sent to the embedding API — each chunk exactly once, vectors returned and stored locally. Local-Only Mode (§6) eliminates both; local embeddings are the long-term default (GRAPH-TRAINING.md §1a).

---

## 2. System overview

```
            ┌────────────── Rhooa Cloud ──────────────┐
            │  Web Hermes app (UI, auth, billing)     │
            │  Inference gateway → LLM                │
            │  Online sources (web search, APIs)      │
            │  ZERO retention of brain context        │
            └───────▲──────────────────────▲──────────┘
                    │ HTTPS (UI, chat)     │ context packet (per query,
                    │                      │ redacted, ephemeral)
        ┌───────────┴──────────────────────┴───────────────────┐
        │                    Browser                           │
        │   Hermes web UI ──── Brain Bridge JS client          │
        └───────────────────────────┬──────────────────────────┘
                                    │ loopback WSS/HTTP
                                    │ (paired, origin-locked,
                                    │  short-lived tokens)
┌───────────────────────────────────▼──────────────────────────┐
│                Employee PC — Rhooa Brain Agent               │
│                    (single signed binary)                    │
│                                                              │
│  Ingestion Pipeline ──► Knowledge Store ──► Retrieval Engine │
│  scan / watch /         SQLite+SQLCipher     token-budgeted  │
│  extract / index        vectors + graph      context builder │
│                         + memory             + privacy gate  │
└──────────────────────────────────────────────────────────────┘
```

**Query flow (MCP pull model):** the Brain is exposed as an **MCP server** (§8a). When a Rhooa agent decides mid-reasoning that it needs company knowledge — and *only then* — it calls an MCP tool like `brain_search`; the call is relayed to the local Agent, which retrieves, packs a token-budgeted result, runs it through the **privacy gate** (redaction + policy), and returns it. No context is pushed preemptively into chat turns: questions that don't need the Brain cost zero transit and zero extra tokens. The server never stores results; raw files and the graph never leave the PC.

---

## 3. The Brain Bridge (browser ↔ local agent) — the critical surface

A web page talking to a localhost service is the classic attack surface (localhost-CSRF, DNS rebinding, port-probing by other sites). The bridge is therefore designed as a hardened, narrow door. The finalized end-to-end flow:

### Phase 0 — Device pairing (once, at install/first use)
The local agent displays a one-time pairing code; the user enters it in the web app while logged in. The agent durably binds itself to that account's `user_id` (seat identity). **From then on, the agent serves only that user.** This is what a session JWT alone cannot prove — a JWT proves "a valid Rhooa user", pairing proves "the owner of *this* Brain". Without this binding, anyone logging into their *own* Rhooa account on the victim's machine would be served the victim's Brain.

### Phase 1 — Login & token issuance (cloud)
User logs in at `app.rhooa.com`. The cloud returns a **short-lived Access Token JWT (~15 min)** in the response payload and a **long-lived Refresh Token in an `HttpOnly`, `Secure`, `SameSite` cookie**. The `/api/auth/refresh` endpoint enforces SameSite + origin checks (CSRF).

### Phase 2 — Local connection (browser)
The Brain Bridge JS client opens a WebSocket to the local agent and sends the Access Token JWT in the handshake.

### Phase 3 — Verification (local agent, fully offline)
On every handshake / token update, in order:
1. **Origin check** — reject unless `Origin` is exactly the Rhooa web app domain (kills DNS rebinding and cross-site localhost attacks).
2. **Rate limit** — local token-bucket; `search` calls capped (e.g. ~30/min), scan status/progress events on a separate looser bucket so indexing UI doesn't hit 429s. Excess → `429`.
3. **Offline JWT verification** — signature checked against the cloud's public key (shipped/cached as JWKS with key IDs so keys can rotate via agent updates), `exp` checked with ~60s clock-skew leeway. **No network call needed** — auth costs the cloud nothing per request.
4. **Seat binding check** — the JWT's `sub` must equal the paired `user_id` from Phase 0. Mismatch → reject.

### Phase 4 — Silent refresh (background)
~2 minutes before JWT expiry the JS client calls `/api/auth/refresh` (refresh cookie), gets a fresh JWT, and pushes it to the agent over the already-open WebSocket. The session stays alive invisibly. On logout, the web app sends `session_end` over the WS so the agent drops the connection immediately rather than waiting out the 15-minute expiry.

### Threat table

| Threat | Mechanism | How it's handled |
|---|---|---|
| Malicious tabs / sites (rebinding, localhost-CSRF) | Origin-locking | Agent only accepts the official Rhooa origin |
| Stolen Access Token | Short-lived JWT (15 min) + seat binding | Expires fast; rejected offline; useless against another user's paired agent |
| Wrong-account access on a shared PC | Pairing (Phase 0) + `sub` check | A valid JWT for a *different* user is rejected |
| Local brute force / runaway scripts | Local token-bucket rate limiting | Agent throttles before DB/CPU are stressed |
| Data theft in the cloud | Zero retention + redacted context packets | Cloud is a pass-through inference processor; stores nothing |

**Honest footnotes for the threat model:** (a) the `Origin` header is guaranteed by the *browser* — same-user local malware can forge headers; what stops it is lacking a valid owner JWT, and fully-privileged same-user malware is out of scope (it could keylog regardless). (b) Loopback TLS is the fiddly implementation point: browsers distrust self-signed localhost certs, but Chrome/Edge/Firefox exempt loopback from mixed-content blocking so plain `ws://127.0.0.1` from an https page usually works — Safari does not; needs per-browser testing and a fallback.

Additional bridge properties (unchanged):
- **Narrow API.** The MCP tool set (`brain_search`, `brain_get_summary`, `brain_lookup_entity`, `brain_propose_memory` — see §8a), the training-control calls (`brain.discover`, `brain.train.start/pause/cancel`, `brain.status` — see §4a), and status events. Never raw file reads, never arbitrary DB queries — a stolen token has a small blast radius.
- **Audited.** Every bridge call lands in the local append-only audit log; anomalous volume throttles and alerts. Mass exfiltration is slow and visible by construction.

**Residual risk to own openly:** if the hosted web app itself is compromised (XSS, poisoned JS dependency), malicious script runs with a valid bridge token in every user's browser. Mitigations: strict CSP, subresource integrity, minimal third-party JS, and the narrow/rate-limited/audited API above which caps how much such an attacker could pull before detection. This is the main security cost of moving from desktop to web, and it should be in the threat model reviewed each release.

---

## 4. One-click scan — Ingestion Pipeline (all local)

1. **Discovery.** Enumerate user-profile folders and drives. Default-exclude: `C:\Windows`, `Program Files`, package caches (`node_modules`, `.venv`, `%TEMP%`), oversized media.
2. **Consent screen (compliance-critical).** Show the discovered folder tree with smart defaults pre-checked; one click accepts, everything is toggleable. The accepted scope is recorded — this screen *is* the audit artifact.
3. **Extraction.** Office/PDF/text/code/email handlers, run in a **sandboxed low-privilege subprocess** (parsing untrusted files is the top local attack surface). Optional OCR.
4. **Sensitive-data gate at ingest.** Local secret/PII detectors (regex + small classifier). Credentials, `.env`, private keys excluded by default; PII tagged so the privacy gate can filter it at query time.
5. **Indexing — cloud-trained, locally-stored.** Structure-aware chunking with content-hash dedup, then redaction-gated batches go to the Rhooa `/brain/ingest` API which returns **embedding + entities + relations in one call** (single-transit principle: each unique chunk travels exactly once, ever; zero-retention endpoint). Vectors and the assembled graph are stored **only locally**. No local ML models, near-zero PC load. Full design: [GRAPH-TRAINING.md](GRAPH-TRAINING.md). (Local-Compute Mode inverts this for strict orgs — nothing transits, local models instead.)
6. **Hierarchical summarization (token-efficiency core).** Built at index time: chunk → document → folder/project → company-level "god node" summaries (graphify-style community summaries). Answering broad questions later costs hundreds of tokens, not thousands.
7. **Continuous watch.** Filesystem watcher feeds deltas; the Brain never rescans and never goes stale.
8. **Resource governor.** Low priority, throttles on CPU/IO pressure, pauses on battery/active use; progress card in the web UI via the bridge.

---

## 4a. Onboarding modes & how training starts from the web UI

**Hard constraint stated honestly:** a bare browser cannot build the full Brain — no whole-disk scanning (only user-picked folders, Chromium-only), no background work after the tab closes, no file watcher, weaker storage isolation than the agent. So onboarding is a two-mode funnel; both modes speak to the **same** `/brain/ingest` API and produce the same graph schema.

### Mode 1 — Instant Web (zero install, first-use)
1. User logs into the web app; no agent detected (the Bridge JS client's WS probe fails).
2. UI offers "Connect your files" → **File System Access API** folder picker (Chrome/Edge; drag-and-drop fallback elsewhere). The user explicitly picks folders — the browser enforces that consent natively.
3. Chunking + a lighter privacy gate run **in-tab** (WASM workers); redaction-gated chunks stream to `/brain/ingest`; vectors + graph come back and are stored in **OPFS/IndexedDB** (persistent-storage permission requested).
4. Search runs in-tab (WASM sqlite-vec). The user has a working mini-brain in minutes, no download.
5. **Declared limits (shown in UI, not hidden):** only picked folders; trains only while the tab is open; no freshness watcher; storage lives in the browser profile.

### Mode 2 — Full Brain (the agent), two delivery paths
- **B2B default — zero download for the user:** IT deploys the signed agent silently via MSI/Intune/GPO. Pairing is automatic: the agent carries a tenant **enrollment token**; on the user's first web login on that machine it binds to their SSO identity — no pairing code, no install step, the employee only ever "logs into a website".
- **Individual upgrade path:** one-click signed installer offered from the UI when the user hits Instant-Mode limits.
- On first connect, the agent **imports the Instant-Mode mini-brain** (OPFS export over the bridge) so nothing is lost, then the full one-click scan takes over.

The web app auto-detects which mode applies at every load: WS probe succeeds → Full Brain; fails → Instant Mode with an upgrade banner.

### Starting the training (Full Brain mode)
The browser never touches the filesystem; it is the remote control, the agent is the machine:

1. **`brain.discover`** (over the authenticated WS): the agent enumerates drives/folders locally and returns the tree with smart defaults pre-checked (sizes, counts, default-excluded system/cache paths).
2. The **consent screen renders in the web UI** from that tree; the user toggles folders and clicks *Build my Brain*.
3. **`brain.train.start {scope}`**: the agent validates JWT + seat binding as always, records the approved scope in the local audit log (the consent artifact), and shows a native tray notification that training began. A strict-org policy file can additionally require a **native OS confirmation click**, so a compromised web app alone can never expand scan scope silently.
4. The agent runs the pipeline autonomously (T0 parse/chunk → privacy gate → `/brain/ingest` batches → local graph assembly), streaming progress events over the WS: files scanned, chunks ingested, tier coverage, ETA — the UI's "Brain fitness" card renders live.
5. **`brain.train.pause/cancel`** available from the UI; the governor auto-throttles regardless. **Training does not depend on the tab**: close the browser and it continues headless; reopening re-subscribes to progress via `brain.status`.
6. After the initial pass, the file watcher owns freshness — "training" is never manually restarted, only re-scoped (which repeats steps 1–3 for the delta).

---

## 5. Knowledge Store (all local)

One encrypted embedded database, zero administration:

- **SQLite + SQLCipher** (AES-256), key in **Windows DPAPI / Credential Manager**.
- **`sqlite-vec`** for vector search; **graph tables** (entities, relations, community summaries) in the same DB.
- **Memory table** — facts learned from conversations and user corrections; how the Brain "rewrites itself" without touching weights. Inspectable and deletable by the user.
- Everything under `%LOCALAPPDATA%\RhooaBrain\` → **one-click full purge** (GDPR erasure).

---

## 6. Security & compliance model

The pitch, stated honestly: **"Your files and your company's knowledge graph never leave your PC. Only the few redacted snippets needed to answer the current question are sent to the model, are never stored, and you can see exactly what was sent."**

| Layer | Control |
|---|---|
| Data at rest | Entire Brain local, SQLCipher AES-256, DPAPI/TPM-held keys — full key hierarchy, algorithms & erasure design in [ENCRYPTION.md](ENCRYPTION.md) |
| Data in motion | Per-query context packets + v1 index-time embedding batches; TLS; **zero server-side retention** (contractual + technical: no logging of packet/chunk bodies at gateway or embedding endpoint) |
| Privacy gate | Before any packet leaves: PII/secret redaction, policy-based exclusion (tagged-sensitive chunks never leave), packet size cap |
| Transparency | "What was sent" inspector in the UI — per message, the user can view the exact packet; all transits recorded in the local audit log |
| **Local-Only Mode** | Policy toggle: retrieval works fully offline against the local graph, answers come from summaries/extractive results (or a local model if installed). For orgs whose policy forbids any content transit. This is also the v2 LoRA landing zone. |
| Bridge | Pairing, origin-lock, short-lived tokens, narrow API, rate limits, audit (§3) |
| Web supply chain | CSP, SRI, dependency pinning/scanning, signed releases of the agent binary |
| Admin policy | IT-signed policy file enforced by the local agent: exclusions, redaction rules, Local-Only enforcement, purge rules |
| Erasure | One-click purge; per-folder/per-file forget |

---

## 7. Retrieval Engine — token efficiency by design

Every query gets a **fixed token budget**; the packet is filled top-down:

1. **Query router** (local heuristics/tiny model): broad vs. specific vs. needs-online-sources.
2. **Coarse-to-fine:** broad questions answered from pre-computed community/god-node summaries (hundreds of tokens); specific ones drill summary → document → exact chunks.
3. **Ranked packing:** hybrid search (vector + BM25 + graph neighborhood), MMR dedup, densest-first packing with citations.
4. **Semantic cache:** repeated/near-duplicate questions short-circuit locally, costing zero cloud tokens.
5. **Server side:** static system prompt ordered for provider prompt-caching; context packet appended last.

All heavy lifting (extraction, summarization, embedding, graph building) happens **once, at index time, on free local compute** — metered cloud tokens are spent only on tightly packed final inference.

---

## 8. Rhooa Agents — per-client agent packs

On top of the Brain sits a pluggable **agent layer**: Rhooa-defined agents (sales assistant, legal reviewer, finance analyst, support triage, …) provisioned **per client**. The Brain is the shared ground truth; agents are cloud-orchestrated personas that consume it.

**Where agents live.** Agents are defined and orchestrated server-side (system prompt, tools, workflow, model settings) inside the web Hermes. Nothing installs on the PC per agent — adding/removing agents for a client is a server-side entitlement change, instantly live for every seat.

**Catalog & entitlements.**
- A central **agent registry** holds versioned agent definitions.
- Each client (tenant) has an **entitlement list** — which agents their seats see. This is the commercial unit: agent packs per client, priced per pack/seat.
- The company's IT policy file (§6) can further deny or restrict agents locally — the PC always has the last word.

**Scoped Brain access — enforced on the PC, not in the cloud.** Every bridge call carries the **agent identity** alongside the user JWT. The local agent enforces per-agent scopes:
- Scopes map to graph regions/tags (e.g. the finance agent searches finance-tagged communities; it cannot touch HR documents even if compromised).
- All agents share the same narrow door — `search(query, budget, scope)`. No agent ever gets raw file access or arbitrary DB queries.
- **Per-agent rate buckets** and **per-agent audit log lines**: "which agent read what, when" is answerable locally — a compliance selling point no cloud-only product can match.

**Agents and memory.** Agents may *propose* `MemoryFact`s from their conclusions, but these enter the graph as `source_class: inferred` (low trust). Only facts the user explicitly confirms become `user_stated`. Agents can never silently rewrite the Brain.

**Agents and online sources.** Agents freely combine Brain context with their online tools (web search, APIs) server-side — the Brain packet they received is already redaction-gated, so composition adds no new exfiltration surface beyond §6's model.

### 8a. The Brain as an MCP server

The Brain's official interface is **MCP (Model Context Protocol)**: the local Agent hosts an MCP server, and every Rhooa agent connects to it as an MCP client — pulling knowledge **only when its reasoning needs it**, never receiving it preemptively.

**Exposed tools (the same narrow door, now as MCP):**
- `brain_search(query, budget)` — token-budgeted hybrid retrieval with citations
- `brain_get_summary(level)` — pre-computed community/god-node summaries (broad questions, cheapest path)
- `brain_lookup_entity(name)` — canonical entity + its graph neighborhood
- `brain_propose_memory(fact)` — writes a `MemoryFact` as `inferred`; only user confirmation upgrades it

MCP is a *protocol adapter over the existing bridge*: every call still passes the full §3 checks (JWT + seat binding, origin where applicable, per-agent scope from the tenant entitlements, per-agent rate bucket, audit log line). An agent's MCP session simply carries its agent identity — scopes are enforced by the local Agent exactly as before.

**Two transports, because the cloud cannot dial into a laptop:**
1. **Browser relay (default):** the cloud agent's MCP call rides down to the page over the existing chat channel and crosses to the local Agent over the already-authenticated WS bridge. Works with zero new connectivity, but only while the user's session is open — which doubles as a natural security property: *the Brain is only queryable while its owner is present*.
2. **Outbound tunnel (opt-in, policy-gated):** the local Agent keeps an outbound WSS connection to the Rhooa gateway, which reverse-proxies MCP calls — enabling scheduled/background agent runs (e.g. a nightly digest agent) with the laptop on but no tab open. **Default OFF**: a cloud-reachable Brain while the user isn't looking is a real trust escalation, so it's an explicit admin-policy decision, clearly labeled, per-agent scoped, fully audited.

**Side benefit (roadmap):** the same MCP server can be opened — policy-gated — to the user's *other* MCP clients (IDEs, desktop assistants), making the Rhooa Brain their personal knowledge backend and deepening lock-in.

---

## 9. Metering & licensing

Inference is hosted, so metering is server-side at the gateway: per-seat **and per-agent** token accounting, quotas, top-ups — standard SaaS billing that directly supports agent-pack pricing per client (§8). The local agent needs only a seat identity (issued at pairing). Local-Only Mode consumes no metered tokens.

---

## 10. v2 — "Train your own AI locally" (LoRA)

The local graph is deliberately the perfect training-data source:

1. **Curate:** generate an instruction dataset locally from graph facts, memory, and key documents.
2. **Train:** QLoRA on the user's GPU; base weights untouched, adapter is a small versioned file in the encrypted store.
3. **Evaluate:** local eval on held-out graph Q/A must pass before activation.
4. **Serve:** adapters apply to the **Local-Only Mode model**, making that mode a full offline assistant — the natural upsell: web Hermes for convenience, trained local brain for sensitive work.

---

## 11. Recommended stack

| Component | Choice | Rationale |
|---|---|---|
| Local agent | **Rust** single signed binary + tray UI | Memory safety for a file-parsing daemon; tiny footprint; Authenticode signing |
| Bridge | Loopback WSS + JSON-RPC, origin-locked | Works from any browser; hardened per §3 |
| Graph training | Rhooa `/brain/ingest` API — embedding + entities + relations per chunk, single-transit, zero-retention | Strongest models, near-zero PC load, no model downloads; Local-Compute Mode (local ONNX/LLM) is the strict-org inversion |
| Store | SQLite + SQLCipher + sqlite-vec | Zero-admin, encrypted, single-directory purge |
| Extraction | Rust-native parsers + sandboxed helpers | Contain the attack surface |
| Web app | Existing whitelabeled Hermes + Brain Bridge JS client (CSP/SRI hardened) | Minimal changes to the host app |
| Local-Only / v2 | llama.cpp runtime, optional install | Quantized 7–8B in ~5–6 GB RAM |

---

## 12. Build phases

- **Phase 1 — MVP:** **Instant Web Mode** (in-tab folder pick → ingest → OPFS mini-brain → search) + local agent skeleton + pairing + bridge; `brain.discover`/`train.start` flow with web consent screen; scan → ingest → search; context packets into web Hermes chat; encrypted store.
- **Phase 2 — v1:** knowledge graph + hierarchical summaries; watcher deltas; privacy gate + "what was sent" inspector; audit log; purge; admin policy file; gateway zero-retention + metering; **agent registry + tenant entitlements + scoped bridge calls (§8)**; MSI/Intune silent deployment + SSO auto-pairing; Instant-Mode → agent brain migration.
- **Phase 3 — v2:** Local-Only Mode with local model; conversation-memory UI; LoRA pipeline (curate → train → eval → activate/rollback).
