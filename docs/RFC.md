# RFC-001: WWII DIY - an AI-run alternate-history WWII game

**Status:** Draft · **Date:** 2026-10-22 · **Authors:** Iclozan Mihnea-Serban, Rusu Mihai, Mogosan Sergiu-Ionut

## 1. Problem

"What if" questions about World War II are some of the most popular in history: what if France had held in 1940, or the US had stayed out of the war? Players want to answer them by making the decisions themselves, but existing games handle this poorly:

- **Grand-strategy games** such as Hearts of Iron IV allow alternate history, but only along paths the designers wrote in advance, and only through menus. They also take a long time to learn.
- **LLM sandbox games** such as [WorldOS](https://worldos.cc) and AI Dungeon (both have WWII worlds) accept any free-text action, but decisions rarely have lasting consequences. The game state lives only in the prompt, so after a few dozen turns the model forgets which territories you hold, which treaties you signed or which army was destroyed. The outcome depends on what the model happens to write, not on what you did.
- These sandboxes also run on large commercial models, and their visuals are mostly static cover art.

We want to build an AI agent that runs an **illustrated, text-based WWII game** in which the player leads a nation and **every decision changes how the war unfolds and how it ends**. The game is deliberately **not historically accurate**. It starts from the real situation in September 1939, and from then on history is whatever the player's choices make it. The game doesn't have to follow history, but it must stay consistent with itself.

## 2. State of the art

| Approach | Examples | Pros | Cons |
|---|---|---|---|
| LLM sandbox games | WorldOS, AI Dungeon | Free-text actions; WWII worlds already exist; worlds created by the community | State drifts and consequences are forgotten; depend on large commercial models; little consistent scene art |
| Grand-strategy games | Hearts of Iron IV | Deep, consistent mechanics; alternate history through national focus trees | Divergence limited to pre-written paths; menus only, no free-form story; long to learn |
| LLM multi-agent research | WarAgent (Hua et al., 2023) | LLM agents role-play WWII-era nations; the paper studies how changed conditions change the course of the war | Not playable by a person; uses GPT-4-class models; no visuals; expensive per run |

## 3. Proposal

The player picks a nation and plays as its leader. Each turn covers one month, starting in September 1939. The game master (GM) is an agent built on a **fine-tuned small language model (SLM)**. The model writes the story and plays the other nations, but the world state and the rules live outside it, in tools it calls through an **MCP server**. Each turn runs like this:

1. The player types any decision or picks one of the suggested ones.
2. The GM reads the world state and retrieves the reference material it needs.
3. A rules tool resolves the decision with probabilities based on the state. The GM writes the changes to the state and to an event log.
4. The other nations react, and the GM narrates the month.
5. The map updates. When the scene changes, the GM requests a new illustration.

The game ends when a nation meets a win or loss condition, or at a cutoff date. The ending is computed from the accumulated state, so it reflects every decision the player made.

Components, mapped to the project requirements:

- **Data packs (data ingestion).** The world is defined by data, not code: nations, leaders, regions, armies, technology and starting relations, plus reference articles. The pipeline validates the structured data, then splits, chunks, embeds and indexes the text. New packs add content without code changes, for example a different start date or a fictional "what if" setup.
- **RAG.** Reference material about the era (nations, leaders, equipment, geography) keeps the narration plausible, with 1940s technology, real places and believable characters. It informs the story, but it never pulls events back to what really happened.
- **Fine-tuned SLM.** A 3–8B open model (candidates: Qwen3, Llama 3.x, Gemma 3) fine-tuned with QLoRA for the GM task. It learns to narrate in the second person, play the other nations, call tools correctly, emit valid JSON state updates and write image prompts. The training data is playthrough transcripts generated with a larger open model, then filtered and corrected by the team.
- **MCP tools.** `get_state` / `update_state`, `resolve_action` (rules and dice), `advance_turn`, `search_reference`, `check_consistency`, `render_map` and `generate_scene_image`.
- **Visuals.** The map is drawn directly from the game state, so it is always correct. Scene illustrations come from an open-weight diffusion model (e.g., SDXL-Turbo or FLUX.1-schnell) with a 1940s illustration style and period-accurate uniforms, flags and insignia for each nation. Each recurring character keeps a fixed description and seed, so they look the same in every scene.
- **Hallucinations.** In this game, a hallucination is not a departure from real history but a contradiction of the game's own world. Examples are troops that appear from nowhere, a leader who returns after being killed, a past decision the story ignores, or technology that doesn't fit the era. Before the player sees a narration, `check_consistency` compares it with the state and the event log.
- **Toxicity and historical content.** WWII includes genocide and extremist ideologies, so the game follows historical content rules. Period flags, symbols and uniforms of every nation, the Axis included, appear in the text, maps and illustrations as historical context, the way war films and strategy games show them. Glorification is not allowed. The game is about military, political and diplomatic decisions, atrocities are not playable actions, and the GM refuses requests to glorify them or to produce propaganda or hate speech. Player input, GM output and image prompts all pass through a safety classifier (e.g., Llama Guard). Images avoid graphic violence and photorealistic depictions of real people.
- **Human feedback.** Players rate each turn (👍/👎 plus an optional comment). The ratings become preference data for the next fine-tuning round (DPO).
- **Evaluation.**
  - Action impact: replay the same start with different decisions and measure how far apart the final states end up.
  - Consistency: how often the narration contradicts the state over long games.
  - Rate of valid tool calls and valid JSON.
  - Narrative quality, scored by an LLM judge and by human raters.
  - Image–text alignment (CLIPScore) and latency per turn.

  The comparisons are the base SLM against the fine-tuned one, and the GM with RAG and the consistency check against the GM without them.

**What makes it different:**

1. A rules engine resolves every decision against a structured state, so decisions always have consequences and the ending reflects all of them.
2. Players make free-text decisions instead of following a list of pre-written paths.
3. The state lives in tools rather than in the prompt, so the game never forgets.
4. Small local models make the game cheap to run, keep data private, and are fine-tuned specifically for the GM task.
5. Maps are always correct, and illustrations look consistent from scene to scene.

**MVP scope:** the whole war from September 1939, played in a web interface with a chat, a map and an image panel. Eight nations are playable: the seven major powers (Germany, the United Kingdom, France, Italy, the Soviet Union, the United States and Japan) and Romania.

**Open questions for the next steps:**

- How detailed should the rules be? Which state variables do we track (territory, army strength, economy, morale, diplomatic relations)?
- Which base SLM and image model fit our GPU budget?
- How large should the fine-tuning dataset be, and how do we generate it?
- What per-turn latency is acceptable once image generation is included?
