# Day 1 — AI / ML / Deep Learning / LLM Landscape

> **Status:** COMPLETED (complete self assessment to confirm understanding)

---

## Learning Objectives

By the end of this lesson, you should understand:

1. The difference between **AI**, **ML**, **deep learning**, **generative AI**, **foundation models**, and **LLM applications**
2. **Supervised**, **unsupervised**, and **reinforcement** learning — what each is and when it applies
3. **Inference vs training** and **model vs application** responsibilities
4. Where **LLMs fit** in software architecture (API-based vs self-hosted)
5. What an **AI Engineer** does versus ML research and traditional software engineering
6. Core **LLM and agentic terminology** for enterprise engineering
7. How the major pieces of the **AI engineering landscape** fit together as a coherent mental model

---

## Why This Matters

Before you build RAG pipelines, agents, or production deployments, you need a **shared vocabulary and map of the territory**. Without it, every new tool feels disconnected, and architectural decisions become guesswork.

This lesson answers questions like:

- What is actually happening when you call an LLM API?
- Where does your code end and the model vendor's responsibility begin?
- When is a simple LLM call enough, and when do you need RAG or an agent?
- What does "AI Engineer" mean as a discipline — distinct from ML research or traditional software engineering?

Enterprise teams building production AI systems care about:

- Designing systems with security and governance built in
- Choosing architectures (RAG vs fine-tuning vs agents) with clear tradeoff reasoning
- Operating systems at scale with observability and evaluation
- Integrating with existing platform teams and business workflows

This lesson establishes the map. Everything else in the course fills in the territory.

---

## Prerequisites

None — this is the entry point. You should already have:

- Senior software engineering experience
- Exposure to AI-assisted development tools (Cursor, Claude, Qodo)
- Awareness of MCP and agent workflows from prior enterprise work (optional but helpful)

---

## Core Concepts

### 1. ML Learning Paradigms

| Paradigm | What it is | Example | Your role |
|----------|------------|---------|-----------|
| **Supervised learning** | Learn from labeled input→output pairs | Fraud classifier, spam filter | Integrate trained models; rarely train from scratch |
| **Unsupervised learning** | Find patterns in unlabeled data | Clustering, anomaly detection | Same — consume as API or service |
| **Reinforcement learning** | Learn via reward signal from environment | Game playing, some alignment (RLHF) | Understand RLHF conceptually; don't implement |

**Engineering note:** Most enterprise LLM work is **not** training these models. You use **pre-trained foundation models** via API.

---

### 2. Generative AI, Foundation Models, and LLMs

| Term | Definition |
|------|------------|
| **Generative AI** | AI that creates new content (text, images, code) |
| **Foundation model** | Large model pre-trained on broad data; adapted for many tasks (GPT, Claude, Llama) |
| **LLM** | Foundation model specialized for language — predicts next token |

**Model vs application:**

| Layer | Responsibility | Who |
|-------|----------------|-----|
| **Model** | Language capability, knowledge cutoff, safety tuning | Vendor (OpenAI, Anthropic, AWS Bedrock) |
| **Application** | Prompts, RAG, tools, auth, UI, observability, cost | **You (AI Engineer)** |

**API-based vs self-hosted:**

| Approach | When to use |
|----------|-------------|
| **API (Bedrock, OpenAI, Anthropic)** | Default for enterprise; managed scaling, compliance options |
| **Self-hosted (ECS, K8s, vLLM)** | Data residency, cost at scale, custom models — more ops burden |

---

### 3. AI vs ML vs Deep Learning vs LLM Applications

| Term | What it means | Your relevance |
|------|---------------|----------------|
| **AI** | Systems that perform tasks requiring human-like intelligence | Umbrella term; use carefully in technical discussions |
| **Machine Learning** | Systems that learn patterns from data (classifiers, regressors, recommenders) | Your MS CS background; not the primary focus of this course |
| **Deep Learning** | ML using neural networks with many layers | Foundation of LLMs; understand conceptually |
| **LLM** | Large language model — predicts next token; trained on massive text | Core technology you integrate, not train |
| **LLM Application** | Software that calls an LLM API to solve a user problem | What you build |

**Key insight:** You are building **LLM applications** and **agentic systems**, not training GPT from scratch. That is the correct mental model for this role.

---

### 4. Inference vs Training

| Phase | What happens | Your involvement |
|-------|--------------|------------------|
| **Training** | Model learns from massive datasets | None (vendor) except rare fine-tuning |
| **Inference** | Model produces output from input | **Primary job** — APIs, prompts, RAG, agents |

---

### 5. What Is an AI Engineer?

**Intuition:** An AI Engineer is a **software engineer who designs and ships production systems where LLMs are a core component** — not someone whose primary job is training neural networks from scratch.

| Role | Primary focus |
|------|---------------|
| ML Research Scientist | Novel models, papers, training at scale |
| ML Engineer | Feature pipelines, model training/deployment (traditional ML) |
| **AI Engineer** | LLM/RAG/agent **application architecture**, production deployment, security, evaluation |
| AI Enablement Lead | Tool adoption, governance, training teams (overlaps; hands-on experience) |
| Software Engineer + AI tools | Uses Copilot/Cursor; does not design AI systems |

**Engineering mechanics:** Your job is to answer:

- What problem are we solving with an LLM?
- What architecture fits (simple completion, RAG, agent, workflow)?
- How do we deploy it (API, Lambda, ECS, Bedrock)?
- How do we secure it (auth, PII, prompt injection)?
- How do we know it works (evaluation, monitoring)?

---

### 6. LLM Terminology (Engineering Level)

| Term | Definition | Why it matters |
|------|------------|----------------|
| **Token** | Smallest unit the model processes (~3/4 of a word in English) | Drives cost, latency, context limits |
| **Context window** | Maximum tokens the model can consider in one request | Limits how much document/history you can send |
| **Prompt** | Input text (system + user + assistant messages) | Your primary "programming" interface |
| **Completion / Generation** | Model output (predicted tokens) | What users see |
| **Inference** | Running a trained model to produce output | Distinct from training; what production systems do |
| **Embedding** | Dense vector representing semantic meaning of text | Foundation of RAG and search |
| **Fine-tuning** | Additional training on domain-specific data | Alternative/complement to RAG |
| **Hallucination** | Model generates plausible but incorrect information | Major production concern |
| **Grounding** | Tying answers to retrieved or provided source material | Reduces hallucination in RAG |
| **Temperature** | Randomness control (0 = deterministic, higher = creative) | Production tuning parameter |
| **Top-p (nucleus sampling)** | Limits token selection to cumulative probability mass | Alternative to temperature |

---

### 7. Agentic AI Terminology

| Term | Definition |
|------|------------|
| **LLM Application** | Input → LLM → Output (single turn or simple multi-turn) |
| **RAG** | Input → Retrieve relevant docs → LLM → Output |
| **Agent** | LLM that **decides actions**, calls **tools**, observes results, and loops until task complete |
| **Tool / Function calling** | LLM invokes structured API (search DB, create ticket, call REST API) |
| **MCP (Model Context Protocol)** | Standard for exposing tools, resources, and prompts to AI clients |
| **Orchestration** | Coordinating multiple steps, agents, or services (e.g., delivery orchestrators) |
| **Agent loop** | Plan → Act (tool) → Observe → Reason → repeat |
| **Memory** | Persisted context across sessions (short-term in context; long-term in DB) |
| **Workflow** | Deterministic or semi-deterministic sequence of steps (may include LLM steps) |
| **Multi-agent** | Multiple specialized agents coordinating on a task |

**Progression:**

```
Simple App RAG Agent
───────── ───── ──────
Input Input Input
 ↓ ↓ ↓
LLM Retrieve LLM
 ↓ ↓ ↓
Output LLM Plan → Tool → Observe → ...
 ↓ ↓
 Output Final Output
```

You already work in the **agent + orchestration** space. This course formalizes the vocabulary.

---

### 8. Enterprise AI Landscape

Typical enterprise AI use cases:

| Use case | Pattern | Example |
|----------|---------|---------|
| Internal knowledge assistant | RAG + auth | "What is our PTO policy?" |
| Document Q&A | RAG + citations | Legal/compliance document search |
| Customer service | RAG + agent + tools | Ticket creation, order lookup |
| Workflow automation | Agent + integrations | Jira → plan → implement pipelines |
| Code assistance | Agent + IDE tools | Cursor, Qodo |
| Manufacturing ops | RAG + structured data tools | Equipment manuals + sensor queries |

Regulated environments add: audit logging, data residency, human approval, least-privilege tool access.

---

## Mental Model

```
 AI ENGINEER SCOPE

 ┌─────────────────────────────────────────────────┐
 │ Business requirements & success criteria │
 └─────────────────────┬───────────────────────────┘
 ↓
 ┌─────────────────────────────────────────────────┐
 │ Architecture choice │
 │ App | RAG | Agent | Workflow | Hybrid │
 └─────────────────────┬───────────────────────────┘
 ↓
 ┌──────────┬──────────┬──────────┬───────────────┐
 │ LLM │ Data │ Tools │ Platform │
 │ provider │ (vector, │ (APIs, │ (AWS, IAM, │
 │ │ docs) │ MCP) │ observability)│
 └──────────┴──────────┴──────────┴───────────────┘
 ↓
 ┌─────────────────────────────────────────────────┐
 │ Production: security, eval, cost, reliability │
 └─────────────────────────────────────────────────┘
```

---

## Engineering Perspective

**When AI/LLM solutions are useful:**

- Unstructured knowledge (documents, policies, manuals)
- Natural language interfaces to complex systems
- Tasks requiring reasoning over varied inputs
- Automation of multi-step workflows with judgment

**When they are NOT useful:**

- Deterministic calculations (use code)
- Strict regulatory decisions without human review
- Real-time sub-10ms responses at massive scale (without careful architecture)
- Problems with abundant labeled data where traditional ML is cheaper and more reliable

**Common architectural choices:**

| Need | Typical choice |
|------|----------------|
| Q&A over company docs | RAG |
| Fixed business process | Workflow + optional LLM steps |
| Dynamic multi-step tasks | Agent |
| Domain-specific language/style | Fine-tuning (or RAG + good prompts) |
| Enterprise AWS deployment | Bedrock, Lambda/ECS, S3, OpenSearch/pgvector |

**Scalability considerations:** Token throughput, embedding batch jobs, vector DB size, concurrent API calls to LLM provider.

**Latency considerations:** Retrieval adds 50–500ms; agent loops multiply LLM calls; streaming improves perceived latency.

**Cost considerations:** Tokens (input + output), embeddings, vector storage, compute for ingestion pipelines.

**Reliability considerations:** LLMs are probabilistic; design fallbacks, caching, and human escalation.

---

## Common Mistakes

1. **Calling everything "AI"** — be precise: RAG, agent, workflow, classifier.
2. **Confusing AI enablement with AI engineering** — enabling teams to use Cursor is valuable but different from designing a production RAG platform.
3. **Assuming bigger model = better solution** — often RAG + smaller model beats large model alone.
4. **Ignoring operational concerns** — no evaluation, no monitoring, no cost tracking.
5. **Treating agents as magic** — agents fail on bad tools, bad prompts, missing guardrails.
6. **Over-promising autonomy** — enterprise agents need human-in-the-loop for high-stakes actions.

---

## Production Considerations

**Security:** Data sent to external LLM APIs may leave your boundary; use private endpoints, Bedrock VPC, or on-prem where required.

**Governance:** Model usage policies, approved tools, audit trails — align with your organization's AI governance framework.

**Observability:** Log prompts (carefully — PII), latency, token usage, retrieval quality, tool call success.

**Reliability:** Timeouts, retries, circuit breakers on LLM and tool calls.

**Cost:** Token budgets per user/session; cache frequent queries.

---

## Example Architecture

**Enterprise internal knowledge assistant (manufacturing company):**

```
Employee (browser)
 ↓
API Gateway + Cognito auth
 ↓
Lambda / ECS — Orchestration API
 ↓
┌─────────────────────────────────────┐
│ RAG Service │
│ ├─ Embed query (Bedrock Titan) │
│ ├─ Search OpenSearch / pgvector │
│ ├─ Metadata filter (site, dept) │
│ └─ Build grounded prompt │
└─────────────────────────────────────┘
 ↓
Bedrock Claude / GPT (via private endpoint)
 ↓
Response + citations
 ↓
CloudWatch logs + cost metrics
```

Authorization: user can only retrieve documents for their plant/role (metadata filtering + IAM).

---

## Hands-On Exercise

**Exercise 1 — Role reflection (written, 5 minutes)**

In your own words, describe what an AI Engineer does in an enterprise and how that differs from traditional software engineering and ML research. Save this in your `notes/` folder — you will revisit it at the end of Week 1.

Structure:

1. What the role is (design + ship production AI systems)
2. How it differs from traditional software engineering and ML research
3. One concrete enterprise example (knowledge assistant, agent, orchestrator)
4. What production concerns matter (security, evaluation, cost, reliability)
5. What skills you want to develop through this course

**Exercise 2 — Terminology mapping (10 minutes)**

For each system you have built or used, label it:

| System | App | RAG | Agent | Workflow |
|--------|-----|-----|-------|----------|
| Cursor chat | | | | |
| Team delivery orchestrator | | | | |
| Qodo PR review | | | | |
| ChatGPT with file upload | | | | |

**Exercise 3 — Architecture sketch (15 minutes)**

Pick one enterprise scenario (e.g., "HR policy assistant"). Draw a one-page box diagram showing: user, auth, API, LLM, data store, observability. Label which boxes you would own as AI Engineer.

---

## Practice Questions

Use these to test your understanding after completing the lesson. Answer in your own words before checking the summary below.

### Core concepts

1. What is the difference between AI, machine learning, and an LLM application?
2. What is a token and why does it matter for cost?
3. What is inference?
4. What is the difference between a chatbot and an AI agent?
5. What is RAG in one sentence?

### Applied thinking

1. How would you describe the AI Engineer role to a colleague joining your team?
2. When would you choose a simple LLM application over RAG over an agent?
3. How does a strong software engineering background help when building AI systems?
4. What operational concerns must be addressed before deploying an LLM feature to thousands of users?
5. What is the difference between AI enablement and AI engineering?

### Design thinking

1. A manufacturing company wants an AI assistant for equipment manuals and maintenance procedures. What questions would you ask before proposing an architecture?
2. How would you compare LLM-based automation versus traditional workflow automation (Step Functions, BPM)?
3. A company has sensitive operational knowledge. Where do LLMs fit versus traditional software systems?
4. How would you introduce AI capabilities into a large enterprise without creating security or governance risk?

---

## Self Assessment

Answer without looking at the lesson:

1. Define AI Engineer in your own words.
2. List five LLM terms and explain each in one sentence.
3. Draw the progression from simple app → RAG → agent.
4. Name three enterprise AI use cases and which pattern (app/RAG/agent) fits each.
5. What is grounding and why does it matter?
6. What is MCP and how does it relate to tool calling?
7. Why is "we'll fine-tune GPT" often the wrong first answer?
8. What production concerns apply to every LLM deployment?

---

## Key Takeaways

1. AI Engineer = **production LLM/RAG/agent systems**, not ML research.
2. **Terminology precision** helps you reason about architecture and communicate with your team.
3. **App → RAG → Agent** is the capability progression; choose based on problem.
4. **Cloud + enterprise + agentic tooling** experience maps directly to building production AI systems.
5. Production systems require **operationalization** — security, eval, cost, reliability.
6. **AI enablement** (tool adoption) complements but does not replace **AI engineering** (system design).
7. Keep a personal glossary in `notes/` as you encounter new terms throughout the course.

---

## Summary

Short reference for the concepts covered today:

- An **AI Engineer** designs and ships production systems where LLMs are integrated with enterprise data, tools, and cloud infrastructure — with security, evaluation, and operational concerns built in.
- An **LLM application** calls a model API; **RAG** adds retrieval from a knowledge base; an **agent** adds autonomous tool use and multi-step reasoning loops.
- **Tokens** drive cost and context limits; **inference** is running a trained model, not training it.
- **Foundation models** are pre-trained by vendors; your work lives in the application layer — prompts, RAG, tools, auth, and observability.
