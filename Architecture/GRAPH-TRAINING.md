# Rhooa Brain — Graph Training Pipeline

How the Brain *learns*: graph construction, enrichment, and continuous updating.
Companion to [ARCHITECTURE.md](ARCHITECTURE.md) (§4 Ingestion, §5 Knowledge Store).

**Model: cloud-trained, locally-stored.** All heavy AI (embeddings, entity/relation extraction, summaries) runs on Rhooa cloud through a zero-retention pipeline; the resulting vectors and graph are stored **only on the user's PC**. The PC contributes only light work — reading files, chunking, storing, searching.

## 0. Constraints that shape everything

1. **Brain data at rest lives only on the PC.** Vectors, graph, summaries, memory — all local, encrypted. The cloud is a stateless processor: content goes up, knowledge comes back, nothing persists server-side (zero retention, contractual + technical).
2. **Single-transit principle.** Each unique chunk travels to Rhooa **exactly once**, and that one trip returns everything: embedding + entities + relations. No second pass, no re-sending. Content-hash caching enforces "once, ever".
3. **Near-zero PC footprint.** No bundled LLM, no local inference during training. What remains local is I/O-light: parsing/chunking (like an antivirus scan, low priority, throttled), graph assembly, community detection (cheap math), vector search at query time. The PC must never feel it.
4. **Every fact auditable and deletable:** provenance on every node/edge; purging a file cascades to everything derived from it.
5. **The Brain is useful minutes after the one-click scan** and gets smarter over hours (cloud speed), not days.

## 1. The extraction ladder

Not every file deserves the same treatment. Tiers now split by *where* and *how much*:

| Tier | What | Where | Cost bearer | When |
|---|---|---|---|---|
| **T0 — Structure** | Files, folders, doc metadata, email headers, code symbols & imports (tree-sitter), hyperlinks, filename mentions | **Local**, deterministic parsers, no ML | ~free | Initial scan, inline |
| **T1 — Embed + Entities** | Chunk embedding **and** NER (people, orgs, amounts, dates) returned by the **same API call** | **Rhooa cloud**, batched | Rhooa (cheap per chunk) | Streaming during scan |
| **T2 — Relations & meaning** | Typed relations ("X manages Y", "contract between A and B"), topics, timelines, per-document gist | **Rhooa cloud**, strong model | Rhooa (expensive) → **priority queue, §3** | Background, rate-controlled |
| **T3 — Community summaries** | Hierarchical "god node" summaries per project/department/company | **Rhooa cloud**, input is graph facts + child summaries (already-transited derivations, not raw files) | Rhooa (moderate) | When a community changes enough (§5) |

T0+T1 alone produce a genuinely useful brain within minutes: who exists, what documents exist, who emailed whom, what belongs where — searchable semantically. T2/T3 add the "understanding" layer over the following hours.

### 1a. The training API call (single-transit)

```
Agent → POST /brain/ingest  (batch of redaction-gated chunks)
      ← per chunk: { embedding, entities[], relations[], gist }
```

- **Privacy gate first:** secret/PII-excluded chunks are never sent — they get BM25/graph-only retrieval, or wait for Local-Compute Mode (§9).
- **Batched + content-hash cached:** unique chunk = one trip, ever. Re-scans cost zero transit.
- **Zero-retention endpoint**, TLS, every batch in the local audit log — same rules as chat context packets.
- **Auth:** the agent authenticates with its **seat identity from pairing** to the Rhooa gateway; real upstream API keys live only server-side. No shared key ever ships in the binary.
- The response is stored locally and the graph is assembled **on the PC** — the cloud never sees or holds the assembled graph.
- Vector store records `embedding_model + version` per vector → future model upgrades re-embed lazily, without a rescan.

## 2. Graph schema

**Nodes:** `Person, Organization, Project, Document, Chunk, Topic, Event, System/Tool, Place, MemoryFact, CommunitySummary`.
**Edges:** typed (`works_on, authored, mentions, sent_to, part_of, imports, relates_to, supersedes, …`).

Every node and edge carries:
- `provenance`: source chunk id(s) → file path → byte range; purge cascades.
- `confidence` (0–1): T0 ≈ 1.0; T1/T2 per model score, capped.
- `source_class`: `user_stated | document | inferred` — trust hierarchy for conflicts (§6).
- `first_seen / last_confirmed` — staleness & decay (§6).

## 3. Prioritization — what earns a T2 pass

The priority queue now controls **Rhooa's cloud bill** (instead of the user's CPU) — same mechanism, same value. Score inputs:

- **Recency** (recently modified ≫ 2019 archive)
- **User attention** (files actually opened/edited; topics asked about in chat)
- **Type weight** (contracts, specs, decks, threads ≫ auto-generated logs)
- **Graph centrality feedback** (documents whose T1 entities are already hubs get understood first)
- **Query-driven boosts:** a chat query landing in thin-coverage graph regions bumps those documents. **The Brain studies what the user actually asks about.**

Cloud speed means the whole T2 backlog for a typical seat clears in hours; the queue's job is cost-shaping and burst-smoothing (server-side rate limits per seat), not multi-day scheduling.

## 4. Entity resolution (local — it's cheap math)

1. **Blocking:** candidate pairs via vector similarity + string metrics ("Κ. Παπαδόπουλος" ≈ "Kostas Papadopoulos" ≈ "kpapadopoulos@…").
2. **Rules first:** exact email/employee-ID/domain matches merge at confidence 1.0.
3. **Cloud tie-break:** truly ambiguous pairs go up with both mention contexts (tiny payloads, already-transited text); below threshold → keep separate (wrong merges are worse than duplicates).
4. **Alias table** on the canonical node; merges reversible, audit-logged.

## 5. Communities & hierarchical summaries

- **Leiden community detection** runs **locally** — it's fast graph math (seconds even on big graphs), incremental: deltas mark communities dirty.
- Hierarchy: entity cluster → project/team → department → company root. Each level's summary is written by the cloud model from **graph facts + child summaries** — never raw files.
- **Dirty-marking with hysteresis:** refresh only when enough change accumulates (e.g. >10% of a community's edges) — never on every file save.
- These summaries are what retrieval reads first (ARCHITECTURE §7): broad questions cost hundreds of tokens, not thousands.

## 6. Continuous learning & conflict resolution

**Delta loop:** file watcher event → affected chunks re-gated and re-sent (new content hash = new single transit) → graph updated locally → communities marked dirty → lazy summary refresh. Never stale, never retrained from scratch.

**Memory writes from chat:** user statements/corrections ("Maria left", "ACME is a *client*, not a vendor") become `MemoryFact` nodes, `source_class: user_stated`.

**Conflict rule — trust hierarchy, then recency:** `user_stated > document > inferred`; ties broken by `last_confirmed`. Losers get a `superseded_by` edge, not deletion — auditable, reversible.

**Decay:** `inferred` edges never re-confirmed and never retrieved lose confidence over months; below threshold they stop being served (still stored, still purgeable).

**Contradiction surfacing:** conflicting document-sourced facts are flagged; retrieval presents both with sources instead of silently picking one.

## 7. Quality loop

- **Auto-eval set:** at ingest, the cloud model returns a few Q/A pairs per sampled document (same single transit); retrieval is periodically tested → hit-rate per graph region.
- **Weak-region feedback:** poor regions re-queued for T2 / re-chunking.
- **Chat signals:** "wrong answer" reactions lower the confidence of the facts that were served.
- Telemetry local-only; opt-in anonymous aggregates (rates, never content).

## 8. PC footprint — what the machine actually does

| Local work | Load profile |
|---|---|
| Initial scan: read + parse + chunk files | Disk-I/O bound, **low process priority**, throttled on CPU/IO pressure, pauses on battery/active use — comparable to an antivirus idle scan |
| Upload batches | Network, trickled + resumable; bandwidth-capped |
| Graph assembly, entity resolution, Leiden | Milliseconds–seconds bursts |
| Query-time vector + graph search | Milliseconds per query |
| **Not on the PC at all** | LLM inference, NER, embedding computation, summarization |

There is no such thing as literal zero (files must be read to be scanned), but nothing here can freeze a machine: no model weights in RAM, no sustained CPU burn. The resource governor + checkpointed queues mean a closed laptop lid mid-scan resumes cleanly.

## 9. Local-Compute Mode (opt-in inversion)

For strict orgs (or the v2 "Train your own AI" story): the same ladder runs with **local models** (ONNX embeddings + small quantized LLM) so that *nothing* transits — accepting the inverted trade-off: model downloads (~2–4 GB), real CPU/GPU usage, days instead of hours. This is a policy-file toggle per ARCHITECTURE §6 (Local-Only Mode), and where v2 LoRA lands. The default v1 path is cloud-trained, locally-stored.

## 10. Phasing

- **MVP:** T0 local + `/brain/ingest` (embeddings + T1 entities) + rules-only entity resolution. Searchable, entity-aware brain on day one.
- **v1:** T2 priority queue, Leiden communities, T3 summaries, memory writes + conflict resolution, decay, eval loop, Brain-fitness UI card (coverage %, what it studied, what's next).
- **v2:** Local-Compute Mode; graph doubles as the curation source for local LoRA (ARCHITECTURE §10).
