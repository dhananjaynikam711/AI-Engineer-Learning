# Day 2 — LLM Fundamentals

---

## Learning Objectives

By the end of this lesson, you should understand:

1. What a **transformer** is and why it enabled modern LLMs
2. How **attention** works conceptually (Q/K/V) and why it matters architecturally
3. What **tokens** are and how they affect cost, latency, and context limits
4. What **embeddings** are and how they relate to RAG (Day 3)
5. How **context windows** constrain system design
6. The difference between **training** and **inference** — and what you operate in production
7. How **temperature**, **top-p**, and **message roles** affect behavior in production APIs

---

## Why This Matters

These fundamentals explain **why production LLM systems behave the way they do**:

- Why context windows are finite and what that means for RAG design
- Why embeddings enable semantic search
- Why inference cost scales with tokens
- Why longer prompts increase latency

You do not need to derive attention mathematically. You need to understand **how the architecture shapes engineering decisions** so you can design systems that work within real constraints.

As an AI Engineer, you integrate LLMs — you do not train them. Understanding fundamentals prevents bad architecture (stuffing 500 pages into one prompt, ignoring token costs, choosing wrong embedding models).

---

## Prerequisites

- Day 1: LLM terminology, inference vs training distinction
- Basic familiarity with vectors (direction + magnitude in high-dimensional space)

---

## Core Concepts

### 1. The Transformer — Intuition First

**Before transformers:** Models processed text sequentially (RNNs). Hard to parallelize; struggled with long-range dependencies ("the cat that sat on the mat **it** was fluffy" — which noun does "it" refer to?).

**Transformer breakthrough (2017, "Attention Is All You Need"):** Process all tokens in parallel; use **attention** to let each token "look at" other tokens and decide what is relevant.

**Why this enabled LLMs:**

- Parallel training on massive GPU clusters → scale to billions of parameters
- Attention captures long-range relationships
- Same architecture works for many tasks via pre-training + fine-tuning / prompting

**Engineering implication:** You consume transformers as **pre-trained models** via API (OpenAI, Anthropic, Bedrock). Your job is integration, not architecture design at the layer level.

---

### 2. Attention — Conceptual Mechanics

**Intuition:** When generating or understanding a word, the model asks: *"Which other words in the input should I pay attention to?"*

Example: In "The maintenance procedure for **Pump A** requires shutting **it** down first" — attention helps link "it" to "Pump A."

**Q / K / V (Query, Key, Value):**

| Component | Analogy | Role |
|-----------|---------|------|
| **Query (Q)** | "What am I looking for?" | Current token's question |
| **Key (K)** | "What do I contain?" | Each token's label/index |
| **Value (V)** | "What information do I provide?" | Each token's content |

For each token, the model computes attention scores against all other tokens (how much to weight each), then combines Values into a contextual representation.

**Equation (for reference — not for derivation):**

```
Attention(Q, K, V) = softmax(QK^T / √d_k) · V
```

What matters engineering-wise:

- `QK^T` — similarity scores between queries and keys
- `softmax` — normalize to weights summing to 1
- Multiply by `V` — weighted sum of information

**Multi-head attention:** Run attention multiple times in parallel (different "perspectives") — syntax, semantics, coreference — then combine.

**Architecture diagram (simplified):**

```
Input tokens → Embedding → ┌─────────────────┐
 │ Multi-Head │
 │ Self-Attention │ × N layers
 └────────┬────────┘
 ↓
 Feed-Forward Network
 ↓
 Next token prediction
```

**Production relevance:**

- Attention is **O(n²)** in sequence length → long context is expensive (compute + memory)
- This is why context windows have limits and why "just send everything" fails

---

### 3. Tokens

**Definition:** The model does not see words. It sees **tokens** — subword units from a tokenizer (BPE, SentencePiece).

Examples:

| Text | Approximate tokens |
|------|-------------------|
| "Hello" | 1 |
| "ChatGPT" | 2 ("Chat" + "GPT") |
| 1000 words of English | ~750–1300 tokens |

**Why tokens matter:**

| Concern | Impact |
|---------|--------|
| **Cost** | Billed per input + output token |
| **Context window** | Max tokens per request (e.g., 128K, 200K) |
| **Latency** | More tokens → more computation |
| **Truncation** | Content beyond window is dropped or must be summarized |

**Engineering practices:**

- Count tokens before sending (tiktoken, provider APIs)
- Chunk documents for RAG instead of stuffing full docs
- Summarize long conversation history
- Cache stable prefix tokens where providers support it (prompt caching)

---

### 4. Embeddings

**Definition:** A fixed-size vector (list of numbers, e.g., 1024 or 1536 dimensions) that represents the **semantic meaning** of text.

**Intuition:** Similar meanings → vectors close together in space.

```
"vehicle maintenance schedule" ≈ "car service interval"
"vehicle maintenance schedule" ≠ "chocolate cake recipe"
```

**How they are produced:** Embedding models (Titan, text-embedding-3, Cohere embed) run text through a neural network; output is the vector.

**Relationship to transformers:** Many embedding models use transformer encoders; LLMs and embedding models are related but serve different purposes:

| Model type | Input | Output | Use |
|------------|-------|--------|-----|
| LLM (decoder) | Text | Next tokens / text | Generation |
| Embedding model | Text | Vector | Search, clustering, RAG retrieval |

**Production relevance:** Day 3 (RAG) depends entirely on embeddings for retrieval. Same embedding model must be used for indexing and querying.

---

### 5. Context Window

**Definition:** Maximum number of tokens the model can process in a single request (input + output combined, depending on provider).

Typical ranges (2025–2026): 8K to 200K+ tokens depending on model.

**What fits in context:**

- System prompt
- Retrieved RAG chunks
- Conversation history
- User question
- Room for model response

**Engineering strategies when content exceeds window:**

| Strategy | When to use |
|----------|-------------|
| RAG (retrieve only relevant chunks) | Large document corpora |
| Summarization | Long conversation history |
| Hierarchical (summarize sections, then query summaries) | Very large single documents |
| Map-reduce over chunks | Analysis of full corpus |
| External memory (DB) | Long-term user history |

**Failure mode:** Silently truncating the beginning or end of input → wrong answers, lost instructions.

---

### 6. Training vs Inference

| Phase | Who does it | What happens | Your involvement |
|-------|-------------|--------------|------------------|
| **Pre-training** | Model vendor | Train on massive text; learn language | None |
| **Fine-tuning** | Vendor or you (SFT, RLHF) | Adapt to tasks/style/domain | Occasionally (enterprise custom models) |
| **Inference** | You (production) | Send prompt → get completion | **Primary job** |

**Inference pipeline:**

```
Client request
 ↓
Tokenize input
 ↓
Model forward pass (attention layers)
 ↓
Sample next token (temperature/top-p)
 ↓
Repeat until stop token or max tokens
 ↓
Detokenize → response
```

**Production parameters you control:**

- Model selection
- Max output tokens
- Temperature / top-p
- Stop sequences
- System prompt

---

### 7. Temperature and Top-p

**Temperature:** Scales randomness of next-token selection.

| Value | Behavior | Use case |
|-------|----------|----------|
| 0 | Deterministic (greedy) | Factual Q&A, code generation, structured output |
| 0.3–0.7 | Balanced | General assistant |
| 1.0+ | Creative, varied | Brainstorming, creative writing |

**Top-p (nucleus sampling):** Consider smallest set of tokens whose cumulative probability ≥ p (e.g., 0.9). Cuts off unlikely tail.

**Engineering guidance:**

- Production Q&A/RAG: **low temperature (0–0.3)**
- Agents with tool selection: **low temperature** (reduce wrong tool calls)
- Do not tune both temperature and top-p aggressively at once — change one

---

### 8. Message Roles (Chat API)

Modern APIs use structured messages:

| Role | Purpose |
|------|---------|
| **system** | Instructions, persona, constraints, grounding rules |
| **user** | End-user input |
| **assistant** | Model's prior responses (conversation history) |
| **tool** | Tool results returned to model (agent flows) |

**Example structure:**

```json
[
 { "role": "system", "content": "You are a manufacturing assistant. Answer only from provided context. Cite sources." },
 { "role": "user", "content": "What is the shutdown procedure for Pump A?" }
]
```

**Engineering practices:**

- System prompt is your **policy layer** — safety, format, grounding
- Version and test system prompts like code
- Separate "developer instructions" from user-visible content where API supports it

---

## Mental Model

```
 LLM INFERENCE (Production)

 System prompt ──┐
 User message ──┼──→ Tokenizer ──→ Transformer ──→ Token sampler ──→ Text
 History ──┤ ↑ ↑ ↑
 RAG context ──┘ (count $) (attention) (temp/top-p)

 Constraints: context window | cost | latency | determinism
```

---

## Engineering Perspective

**When LLM fundamentals knowledge is useful:**

- Sizing RAG chunks to fit context with room for answer
- Explaining why retrieval beats giant prompts
- Cost estimation for product features
- Debugging "model forgot instructions" (truncation, weak system prompt)

**When it is not useful:**

- Deciding convolution filter sizes
- Implementing CUDA kernels
- Choosing number of attention heads for custom training

**Scalability:** Throughput = requests × tokens × model size; use batching for embeddings, async for generation.

**Latency:** Time to first token (TTFT) + tokens per second; streaming helps UX.

**Cost:** `input_tokens × input_price + output_tokens × output_price`; embedding cost for RAG indexing.

**Reliability:** Probabilistic outputs — same prompt can vary (especially temperature > 0).

---

## Common Mistakes

1. **Ignoring token counts** — surprise bills, truncated prompts
2. **Confusing embeddings with LLM completions** — different APIs, different models
3. **Setting temperature high for factual tasks** — increases hallucination
4. **Assuming unlimited context** — always design for limits
5. **Sending duplicate content every turn** — waste tokens; use caching or reference by ID
6. **Thinking "more parameters = always better"** — latency, cost, and RAG quality often matter more

---

## Production Considerations

**Performance:** Stream responses; limit max output tokens; use smaller models for classification/routing.

**Cost:** Monitor tokens per request; alert on anomalies; prompt caching for repeated system prefixes.

**Observability:** Log token counts, model ID, latency, temperature; trace retrieval + generation separately in RAG.

**Security:** System prompt injection via user content — validate and separate untrusted input (Day 4+).

---

## Example Architecture

**Multi-model routing (enterprise cost optimization):**

```
User query
 ↓
Router (small/fast model or rules)
 ├─ Simple FAQ → Small LLM (cheap)
 ├─ Complex doc Q&A → RAG + Large LLM
 └─ Tool action → Agent loop
```

Each path has different token budgets and temperature settings.

---

## Hands-On Exercise

**Exercise 1 — Token estimation (10 minutes)**

Take a paragraph from an internal doc (or any 300-word text). Estimate token count (~0.75 × word count). If you have Python:

```python
# pip install tiktoken
import tiktoken
enc = tiktoken.encoding_for_model("gpt-4")
text = "..." # your paragraph
print(len(enc.encode(text)))
```

**Exercise 2 — Context budget (15 minutes)**

Design a context budget for a RAG Q&A system with a 128K context model:

| Component | Allocated tokens |
|-----------|------------------|
| System prompt | ? |
| Retrieved chunks (5 × 800 tokens) | ? |
| Conversation history | ? |
| User question | ? |
| Response reserve | ? |
| **Total** | ≤ 128K |

Explain what you would cut first under pressure.

**Exercise 3 — Parameter tuning (10 minutes)**

For each scenario, specify temperature (0–1) and justify:

1. Compliance policy Q&A with citations
2. Marketing tagline brainstorming
3. Agent selecting which API to call

**Exercise 4 — Explain to a peer (5 minutes)**

Explain Q/K/V attention to a senior backend engineer in 3 sentences without math.

---

## Practice Questions

Use these to test your understanding after completing the lesson. Answer in your own words before checking the summary below.

### Core concepts

1. What is a transformer and why did it matter for LLMs?
2. What is a token?
3. What is an embedding?
4. What is the context window?
5. What is the difference between training and inference?

### Applied thinking

1. Explain attention using Q, K, and V without deriving the equation.
2. Why does attention make very long context expensive?
3. How would you estimate monthly LLM cost for 10,000 daily users?
4. When would you use temperature 0 vs 0.7 in production?
5. Why must the same embedding model be used for indexing and querying in RAG?
6. How would you manage LLM costs for an internal assistant used by thousands of employees?
7. What context window strategy would you use for long equipment manuals?

### Design thinking

1. Your RAG system sends 50 document chunks per query to stay safe. Context allows 20. How do you redesign?
2. How would you implement conversation memory for a 50-turn chat without exceeding context limits?

---

## Self Assessment

1. Why are transformers parallelizable where RNNs were not?
2. In one sentence each: Query, Key, Value in attention.
3. Name three engineering consequences of token-based billing.
4. What is the difference between an LLM and an embedding model?
5. What happens when input exceeds the context window?
6. What temperature would you use for a production RAG compliance bot?
7. What are the four common chat message roles?
8. Why is inference, not training, your primary production concern?

---

## Key Takeaways

1. **Transformers + attention** enabled scalable LLMs; you integrate, not invent.
2. **Q/K/V** — tokens ask (Q), advertise (K), and deliver content (V); attention weights combine them.
3. **Tokens** drive cost, latency, and context — count them in design.
4. **Embeddings** are semantic vectors — foundation of RAG retrieval (Day 3).
5. **Context window** is a hard constraint — RAG and summarization exist to work around it.
6. **Inference** is production; control temperature, top-p, system prompts, max tokens.
7. Attention is **O(n²)** — long context has real compute cost.

---

## Summary

Short reference for the concepts covered today:

- **Transformers** use self-attention so each token can relate to every other token in parallel, which enabled training at LLM scale.
- **Attention** uses queries, keys, and values to compute weighted combinations of context — you do not implement this, but it explains context limits and cost.
- **Tokens** are the billing and context unit; **embeddings** are dense vectors for semantic similarity, produced by a separate embedding model.
- In production you optimize **inference**: token budgets, temperature, context allocation, and model routing — not training from scratch.
