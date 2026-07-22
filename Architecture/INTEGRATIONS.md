# Rhooa Brain — App Integrations

What's worth feeding into the Brain beyond the local disk, and how connectors fit the *cloud-trained, locally-stored* model. Companion to [ARCHITECTURE.md](ARCHITECTURE.md) and [GRAPH-TRAINING.md](GRAPH-TRAINING.md).

## 1. What's worth ingesting — value tiers

The Brain's graph gets its power from *relationships*, and apps carry relationship signal that files never do: an email thread says who promised what to whom; a calendar says who actually meets; a CRM says which company is a client. Ranked by knowledge-per-byte:

### Tier 1 — the company's living memory (ship first)
| Source | Why it's gold for the graph |
|---|---|
| **Email — Outlook / Gmail** | Decisions, commitments, negotiations; `Person—sent_to—Person` edges; threads = ready-made timelines |
| **Calendar — Outlook / Google** | Who meets whom, about what, how often → strongest `works_on` / `relates_to` edge source there is |
| **Files — OneDrive / SharePoint / Google Drive / Dropbox** | The cloud half of what the disk scan misses; usually *more* current than local copies |
| **Chat — Teams / Slack** | Where decisions actually happen now; channel membership maps teams and projects for free |

**One API covers most of this:** **Microsoft Graph** alone = Outlook mail + calendar + OneDrive + SharePoint + Teams under a single OAuth. **Google Workspace APIs** = Gmail + Drive + Calendar + Docs. Direct integration with these two giants covers ~80% of Tier-1 value for a typical company — no wrapper needed for them.

### Tier 2 — structured business truth
- **CRM** — HubSpot, Salesforce, Pipedrive: canonical client/deal entities; anchors entity resolution ("ACME" in an email = ACME the account)
- **Project & tasks** — Jira, Notion, Asana, ClickUp, Monday, Trello: projects, ownership, status
- **Support** — Zendesk, Freshdesk, Intercom: what customers actually struggle with

### Tier 3 — contextual / vertical
- **Accounting/ERP** — QuickBooks, Xero (Greek market: SoftOne / Epsilon Net — usually via file exports, which the disk scan already catches)
- **Meetings** — Zoom/Meet transcripts, Fireflies-type notes: decisions in spoken form
- **Dev** — GitHub/GitLab (tech clients): repos, PRs, issues
- **E-sign** — DocuSign: contracts with authoritative dates/parties

Tier 2/3 arrive as **agent-pack companions** (§ARCHITECTURE 8): the client who buys the Sales agent pack gets the CRM connector; the Support agent pack brings Zendesk. Integrations and agents sell each other.

## 2. Unified API wrapper — the "one that has them all"

Current landscape (verified July 2026):

| Platform | Fit for us |
|---|---|
| **Nango** — open-source, self-hostable, ~800 APIs, code-first (Auth/Proxy/Functions primitives, TypeScript sync logic) | **Recommended.** Self-hosted on Rhooa infra → no third-party subprocessor ever touches client content; full control of sync logic; SOC 2 certified cloud option if we outgrow self-hosting |
| **Merge.dev** — most mature unified models (HRIS/ATS/CRM/files/ticketing) | Strong, but data proxies through *their* cloud → adds a subprocessor to the compliance story; pricier |
| **Unified.to** — broadest shallow catalog | Entry-level option; limited customization/observability |
| **Apideck** — per-request pricing | Thin per-category coverage |
| **Composio** — agent/MCP-oriented tool catalog | Interesting later for *agent actions* (write-side: send email, create task), not for bulk knowledge sync |

**Decision:** direct Microsoft Graph + Google Workspace connectors (they're the crown jewels and deserve first-class code), **self-hosted Nango** for the Tier-2/3 long tail, Composio-style MCP tooling considered later only for agent *actions*, never for ingestion.

## 3. Connector architecture — keeping "locally-stored" honest

Cloud-app data must end up in the same place disk data does: **the encrypted Brain on the user's PC** — and raw content must obey the same single-transit, zero-retention rules.

```
Provider (M365/Google/…)          Rhooa Cloud                      Employee PC
     │  webhooks: "X changed" ──► Dirty-flag store (ids only,          │
     │                            no content, persistent)              │
     │                                 │                               │
     │   agent online → pull      Connector runtime (Nango,       Local Agent
     │ ◄─────────────────────────  self-hosted) streams delta          │
     │                                 │                               │
     │                            /brain/ingest (embedding +           │
     │                            entities + relations, in-flight, ──► vectors + graph
     │                            zero retention)                      stored locally
```

Rules that make it compliant:

1. **OAuth handled by Rhooa cloud** (redirect URIs, token refresh — Nango's job); tokens encrypted per-seat, scopes **read-only**, per-user consent (admin consent flow for org-wide M365/Workspace grants).
2. **Webhooks store dirty flags only** — provider object ids, never content. This is the only integration state that persists server-side.
3. **Syncs run only while the seat's agent is connected** (agent-pull): the agent requests its deltas, the connector streams provider content through `/brain/ingest` *in-flight*, and derived knowledge (vectors, entities, relations) lands directly down the open connection into the local store. Nothing content-bearing persists in the cloud — same guarantee as the disk pipeline.
4. **Same privacy gate**, same audit log ("connector X ingested N items from source Y at T"), same purge cascade: disconnecting an integration deletes everything derived from it, locally, in one click.
5. **Instant Web Mode** gets connectors too (it's all cloud-side OAuth + the same ingest) — cloud-app content is actually *easier* zero-install than disk content, which softens Instant Mode's limits nicely.

## 4. What integrations do to the graph

- **Entity anchoring:** CRM accounts and M365/Workspace directory entries become high-confidence canonical nodes — entity resolution (GRAPH-TRAINING §4) suddenly has ground truth to merge against ("K. Papadopoulos" the email sender = the AD user = the CRM contact owner).
- **Edge density:** email/calendar/chat produce the `Person↔Person↔Project` edges that make community detection (and therefore god-node summaries) dramatically better than files alone.
- **Freshness:** webhook-driven deltas mean the Brain knows about this morning's thread — the disk scan could never.
- **Dedup for free:** the same PDF in Drive, SharePoint and an email attachment collapses to one chunk set via the existing content-hash cache.

## 5. Phasing

- **v1.0:** Microsoft Graph (mail, calendar, OneDrive/SharePoint, Teams) + Google Workspace — direct connectors. That's most of the value.
- **v1.1:** self-hosted Nango runtime + Slack, HubSpot, Notion, Jira (most-requested long tail).
- **v1.2+:** Tier-2/3 connectors bundled with their agent packs; MCP action tooling for agents (write-side) evaluated separately.
