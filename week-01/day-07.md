# Day 7 — Advanced RAG: Query Understanding, Context Engineering & RAG Patterns

---

## Spaced Review (Days 1–6)

Answer mentally before continuing:

1. What are the four major concerns of enterprise RAG?
2. What is the difference between precision and recall?
3. What are the three pillars of observability?
4. Why can response caching be dangerous in RAG?
5. How does a production feedback loop connect to evaluation?

---

## Learning Objectives

By the end of this lesson, you should be able to explain:

1. Why simple query → vector search → LLM is not enough for complex real-world questions
2. What query understanding is and what information it extracts from a user question
3. The difference between query rewriting, query expansion, and multi-query retrieval
4. What query decomposition and multi-step retrieval are, and when they help
5. What context engineering is and why more context is not always better
6. How context ordering, compression, and assembly affect generation quality
7. What parent-child and hierarchical retrieval are
8. How metadata-aware retrieval narrows the search space
9. Why structured data questions may not belong in vector search
10. What query routing is and when to route to RAG, SQL, or APIs
11. What agentic RAG is and how it differs from traditional RAG
12. Why agentic RAG introduces additional risks and requires strong boundaries
13. Who should enforce authorization in an agentic system
14. When to introduce advanced RAG techniques versus keeping the system simple

---

## Why This Matters

Week 1 progression:

| Day | Topic |
|-----|-------|
| Day 1 | AI / ML / LLM landscape |
| Day 2 | LLM fundamentals |
| Day 3 | RAG fundamentals |
| Day 4 | Enterprise RAG architecture |
| Day 5 | RAG evaluation |
| Day 6 | Production RAG |
| **Day 7** | **Advanced RAG patterns and context engineering** |

Days 1–6 built the RAG foundation and production side. Day 7 steps back to connect everything and explore how to make RAG systems substantially better when a simple:

> Query → Vector Search → LLM

architecture is not enough.

Today is **not about learning another giant list of technologies**. The goal is to understand **how to make retrieval and context smarter** — and when to add complexity based on demonstrated problems, not because a pattern sounds interesting.

The primary focus is **engineering understanding**. The priority is understanding what each pattern solves, when to use it, and how evaluation tells you whether it helped.

---

## Prerequisites

Before starting Day 7, you should understand:

- Basic and enterprise RAG architecture (Days 3–4)
- Query rewriting and hybrid search (Day 4)
- RAG evaluation and golden datasets (Day 5)
- Production observability, caching, and cost optimization (Day 6)

If you cannot explain when to evaluate retrieval before generation, or why authorization must happen before the LLM, review Days 4–6 before continuing.

---

## Core Concepts

### 1. What Problem Are We Solving?

Basic RAG:

```
User Question → Embedding → Vector Search → Top K Chunks → LLM → Answer
```

This works surprisingly well. But real-world questions are not always simple.

Consider:

> "What changed between the 2024 and 2026 safety procedures for handling Chemical X at Plant 2, and what should an employee currently do if there is a spill?"

We potentially need to:

1. Understand the question
2. Identify multiple sub-questions
3. Find multiple documents
4. Compare document versions
5. Determine the current procedure
6. Retrieve the emergency procedure
7. Combine the information
8. Generate a coherent answer

A simple vector search may struggle. This leads us into **advanced RAG patterns**.

---

### 2. The Advanced RAG Mental Model

Instead of:

```
Query → Search → LLM
```

we can build:

```
                   USER QUERY
                       ↓
                Query Understanding
                       ↓
                 Query Planning
                       ↓
              ┌────────┴────────┐
              ↓                 ↓
         Search A           Search B
              ↓                 ↓
              └────────┬────────┘
                       ↓
                  Filtering
                       ↓
                  Reranking
                       ↓
               Context Assembly
                       ↓
                    LLM
                       ↓
                 Validation
                       ↓
                   Answer
```

The system becomes more intelligent about **how it retrieves information**, rather than simply retrieving the nearest vectors.

---

### 3. Query Understanding

Users do not always ask questions in the form that is best for retrieval.

For example:

> "What's the current thing for Chemical X?"

A human understands the likely intent. A retrieval system may not.

Query understanding attempts to determine:

- What is the user asking?
- What entities are involved?
- What time period matters?
- What location matters?
- What type of information is required?
- Are there multiple questions?

Example:

```
User Query: "What is the current procedure for Chemical X at Plant 2?"

Extracted:
  Entity: Chemical X
  Location: Plant 2
  Document Type: Procedure
  Time: Current
  Intent: Operational procedure
```

Now retrieval has more information to work with.

---

### 4. Query Rewriting

We introduced query rewriting on Day 4. Here we go deeper.

The original:

```
"What about the current procedure?"
```

may depend on conversation history. The system could rewrite it into:

```
"What is the current Chemical X handling procedure at Plant 2?"
```

The important distinction:

> **Query rewriting changes the representation of the question for retrieval, not the user's underlying intent.**

The rewritten query should preserve the original meaning.

---

### 5. Query Expansion

Sometimes one query does not contain enough terminology to retrieve all relevant documents.

User asks:

> "How do I protect myself when handling Chemical X?"

Potential search concepts: PPE, protective equipment, gloves, eye protection, respirator, chemical handling.

```
             User Query
                  ↓
            Query Expansion
                  ↓
       ┌──────────┼──────────┐
       ↓          ↓          ↓
     PPE       Chemical    Handling
       ↓          ↓          ↓
       └──────────┼──────────┘
                  ↓
               Retrieval
```

This can improve recall. But it can also introduce irrelevant terms. **Evaluate it** — do not assume expansion always helps.

---

### 6. Multi-Query Retrieval

Another approach generates multiple search queries from one user question.

Original:

> "What are the safety requirements for Chemical X?"

Generated queries:

```
Query 1: Chemical X PPE requirements
Query 2: Chemical X safety procedures
Query 3: Chemical X handling requirements
Query 4: Chemical X employee safety precautions
```

Each query retrieves candidates. Then:

```
Results 1 + Results 2 + Results 3 + Results 4 → Deduplicate → Rerank → Top Results
```

This can improve recall at the cost of additional retrieval work.

---

### 7. Multi-Step Retrieval

Some questions require information from multiple stages.

Example:

> "Who is authorized to handle Chemical X and what training do they need?"

```
Question → Who is authorized? → Authorization Policy → Identify required role
         → Training requirements → Training Policy
```

The system may need sequential retrieval:

```
Query → Retrieve A → Extract information → Generate next search → Retrieve B → Combine
```

This pattern is particularly useful for complex questions that depend on intermediate answers.

---

### 8. Query Decomposition

Complex questions can be broken into smaller questions.

Example:

> "Compare the old and current Chemical X procedures and tell me what changed."

Decomposition:

```
Question 1: What did the old procedure say?
Question 2: What does the current procedure say?
Question 3: What changed between them?
```

Then:

```
Q1 → Retrieval | Q2 → Retrieval | Q3 → Comparison → Final answer combines results
```

---

### 9. Why Decomposition Helps

Consider:

> "What are the current storage requirements, PPE requirements, emergency procedures, and training requirements for Chemical X?"

A single embedding may represent the entire question poorly.

```
                  Original Question
                         ↓
                    Decompose
                         ↓
        ┌────────┬────────┬────────┐
        ↓        ↓        ↓        ↓
      Storage    PPE   Emergency Training
        ↓        ↓        ↓        ↓
        └────────┴────────┴────────┘
                         ↓
                     Reranking
                         ↓
                     Context
                         ↓
                        LLM
```

Each sub-question can retrieve more targeted information.

---

### 10. Context Engineering

We have already talked about prompt construction. **Context engineering** is the broader discipline of deciding what information the model sees.

> **The quality of information provided to the LLM often matters more than simply choosing a larger model.**

Context can include:

- Retrieved documents
- Conversation history
- User information
- Tool results
- Instructions
- Structured data
- Previous actions
- Metadata

The system needs to decide: **what information should the model see at this moment?**

---

### 11. More Context Is Not Better Context

Suppose we retrieve 50 chunks and send all of them to the LLM.

Potential problems:

- Higher cost and latency
- More irrelevant information
- Conflicting information
- Greater context dilution
- Increased opportunity for malicious content (Day 4)

Instead:

```
50 candidates → Filter → Rerank → Top 5 → Compress → LLM
```

The objective is not maximum context. It is:

> **Maximum useful context.**

---

### 12. Context Ordering

The order of retrieved information can matter.

```
System instructions

Context:
Chunk A
Chunk B
Chunk C
Chunk D

Question
```

Useful information should be:

- Clearly separated
- Labeled with source metadata
- Organized by relevance
- Organized by chronology when comparing versions

Example:

```
SOURCE 1
Title: Chemical X Safety Procedure
Version: 2026
Section: PPE

CONTENT:
...
```

Structure helps the model use context correctly.

---

### 13. Context Compression

Retrieved chunks may contain a lot of unnecessary text.

Instead of:

```
10 chunks × 1,000 tokens = 10,000 tokens
```

extract the portions relevant to the question:

```
Retrieved Documents → Relevant Information Extraction → Compressed Context → LLM
```

This can reduce token usage, latency, and noise. But compression introduces another processing step and can accidentally remove important information. **Measure whether compression improves the system.**

---

### 14. Parent-Child Retrieval

Small chunks allow precise search. But a tiny chunk may not contain enough context for generation.

```
Parent Document
       │
       ├── Child Chunk 1
       ├── Child Chunk 2
       ├── Child Chunk 3
       └── Child Chunk 4
```

Search using child chunks. If Child Chunk 3 matches, retrieve the parent section or surrounding context for generation.

> **Precise retrieval + richer generation context.**

---

### 15. Hierarchical Retrieval

Large documents can be represented hierarchically:

```
Document
│
├── Chapter 1
│   ├── Section 1.1
│   └── Section 1.2
├── Chapter 2
│   ├── Section 2.1
│   └── Section 2.2
└── Chapter 3
```

The system can first determine which chapter, then which section, then which specific chunks. This helps with large structured documents.

---

### 16. Metadata-Aware Retrieval

Metadata improves retrieval beyond authorization (Day 4).

Query: *"What is the current procedure?"*

Filter first:

```
document_type = procedure
is_current = true
location = Plant 2
```

Then semantic search within the relevant subset:

```
All Documents → Metadata Filter → Relevant Candidate Set → Semantic / Keyword Search → Reranking
```

This can improve both quality and efficiency.

---

### 17. Structured Data and RAG

Not all enterprise information belongs in a vector database.

> "How many Chemical X incidents occurred at Plant 2 last year?"

This is potentially a database question. A vector search system is not necessarily the best tool.

```
Documents → RAG
Structured Data → SQL / Database
```

The system may need to determine which source is appropriate.

---

### 18. Query Routing

The system determines where a query should go:

```
                     Query
                       ↓
                    Router
                 /     |      \
                /      |       \
               ↓       ↓        ↓
           Vector     SQL      API
             RAG
```

| Question | Route |
|----------|-------|
| "What is the Chemical X procedure?" | RAG |
| "How many incidents occurred?" | Database |
| "What is the current weather at Plant 2?" | External API |

The router chooses the appropriate tool or retrieval mechanism.

---

### 19. Agentic RAG

**Traditional RAG:**

```
Query → Retrieve → Generate
```

**Agentic RAG:**

```
                     User
                      ↓
                    Agent
                      ↓
              Decide what to do
                 /    |    \
                /     |     \
               ↓      ↓      ↓
            Search   SQL    API
               │      │      │
               └──────┼──────┘
                      ↓
                    Agent
                      ↓
                  Evaluate
                      ↓
               Search again?
                      ↓
                    Answer
```

The system dynamically decides which actions to take.

---

### 20. Why Agentic RAG Is Powerful

Consider:

> "Find the latest Chemical X procedure, compare it with last year's procedure, check the incident database for the last six months, and summarize whether the new procedure appears to have reduced incidents."

This potentially requires:

```
1. Search current procedure
2. Search previous procedure
3. Retrieve incident data
4. Analyze incidents
5. Compare procedures
6. Produce conclusion
```

A static RAG pipeline becomes awkward. An agent can orchestrate these steps.

---

### 21. Why Agentic RAG Is Dangerous

More autonomy means more potential failure modes. An agent might:

- Call the wrong tool
- Retrieve unnecessary information
- Loop indefinitely
- Spend excessive money
- Access unauthorized information
- Misinterpret tool output
- Take an unsafe action

Therefore:

> **Agentic systems require strong boundaries.**

These include tool permissions, maximum iterations, timeouts, token budgets, approval requirements, authorization, input/output validation, and monitoring.

---

### 22. RAG vs Agentic RAG

| Aspect | Traditional RAG | Agentic RAG |
|--------|-----------------|-------------|
| Workflow control | Application controls the workflow | Model participates in deciding the workflow |
| Flexibility | Fixed pipeline | Dynamic tool and search selection |
| Complexity | Lower | Higher |
| Risk | Lower | Higher — more failure modes |

Traditional RAG:

```
Application → Retrieve → LLM → Answer
```

Agentic RAG:

```
Application → Agent → Decide (Search / SQL / API / Search again) → Final answer
```

The second is more flexible but also more complex.

---

### 23. Tool Use

An agent can use tools:

```
Tool 1: Search documents
Tool 2: Query database
Tool 3: Get current procedure
Tool 4: Calculate statistics
Tool 5: Retrieve external information
```

The model decides which tool to call based on the task. The application should still enforce authorization. **The model should never be treated as the security boundary.**

---

### 24. The Security Boundary

A principle from everything we have studied:

> **The LLM can make decisions, but the application enforces permissions.**

```
Agent → "I want to access confidential document X."
     → Application Authorization Layer → ALLOW / DENY
```

Not:

```
Agent → "I think I'm allowed."
```

The same principle applies to tools.

---

### 25. Advanced RAG Architecture

```
                         USER
                           │
                           ↓
                    Authentication
                           │
                           ↓
                      Query API
                           │
                           ↓
                  Query Understanding
                           │
                           ↓
                       Router
                           │
             ┌─────────────┼──────────────┐
             ↓             ↓              ↓
            RAG           SQL            API
             │             │              │
             ↓             ↓              ↓
        Retrieval       Database      External Data
             │
       ┌─────┴──────┐
       ↓            ↓
    Semantic     Keyword
     Search       Search
       │            │
       └─────┬──────┘
             ↓
          Reranking
             ↓
       Context Assembly
             ↓
       Context Compression
             ↓
            LLM
             ↓
         Validation
             ↓
           Answer
```

For agentic applications:

```
                    USER
                      ↓
                    AGENT
                      ↓
                 TOOL ROUTER
                /     |      \
               ↓      ↓       ↓
             RAG     SQL      APIs
               \      |       /
                \     |      /
                 ↓    ↓     ↓
                    AGENT
                      ↓
                 Final Answer
```

---

### 26. When Should You Use Advanced RAG?

Do not automatically add complexity. Start with simple RAG. If evaluation shows problems, identify the cause.

| Problem | Potential solution |
|---------|-------------------|
| Low recall | Hybrid search, query expansion, multi-query retrieval |
| Complex questions not answered | Query decomposition, multi-step retrieval |
| Chunks contain too little context | Parent-child retrieval, context expansion |
| Queries require multiple data sources | Query routing, tool use, agentic RAG |

> **Add complexity to solve a demonstrated problem.**

---

## Mental Model

```
              BASIC RAG
                  │
                  ↓
       Query → Retrieve → Generate
                  │
                  ↓
            ENTERPRISE RAG
                  │
                  ↓
     Authorization + Metadata
                  │
                  ↓
         Evaluation + Monitoring
                  │
                  ↓
             ADVANCED RAG
                  │
        ┌─────────┼─────────┐
        ↓         ↓         ↓
     Query     Retrieval   Context
  Understanding  Strategies Engineering
        │         │         │
        └─────────┼─────────┘
                  ↓
            QUERY ROUTING
                  │
                  ↓
            TOOL USE / SQL
                  │
                  ↓
             AGENTIC RAG
```

> **RAG is not one architecture. It is a collection of patterns that can be combined according to the problem you are trying to solve.**

> **Do not introduce advanced components because they are interesting. Introduce them because evaluation shows the simpler system is not solving the problem well enough.**

---

## Tradeoffs

| Decision | Simpler | More advanced | Guidance |
|----------|---------|---------------|----------|
| Query handling | Direct embedding | Rewrite / expand / decompose | Add when recall or complex questions fail evaluation |
| Retrieval | Single query | Multi-query / multi-step | Multi-query improves recall; multi-step for dependent questions |
| Context | Send all chunks | Compress / parent-child | Compress when cost or noise is high; parent-child when chunks lack context |
| Data source | Vector only | Route to SQL / API | Route when question type does not match document search |
| Architecture | Static pipeline | Agentic RAG | Agentic only when workflow is genuinely dynamic and boundaries are enforced |

---

## Common Failure Modes

| Failure | Symptom | Fix |
|---------|---------|-----|
| **Over-expansion** | Irrelevant chunks retrieved | Tighten expansion; evaluate recall vs precision |
| **Bad rewrite** | Changed user intent | Validate rewrites; compare to original |
| **Over-decomposition** | Too many sub-queries, high cost | Decompose only for genuinely complex questions |
| **Compression loss** | Missing facts in answer | Measure groundedness after compression |
| **Wrong route** | SQL question sent to RAG | Improve router training or rules |
| **Agent loop** | Infinite search retries | Max iterations, timeouts, circuit breakers |
| **Unauthorized tool call** | Data leakage | Enforce permissions at application layer |
| **Premature complexity** | Hard to debug, no quality gain | Start simple; add patterns when evaluation shows need |

---

## Hands-On Exercise

### Exercise 1 — Design Advanced RAG for a Complex Question (45 minutes)

Design an advanced RAG system for this question:

> **"Compare the current Chemical X procedure with the 2024 procedure, explain what changed, and tell me whether the number of Chemical X incidents at Plant 2 decreased after the new procedure was introduced."**

Assume the enterprise has: Confluence, PDF procedures, historical procedures, an incident database, and an employee authorization system.

Answer:

1. **Query understanding** — What does the system need to identify from the question?
2. **Query decomposition** — What smaller questions might you create?
3. **Routing** — Which parts require RAG? Which part requires the database?
4. **Retrieval** — How would you find the current procedure? The 2024 procedure?
5. **Authorization** — How do you ensure the user can access both procedures and incident data?
6. **Context** — What information should ultimately reach the LLM?
7. **Generation** — How should the LLM combine the information?
8. **Validation** — How would you verify the answer is supported by retrieved information?
9. **Observability** — What would you log so you could reproduce the answer later?
10. **Agentic approach** — Would you use a traditional workflow or an agent? Why?

Focus on reasoning about the architecture, not on being perfect.

### Exercise 2 — Pattern Selection (15 minutes)

For each problem below, name one advanced RAG pattern that could help and explain why:

1. Users ask vague follow-up questions like "What about the current one?"
2. Retrieval misses relevant documents because users use different words than the documents
3. A single question asks about PPE, storage, and emergency procedures at once
4. Retrieved chunks are precise but too small for the LLM to generate a complete answer
5. Half the questions need document search and half need incident counts from a database

### Exercise 3 — Context Budget (15 minutes)

Retrieval returns 40 chunks averaging 400 tokens each. Your token budget for context is 6,000 tokens. Describe the pipeline steps you would use to produce maximum useful context within the budget.

---

## Knowledge Check

Answer these without looking at the previous sections:

1. What is query understanding?
2. What is query rewriting?
3. What is query expansion?
4. What is multi-query retrieval?
5. What is query decomposition?
6. Why can complex questions benefit from decomposition?
7. What is context engineering?
8. Why is more context not always better?
9. What is parent-child retrieval?
10. What is hierarchical retrieval?
11. What is query routing?
12. Why should not every question go through vector search?
13. What is agentic RAG?
14. How does agentic RAG differ from traditional RAG?
15. Why does agentic RAG introduce additional risks?
16. What is the role of tools in an agentic system?
17. Who should enforce authorization — the LLM or the application?
18. When should you introduce advanced RAG techniques?

---

## Self Assessment

1. Draw the advanced RAG mental model from memory.
2. Explain the difference between query rewriting, expansion, and decomposition with an example for each.
3. Describe parent-child retrieval and when you would use it.
4. Give an example question that should route to SQL instead of RAG.
5. What boundaries would you put on an agentic RAG system?
6. How does Day 7 connect to Days 3–6?
7. When would you choose multi-query retrieval over query expansion?
8. Explain "maximum useful context" vs "maximum context."
9. Design a decision tree: when does a question stay in simple RAG vs need decomposition vs need an agent?
10. Why should advanced patterns be introduced only after evaluation shows a problem?

---

## Key Takeaways

1. Simple **query → vector search → LLM** fails on complex, multi-part, or multi-source questions.
2. **Query understanding** extracts entities, intent, time, and location to improve retrieval.
3. **Query rewriting** preserves intent while improving retrieval representation; **expansion** and **multi-query** improve recall at a cost.
4. **Decomposition** and **multi-step retrieval** handle complex questions that need multiple searches.
5. **Context engineering** decides what the model sees — quality matters more than quantity.
6. **Parent-child** and **hierarchical retrieval** balance precise search with rich generation context.
7. **Metadata-aware retrieval** narrows candidates before semantic search.
8. Not every question belongs in vector search — **query routing** sends questions to RAG, SQL, or APIs.
9. **Agentic RAG** orchestrates dynamic workflows but requires strong boundaries and authorization at the application layer.
10. **Add complexity to solve demonstrated problems** — evaluate every advanced pattern before adopting it.

---

## Summary

Day 7 connects the Week 1 RAG foundation into **advanced patterns**. Move from simple retrieval to **query understanding, decomposition, routing, and context engineering**. Use **parent-child** and **hierarchical retrieval** for document structure. Route structured questions to **SQL or APIs**. Understand **agentic RAG** as a more flexible but riskier architecture. The overarching lesson: RAG is a **collection of patterns** — choose them based on evaluation evidence, not novelty.

---

## What You Should Be Able to Explain

Without looking at notes, you should be able to describe:

```
How advanced RAG extends basic RAG
→ query understanding, rewriting, expansion, decomposition, and multi-step retrieval
→ context engineering: ordering, compression, and maximum useful context
→ parent-child and hierarchical retrieval
→ metadata-aware retrieval and query routing
→ when to use agentic RAG and what boundaries it requires
→ how to choose patterns based on evaluation, not complexity for its own sake
```

You do not need to memorize specific frameworks or agent libraries. The goal is to understand which pattern solves which problem and how Week 1 concepts fit together.

---

## Next

**Week 2 — Day 8: Prompt Engineering** — build reliable applications around LLM APIs.

Complete the exercises and self assessment above before moving on.
