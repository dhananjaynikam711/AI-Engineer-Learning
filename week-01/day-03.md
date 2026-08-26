# Day 3 — RAG Fundamentals

---

## Learning Objectives

By the end of this lesson, you should understand:

1. What **RAG** is and when to use it versus fine-tuning or prompt engineering alone
2. The end-to-end **RAG pipeline** from documents to grounded answers
3. **Chunking strategies** and how chunk size affects retrieval quality
4. **Embedding models** and vector search mechanics (including similarity metrics)
5. **Hybrid search**, **metadata filtering**, and **reranking** — when and why
6. How **citations** and **grounding instructions** reduce hallucination
7. How to **design an enterprise RAG system** — pipeline, authorization, and production tradeoffs

---

## Why This Matters

RAG is the most common enterprise LLM pattern:

- Internal knowledge assistants
- Document Q&A (policies, manuals, contracts)
- Customer support over product docs
- Regulated environments where answers must trace to sources

Understanding RAG end-to-end is essential for building production knowledge systems. A useful design checkpoint to work toward:

> "How would I build an enterprise RAG system for this use case?"

This lesson gives you the complete pipeline mental model plus production tradeoffs.

---

## Prerequisites

- Day 1: RAG definition; app vs RAG vs agent
- Day 2: Embeddings, tokens, context window, inference, message roles

---

## Core Concepts

### 1. What Is RAG?

**Definition:** Retrieval-Augmented Generation combines **information retrieval** (find relevant documents) with **LLM generation** ( synthesize an answer from retrieved content).

**Without RAG:**

```
User question → LLM (parametric memory only) → Answer
```

Model only "knows" what was in training data — stale, no private docs, hallucination risk.

**With RAG:**

```
User question → Retrieve relevant chunks → LLM + chunks as context → Grounded answer
```

**Intuition:** Open-book exam vs closed-book exam. RAG gives the model the "book" (your corpus) at query time.

---

### 2. The RAG Pipeline

```
Documents (PDF, HTML, wiki, tickets)
 ↓
 Ingestion / parsing
 ↓
 Chunking
 ↓
 Embedding
 ↓
 Vector database (+ metadata)
 ↓
 ─ ─ ─ ─ query time ─ ─ ─ ─
 ↓
 Embed user question
 ↓
 Vector search (+ filters)
 ↓
 Optional: rerank
 ↓
 Build prompt (system + context + question)
 ↓
 LLM
 ↓
 Answer + citations
```

Each stage is an engineering decision point.

---

### 3. Ingestion and Parsing

**Goal:** Extract clean text (and structure) from source formats.

| Source | Challenges |
|--------|------------|
| PDF | Layout, tables, headers, scanned images (OCR) |
| HTML / wiki | Navigation boilerplate, links |
| Word / DOCX | Styles, embedded objects |
| Tickets / CRM | Structured fields + free text |
| Code repos | Syntax, file boundaries |

**Engineering practices:**

- Preserve metadata: source URL, title, author, date, page number, department, classification
- Idempotent ingestion (re-run safe); track document version
- Incremental updates (only re-embed changed docs)

**Enterprise example:** Manufacturing equipment manuals in PDF → extract per-page text with page numbers for citation.

---

### 4. Chunking

**Goal:** Split documents into pieces small enough to embed, retrieve, and fit in context — large enough to be meaningful.

**Why chunking matters:**

- Embedding models have input limits (e.g., 8K tokens)
- Retrieval returns whole chunks — too large = noise; too small = lost context

**Common strategies:**

| Strategy | Description | Best for |
|----------|-------------|----------|
| **Fixed size** | N tokens/chars with overlap | General purpose |
| **Recursive** | Split by paragraphs → sentences → words | Structured docs |
| **Semantic** | Split where embedding similarity drops | Variable-density content |
| **Document-structure** | By heading, section, page | Manuals, policies, legal |
| **Parent-child** | Small chunks for search; retrieve parent for context | Precision + context |

**Overlap:** 10–20% overlap between consecutive chunks prevents sentences from being split across boundaries without context.

**Typical sizes:**

| Size | Tradeoff |
|------|----------|
| 256–512 tokens | Precise retrieval; may lack context |
| 512–1024 tokens | Common default for enterprise docs |
| 1024–2048 tokens | More context per chunk; noisier retrieval |

**Failure modes:**

- Chunks split mid-table → garbage retrieval
- Chunks too large → retrieval misses specific fact buried in chunk
- No metadata → cannot filter or cite precisely

---

### 5. Embeddings and Vector Search

**Indexing (offline):**

```
chunk text → embedding model → vector [0.12, -0.45, ...] → store in vector DB
```

**Query (online):**

```
user question → same embedding model → query vector → nearest neighbors in vector DB
```

**Similarity metrics:**

| Metric | Notes |
|--------|-------|
| **Cosine similarity** | Most common; angle between vectors |
| **Dot product** | Related; sensitive to magnitude |
| **Euclidean (L2) distance** | Some DBs expose as distance (lower = closer) |

**Important:** Index and query with the **same embedding model and dimensions**. Mixing models breaks retrieval.

**Top-k:** Retrieve k nearest chunks (e.g., 5–20). Too few = miss info; too many = noise and token waste.

**Score thresholds:** Discard chunks below similarity threshold → "I don't know" is better than wrong answer.

---

### 6. Vector Databases

**Role:** Store vectors + metadata; perform approximate nearest neighbor (ANN) search at scale.

**Technology examples (conceptual first):**

| Option | Characteristics |
|--------|-----------------|
| **pgvector** (PostgreSQL) | Leverage existing RDS/Aurora; good for moderate scale; SQL + vectors |
| **OpenSearch / Elasticsearch** | Hybrid search native; AWS integration |
| **Pinecone / Weaviate / Qdrant** | Managed vector specialty |
| **Chroma / FAISS** | Dev/prototype; local |

**Selection factors:**

- Existing stack (you know RDS → pgvector is natural)
- Scale (millions of vectors?)
- Hybrid search need
- Metadata filter performance
- Ops burden (managed vs self-hosted)

**Metadata filtering (critical for enterprise):**

```sql
-- Conceptual: vectors WHERE department = 'HR' AND classification <= user_clearance
```

User at Plant A should not retrieve Plant B documents — enforce via metadata filter **before or during** vector search, not hope the LLM ignores them.

---

### 7. Hybrid Search

**Problem:** Pure vector search misses exact keyword matches (SKUs, error codes, regulation numbers).

**Hybrid:** Combine vector similarity + keyword search (BM25).

```
Vector results ──┐
 ├──→ Merge / rerank ──→ Final top-k
BM25 results ──┘
```

**When to use:** Technical docs, legal citations, product IDs, log error strings.

---

### 8. Reranking

**Problem:** Bi-encoder retrieval (embed query, embed doc, compare) is fast but approximate. Cross-encoder reranking is slower but more accurate.

**Pattern:**

1. Retrieve top 50 with vector search (fast)
2. Rerank to top 5 with reranker model (Cohere rerank, cross-encoder)
3. Send top 5 to LLM

**Tradeoff:** Extra latency (+100–300ms) and cost vs better precision.

---

### 9. Prompt Construction and Grounding

**System prompt (policy layer):**

```
Answer ONLY using the provided CONTEXT.
If CONTEXT does not contain the answer, say "I don't have enough information."
Cite sources using [1], [2] format.
Do not invent procedures or policy details.
```

**User prompt (context injection):**

```
CONTEXT:
[1] (source: HR/policy.pdf, page 12)
... chunk text ...

[2] (source: HR/benefits.pdf, page 3)
... chunk text ...

QUESTION: What is the parental leave policy?
```

**Grounding:** Tying generation to retrieved text. Reduces but does not eliminate hallucination.

---

### 10. Citations

**Why:** Trust, auditability, compliance, user verification.

**Implementation:**

- Store chunk metadata: `document_id`, `page`, `url`, `section`
- Pass to LLM in context labels
- Post-process: map `[1]` to structured citation object in API response
- Optional: highlight in UI

**Enterprise:** Regulated industries may require citation for every factual claim.

---

### 11. Hallucination Reduction (RAG-Specific)

| Technique | Mechanism |
|-----------|-----------|
| Grounding instructions | Constrain to context |
| Low temperature | Less creative fabrication |
| Score threshold | Don't answer if retrieval weak |
| "I don't know" fallback | Explicit out-of-domain handling |
| Citations required | Model must reference chunks |
| Evaluation | Measure groundedness vs source |

**Reality:** RAG reduces hallucination; it does not eliminate it. Model may misread or over-generalize retrieved text.

---

### 12. RAG vs Fine-Tuning vs Prompt Engineering

| Approach | When to use |
|----------|-------------|
| **Prompt engineering** | Behavior/format changes; small knowledge in prompt |
| **RAG** | Dynamic/large/private knowledge; need citations; frequent updates |
| **Fine-tuning** | Style, format, domain language; task-specific behavior; stable knowledge |
| **Combined** | Fine-tune for tone + RAG for facts (common enterprise pattern) |

**Recommended framing:** "Default to RAG for enterprise knowledge that changes and must be cited. Fine-tune when you need consistent output format or domain language and knowledge is stable. Don't fine-tune to teach facts that belong in a retrieval corpus."

---

## Mental Model

```
 ENTERPRISE RAG (Query Path)

Employee → API (auth) → Embed query → Vector DB (+ ACL filter)
 ↓
 Top-k chunks
 ↓
 Prompt builder + system rules
 ↓
 LLM
 ↓
 Answer + citations + confidence
 ↓
 Logs / eval metrics
```

---

## Engineering Perspective

**When RAG is useful:**

- Private or frequently updated knowledge
- Need source attribution
- Corpus too large for context window
- Multiple document types and sources

**When RAG is NOT useful:**

- Purely structured data queries → SQL/API
- Real-time data → direct API integration (agent tool)
- Small static FAQ → prompt engineering or fine-tune may suffice
- Tasks requiring reasoning over entire corpus at once → batch analytics, not chat RAG

**Scalability:**

- Ingestion: async pipelines (S3 → SQS → Lambda → embed → store)
- Query: cache frequent queries; read replicas; ANN indexes
- Re-indexing: version embeddings when model changes

**Latency budget (typical):**

| Stage | Time |
|-------|------|
| Embed query | 50–150ms |
| Vector search | 20–100ms |
| Rerank (optional) | 100–300ms |
| LLM generation | 1–10s (streaming) |

**Cost:**

- Embedding indexing: one-time per chunk + re-index on change
- Per query: embed + LLM tokens (retrieved chunks in input)

**Reliability:**

- Empty retrieval → explicit fallback message
- Stale index → sync strategy and freshness metadata
- Partial ingestion failures → monitoring and DLQ

---

## Common Mistakes

1. **Chunking entire PDF as one vector** — poor retrieval precision
2. **Wrong embedding model at query time** — silent quality collapse
3. **No metadata / ACL filtering** — data leakage across tenants or departments
4. **Retrieving too many chunks** — fills context, increases cost, adds noise
5. **No "I don't know" path** — model hallucinates when retrieval fails
6. **Ignoring ingestion quality** — garbage text in → garbage answers out
7. **No evaluation** — cannot tell if changes help or hurt
8. **Treating RAG as set-and-forget** — docs change; index must update

---

## Production Considerations

**Security:**

- AuthZ on retrieval (metadata filters tied to user identity)
- PII in documents — redact at ingestion or restrict access
- Prompt injection via malicious documents (indirect injection)

**Observability:**

- Log: query, retrieved chunk IDs, scores, latency, token usage
- Metrics: retrieval hit rate, empty retrieval rate, user feedback

**Reliability:**

- Idempotent ingestion workers
- DLQ for failed embeds
- Circuit breaker on LLM provider

**Governance:**

- Document approval workflow before indexing
- Audit: who queried what, which sources cited

**Performance:**

- Batch embedding during ingestion
- ANN index tuning (recall vs speed)

**Cost:**

- Smaller LLM if RAG provides good context
- Cache embeddings for repeated queries

---

## Example Architecture

**Enterprise document search (AWS-aligned — matches patterns you know):**

```
 INGESTION (async)
S3 (raw docs) → Event → SQS → Lambda (parse/chunk)
 ↓
 Bedrock Titan Embed
 ↓
 Aurora PostgreSQL + pgvector
 (chunks + metadata + ACL fields)

 QUERY (sync)
User → CloudFront → API Gateway → ECS/Fargate (RAG API)
 ↓
 Cognito JWT → user ACL claims
 ↓
 Embed question (Bedrock)
 ↓
 pgvector search + metadata filter
 ↓
 Bedrock Claude (grounded prompt)
 ↓
 Response + citations
 ↓
 CloudWatch + structured logs
```

**Example reference:** A split vectorizer + question-api deployment on Kubernetes with PDF ingestion, managed embeddings, and pgvector mirrors this pattern.

---

## Hands-On Exercise

**Exercise 1 — Whiteboard enterprise RAG (20 minutes)**

Answer on paper or whiteboard without looking at the lesson:

*"How would you build an enterprise RAG system for HR policy documents for 5,000 employees with department-level access control?"*

Include: ingestion, chunking, storage, auth, query path, citations, monitoring.

**Exercise 2 — Chunking decision (10 minutes)**

Given a 40-page equipment maintenance PDF with sections, tables, and warnings:

- What chunking strategy would you choose?
- What metadata would you store per chunk?
- What chunk size and overlap?

**Exercise 3 — Failure scenario (10 minutes)**

User asks: "What is the shutdown procedure for Pump X?"

Retrieval returns chunks about Pump Y with score 0.72. Threshold is 0.75.

What should the system do? What would you log? What product behavior do you want?

**Exercise 4 — RAG vs alternatives (5 minutes)**

For each, choose RAG, fine-tuning, prompt-only, or SQL/API — and one-sentence why:

1. Company wiki Q&A updated weekly
2. Classify support tickets into categories
3. Look up current inventory for SKU-12345
4. Match company's formal tone in all customer emails

---

## Practice Questions

Use these to test your understanding after completing the lesson. Answer in your own words before checking the summary below.

### Core concepts

1. What is RAG?
2. What is a chunk and why do we chunk documents?
3. What is an embedding in the context of RAG?
4. What is top-k retrieval?
5. How do citations help in enterprise RAG?

### Applied thinking

1. Walk through the full RAG pipeline from document upload to answer.
2. RAG vs fine-tuning — when do you choose each?
3. How do you enforce document-level authorization in retrieval?
4. What is hybrid search and when do you need it?
5. How do you handle "I don't know" when retrieval returns weak results?
6. What metrics would you track for a production RAG system?

### Design thinking

1. Design an enterprise RAG system for manufacturing knowledge (manuals, SOPs, incident reports) with 10,000 users across 5 plants.
2. Documents update daily. How do you keep the vector index fresh without full re-index?
3. LLM costs are too high. What optimizations do you consider?
4. Design RAG for manufacturing SOPs and safety procedures where wrong answers create liability.
5. How do you prevent cross-plant document leakage in a multi-site RAG system?

---

## Self Assessment

1. Draw the RAG pipeline from memory (至少 8 boxes).
2. Name three chunking strategies and when to use each.
3. Why must indexing and querying use the same embedding model?
4. What is metadata filtering and why is it non-optional in enterprise?
5. Explain hybrid search in two sentences.
6. What is reranking and what tradeoff does it introduce?
7. List four ways to reduce hallucination in RAG.
8. When would you NOT use RAG?
9. What should happen when no chunk passes the similarity threshold?
10. How does RAG relate to your context window constraints from Day 2?

---

## Key Takeaways

1. **RAG = retrieve then generate** — open-book LLM over your corpus.
2. **Ingestion and chunking** determine retrieval quality ceiling.
3. **Embeddings + vector search** find semantically similar chunks; same model for index and query.
4. **Metadata filtering** enforces enterprise authorization at retrieval time.
5. **Hybrid search + reranking** improve precision for keyword-heavy and high-stakes queries.
6. **Grounding prompts + citations + thresholds** reduce hallucination and enable trust.
7. **RAG vs fine-tuning:** RAG for dynamic, citable knowledge; fine-tune for behavior/style.
8. **Production RAG** requires observability, incremental indexing, and explicit "I don't know" paths.

---

## Summary

Short reference for the concepts covered today:

- **RAG** retrieves relevant document chunks at query time, injects them into the prompt, and asks the LLM to answer from that context — with citations.
- Chunk by document structure for manuals; store vectors with ACL metadata; filter by user claims before search; fall back when scores are below threshold.
- Use **hybrid search** when exact identifiers matter; use **reranking** when precision is critical; never mix embedding models between index and query.
- **RAG** is the default for enterprise knowledge that changes and must be auditable; **fine-tuning** is for stable behavior, not replacing a document corpus.

---

## Connection to Prior Experience

Map this lesson to systems your team may already know:

| Prior experience | RAG concept |
|-----------------|-------------|
| Document ingestion pipelines | Ingest + parse + chunk |
| SQL + RDS / PostgreSQL | pgvector storage pattern |
| S3 + Lambda + SQS | Event-driven ingestion |
| Enterprise auth (Cognito, IAM) | Metadata filtering / ACL |
| Agent orchestrators | RAG as a tool inside an agent loop |

Example reference implementation: a split **vectorizer + question-api** architecture with Kubernetes, PDF ingestion, Titan (or similar) embeddings, and pgvector — the pattern used in many enterprise RAG prototypes.

Day 4 (Tool Calling) connects RAG to agents: retrieval becomes one tool among many.
