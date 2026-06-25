# Interview agent

End-to-end system for discovering, evaluating, and tracking automation candidates — from stakeholder interview to prioritised Linear ticket, backed by a scalable OKF knowledge base in Obsidian.

---

## What this is

This project automates the automation discovery process itself. Stakeholder interviews are transcribed, structured by an extraction agent, scored by a sizing agent, and converted into Linear tickets — with a persistent knowledge base (WikiStore) at the centre that grows with every discovery session.

The WikiStore is the shared memory of the system. Agents read from it to make better decisions (deduplication, context, sizing benchmarks) and write to it after each step. It is LLM-managed and human-curated — the team controls what information enters the system, the agents handle the bookkeeping.

---

## Architecture

```
Raw sources
├── Interview recording → transcription
└── Additional stakeholder documents
        │
        ▼
Extraction skill          ──writes──▶   WikiStore (Obsidian vault)
        │                                ├── /raw-sources/       ← immutable
        │                                ├── /wiki/              ← OKF .md files
        ▼                                │    ├── index.md       ← agent-agnostic schema
Sizing skill              ◀──reads──    │    ├── {process}.md   ← one file per process
        │                                │    └── ticket-log.md  ← lightweight ticket refs
        ▼                                └── embeddings index    ← vector store for retrieval
Ticket creation agent     ◀──reads──
        │
        ▼
Linear                    ──appends log entry──▶  WikiStore
```

---

## Key design decisions

**WikiStore is decoupled from Linear.** Linear is the source of truth for tickets. Obsidian is the source of truth for process knowledge. The ticket creation agent writes to both — a full ticket in Linear, a lightweight log entry (ID, process name, score, date) in the wiki. No live sync, no duplication of ticket content.

**OKF format.** All wiki entries follow the [Open Knowledge Format](https://cloud.google.com/blog/products/data-analytics/how-the-open-knowledge-format-can-improve-data-sharing) — markdown files with YAML frontmatter. This makes the knowledge base agent-agnostic: any LLM can read and write entries without a custom integration. The `index.md` at the vault root acts as the schema and navigation guide for agents.

**Two-phase retrieval strategy.** The system starts with keyword matching for simplicity, with a planned migration to embedding-based retrieval as the wiki grows. See the Indexing & retrieval section for the full breakdown.

**Specialist agents, shared memory.** Each agent has a single responsibility. The extraction skill does not size. The sizing skill does not create tickets. All agents read from the same WikiStore, which means they benefit from accumulated context without being coupled to each other.

**Scalability is a first-class concern.** The YAML frontmatter schema is defined upfront so retrieval stays fast as the wiki grows. The index file acts as a cheap first-pass filter — agents scan it before loading full entries. These conventions must be maintained consistently; retrofitting them later is expensive.

---

## Skills

| Skill | Trigger | Output |
|---|---|---|
| `interview-extraction` | Paste a transcript | OKF wiki entry + raw transcript saved to WikiStore |
| `automation-sizing` | Pass extracted process data | Scored sizing card + priority tier |

---

## WikiStore structure

```
/obsidian-vault
├── index.md                  ← OKF schema, navigation guide for agents
├── /raw-sources/             ← immutable originals
│   ├── {process}/transcript.md
│   └── {process}/docs/
├── /wiki/                    ← LLM-managed OKF entries
│   ├── {process-name}.md     ← one per discovered process
│   └── ticket-log.md         ← append-only log of created tickets
└── /embeddings/              ← vector index (not human-edited)
```

Each process entry (`{process-name}.md`) links back to its raw source via the OKF `resource` field, so any entry can be traced back to the original transcript.

---

## Indexing & retrieval strategy

Retrieval is the main scalability bottleneck — as the wiki grows, agents must find the right entries quickly without loading everything into context. The strategy is two-phased: start simple, migrate to semantic search once volume justifies it.

### Indexing

Every OKF wiki entry includes a YAML frontmatter block that acts as the index. This is populated by the extraction skill at write time and is the foundation for both retrieval approaches.

```yaml
---
type: process
title: Invoice reconciliation
description: Monthly matching of supplier invoices against PO records.
business_unit: Finance
tags: [finance, invoicing, reconciliation]
status: sized
sizing_tier: quick-win
ticket_id: ENG-204
timestamp: 2026-06-20T10:30:00Z
resource: /raw-sources/invoice-reconciliation/transcript.md
---
```

The `index.md` at the vault root maintains a flat summary table of all entries — agents scan this first to identify candidates before loading full documents.

**Key principle:** the frontmatter schema must be consistent across all entries. Gaps or inconsistencies degrade retrieval quality for both phases. Define the schema once and enforce it in the extraction skill.

---

### Phase 1 — Keyword matching *(current)*

Agents filter wiki entries by matching terms in the YAML frontmatter fields — tags, business unit, status, sizing tier. This is the starting point: low infrastructure overhead, fast to implement, and sufficient for a small wiki.

**How it works:** an agent queries the index for entries where `business_unit = Finance` and `tags` contains `invoicing`. The index returns matching file paths; the agent loads only those.

**Limitations at scale:** keyword matching only finds exact or near-exact terms. Two processes described differently but functionally similar will not surface as related. Deduplication becomes unreliable as volume grows.

**Suitable for:** early prototype, up to ~50–100 wiki entries.

---

### Phase 2 — Embedding-based retrieval *(planned)*

When the wiki reaches a scale where keyword matching degrades — typically when deduplication misses start appearing, or agent context becomes noisy — the system migrates to vector embeddings.

**How it works:** when the extraction skill writes a new wiki entry, it also generates a vector embedding of the entry content and stores it in a vector index alongside the vault. At query time, an agent embeds its query and retrieves the most semantically similar entries by cosine distance — no exact keyword match needed.

**Advantages over keyword matching:**
- Finds semantically similar processes even when described with different terminology
- More reliable deduplication ("has something like this already been ticketed?")
- Scales predictably — retrieval cost stays low regardless of wiki size

**Infrastructure required:** a lightweight vector store alongside Obsidian (e.g. Chroma, FAISS, or a hosted option). The extraction skill needs a small update to generate and store embeddings on write.

**Migration path:** keyword and embedding retrieval can run in parallel during transition. Embeddings do not replace the YAML frontmatter index — both are used together, with frontmatter as a pre-filter and embeddings for semantic ranking.

---

## Sizing model

TBC

---

## Open questions

- [ ] Finalise YAML frontmatter schema for extraction skill output
- [ ] Choose vector store for Phase 2 (Chroma vs FAISS vs hosted option)
- [ ] Define re-scoring cadence (when does a process get re-sized?)
- [ ] Agree on ticket creation flow — manual approval step or fully automated?
