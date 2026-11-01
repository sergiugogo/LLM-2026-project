# WWII DIY — Documentation

An AI-run alternate-history WWII game. The player leads a nation from September
1939; every decision changes how the war unfolds and how it ends.

**Authors:** Iclozan Mihnea-Serban, Rusu Mihai, Mogosan Sergiu-Ionut

| Document | What it covers |
|---|---|
| [RFC-001](RFC.md) | Milestone 1 — the problem, state of the art, and the proposal |
| [ARCHITECTURE.md](ARCHITECTURE.md) | Milestone 2 — task, data, ingestion, agent decisions, tools, MCP servers, model choices |
| [RAG_DESIGN.md](RAG_DESIGN.md) | Milestone 2 — chunking, embeddings, vector database, the seven RAG variants we compare |
| [GOLDEN_DATASET.md](GOLDEN_DATASET.md) | Milestone 2 — five golden sets, how they are built and governed |
| [PLAN.md](PLAN.md) | Milestone 2 — week-by-week plan to the Week 14 presentation, and the ten experiments |

---

## Milestone 2 requirement coverage

Each row points at the exact section that answers it.

| Requirement | Where |
|---|---|
| **Problem and data** — what task the system will handle | [ARCHITECTURE §1](ARCHITECTURE.md#1-the-task-the-system-handles) |
| **Problem and data** — which dataset | [ARCHITECTURE §3](ARCHITECTURE.md#3-data) — six sources, each with its licence |
| **Problem and data** — how it supports ingesting new data | [ARCHITECTURE §4](ARCHITECTURE.md#4-data-ingestion-the-data-pack-format) — versioned data packs |
| **Golden Dataset** — representative questions and expected answers | [GOLDEN_DATASET.md](GOLDEN_DATASET.md) — 500 items across five sets, with schemas and build process |
| **RAG design** — chunking strategy | [RAG_DESIGN §2](RAG_DESIGN.md#2-chunking) |
| **RAG design** — embedding model | [RAG_DESIGN §3](RAG_DESIGN.md#3-embedding-model) |
| **RAG design** — vector database | [RAG_DESIGN §4](RAG_DESIGN.md#4-vector-database) |
| **RAG design** — which approaches we use or compare | [RAG_DESIGN §6](RAG_DESIGN.md#6-rag-approaches-we-compare) — R0 to R6; **R4 ships** |
| **Agent and tools** — what decisions the agent makes | [ARCHITECTURE §5](ARCHITECTURE.md#5-what-the-agent-decides) |
| **Agent and tools** — what tools it uses | [ARCHITECTURE §6](ARCHITECTURE.md#6-tool-inventory) — 14 tools |
| **Agent and tools** — how they are exposed through an MCP server | [ARCHITECTURE §7](ARCHITECTURE.md#7-mcp-server-design) — three servers, split by concern |
| **Model choices** — which SLM and other models, and why | [ARCHITECTURE §8](ARCHITECTURE.md#8-model-choices) |
| **Plan** — week by week through Week 14 | [PLAN §3](PLAN.md#3-week-by-week) |
| **Plan** — the experiments we intend to complete | [PLAN §4](PLAN.md#4-experiments) — E1 to E9 |

---

## The three ideas the design rests on

**The model never owns the truth.** The world state lives in Postgres behind
`mcp-world`. The agent proposes a state delta; `apply_state_delta` is the only
writer and rejects anything computed against a stale state version. This is the
answer to the drift that makes existing LLM sandbox games forget the war.
→ [ARCHITECTURE §1.1](ARCHITECTURE.md#11-the-failure-mode-we-are-designing-against),
[§5](ARCHITECTURE.md#5-what-the-agent-decides)

**Retrieval is time-aware, because the future has not happened yet.** In November
1939 the most semantically relevant chunk for "should I invade Norway" is the
article about the April 1940 invasion — correctly retrieved and exactly wrong. A
hard `year_range` filter inside the index is the fix, and the anachronism rate is
a first-class metric.
→ [RAG_DESIGN §2.3](RAG_DESIGN.md#23-why-year_range-is-a-first-class-field)

**Evaluation is split by failure mode, not aggregated.** Five golden sets, because
a retrieval regression and a tool-calling regression are indistinguishable in a
single narrative score. Two of them measure the thing nobody measures: refusal of
invented answers, and *over*-refusal of legitimate historical content.
→ [GOLDEN_DATASET §1](GOLDEN_DATASET.md#1-why-there-are-five-golden-sets-and-not-one)

---

## Conventions in these documents

Every design subsection is written as **Decision → Why → Rejected**, so the
alternative we turned down and the reason it lost are visible without reading the
surrounding prose.

Where a choice is not yet settled, it appears in
[ARCHITECTURE §10](ARCHITECTURE.md#10-decisions-still-open) with the date we will
decide it and the experiment that decides it — not as an omission.

Diagrams are Mermaid, so they render on GitHub and a reviewer can comment on a
specific edge.
