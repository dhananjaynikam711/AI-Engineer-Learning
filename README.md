# Senior AI Engineer — LLM, RAG, Agents & Production AI

An **84-day**, engineering-focused curriculum for experienced software engineers building production LLM, RAG, and agent systems.

---

## What This Course Is

A structured path from **LLM foundations** through **RAG**, **application engineering**, **agents**, **MCP**, **production architecture**, **security**, **evaluation**, and **system design** — ending with capstone review and consolidation.

This is a **learning course**, not a beginner programming course, a mathematical ML course, or an interview prep bootcamp. Lessons prioritize engineering understanding: architecture, tradeoffs, production behavior, and how components fit together.

---

## Who This Course Is For

Experienced software engineers with backgrounds in application development, distributed systems, and cloud platforms who want to design, build, secure, evaluate, and operate enterprise AI systems.

Typical background:

- Application engineering (.NET, Angular, Python, SQL, or similar)
- Cloud and distributed systems (AWS or equivalent)
- Exposure to LLM tools in development workflows

**Goal:** Senior AI Engineer depth — able to reason about LLM/RAG/agent systems end to end, from ingestion through production monitoring.

---

## Learning Philosophy

### Engineering understanding first

Build real understanding you can apply when designing and shipping systems. Do not frame lessons around interview answers or rehearsed talking points.

Do **not** excessively focus on mathematics. When math appears:

1. Explain the intuition
2. Show the important equation if useful
3. Explain what the variables mean
4. Explain why it matters to an engineer
5. Move on

### For every concept, prioritize

- What is it?
- Why does it exist?
- How does it work?
- When do I use it?
- When should I **not** use it?
- Tradeoffs: architecture, cost, latency, reliability, security, evaluation
- How the concept applies in production system design

### Teaching layers

| Level | Focus |
|-------|--------|
| 1 — Intuition | Simple explanation |
| 2 — Technical | How it actually works |
| 3 — Engineering | Design decisions and tradeoffs |
| 4 — Production | Real enterprise architecture |
| 5 — Practice | Exercises, design problems, self assessment |

---

## Repository Layout

| Path | Purpose |
|------|---------|
| [`COURSE_PLAN.md`](COURSE_PLAN.md) | Full 12-week / 84-day roadmap |
| [`PROGRESS.md`](PROGRESS.md) | Personal progress tracking (local; not required for colleagues) |
| [`week-XX/day-XX.md`](week-01/) | Daily lessons |
| [`projects/`](projects/) | Six incremental hands-on projects (added as you reach them) |
| `notes/` | Personal study notes (gitignored) |

---

## How to Use This Course

### Daily workflow

1. Read the lesson for the day
2. Complete **Hands-On Exercises**
3. Work through the **Knowledge Check** and **Self Assessment** without looking back at the lesson
4. Update your personal `PROGRESS.md` and notes when you are satisfied you understand the material
5. Move to the next day

### Completion criteria

Do **not** mark a day complete until you can explain the core concepts in your own words and complete the exercises. Understanding matters more than speed.

### Lesson format

Each full lesson includes:

Learning Objectives · Why This Matters · Prerequisites · Core Concepts · Mental Model · Architecture Diagrams · Tradeoffs · Common Failure Modes · Hands-On Exercise · Knowledge Check · Self Assessment · Key Takeaways · Summary

Days marked as stubs (review days, future weeks) are expanded when you reach them.

---

## Course Progression

| Week | Focus |
|------|-------|
| **Week 1** | LLM Foundations + RAG |
| **Week 2** | LLM Application Engineering |
| **Week 3** | Advanced RAG |
| **Week 4** | AI Agents |
| **Week 5** | MCP & Agent Ecosystems |
| **Week 6** | Production AI Architecture |
| **Week 7** | AI Security |
| **Week 8** | AI Evaluation & Quality |
| **Week 9** | Advanced Model Engineering |
| **Week 10** | AI System Design |
| **Week 11** | Enterprise AI Architecture |
| **Week 12** | Capstone Review & Consolidation |

See [`COURSE_PLAN.md`](COURSE_PLAN.md) for the full day-by-day breakdown.

---

## Week 1 — Available Lessons

| Day | Topic | Lesson |
|-----|-------|--------|
| 1 | AI / ML / Deep Learning / LLM Landscape | [week-01/day-01.md](week-01/day-01.md) |
| 2 | LLM Fundamentals | [week-01/day-02.md](week-01/day-02.md) |
| 3 | RAG Fundamentals | [week-01/day-03.md](week-01/day-03.md) |
| 4 | Enterprise RAG | [week-01/day-04.md](week-01/day-04.md) |
| 5 | RAG Evaluation | [week-01/day-05.md](week-01/day-05.md) |
| 6 | Week 1 Review | _stub — generated when you reach this day_ |
| 7 | Rest / Spaced Review | _stub — generated when you reach this day_ |

Weeks 2–12 follow the same pattern: lessons are added to the repo as they are written.

---

## Current Status

| Content | Status |
|---------|--------|
| Days 1–5 | **Available** — full lessons in [`week-01/`](week-01/) |
| Days 6–7 | Stubs — Week 1 review and spaced review |
| Days 8–84 | Planned — see [`COURSE_PLAN.md`](COURSE_PLAN.md) |
| Projects | Not yet started — see projects roadmap in course plan |

**Suggested next step:** [Day 5 — RAG Evaluation](week-01/day-05.md), then Week 1 review (Day 6).

---

## Week 1 at a Glance

What you should understand after completing Week 1:

```
Day 1  → AI/ML/DL/LLM landscape; model vs application; inference vs training
Day 2  → Transformers, tokens, embeddings, context windows, LLM inference
Day 3  → RAG pipeline: ingest, chunk, embed, retrieve, generate, ground
Day 4  → Enterprise RAG: ingestion, ACL, hybrid search, freshness, security
Day 5  → RAG evaluation: retrieval, generation, golden datasets, regression testing
Day 6  → Week 1 consolidation
Day 7  → Spaced review
```

---

## Projects Roadmap

Hands-on projects reinforce the curriculum. See [`COURSE_PLAN.md`](COURSE_PLAN.md) for timing.

| # | Project | After |
|---|---------|-------|
| 1 | Simple LLM Application | Week 2 |
| 2 | Basic RAG | Week 1–3 |
| 3 | Enterprise RAG | Week 3 |
| 4 | Tool-Using Agent | Week 4 |
| 5 | MCP Agent | Week 5 |
| 6 | Production Enterprise AI Platform | Week 6+ |

---

## Sharing With Colleagues

Share the repository folder. Each learner maintains their own `notes/` and personal progress files. Group study works well for design exercises and peer discussion of architecture tradeoffs.

---

## Local Tooling (Not in Git)

These directories are excluded from version control and are safe to use locally:

- `.cursor/` — Cursor IDE settings
- `.specstory/` — SpecStory session history
- `.cursorindexingignore` — Cursor indexing exclusions
- `notes/`, `interview-prep/`, `knowledge-check/` — personal study material

---

_Last updated: 2026-08-26_
