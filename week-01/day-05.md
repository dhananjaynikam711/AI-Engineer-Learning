# Day 5 — RAG Evaluation

---

## Spaced Review (Days 1–4)

Answer mentally before continuing:

1. What are the four major concerns of enterprise RAG?
2. Why should authorization happen before information reaches the LLM?
3. What is hybrid search and when is it useful?
4. What is the difference between the ingestion pipeline and the query pipeline?
5. Why should retrieved documents be treated as untrusted data?

---

## Learning Objectives

By the end of this lesson, you should be able to explain:

1. Why RAG evaluation is different from traditional application testing
2. Why a bad final answer does not necessarily mean the LLM is at fault
3. The three major evaluation areas: retrieval, generation, and end-to-end
4. What a golden evaluation dataset is and why it matters
5. Precision, recall, and Top-K retrieval metrics at a conceptual level
6. How to evaluate reranking separately from initial retrieval
7. What context relevance, groundedness, and answer relevance mean
8. The difference between grounded, relevant, and correct answers
9. What LLM-as-a-judge is and its limitations
10. Why human evaluation and regression testing matter
11. What should be benchmarked when RAG components change
12. How to design an evaluation dataset with realistic categories
13. Why "no answer" questions belong in the benchmark
14. Operational metrics beyond accuracy (latency, cost, failure rate)
15. How the RAG evaluation loop connects build, measure, analyze, and improve

---

## Why This Matters

Day 4 was about designing the RAG system. Day 5 answers the next engineering question:

> **How do we know whether our RAG system is actually working well?**

This is especially important because a RAG system can *look* good while quietly:

- Retrieving the wrong documents
- Using stale information
- Generating answers that are not supported by the retrieved context

The primary focus is **engineering understanding** — you do not need to memorize every evaluation framework or metric formula today. The priority is understanding **what you are measuring, why you are measuring it, and how the measurements help you improve the system**.

**Day 4 taught you how to build the RAG pipeline. Day 5 teaches you how to measure whether the pipeline actually works.**

---

## Prerequisites

Before starting Day 5, you should understand:

- The basic RAG pipeline (Day 3)
- Enterprise RAG architecture: ingestion, retrieval, reranking, authorization, freshness (Day 4)
- Grounding and "information not found" responses (Day 4)

If you cannot describe the offline ingestion pipeline and the online query pipeline from Day 4, review before continuing.

---

## Core Concepts

### 1. The Evaluation Mental Model

Evaluate **each stage of the RAG pipeline**, not just whether the final answer "looks good":

```
                    RAG EVALUATION

                       User Query
                           │
                           ↓
                    Query Processing
                           │
                           ↓
                       Retrieval
                           │
                           ↓
                       Reranking
                           │
                           ↓
                     Context Quality
                           │
                           ↓
                         LLM
                           │
                           ↓
                    Generated Answer
                           │
                           ↓
                     Final Evaluation
```

Ask different questions at every stage.

---

### 2. Why RAG Evaluation Is Different

For a traditional application:

```
Input → Function → Expected Output
```

Testing can be relatively straightforward:

```
Input: 2 + 2
Expected: 4
```

LLM applications are different. There can be many valid ways to answer the same question.

For example, "What PPE is required when handling Chemical X?" can be worded differently while conveying the same information.

We cannot always rely on:

```
actual_output == expected_output
```

Instead, we evaluate several dimensions.

---

### 3. The Pipeline We Are Evaluating

Recall the Day 4 architecture:

```
Documents → Ingestion → Normalization → Chunking → Metadata
→ Embeddings / Indexing → Retrieval → Filtering → Reranking
→ Context → LLM → Answer
```

A major lesson:

> **A bad final answer does not necessarily mean the LLM is bad.**

The failure could have happened much earlier:

```
User asks correct question
        ↓
Wrong chunks retrieved
        ↓
LLM receives wrong information
        ↓
LLM generates plausible answer
```

The LLM may have done exactly what we asked it to do. Evaluation needs to locate **where the failure occurred**.

---

### 4. Three Big Evaluation Areas

| Area | Question |
|------|----------|
| **Retrieval** | Did we retrieve the right information? |
| **Generation** | Did the LLM produce a good answer from the retrieved information? |
| **End-to-end** | Did the complete system answer the user's question correctly and safely? |

```
              RAG Evaluation
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
    Retrieval   Generation   End-to-End
```

---

### 5. Retrieval Evaluation

Suppose the user asks: *"What PPE is required when handling Chemical X?"*

The knowledge base contains:

```
Chunk A — Chemical X must be stored below 25°C.
Chunk B — Workers handling Chemical X must wear chemical-resistant gloves and eye protection.
Chunk C — Chemical X should be transported using approved containers.
Chunk D — Chemical X was introduced in 2018.
```

The ideal retrieval result should contain **Chunk B**.

If the system instead retrieves Chunks A, C, and D, the generation model does not have the information it needs.

Therefore:

> **Before evaluating the answer, evaluate retrieval.**

---

### 6. Golden Evaluation Dataset

One of the most important concepts is the **golden evaluation dataset** — a set of representative questions for the system.

| Question | Expected Relevant Source |
|----------|--------------------------|
| What PPE is required for Chemical X? | Safety Procedure §2 |
| How should Chemical X be stored? | Safety Procedure §3 |
| What is the emergency response? | Emergency Procedure §4 |
| Who is authorized to handle Chemical X? | Employee Safety Policy |

This becomes the benchmark for measuring changes over time.

---

### 7. Why the Golden Dataset Matters

Suppose you change your chunking strategy:

```
Chunking Strategy A → RAG → 82% evaluation score
Chunking Strategy B → RAG → 88%
```

Now you have evidence that the change improved the system.

Without a benchmark, you might simply think "it seems better." That is not enough for a production system.

---

### 8. Precision

> Of the results we retrieved, how many were relevant?

Imagine retrieving 10 chunks. Only 6 are relevant:

```
Precision = 6 / 10 = 60%
```

Precision measures: **how much of what we retrieved was useful?**

---

### 9. Recall

> Of all the relevant information that existed, how much did we successfully retrieve?

Suppose there are 5 relevant chunks in the knowledge base. The system retrieves 4:

```
Recall = 4 / 5 = 80%
```

| Metric | Question |
|--------|----------|
| **Precision** | How much of what I retrieved was relevant? |
| **Recall** | How much of the relevant information did I retrieve? |

---

### 10. Precision vs Recall

**High precision, low recall:**

```
Retrieved: ✓ Correct PPE procedure
Missed:    ✗ Emergency procedure, ✗ Storage procedure
```

Results are highly relevant but incomplete.

**Low precision, high recall:**

```
Retrieved: ✓ PPE, ✓ Emergency, ✓ Storage
           ✗ Employee handbook, ✗ HR document, ✗ Unrelated procedure
```

Relevant information was found, but with lots of unnecessary content.

The right balance depends on the application.

---

### 11. Top-K Retrieval

You will frequently see **Recall@K** and **Precision@K**. The `K` represents how many results are examined:

```
Top 5 | Top 10 | Top 20
```

If we ask "Did the correct document appear in the top 5 results?" we are evaluating **Recall@5**.

This is useful because the RAG system may only pass the top few results to the reranker or LLM.

---

### 12. Reranking Evaluation

From Day 4:

```
Query → Initial Retrieval → 50 candidates → Reranker → Top 5 → LLM
```

Evaluate the reranker separately.

Initial retrieval:

```
1. Chunk A
2. Chunk B
3. Chunk C
4. Chunk D  ← relevant
5. Chunk E
```

The reranker should ideally move the relevant chunk toward the top:

```
1. Chunk D  ← relevant
2. Chunk B
3. Chunk A
4. Chunk E
5. Chunk C
```

Evaluate whether reranking improves the quality of the final candidate set.

---

### 13. Context Relevance

Suppose retrieval returns Chunks B, C, F, and H. The question is:

> **Does the context actually contain information useful for answering the question?**

This is different from measuring vector similarity alone. A chunk can be semantically similar but not useful.

Question: *"What PPE is required?"*

Retrieved: *"Chemical X must be stored below 25°C."*

That is related to Chemical X but does not answer the question.

Context evaluation asks: **does the retrieved context contain the information needed to answer the query?**

---

### 14. Generation Evaluation

Assume retrieval worked correctly. Retrieved context:

```
Chemical X handling requires:
- Chemical-resistant gloves
- Eye protection
```

Good LLM response:

> "Workers must wear chemical-resistant gloves and eye protection."

Problematic response:

> "Workers must wear chemical-resistant gloves, eye protection, and a respirator."

If the source never mentioned a respirator, we have a grounding problem.

---

### 15. Groundedness / Faithfulness

Ask: **can every important claim in the answer be supported by the provided context?**

Context: *"Chemical X requires gloves and eye protection."*

| Answer | Grounded? |
|--------|-----------|
| "Chemical X requires gloves and eye protection." | Yes |
| "Chemical X requires gloves, eye protection, and a respirator." | No — respirator not in context |

The model introduced information that was not in the context.

---

### 16. Answer Relevance

> **Does the answer actually answer the user's question?**

Question: *"What PPE is required?"*

Answer: *"Chemical X was introduced in 2018."*

Even if true, it is not relevant.

```
Correct information ≠ Relevant answer
```

We need both.

---

### 17. Three Important Generation Questions

When evaluating an answer, ask:

| Question | Meaning |
|----------|---------|
| **Is it relevant?** | Does it answer the user's question? |
| **Is it grounded?** | Is it supported by the retrieved context? |
| **Is it correct?** | Does it agree with the authoritative source? |

```
                 Answer
                    │
        ┌───────────┼───────────┐
        ↓           ↓           ↓
    Relevance   Groundedness  Correctness
```

These are not necessarily identical.

---

### 18. End-to-End Evaluation

Ultimately, users care about: *"Did I get the right answer?"*

Pass example:

```
Question: What PPE is required?
Retrieved: Correct safety procedure
Generated: Correct answer
Result: PASS
```

Fail example:

```
Question → Wrong document retrieved → LLM produces plausible answer → FAIL
```

Evaluating only the LLM is not enough.

---

### 19. LLM-as-a-Judge

Manually evaluating thousands of answers is difficult. One approach uses another LLM as an evaluator:

```
Question + Retrieved Context + Generated Answer + Evaluation Criteria
        ↓
     Judge LLM
        ↓
     Score
```

The judge might evaluate relevance, groundedness, completeness, correctness, and citation quality:

```
Groundedness: 4/5
Relevance: 5/5
Completeness: 3/5
```

---

### 20. LLM-as-a-Judge Is Not Perfect

An LLM judge is itself a model. It can make mistakes:

- Bias and inconsistent scoring
- Difficulty evaluating nuanced answers
- Preference for certain wording
- Incorrectly accepting hallucinations

LLM-as-a-judge should be treated as an evaluation tool, not absolute truth.

For important systems, combine:

```
Automated metrics + LLM evaluation + Human evaluation
```

---

### 21. Human Evaluation

Human reviewers evaluate a sample of answers:

```
Answer 1 → PASS
Answer 2 → FAIL
Answer 3 → PASS
```

Human evaluation is expensive but valuable for:

- Creating benchmark datasets
- Validating automated evaluators
- Testing edge cases
- High-risk domains

---

### 22. Regression Testing

Suppose the current system performs at 88% accuracy. You change the embedding model and reach 91%. Great — but perhaps safety questions dropped from 95% to 93%.

The overall score improved while an important category got worse.

Evaluation should be broken down into meaningful categories, not just a single headline number.

---

### 23. RAG Regression Pipeline

A production development workflow:

```
Change → Build → Run Evaluation Dataset → Compare Against Baseline
→ Check Metrics → Check Regressions → Deploy
```

```
                  Golden Dataset
                       ↓
                ┌──────────────┐
                │ Current RAG  │
                └──────┬───────┘
                       ↓
                    Metrics
                       ↓
              Compare to Baseline
                       ↓
                Pass / Fail
```

This turns RAG evaluation into an engineering process rather than a one-time experiment.

---

### 24. What Should We Benchmark?

Changes can occur in any pipeline component:

| Component | Examples of changes |
|-----------|---------------------|
| **Ingestion** | Parser, OCR |
| **Chunking** | Chunk size, overlap, structure-aware strategy |
| **Embeddings** | Embedding model, configuration |
| **Retrieval** | Vector search, keyword search, hybrid search |
| **Reranking** | Reranker model, number of candidates, Top-K |
| **Prompt** | System instructions, context formatting, output requirements |
| **LLM** | Model, temperature, context size |

Every one of these changes can potentially affect performance.

---

### 25. Evaluation Dataset Design

A good evaluation dataset should represent real usage. Do not create 100 questions that all look like "What is Chemical X?"

Include different categories:

```
Simple factual questions
Multi-step questions
Ambiguous questions
Specific identifier questions
Questions requiring multiple documents
Questions with no answer
Outdated-information questions
Permission-sensitive questions
Adversarial questions
```

---

### 26. "No Answer" Questions

Suppose the knowledge base contains no information about the Brazil retirement policy. The correct behavior may be:

> "I don't have enough information in the available sources to answer that."

The system should not receive full credit for confidently inventing an answer.

The evaluation dataset should contain questions where the correct answer is **NOT FOUND**. This tests whether the system knows when it does not know.

---

### 27. Evaluation Categories

| Category | Example |
|----------|---------|
| Factual | What PPE is required? |
| Multi-document | Compare two procedures |
| Exact lookup | What is procedure ID MX-4728? |
| Ambiguous | What about the previous procedure? |
| No-answer | What is the Brazil retirement policy? |
| Permission | What confidential information is available? |
| Freshness | What is the current procedure? |
| Adversarial | Malicious retrieved instructions |

---

### 28. Evaluation Matrix

| Component | Question |
|-----------|----------|
| Retrieval | Did we find the right information? |
| Precision | Were retrieved results relevant? |
| Recall | Did we retrieve enough relevant information? |
| Reranking | Did relevant results move to the top? |
| Context | Does the context answer the question? |
| Groundedness | Is the answer supported by context? |
| Relevance | Does the answer answer the question? |
| Correctness | Does it agree with authoritative information? |
| Safety | Did it avoid unsafe or unauthorized behavior? |
| End-to-end | Did the system solve the user's task? |

---

### 29. Evaluation Beyond Accuracy

Production systems also need operational metrics:

| Metric | Question |
|--------|----------|
| **Latency** | How long does a response take? |
| **Cost** | How many tokens / model calls are required? |
| **Throughput** | How many requests can the system handle? |
| **Failure rate** | How often do requests fail? |
| **Retrieval metrics** | How often is the correct information retrieved? |
| **Generation metrics** | How often are answers grounded and correct? |

A system can be highly accurate but too slow or expensive to operate.

---

## Mental Model

If you remember only one thing today, remember this:

```
              USER QUESTION
                    │
                    ↓
             ┌─────────────┐
             │  RETRIEVAL  │
             └──────┬──────┘
                    │
          Did we find the
          right information?
                    │
                    ↓
             ┌─────────────┐
             │   CONTEXT   │
             └──────┬──────┘
                    │
          Is the context
          sufficient?
                    │
                    ↓
             ┌─────────────┐
             │     LLM     │
             └──────┬──────┘
                    │
                    ↓
              GENERATED ANSWER
                    │
          ┌─────────┼─────────┐
          ↓         ↓         ↓
      Relevant?  Grounded?  Correct?
          │         │         │
          └─────────┼─────────┘
                    ↓
              FINAL EVALUATION
```

---

## The RAG Evaluation Loop

```
                  Build RAG
                     ↓
                Create Dataset
                     ↓
                  Evaluate
                     ↓
                Analyze Failures
                     ↓
           Identify Pipeline Problem
                     ↓
        ┌────────────┼────────────┐
        ↓            ↓            ↓
     Chunking     Retrieval     Prompt
        │            │            │
        └────────────┼────────────┘
                     ↓
                  Modify
                     ↓
                  Evaluate
                     ↓
              Compare Baseline
                     ↓
              Deploy if Improved
                     ↓
              Monitor Production
                     ↓
              Add New Failures
                     ↓
             Update Evaluation Set
```

> **Evaluation is a continuous feedback loop.**

## Knowledge Check

Answer these without looking at the previous sections:

1. Why is RAG evaluation different from traditional input/output testing?
2. Why does a bad final answer not necessarily mean the LLM failed?
3. What are the three major evaluation areas?
4. What is a golden evaluation dataset?
5. What is the difference between precision and recall?
6. What does Recall@5 mean?
7. Why should reranking be evaluated separately?
8. What is context relevance?
9. What is groundedness?
10. What is the difference between a grounded answer and a relevant answer?
11. What is LLM-as-a-judge and what are its limitations?
12. Why is regression testing important?
13. Why should evaluation datasets include "no answer" questions?
14. Name three operational metrics beyond accuracy.
15. What is the RAG evaluation loop?

---

## Self Assessment

1. Draw the stage-by-stage evaluation mental model from memory.
2. Explain precision and recall with a concrete example.
3. Describe how you would evaluate a chunking strategy change.
4. Give an example where an answer is relevant but not grounded.
5. Give an example where an answer is grounded but not correct.
6. When would you use LLM-as-a-judge vs human evaluation?
7. What pipeline stage would you investigate first if answers cite old procedures?
8. List five categories that belong in a realistic evaluation dataset.
9. How does Day 5 build on Day 4?
10. Explain why evaluation is a continuous loop, not a one-time activity.

---

## Key Takeaways

1. Evaluate each pipeline stage, not just the final answer.
2. A bad answer often starts with bad retrieval — locate where the failure occurred.
3. A **golden evaluation dataset** provides a repeatable benchmark for changes.
4. **Precision** measures retrieved relevance; **recall** measures completeness.
5. Evaluate **reranking**, **context relevance**, and **generation** separately.
6. Good answers must be **relevant**, **grounded**, and **correct** — these differ.
7. **LLM-as-a-judge** scales evaluation but is not absolute truth.
8. Combine automated metrics, LLM evaluation, and human review for important systems.
9. **Regression testing** catches category-level regressions hidden by overall scores.
10. Benchmark every component that can change: chunking, embeddings, retrieval, prompt, LLM.
11. Include **no-answer**, **freshness**, and **adversarial** questions in the dataset.
12. Operational metrics (latency, cost, failure rate) matter alongside accuracy.
13. **Evaluation is a continuous feedback loop** — build, measure, analyze, improve, monitor.

---

## Summary

Day 5 centers on one engineering question:

> **How do we know whether our RAG system is actually working well?**

Divide evaluation into **retrieval**, **generation**, and **end-to-end**. Use a **golden dataset** to benchmark changes. Measure **precision**, **recall**, **context quality**, **groundedness**, and **relevance** at the appropriate stage. Use **LLM-as-a-judge** and **human evaluation** as complementary tools. Run **regression tests** before deploying changes. Treat evaluation as a **continuous loop** that feeds back into chunking, retrieval, prompts, and monitoring.

---

## What You Should Be Able to Explain

Without looking at notes, you should be able to describe:

```
How to evaluate each stage of a RAG pipeline
→ why retrieval must be evaluated before generation
→ how a golden dataset supports regression testing
→ the difference between precision, recall, groundedness, and relevance
→ when LLM-as-a-judge is useful and when human evaluation is required
→ how to design a realistic evaluation dataset
→ how the evaluation loop connects to production monitoring
```

You do not need to memorize specific evaluation frameworks or vendor tools. The goal is to understand what to measure, why it matters, and how to use the results to improve the system.

---

## Next

**Day 6 — Week 1 Review** — consolidate Days 1–5 and confirm understanding of the LLM and RAG foundations before moving to Week 2.

Complete the self assessment above before moving on.
