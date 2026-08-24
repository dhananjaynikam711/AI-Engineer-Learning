# Day 4 — Enterprise RAG

> **Status:** IN PROGRESS (complete exercises and self assessment to mark done)

---

## Spaced Review (Days 1–3)

Answer mentally before continuing:

1. What is the difference between inference and training?
2. Why must the same embedding model be used at index time and query time?
3. What happens when retrieval returns no chunks above your similarity threshold?
4. RAG vs fine-tuning — one sentence each for when to use.
5. Name three components of the basic RAG pipeline in order.

---

## Learning Objectives

By the end of this lesson, you should understand:

1. How **enterprise RAG** differs from prototype/demo RAG
2. **Structure-aware** and **hierarchical chunking** for complex documents
3. **Metadata filtering** and **access control** — authorization *before* LLM context
4. **Document versioning**, **freshness**, and **staleness** handling
5. **Hybrid search** (semantic + keyword), **reranking**, **query rewriting**, and **query expansion**
6. How to design a **multi-tenant RAG** system with tenant isolation
7. How to **design multi-tenant RAG** with department- and site-level access control

---

## Why This Matters

Day 3 taught the RAG pipeline. **Enterprise RAG** is what separates a demo from something you can deploy in finance, manufacturing, or healthcare.

Prototype RAG fails in enterprise because:

- Users see documents they should not access
- Retrieval misses exact identifiers (part numbers, regulation IDs, error codes)
- Stale policies produce wrong answers with high confidence
- No audit trail for compliance
- One bad document poisons answers for everyone

Senior AI Engineers are expected to design RAG where **authorization, retrieval quality, and freshness** are first-class — not afterthoughts bolted on after the LLM works.

**Enterprise framing:** Manufacturing knowledge (SOPs, safety procedures, equipment manuals) requires **plant-level access**, **citation to source**, and **zero tolerance for cross-site data leakage**.

---

## Prerequisites

- **Day 1:** Model vs application; where LLMs fit in architecture
- **Day 2:** Embeddings, tokens, context window
- **Day 3:** Basic RAG pipeline, chunking, vector search, grounding, citations

If you cannot whiteboard Day 3's ingest → chunk → embed → retrieve → generate flow, review before continuing.

---

## Core Concepts

### Level 1 — Intuition: Demo RAG vs Enterprise RAG

| Demo RAG | Enterprise RAG |
|----------|----------------|
| All docs in one bucket | Docs tagged by dept, site, classification |
| Fixed-size chunks | Structure-aware chunks (headings, pages, tables) |
| Vector search only | Hybrid search + reranking |
| Re-index manually | Versioned docs, incremental sync, freshness metadata |
| No auth on retrieval | **Filter by user permissions before search** |
| "It works on my PDFs" | Evaluation pipeline, audit logs, SLAs |

**Core principle:** In enterprise RAG, **retrieval is a security boundary**, not just a search optimization.

---

### Level 2 — Structure-Aware and Hierarchical Chunking

**Problem with naive fixed-size chunking:**

- Splits mid-table → garbage embeddings
- Loses section hierarchy ("Section 3.2" context lost)
- Retrieves fragment without parent context

**Structure-aware chunking:** Split on document structure — headings, pages, paragraphs, list items — not arbitrary token counts.

| Strategy | How | Best for |
|----------|-----|----------|
| **By heading/section** | One chunk per H1/H2 section | Policies, manuals, wikis |
| **By page** | Page boundary + overlap | PDFs with layout (your ingest pipelines) |
| **Recursive** | Paragraphs → sentences when too large | Mixed-density docs |
| **Parent-child** | Small child chunks for search; retrieve parent for LLM context | Precision + context |

**Hierarchical chunking example (equipment manual):**

```
Document: Pump-A-Maintenance-Manual.pdf
├── Section: Safety Warnings (parent)
│ ├── Chunk: Warning 3.1 (child — embedded for search)
│ └── Chunk: Warning 3.2
├── Section: Shutdown Procedure (parent)
│ ├── Step 1 chunk
│ └── Step 2 chunk
```

**Retrieval pattern:** Search hits **child** chunk (precise match) → fetch **parent** section for LLM context (full procedure).

**Engineering implication:** Store `parent_id`, `section_title`, `page_number`, `document_version` in metadata on every chunk.

---

### Level 3 — Metadata, Access Control, and Authorization

**Metadata** is not optional in enterprise RAG. Every chunk carries:

| Metadata field | Purpose |
|----------------|---------|
| `document_id` | Stable identifier |
| `version` | Which revision was indexed |
| `source_url` / `s3_key` | Citation + re-fetch |
| `department`, `site`, `plant_id` | Access control filter |
| `classification` | public / internal / confidential |
| `effective_date`, `expires_at` | Freshness |
| `owner`, `approved_by` | Governance |

**Authorization model — enforce at retrieval, not in the prompt:**

```
WRONG: Retrieve all → send to LLM → hope it ignores unauthorized chunks
RIGHT: User JWT → resolve claims → filter vector query → only authorized chunks
```

**Flow:**

1. User authenticates (Cognito, SSO, enterprise IdP)
2. API resolves claims: `{ sites: ["plant-a"], departments: ["maintenance"], clearance: "internal" }`
3. Vector search includes filter: `site IN user.sites AND classification <= user.clearance`
4. Only permitted chunks enter the prompt

**Multi-tenant RAG:** Each tenant (business unit, customer, plant) has isolated metadata namespace. Never share vector index rows across tenants without explicit filter — prefer separate collections or partition keys.

**Connection to experience:** Same pattern as row-level security in SQL/RDS — apply at query time, not application hope.

---

### Level 4 — Document Versioning and Freshness

**Versioning:**

- Each document revision gets a new `version` or `content_hash`
- On update: re-chunk, re-embed, upsert (or delete old version chunks)
- Citations must reference **specific version** user saw

**Freshness strategies:**

| Strategy | Mechanism |
|----------|-----------|
| **Event-driven re-index** | S3 upload → SQS → Lambda re-ingests changed doc |
| **Scheduled sync** | Nightly crawl of SharePoint/wiki |
| **TTL metadata** | Flag chunks older than N days for re-validation |
| **Source-of-truth pointer** | Store `indexed_at`; display "last updated" in UI |

**Stale knowledge failure:** User asks about 2026 policy; index has 2024 PDF. LLM answers confidently from stale chunk.

**Mitigations:**

- Prefer source systems with version APIs
- Include `effective_date` in metadata; filter `effective_date <= today`
- Evaluation golden set detects stale answers after doc updates
- "Knowledge as of {date}" in UI when freshness uncertain

---

### Level 5 — Hybrid Search, Reranking, and Query Pipeline

**Semantic search alone fails when:**

- Exact SKU, error code, regulation number, part ID needed
- Rare technical terms not well represented in embeddings
- User query uses different words than document

**Hybrid search:** Combine **vector (semantic)** + **keyword (BM25)** results.

```
User query
 ├─→ Embed query ──→ Vector search (top 50)
 └─→ Tokenize query ──→ BM25 / keyword search (top 50)
 ↓
 Merge + dedupe (RRF or weighted score)
 ↓
 Optional rerank (top 50 → top 5)
 ↓
 Build prompt → LLM
```

**Reciprocal Rank Fusion (RRF):** Merge ranked lists without normalizing incompatible scores — common in OpenSearch hybrid queries.

**Reranking:** Cross-encoder or dedicated reranker model scores (query, chunk) pairs. Slower but more accurate than bi-encoder retrieval alone.

| Stage | Speed | Accuracy |
|-------|-------|----------|
| Bi-encoder retrieval | Fast | Good |
| Hybrid (BM25 + vector) | Medium | Better for exact matches |
| Reranker | Slower | Best precision |

**Query rewriting:** LLM or rules transform user query before search.

- "PTO" → "paid time off leave policy"
- Typo correction, acronym expansion

**Query expansion:** Generate alternate phrasings; search with multiple queries; merge results.

- Original: "shutdown pump A"
- Expanded: ["pump A shutdown procedure", "deactivate pump A", "Pump-A emergency stop"]

**Tradeoff:** Each rewrite/expansion adds latency and cost. Use for high-stakes queries or when initial retrieval score is low.

---

### Level 6 — Enterprise Retrieval Pipeline (End-to-End)

**Offline (ingestion):**

```
Source (SharePoint / S3 / wiki / ticket system)
 ↓
Extract + normalize text
 ↓
Structure-aware chunk + metadata enrichment
 ↓
Embed (batch)
 ↓
Upsert to vector store with ACL fields + version
 ↓
Index audit log
```

**Online (query):**

```
User question + auth token
 ↓
Resolve user claims
 ↓
Query pipeline:
 optional rewrite/expansion
 hybrid search + metadata filter
 optional rerank
 score threshold check
 ↓
If no chunks pass → "I don't know" (no LLM call or minimal call)
 ↓
Build grounded prompt + citations
 ↓
LLM generate
 ↓
Log: query, chunk IDs, scores, latency, user, model
```

---

## Mental Model

```
 ENTERPRISE RAG = RETRIEVAL + SECURITY + FRESHNESS

 ┌─────────────── INGEST (offline) ───────────────┐
 │ Doc → structure chunk → metadata → embed │
 │ version, ACL, site, classification │
 └──────────────────────┬─────────────────────────┘
 ↓
 ┌─────────────── QUERY (online) ───────────────┐
 │ Auth → claims → filter → hybrid search │
 │ → rerank → threshold → ground → LLM │
 └───────────────────────────────────────────────┘

 If any layer fails: prefer NO ANSWER over WRONG ANSWER
```

---

## Architecture Diagram

**Enterprise RAG on AWS (maps to your stack):**

```
 INGESTION
┌─────────┐ ┌─────────┐ ┌──────────────┐ ┌─────────────────┐
│ S3 / │───→│ SQS │───→│ Lambda/ECS │───→│ Embed (Bedrock │
│ SharePoint │ event │ │ parse/chunk │ │ Titan, etc.) │
└─────────┘ └─────────┘ └──────────────┘ └────────┬────────┘
 ↓
 ┌─────────────────┐
 │ Aurora/pgvector │
 │ or OpenSearch │
 │ + ACL metadata │
 └─────────────────┘

 QUERY
┌─────────┐ ┌──────────────┐ ┌──────────────────────────────┐
│ User │───→│ API Gateway │───→│ RAG Service (ECS/Lambda) │
│ (SSO) │ │ + Cognito │ │ 1. Resolve claims │
└─────────┘ └──────────────┘ │ 2. Query rewrite (optional) │
 │ 3. Hybrid search + ACL filter│
 │ 4. Rerank │
 │ 5. Threshold + prompt build │
 │ 6. Bedrock Claude / LLM │
 └──────────────┬───────────────┘
 ↓
 ┌──────────────────────────────┐
 │ Response + citations │
 │ CloudWatch / structured logs │
 └──────────────────────────────┘
```

**Technology-neutral note:** Same pattern on Azure (Blob + AI Search), GCP, or on-prem — layers stay the same; swap storage and embedding providers.

---

## Engineering Perspective

**When enterprise RAG patterns are required:**

- Multi-site / multi-department document access
- Regulated or safety-critical content (manufacturing, finance, healthcare)
- Large corpus with mixed doc types (PDF, wiki, tickets)
- Need audit trail and citations

**When basic RAG suffices:**

- Single-team internal tool, all users see same docs
- Small static corpus (<500 docs), low stakes
- Prototype / POC phase (but plan enterprise path early)

**Scalability:**

- Ingestion: async queue workers; batch embeddings; parallel by document
- Query: read replicas; ANN index tuning; cache frequent queries + embeddings
- OpenSearch/pgvector: partition by tenant or site if metadata filters become hot paths

**Latency budget (typical enterprise query):**

| Stage | Time |
|-------|------|
| Auth + claims | 10–50ms |
| Query rewrite (optional) | 200–800ms |
| Hybrid search | 50–200ms |
| Rerank | 100–400ms |
| LLM generation | 1–10s |

**Reliability:** Idempotent ingestion; DLQ for failed embeds; circuit breaker on LLM; fallback message when retrieval empty.

---

## Tradeoffs

| Decision | Option A | Option B | Guidance |
|----------|----------|----------|----------|
| Chunking | Fixed size | Structure-aware | Enterprise docs → structure-aware |
| Search | Vector only | Hybrid | Add BM25 when exact IDs/terms matter |
| Reranking | None | Cross-encoder | High-stakes Q&A → rerank top 50 → 5 |
| ACL | Post-retrieval filter | Pre-filter in query | **Always pre-filter in query** |
| Index | Shared index + metadata | Separate index per tenant | Shared OK with strict filters; separate for hard isolation |
| Freshness | Full re-index nightly | Event-driven on change | Prefer event-driven |
| Query rewrite | Always | On low retrieval score | Conditional saves cost |

---

## Common Failure Modes

| Failure | Symptom | Fix |
|---------|---------|-----|
| **ACL after retrieval** | Data leakage across departments | Filter in vector query |
| **Stale index** | Wrong policy answers | Event-driven re-index + version metadata |
| **Chunk too small** | Missing context in answer | Parent-child retrieval |
| **Chunk too large** | Retrieval misses specific fact | Smaller chunks or hierarchical |
| **Vector-only search** | Misses part number "XJ-4429" | Hybrid search |
| **No threshold** | LLM hallucinates when retrieval weak | Score threshold + "I don't know" |
| **Poisoned document** | Malicious text in corpus affects answers | Source approval, sanitization, indirect injection defenses (Week 7) |
| **Embedding model change** | Silent quality collapse | Re-index entire corpus; version embedding model in metadata |
| **Citation without version** | User cannot verify which policy revision | Include version + page in citation |

---

## Production Considerations

- **SLA:** Define p95 latency and availability for query API
- **Idempotency:** Ingestion keyed by `document_id + version`
- **Backpressure:** Queue depth alerts on ingestion backlog
- **Rate limits:** Per-user query limits to control cost
- **Human review:** Approval workflow before indexing external or user-uploaded docs

---

## Security Considerations

- **Authorization before context** — non-negotiable
- **Tenant isolation** — metadata or physical separation
- **Indirect prompt injection** — documents can contain "ignore instructions" text; combine retrieval filtering with system prompt hardening
- **PII in corpus** — redact at ingest or restrict access; log scrubbing
- **Audit:** Log who queried what, which chunks retrieved (not always full text — consider policy)

---

## Observability Considerations

Log and metric per query:

| Signal | Why |
|--------|-----|
| `retrieval_chunk_ids` | Debug wrong answers |
| `retrieval_scores` | Tune threshold |
| `filter_applied` | Verify ACL |
| `chunks_before/after threshold` | Detect retrieval degradation |
| `embedding_model`, `index_version` | Regression tracking |
| `latency_ms` per stage | Bottleneck analysis |
| `empty_retrieval_count` | Corpus or ACL gaps |

**Alert on:** Spike in empty retrievals, drop in average top score, ingestion DLQ depth.

---

## Cost Considerations

| Cost driver | Control |
|-------------|---------|
| Embedding at ingest | Batch; skip unchanged docs (content hash) |
| Query embedding | Cache query vectors for repeated questions |
| Reranker | Apply only when top score below threshold |
| Query rewrite LLM | Conditional, not every query |
| LLM input tokens | Limit chunks sent; parent-child to avoid duplication |
| Vector storage | Archive old versions; tier cold docs |

---

## Hands-On Exercise

**Exercise 1 — Metadata schema (15 minutes)**

Design a metadata schema for chunks in a **multi-plant manufacturing** knowledge base. List at least 10 fields and explain which are used for ACL vs citation vs freshness.

**Exercise 2 — ACL filter (10 minutes)**

User claims: `{ plant_id: "kingsport", department: "maintenance", clearance: "internal" }`

Write a pseudocode filter applied **before** vector search. What happens if a chunk has `plant_id: "longview"`?

**Exercise 3 — Hybrid search scenario (10 minutes)**

User searches: `error code E-4429 pump seal failure`

Would vector-only search suffice? What would you add? Sketch the query pipeline stages.

---

## Design Exercise

**Design: Enterprise HR + Safety Document Assistant**

Requirements:

- 5,000 employees across 3 plants
- Documents: HR policies (company-wide), safety SOPs (plant-specific), equipment manuals (department-specific)
- SSO authentication; users must only see authorized docs
- Citations required; audit log for compliance
- Documents update weekly

Deliver on paper:

1. Ingestion architecture (event-driven)
2. Chunking strategy per document type
3. Metadata schema
4. Query pipeline (include hybrid + threshold decision)
5. What you log for observability
6. How you handle a document update mid-week

---

## Practice Questions

Use these to test your understanding after completing the lesson. Answer in your own words before checking the summary below.

### Core concepts

1. How does enterprise RAG differ from basic RAG?
2. What is structure-aware chunking?
3. What is parent-child chunking?
4. Why enforce access control at retrieval time?
5. What is hybrid search?

### Applied thinking

1. Walk through your enterprise RAG query path from SSO login to cited answer.
2. How do you handle document versioning in a RAG index?
3. When do you add reranking? What is the cost tradeoff?
4. Explain query rewriting vs query expansion.
5. How do you detect and handle stale knowledge?
6. What metadata fields are essential for multi-tenant RAG?

### Design thinking

1. Design RAG for 10,000 employees with department and site-level access control.
2. A user reports seeing another plant's safety procedure in an answer. How do you diagnose and fix?
3. Retrieval quality dropped after switching embedding models. What is your migration plan?
4. How would you build a RAG system for manufacturing SOPs where a wrong answer could create safety liability?
5. How do you handle document access across multiple sites and languages in a global enterprise?
6. How do you balance comprehensive retrieval (more chunks) vs context window and cost limits?

---

## Self Assessment

1. Why is authorization a retrieval problem, not an LLM problem?
2. Draw parent-child chunk hierarchy for a equipment manual.
3. Name three metadata fields for ACL and three for citations.
4. When would you use hybrid search over vector-only?
5. What is reranking and what latency does it add?
6. How does event-driven ingestion improve freshness?
7. What should happen when no chunk passes the similarity threshold?
8. Name two observability metrics for enterprise RAG.
9. What is indirect prompt injection via documents?
10. How does Day 4 build on Day 3?

---

## Key Takeaways

1. **Enterprise RAG = Day 3 pipeline + ACL + freshness + hybrid retrieval + observability.**
2. **Authorize at retrieval** — filter vector queries by user claims, never trust the LLM to ignore chunks.
3. **Structure-aware and hierarchical chunking** improve precision and context for real documents.
4. **Hybrid search + reranking** address semantic search weaknesses with exact terms.
5. **Versioning and event-driven re-index** prevent stale knowledge failures.
6. **Query rewriting/expansion** helps but adds cost — use conditionally.
7. **Prefer no answer over wrong answer** when retrieval confidence is low.
8. **Log retrieval traces** — you cannot debug enterprise RAG without them.

---

## Summary

Short reference for the concepts covered today:

- **Enterprise RAG** extends basic RAG with structure-aware chunking, metadata-rich indexing, authorization filters at retrieval time, hybrid search, optional reranking, document versioning with event-driven re-indexing, and observability on retrieval scores and chunk IDs.
- Never send unauthorized chunks to the LLM — user claims from SSO resolve to metadata filters on the vector query before any text enters the prompt.
- For regulated content, use parent-child chunking, hybrid search for part numbers and error codes, score thresholds with an explicit I-don't-know path, and citations tied to document version and page.
- When embedding models change, re-index the full corpus and track index version in logs to correlate quality regressions.

---

## Connection to Prior Experience

| Your background | Enterprise RAG mapping |
|-----------------|------------------------|
| **S3 + SQS + Lambda** | Event-driven document ingestion pipeline |
| **RDS / SQL** | pgvector with metadata filters; familiar ACL patterns |
| **Cognito / enterprise auth** | JWT claims → retrieval filters |
| **CloudWatch** | Retrieval trace logging, latency dashboards |
| **Step Functions** | Multi-step ingest workflows (parse → chunk → embed → validate) |
| **rag-pgvector pattern** | Extend with ACL metadata, hybrid search, versioning |
| **ISEC / AI governance** | Document approval before index; audit logging |

**Honest gap:** Production hybrid search + reranking + formal eval pipeline — until Project 3 and Week 3.

---

## Next

**Day 5 — RAG Evaluation** — measure whether enterprise RAG actually works (retrieval vs generation failures, golden datasets, LLM-as-judge).

When ready: complete the self assessment, then say **"Quiz me on Day 4"** or **"Generate Day 5"**.
