# Milestone 2 — Golden Dataset

**Companions:** [ARCHITECTURE.md](ARCHITECTURE.md), [RAG_DESIGN.md](RAG_DESIGN.md),
[PLAN.md](PLAN.md) · **Precedes:** [RFC-001](RFC.md)

---

## 1. Why there are five golden sets and not one

The system has five independently breakable surfaces, and a single mixed
question-and-answer file would measure none of them cleanly.
A retrieval regression and a tool-calling regression look identical in an
aggregate narrative score, and we would have no way to tell which one to fix.

| Set | Surface under test | Items | Owner |
|---|---|---|---|
| **G1** | Retrieval + grounded answering over the corpus | 180 | Workstream B |
| **G2** | Tool-call sequence and state-delta correctness | 120 | Workstream A |
| **G3** | Consistency checking / hallucination detection | 80 | Workstream B |
| **G4** | Narrative quality | 60 | Workstream C |
| **G5** | Safety: refusal **and** over-refusal | 60 | Workstream C |
| | **Total** | **500** | |

500 hand-reviewed items across three people is roughly 34 person-hours of
review, itemised per set in §8. That is budgeted in [PLAN.md](PLAN.md) across W6-W9 and is the single
largest non-coding cost in the project. We are stating the number because an
evaluation plan whose cost is not accounted for is an evaluation plan that does
not get built.

All five live in `eval/golden/*.jsonl`, with JSON Schemas in `eval/schema/`, and
are committed to git — they are a deliverable, not a by-product.

---

## 2. G1 — Reference QA

Tests [RAG_DESIGN.md](RAG_DESIGN.md) end to end: chunking, encoder, index,
retrieval strategy and grounded answering.

### 2.1 Composition

Stratified on purpose, because a flat set of easy factual lookups would report a
high score and hide every failure we actually expect.

| Slice | Share | Items | What it catches |
|---|---|---|---|
| Single-hop factual | 40% | 72 | Baseline retrieval and extraction |
| Multi-hop / comparative | 25% | 45 | Whether R6 (entity-anchored) earns its place |
| Temporally constrained — "as of month X" | 15% | 27 | The anachronism failure in RAG_DESIGN §2.3 |
| Entity-attribute lookup | 10% | 18 | Whether Wikidata-derived fields are reachable |
| **Unanswerable / out-of-corpus** | 10% | 18 | Refusal instead of invention |

The unanswerable slice is the one we would defend hardest.
Without it, a model that confidently invents an answer to everything scores
*better* than one that correctly abstains, and the whole evaluation rewards the
behaviour the project exists to prevent.

### 2.2 Item schema

```json
{
  "id": "G1-0042",
  "question": "Which tank did Romania's 1st Armoured Division field in mid-1941?",
  "expected_answer": "The Renault R-35 and the Czech-built Skoda R-2 (LT vz. 35).",
  "answer_type": "short_factual",
  "slice": "entity_attribute",
  "as_of_date": "1941-06",
  "gold_doc_ids": ["wiki:1st_Armored_Division_(Romania)", "wiki:Skoda_R-2"],
  "gold_spans": [{"doc_id": "wiki:Skoda_R-2", "start": 1840, "end": 1998}],
  "difficulty": "medium",
  "tags": ["romania", "equipment", "1941"],
  "unanswerable": false,
  "notes": "R-2 is the Romanian designation; checks synonym handling."
}
```

`gold_doc_ids` makes retrieval metrics computable without a model in the loop,
which means Recall@10 and nDCG@10 are cheap enough to run on every commit, and
only the generation metrics need a GPU.

### 2.3 How it is built

```mermaid
flowchart TB
    A["Stratified sample of indexed chunks, by source and year"] --> B["Teacher model drafts a question, answer and source span per chunk"]
    B --> C["Automatic filters"]
    C --> C1["Reject if answerable without the chunk, i.e. closed-book model gets it right"]
    C --> C2["Reject if the answer is not entailed by the cited span"]
    C --> C3["Reject near-duplicates by MinHash"]
    C1 --> D
    C2 --> D
    C3 --> D
    D["Human review: 3 disjoint thirds, 20% overlap slice"] --> E["Inter-annotator agreement on the overlap, Cohen kappa"]
    E -->|"kappa < 0.7"| F["Tighten the guidelines, re-review"]
    E -->|"kappa >= 0.7"| G["Hand-written adversarial additions"]
    G --> H["Leakage check: MinHash against the fine-tuning set"]
    H --> I["Freeze as a tagged version"]
```

Three parts of that are worth calling out.

**Filter C1 is the one that makes the set measure retrieval.** If a closed-book
model already answers an item correctly, that item tells us nothing about the
retrieval stack and only inflates every score. Dropping those is what keeps R0
(no retrieval) an honest floor in **E2**.

**The 20% overlap slice** is how we find out whether "expected answer" means the
same thing to all three of us, before we have built a semester of results on top
of it. We report κ in the Week 9 progress update.

**Adversarial additions are written by hand**, because a teacher model will not
generate the traps that matter: near-name collisions ("Operation Market Garden"
versus "Operation Market Time"), questions whose naive lexical match pulls the
wrong year, and entities that changed name mid-war.
Target: 20 of the 180.

---

## 3. G2 — Tool-call and state-delta correctness

Tests the boundary in [ARCHITECTURE.md](ARCHITECTURE.md) §5: the agent proposes,
code adjudicates.
This is the set that protects the project's central claim, because a wrong state
delta is the drift the RFC was written against.

### 3.1 Composition

The action taxonomy × representative states, 120 scenarios:

| Action class | Items | | Action class | Items |
|---|---|---|---|---|
| Attack / offensive | 16 | | Espionage | 8 |
| Fortify / defend | 12 | | Propaganda / morale | 8 |
| Diplomacy — offer, accept, break | 18 | | Logistics / supply | 10 |
| Trade / resources | 10 | | Retreat / withdrawal | 8 |
| Research / technology | 10 | | **Out-of-scope or must refuse** | 10 |
| | | | **Ambiguous, must ask** | 10 |

The last two classes are 17% of the set and they are the point.
A GM that resolves everything is worse than one that occasionally says "which
front do you mean?", and nothing in the other classes would reveal the
difference.

### 3.2 Item schema

```json
{
  "id": "G2-0031",
  "action_class": "diplomacy_offer",
  "state_snapshot": "eval/golden/states/1941-03-romania.json",
  "player_action": "Offer Hungary a non-aggression pact in exchange for dropping their claim on Northern Transylvania.",
  "expected_tool_sequence": [
    "get_state_digest", "search_reference", "resolve_action", "apply_state_delta"
  ],
  "expected_delta_keypaths": [
    "relations.ROU.HUN.treaty",
    "relations.ROU.HUN.trust",
    "event_log[+1]"
  ],
  "forbidden_keypaths": ["regions.*.controller", "armies.*"],
  "must_not_invent": ["armies.ROU.new_division"],
  "expected_outcome_class": "offer_pending",
  "requires_clarification": false
}
```

`forbidden_keypaths` and `must_not_invent` are the useful half of this schema.
Checking that the *right* keys changed is easy; checking that nothing else
changed is what catches a model quietly conjuring a division or flipping
territory control as a side effect of a diplomatic offer.

### 3.3 Metrics

| Metric | Target |
|---|---|
| Tool-selection accuracy — correct set of tools | ≥ 0.95 |
| Tool-order validity — `resolve_action` strictly before `apply_state_delta` | **1.00** |
| JSON schema validity rate on deltas | ≥ 0.99 |
| State-delta F1 over changed key-paths | ≥ 0.90 |
| Forbidden-keypath violation rate | ≤ 0.01 |
| Illegal-action rejection rate | ≥ 0.90 |
| Clarification precision on the ambiguous slice | ≥ 0.80 |

Tool-order validity is the only metric in this document with a target of exactly
1.00, and that is deliberate: `apply_state_delta` before `resolve_action` means an
unadjudicated write, which is a correctness bug rather than a quality shortfall.
It is enforced by the server contract as well as measured, so a failure here is a
bug in two places at once.

G2 is also the rules engine's own regression suite, which is why
[ARCHITECTURE.md](ARCHITECTURE.md) and this set have the same owner.
Writing the expected deltas by hand against the rules spec is how we find out
whether the rules spec is actually specified.

---

## 4. G3 — Consistency probes

Tests `check_consistency`. 80 items, built by taking 40 clean turn traces from
real playthroughs and producing a contradicting variant of each.

| Injected contradiction | Items | Source of truth violated |
|---|---|---|
| Phantom unit — narrates a formation not in state | 12 | `armies` |
| Dead leader acts | 10 | `leaders` + event log |
| Territory never captured, narrated as held | 12 | `regions` |
| Treaty or war status contradicted | 8 | `relations` |
| Anachronistic technology | 8 | `technology.year_introduced` |
| Past player decision ignored | 10 | event log |
| **Clean, no contradiction** | 20 | — |

```json
{
  "id": "G3-0017",
  "state_snapshot": "eval/golden/states/1942-08-germany.json",
  "narration": "Field Marshal Reichenau personally inspects the Stalingrad front...",
  "has_contradiction": true,
  "contradiction_type": "dead_leader_acts",
  "violated_keypath": "leaders.DEU.reichenau.status",
  "evidence": "Event log turn 1942-01: Reichenau died of a stroke."
}
```

**25% of the set is clean.** A checker evaluated only on contradicting items is
rewarded for flagging everything, and a checker that flags everything triggers a
revision on every turn — which costs the latency budget in **E7** and degrades
narration quality for no benefit. So we measure false positives explicitly.

| Metric | Target |
|---|---|
| Contradiction detection precision / recall / F1 | F1 ≥ 0.80 |
| False-positive rate on the 20 clean turns | ≤ 0.10 |
| `violated_keypath` localisation accuracy | ≥ 0.70 |

Localisation accuracy is there because a checker that says "something is wrong"
cannot drive the single revision pass in [ARCHITECTURE.md](ARCHITECTURE.md) §5.2 —
the revision prompt needs to know *what* to fix.

---

## 5. G4 — Narrative quality

60 `(state, action)` prompts spread across early war, mid war and late war, and
across all eight playable nations.
Each is scored 1-5 on five rubric dimensions:

| Dimension | 1 | 5 |
|---|---|---|
| **Grounding** | contradicts or ignores the state | every claim traceable to state, log or retrieved context |
| **Period plausibility** | anachronistic tech, places or language | consistent with 1940s capability and idiom |
| **Voice** | drifts out of second person, or into pastiche | consistent second-person address, restrained period register |
| **Divergence** | railroads back to real history | follows the player's divergence and lets it compound |
| **Playability** | suggested actions vague or already impossible | 3 distinct, legal, consequential options |

**Divergence** is a rubric dimension we would not have if we were building a
history tutor. It exists because the base model's strongest prior is real history,
so "the model quietly steers the war back to what actually happened" is our most
likely narrative failure, and it is invisible to grounding and plausibility
scoring — a railroaded narration can be perfectly grounded.

**Scoring.** An LLM judge from a different family than the GM — Llama-3.3-70B
against a Qwen3 GM — scores all 60, with the rubric and two reference exemplars
per dimension in the prompt.
All three of us independently score a 20-item calibration subset.
**E6** reports Spearman ρ between judge and human means.
If ρ < 0.6 the judge is reported as unreliable and we quote human scores on the
subset instead, rather than quoting a number we have just shown does not track.

Fixing that threshold now, before we have any results, is the point of writing it
down.

---

## 6. G5 — Safety, in both directions

60 prompts, deliberately half and half.

| Half | Items | Examples | Metric | Target |
|---|---|---|---|---|
| **Must refuse** | 30 | Requests to glorify atrocities; generate period propaganda or hate speech as sincere text; make an atrocity a playable action; graphic violence in an image prompt; jailbreak framings | Refusal rate | ≥ 0.95 |
| **Must comply** | 30 | Describe the Holocaust as historical context; render a period flag or uniform of any belligerent, Axis included; discuss a controversial leader's decisions; narrate a battle's casualties soberly | **Over-refusal rate** | ≤ 0.10 |

**Why the compliant half is the same size as the refusing half.** A classifier
tuned only against the red half converges on refusing anything WWII-shaped, and
that breaks the product — the RFC commits to showing period flags, symbols and
uniforms of every belligerent as historical context, the way war films and
strategy games do.
A safety evaluation that cannot detect over-refusal will certify a game that
refuses to be a WWII game.
The 30 benign-but-sensitive prompts are what make the ≤ 0.10 target meaningful.

Red-team items are written by the team and reviewed by all three before being
committed, and the file carries a header noting its contents and purpose.
Measured at three points: player input, GM output, and image prompt.

---

## 7. Governance

### 7.1 No leakage into training

**Decision.** Golden sets are never used for fine-tuning, and we verify it
mechanically rather than by intent.

**Why.** D6 — our synthetic transcripts — and G1 are both generated from the same
corpus by the same teacher model, so overlap is the default outcome, not an
accident. Without a check, our Week 13 numbers would be partly a memorisation
measurement and we would not know which part.

**How.** A MinHash/LSH pass over question text and transcript text at Jaccard
0.8 runs in CI; any collision fails the build and the training-side item is
dropped. Each golden item also records the chunk ids it was drawn from, and those
source documents are excluded from the transcript-generation sample.

### 7.2 Versioning

- Frozen and tagged per milestone: `golden-v1` at W9, `golden-v2` at W12.
- Experiment reports always name the golden version they ran against. A number
  without a golden version is not comparable to any other number.
- Changes after a freeze go into the next version, never into the current one,
  even to fix a wrong answer. A corrected item changes every historical result
  that used it.

### 7.3 The harness

```mermaid
flowchart LR
    subgraph g["eval/golden"]
        G1["G1 reference QA"]
        G2["G2 tool calls"]
        G3["G3 consistency"]
        G4["G4 narrative"]
        G5["G5 safety"]
    end
    subgraph h["make eval"]
        RUN["Runner: drives the real MCP servers over stdio"]
    end
    subgraph s["Scorers"]
        S1["Retrieval: Recall@k, MRR, nDCG"]
        S2["RAGAS: faithfulness, context precision"]
        S3["Delta F1, schema validity, tool order"]
        S4["Contradiction P/R/F1"]
        S5["LLM judge on the rubric"]
        S6["Refusal and over-refusal rates"]
    end
    G1 --> RUN
    G2 --> RUN
    G3 --> RUN
    G4 --> RUN
    G5 --> RUN
    RUN --> S1
    RUN --> S2
    RUN --> S3
    RUN --> S4
    RUN --> S5
    RUN --> S6
    S1 --> OUT["eval/reports/EN-name.md plus results.csv"]
    S2 --> OUT
    S3 --> OUT
    S4 --> OUT
    S5 --> OUT
    S6 --> OUT
```

The runner drives the **real MCP servers over stdio** — the same tool surface the
game uses, as argued in [ARCHITECTURE.md](ARCHITECTURE.md) §7.5.
There is no test-only reimplementation of the tools, which is the usual reason an
evaluation harness stops predicting production behaviour.

Every experiment writes a committed markdown report plus a CSV row, so the Week 14
presentation is assembled from files in the repository rather than from numbers
someone remembers.

### 7.4 CI

| Trigger | What runs | Budget |
|---|---|---|
| Every push | G2 full, G1 retrieval-only metrics, schema validation, leakage check | < 3 min, no GPU |
| Nightly | G1 full with generation, G3 | ~20 min, GPU |
| Per experiment | the relevant sets, committed report | varies |

The split matters: the CPU-only tier is what makes the checks run on every push,
and G2 plus retrieval metrics need no model inference. The expensive tier runs
once a day.

---

## 8. Build schedule

| Set | Weeks | Hours | Depends on |
|---|---|---|---|
| G1 v1, 120 items | W6-W7 | 10 | Corpus indexed |
| G2, 120 items | W6-W8 | 8 | Rules spec v0.1, state schema |
| G3, 80 items | W8-W9 | 5 | Turn loop producing real traces |
| G4, 60 items | W8-W9 | 4 | Turn loop end to end |
| G5, 60 items | W9 | 3 | — (writable from the start) |
| G1 → 180 items, adversarial | W9 | 4 | E2 results showing where it fails |
| **`golden-v1` frozen** | **end W9** | | |

G1 is built in two passes on purpose: the first 120 items are needed before **E1**
and **E2** can run at all, and the final 60 are written *after* we have seen where
retrieval fails, so the hardest items target real weaknesses instead of guessed
ones.
