# Viral Script Debugging Engine (MetaDebate) — Project Context

**Location:** `C:\Users\kittu\OneDrive\Desktop\My Projects\meta`
**Git:** yes. Two remotes: `origin` = `github.com/vajeedashaik/meta`, `hf` = HuggingFace Space `vajeeda/MetaDebate`. ~30 commits, last is "Add Dockerfile".
**Type:** Hackathon submission (Meta × OpenEnv Hackathon 2026, Theme 1 Multi-Agent + Theme 4 Self-Improvement). A working RL environment, a training pipeline, an HTTP server and a Next.js demo UI.
**Built across:** 13 phases (Phase 0–12) driven by prompt files in `prompts/`, then a "pre-submission fixes" pass (`fixes.md`) and a UI overhaul.
**Space title (README front-matter):** `MetaDebate`, Docker SDK, port 7860.

## 1. What this project is

A multi-agent reinforcement-learning **environment** where an LLM **Arbitrator** learns, through adversarial debate, which fix to apply to a short-form video script (Reels, Shorts, TikTok). It is not a content generator. The thing being trained is the decision policy: which flaw to fix, how, and what to protect.

The problem it targets: 95% of creators never break 10K views, existing tools are one-shot rewrites with no trade-off reasoning, and nothing learns from outcomes.

Each episode is one improvement trajectory of up to 5 steps. Every step runs four agents:

| Agent | Role | Trained? |
|---|---|---|
| **Critic** | Reads the script, produces 3–6 falsifiable `CritiqueClaim`s across 6 flaw classes | No (LLM via API) |
| **Defender** | Finds the script's `core_strength_quote`, flags Critic claims that would destroy it or regional voice | No (LLM via API) |
| **Arbitrator** | Sees the debate, picks one action as JSON | **Yes, GRPO** |
| **Rewriter** | Applies only the Arbitrator's instruction, returns new script plus unified diff | No (LLM via API) |

Flaw classes: `hook_weakness`, `pacing_issue`, `cultural_mismatch`, `cta_buried`, `coherence_break`, `retention_risk`.
Actions: `hook_rewrite`, `section_reorder`, `cultural_ref_sub`, `cta_placement`.

Two headline ideas:
- **Theme 1 (multi-agent):** the Arbitrator must weigh a Critic that attacks against a Defender that protects. Accepting every critique wrecks coherence, rejecting all stagnates.
- **Theme 4 (self-improvement):** a **Critic Escalation Engine** watches for mastered flaw classes and generates harder self-made scripts, so difficulty rises as fast as the policy improves.

## 2. Tech stack

**Backend / RL (Python 3.10+, Docker image uses 3.11-slim)**
- Pydantic v2 models, FastAPI + Uvicorn server, NumPy
- `sentence-transformers` (all-MiniLM-L6-v2) for R2/R5 embeddings, with a TF-IDF cosine fallback if it fails to load
- scikit-learn `MultiOutputRegressor(GradientBoostingRegressor)` for the retention model (R10)
- Training: `trl` (GRPO), `unsloth` (4-bit, with a transformers + peft + bitsandbytes fallback), `torch`, `accelerate`
- Base model for the Arbitrator: `unsloth/Qwen2.5-7B-Instruct-bnb-4bit`, LoRA r=16
- `rich` for terminal demos, `matplotlib` for plots, `pytest` for tests, `openenv`
- LLM backends (`agents/llm_backend.py`): `hf` (default, HuggingFace Inference API), `groq`, `qwen` (local pipeline), `anthropic`, `openai`. Every call is wrapped in a thread with a 30s timeout and raises `TimeoutError`.

**Frontend (`web-ui/`)**
- Next.js 14 App Router, React 18, TypeScript, Tailwind 3, Framer Motion, Recharts, Radix Slot, lucide-react, shadcn-style `button`/`card`
- Reads `NEXT_PUBLIC_ENGINE_API_BASE` (default `http://localhost:7860`)

**Deploy:** `Dockerfile` (python:3.11-slim, `pip install -r requirements.txt`, uvicorn on 7860), HuggingFace Space. `.gitattributes` tracks `mp4` and `joblib` with git-lfs.

## 3. Repository layout

The repo has two Python roots. Root-level `app.py`, `client/`, `scripts/`, `demo/`, `notebooks/` sit outside the package `viral_script_engine/`, which holds the real engine.

```
meta/
├── app.py                     FastAPI wrapper (reset / step / state / health), CORS *, in-memory session dict
├── openenv.yaml               OpenEnv manifest, tools renamed env_reset/env_step/env_state/env_health
├── Dockerfile, requirements.txt, setup.py
├── README.md (HF front-matter), BLOG.md (mini-blog), fixes.md (big pre-submission master prompt)
├── claude.md / CLAUDE.md      Phase-gated workflow rules (see section 12)
├── client/env_client.py       HTTP-only ViralScriptEnvClient, never imports server code
├── demo/run_demo.py           5-act rich terminal demo (--script S03 --compare | --interactive)
├── scripts/                   deploy.sh, dev.sh, test.sh, run_dummy_episode.py, run_platform_comparison.py,
│                              run_longitudinal_demo.py, smoke_test_remote.py, submission_check.py,
│                              train_retention_model.py, inspect_generations.py, replace_training_plot.py
├── notebooks/training_colab.ipynb        original Colab notebook
├── viral_script_engine_colab.ipynb       newer 10-section notebook (28 cells)
├── prompts/                   phase-0…12 prompts, phase-index(2), hf.md, landing-page.md, ui-design.md, update-data.md
├── docs/, session/, .claude/skills/      workflow scaffolding (mostly templates, see section 12)
├── data/creator_histories/S01.json       persisted creator memory (also a copy inside the package)
├── logs/                      difficulty_tracker.json, errors.log, test.log
├── viral-script-graphs/       1.png 2.png 3.png
├── web-ui/                    Next.js app (section 9)
└── viral_script_engine/       the engine (below)
```

Inside `viral_script_engine/`:

| Folder | Contents |
|---|---|
| `agents/` | `critic.py`, `defender.py`, `rewriter.py`, `baseline_arbitrator.py`, `llm_backend.py`, `moderation_agent.py`, `originality_agent.py`, `reasoning_parser.py` |
| `environment/` | `env.py` (`ViralScriptEnv`), `ab_env.py` (`ABScriptEnv`), `actions.py`, `observations.py`, `episode_state.py`, `trajectory.py` |
| `rewards/` | `r1`…`r10`, `reward_aggregator.py`, `process_reward.py`, `process_verifier.py`, `contrastive_reward.py`, `base.py` |
| `escalation/` | `difficulty_tracker.py`, `critic_escalation_engine.py` |
| `training/` | `train_grpo.py`, `rollout_function.py`, `eval_trained_model.py`, `reward_curves.py` |
| `personas/` | `creator_profile.py`, `profile_generator.py`, `persona_kb.py` |
| `platforms/` | `platform_spec.py`, `platform_kb.json` |
| `memory/` | `creator_history.py`, `memory_compressor.py`, `history_store.py` |
| `retention/` | `feature_extractor.py`, `curve_predictor.py`, `curve_scorer.py`, `model.joblib`, `training_data/` |
| `evaluation/` | `critic_evaluator.py` (Phase 0 gate) |
| `data/` | `test_scripts/scripts.json`, `cultural_kb.json`, `persona_advice_kb.json`, `shadowban_triggers.json`, `viral_templates.json`, `curriculum/*.jsonl`, `golden_fixtures/` |
| `scripts/` | `run_critic_gate.py`, `run_dummy_episode.py`, `run_baseline.py`, `run_escalation_demo.py`, `run_ab_episode.py` |
| `tests/` | 13 test files, ~180 test functions |
| `logs/` | baseline and training result JSON, plots, per-episode logs, escalation progression |

## 4. The environment (`ViralScriptEnv`)

Gym-style: `reset(seed, options) -> (obs, info)`, `step(action, raw_output=None) -> (obs, reward, terminated, truncated, info)`, `state() -> dict`. Also `reset_from_config(episode_config)` for curriculum JSONL rows.

**Difficulty tiers** map to script IDs: easy = S01–S04, medium = S05–S07, hard = S08–S10, `self_generated` = escalation output. Unknown tier falls back to all scripts.

**Test scripts (10, in `data/test_scripts/scripts.json`)**

| IDs | Region | Platform | Niches |
|---|---|---|---|
| S01–S03 | Mumbai Gen Z | Reels/Reels/Shorts | personal finance, fashion, tech |
| S04–S06 | Tier-2 Hindi belt | Reels/Reels/Shorts | agriculture, small business, local culture |
| S07–S08 | Pan-India English | Shorts/Reels | startup advice, productivity |
| S09–S10 | Hinglish | Reels/Shorts | finance, tech |

Each has `known_flaws` used by the Phase 0 critic gate.

**Action** (`ArbitratorAction`): `action_type`, `target_section` (`hook|body|cta|full`), `instruction`, `critique_claim_id` (e.g. `C2`), `reasoning`.

**Observation** carries: current and original script, region, platform, niche, step/max_steps, full `debate_history` (claims, defender response, action, diff, per-step rewards, moderation/originality output, reasoning chain), `reward_components`, difficulty, episode_id, moderation/originality flags, `creator_profile`, `creator_history`, `history_context` (prompt string).

**One `step()`:**
1. Critic critiques the current script.
2. Defender defends it.
3. If `raw_output` was supplied, parse the Arbitrator's reasoning chain and compute the process reward (before the rewrite).
4. Rewriter applies the action.
5. Score R1–R10 on the new script.
6. Aggregator applies safety hard-zero, catastrophic-drop and repetition rules.
7. Append a `DebateRound`.
8. Terminate if `step_num >= max_steps` (5) or `total >= 0.9`.
9. On termination: record the episode in the `DifficultyTracker`, compress the episode into creator memory and persist it.

**Timeouts (anti-hang):** any Critic/Defender/Rewriter `TimeoutError`, or a step exceeding 120s wall clock, returns `reward=0.0, truncated=True` with `info["timeout"]`, `timeout_agent`, `timeout_count`.

## 5. Reward system

Ten components plus a process reward. Total = weighted mean over the components that are non-`None` (weights renormalised), minus penalty, clipped to [0, 1].

| # | Name | What it measures | How |
|---|---|---|---|
| R1 | Hook Strength | Does the first ~3 sentences (max 50 words) grab attention | 5 rule checks (promise, curiosity, specificity, front-load, anti-filler) plus a platform-aware hook-length fit; dead openers like "hey guys" penalised |
| R2 | Coherence | Rewrite still matches original intent | Embedding cosine (MiniLM), TF-IDF fallback, platform-aware, cached by hash |
| R3 | Cultural Alignment | Region voice preserved | `(valid_refs + correct_idioms − invalid − anachronistic) / (len(valid_refs)+len(idioms))` against `cultural_kb.json`; unknown region returns 0.5 |
| R4 | Debate Resolution | Did the targeted claim actually get resolved | Re-runs Critic on the new script; 1.0 if no matching claim of the same class (within 5s timestamp window, or any if timestamp N/A), lower if it persists or new claims appear |
| R5 | Defender Preservation | Was the Defender's core strength kept | Sentence-level similarity of `core_strength_quote` against the rewrite |
| R6 | Safety | Shadowban / policy triggers | From `ModerationAgent` (regex/keyword, no LLM): 0 flags=1.0, low=0.8, medium=0.5, **high=0.0 hard zero** |
| R7 | Originality | Not a template clone | From `OriginalityAgent` (fuzzy match vs `viral_templates.json`, threshold 0.75): ≥0.8→1.0, 0.6–0.8→score, 0.4–0.6→0.3, <0.4→0.0 |
| R8 | Persona Fit | Action suits creator tier | `persona_advice_kb.json`: priority=1.0, neutral=0.5, deprioritised=0.2, forbidden=0.0, +0.1 if it targets a known recurring weakness |
| R9 | Platform Pacing | Sentence length, section ratios, CTA position vs platform spec | Rule-based, zero LLM calls |
| R10 | Retention Curve | Predicted watch-time curve improved | GBR model predicts retention at t=[0,3,6,10,15,20,25,30,45,60]s; `0.50·overall + 0.35·targeted − 0.15·regression`. Skipped (None) if `model.joblib` isn't trained |

**Weights** (`observations.py`): R1 .12, R2 .10, R3 .10, R4 .10, R5 .08, R6 .07, R7 .07, R8 .08, R9 .08, R10 .10.

**Process reward (Phase 7):** the Arbitrator outputs a 3-part reasoning chain (`priority_assessment`, `conflict_check`, `defender_consideration`) before its action. `ProcessVerifier` checks each against ground truth. Weights 0.40 / 0.35 / 0.25. Contribution capped at 0.15 of step reward and added before anti-gaming, so outcome still dominates.

**R10 targeted timepoints:** hook_rewrite→0/3/6s, section_reorder→10/15/20s, cultural_ref_sub→15–30s, cta_placement→45/60s.

**Contrastive reward (A/B, Phase 10):** `final = clip(max(cumA, cumB) + tanh(3·Δ)·0.2, 0, 1)`.

## 6. Anti-gaming (`rewards/reward_aggregator.py`)

Evaluated in this order:
1. **R6 hard zero:** any high-severity moderation flag sets total to 0.0 and penalty 1.0.
2. **Catastrophic drop:** if any component falls more than **0.25** below its episode-start value, total is 0.0 (logged with the component name).
3. **Action repetition:** same action type 3 times in a row costs **−0.15**.

Each step emits an `AntiGamingLog` (rule, component, pre/post totals). Note: the README describes rule 1 as "−0.3 penalty when total drops >0.3" but the code zeroes the step at a 0.25 per-component drop. The code is the source of truth.

## 7. Self-improvement loop (Theme 4)

- `DifficultyTracker` (persists to `logs/difficulty_tracker.json`): per critique class, tracks total/resolved episodes, consecutive resolutions, last-10 R4 scores. A class is **mastered** after `mastery_threshold=3` consecutive resolutions (R4 ≥ 0.8).
- `CriticEscalationEngine`: for a mastered class, asks an LLM to write a harder script with a conflicting flaw, a `why_its_harder`, `optimal_action_order` and a `trap_action` (the obvious fix that lowers total reward). Returns an `EscalatedChallenge` dataclass, difficulty `self_generated`.
- In `reset()`: if any class is mastered, use an escalated challenge instead of a normal script. `info["escalation_used"]` is set.
- Dominant class of an episode = most common `critique_class` among the first step's claims.

Progression: `easy → medium → hard → self_generated`.

## 8. Extra modules (Phases 6–12)

| Phase | Module | Summary |
|---|---|---|
| 6 | `ModerationAgent`, `OriginalityAgent` | Rule-based; 6 moderation categories (hate speech, misleading health, copyright bait, engagement bait, spam, policy violations); 4 template types (hooks, structures, CTAs, transitions) |
| 7 | `ReasoningParser`, `ProcessVerifier`, `ProcessReward` | Structured reasoning chain, verified before rewrite |
| 8 | `CreatorProfile`, `ProfileGenerator`, R8 | Tiers: beginner (<1k), growing (1–10k), established (10–100k), verified (100k+). Profile tier is picked from episode difficulty. Includes posting frequency, engagement, retention rate, weak/strong points, voice descriptors |
| 9 | `PlatformRegistry`, R9 | Reels, Shorts, Feed, TikTok specs (hook window 3/2/5/2s, optimal words 120/80/200/100, CTA position, pacing norm). R1 and R2 became platform-aware |
| 10 | `ABScriptEnv`, `Trajectory`, `ContrastiveReward` | Two parallel envs from the same script. Trajectory A forced to act on the Critic's top claim first, B on the Defender's concern first, later steps free. Arbitrator sees both |
| 11 | `CreatorHistoryBuffer`, `MemoryCompressor`, `HistoryStore` | Rule-based (no LLM) compression of each episode into `EpisodeMemory`; sliding window of last 5; recurring weak points (≥3 of 5), strengths (≥0.7 in ≥4 of 5), most effective action, voice stability, trend (improving/plateauing/declining). Persisted per creator, injected into the prompt as `history_context` |
| 12 | `FeatureExtractor`, `RetentionCurvePredictor`, `RetentionCurveScorer`, R10 | 14 structural features plus platform one-hot, zero LLM calls; 150 rule-based synthetic samples; reported avg MAE 0.031; monotonic-curve enforcement; original curve cached per episode |

## 9. HTTP API, client and web UI

**Server (`app.py`)** keeps `_envs: dict` keyed by `session_id`, in memory only.

```
POST /reset   {session_id, difficulty="easy", options={}}  -> {observation, info}
POST /step    {session_id, action}                          -> {observation, reward, terminated, truncated, info}
GET  /state/{session_id}
GET  /health  -> {status:"ok", environment:"ViralScriptDebugEngine", version:"1.0.0"}
```
Unknown session returns 404. Each `/reset` builds a new `ViralScriptEnv`. `openenv.yaml` tool names are `env_reset|env_step|env_state|env_health` (hackathon rules forbid the reserved names `reset/step/state/close` in the manifest). The HTTP routes keep the plain names.

**Client (`client/env_client.py`):** `ViralScriptEnvClient(base_url)` with `reset(difficulty)`, `step(action)`, `new_session()`. HTTP only.

**Web UI (`web-ui/`, `npm run dev` → :3000, root redirects to `/landing`)**

| Route | Purpose |
|---|---|
| `/landing` | Dark hero page with a full-bleed looping background video (Google Drive preview embed) |
| `/dashboard` | System overview |
| `/episode` | Step-through episode viewer: script panel, Critic/Defender panels, Arbitrator reasoning, reward bars (R1–R10), diff, controls |
| `/ab` | A/B battle (critic-first vs defender-first) with counterfactual rewind |
| `/retention` | Retention curve chart plus explainer |
| `/memory` | Creator memory view |
| `/learning`, `/learning-playback` | Learning graph and AI learning timeline |

Components include `PipelineViz`, `PhaseTimeline`, `RewardBars`, `RewardDeltaBadge`, `JudgeExplanation` (Judge Mode), `LearningGraph`, `LearningTimeline`, `RetentionChart`, `ABBattle`, `CreatorMemory`, `BackgroundOrbs`. `lib/mock-data.ts` holds a hard-coded demo episode (Bengaluru productivity script) that the pages use as a fallback; `lib/api.ts` is the typed fetch layer for the live server.

Design brief (`prompts/ui-design.md`): light, Meta-internal-tool feel, #1877F2 blue accent, rounded-2xl cards, heavy Framer Motion. Message: "this AI is learning how to think, not just generate text."

## 10. Training pipeline

- **Algorithm:** GRPO via TRL. Reward function `rollout_fn(completions, prompts, **kw) -> List[float]` in `training/rollout_function.py`. It parses episode config from the prompt header, resets the **live** environment, generates an action (JSON with reasoning fields), steps up to 5 times, returns episode reward. Falls back to a default `hook_rewrite` action if the model's JSON is invalid; JSON extraction uses a balanced-brace walker.
- **Arbitrator system prompt** requires JSON with `priority_assessment`, `conflict_check`, `defender_consideration`, then the action fields.
- **Config:** lr 5e-6, max_grad_norm 0.1, warmup 10, batch 4, 8 generations, grad accum 4, save every 50 steps, 256 new tokens, temp 0.8. `GRPOConfig` is built by introspecting valid params because TRL removed several args in 0.15+. W&B only if `WANDB_API_KEY` is set.
- **Curriculum:** `data/curriculum/{easy,medium,hard}_tier.jsonl` (12 / 13 / 9 rows, built by `build_curriculum.py` + `generate_synthetic_scripts.py`).
- **Commands:**
  - `python training/train_grpo.py --dry-run` (mock model, live env, no GPU; writes `logs/dry_run_log.json`)
  - `python training/train_grpo.py --tier easy,medium --steps 200`
  - `python training/eval_trained_model.py --model outputs/checkpoints/final_model` (same 20-episode schedule as baseline: 8 easy, 8 medium, 4 hard)
  - `python scripts/replace_training_plot.py --training-log logs/training_results.json`
- **A/B rollout:** `build_ab_rollout_fn` for the contrastive environment.

## 11. Results, and how much to trust them

Claimed (README and BLOG): Qwen2.5-7B, 4-bit, 200 GRPO steps on a T4, ~90 min. Baseline → trained total 0.50 → 0.76 (README) or 0.51 → 0.78 (BLOG). Retention "3×" claim: viewers drop at 6s (57% remain) before vs 20s (70% remain) after. README percentages are relative gains, BLOG percentages are absolute point gains for the same numbers (e.g. hook 0.42→0.71).

**Caveat, important:** the repo's own files show these are not from a completed real run.
- `logs/training_results_synthetic.json` and the README note say the plot is a placeholder: "Plot will be replaced with real GRPO training curves after onsite compute run." `fixes.md` (Fix 4) adds a synthetic-watermark path and a `replace_training_plot.py` script for swapping in the real one.
- `outputs/checkpoints/` is effectively empty.
- `session/context.md` lists real GRPO training as **not done**, blocked on GPU.
- `prompts/update-data.md` supplies the "real numbers" by hand for the website.
- The README "penalties fired" log excerpts (Episodes 7, 14, 19) are illustrative; the actual rule in code differs (section 6).

Treat the table as a target or demo figure until a real `logs/training_results.json` exists. Also, the README links the Space at `vajeeda/viral-script-debugging-engine` while the git remote is `vajeeda/MetaDebate`; confirm which is live.

Baseline logs (`baseline_results.json`, `baseline_reward_curves.png`) come from `scripts/run_baseline.py` with `BaselineArbitratorAgent`.

## 12. Development workflow and status

**Phase log (all dated 2026-04-26, all COMPLETE with gate PASS):**
P3 curriculum + GRPO pipeline, P4 escalation engine, P5 HF deploy + demo + README (`submission_check` 10/10), P6 safety/originality R6/R7, P7 process reward, P8 personas R8, P9 platform pacing R9, P10 A/B contrastive, P11 creator memory, P12 retention R10 (model MAE 0.031). After that: "mvp version 2", "final mvp created", mp4 swaps, README, Dockerfile.

**Test status per session records:** phase tests 6 (P4) → 14 (P12), ~180 test functions total. Gate check: `python scripts/run_dummy_episode.py --difficulty easy --steps 3 --verbose`. Run pytest from the **repo root**, not from `viral_script_engine/`.

**Workflow rules (`claude.md`)** — phase-gated: on session start read `docs/index.md`, `session/phase-log.md`, `session/context.md`, then the current `prompts/phase-X.md`; don't advance phases without user confirmation; write tests per feature; every error logs file, function, reason, fix hint; on session end give a one-line commit message, don't push, update `session/summary.md`, `phase-log.md`, `docs/progress.md`, `docs/learnings.md`. Note that `docs/architecture.md`, `prd.md`, `testing.md`, `deployment.md` and `.claude/skills/supabase.md` are **unfilled templates** copied from another (Next.js + Supabase + Jest) project. They do not describe this repo. The Supabase rules in `claude.md` are irrelevant here.

**Submission pre-flight (`fixes.md`):** Part A compliance fixes (reserved tool names, `smoke_test_remote.py`, client/server separation, synthetic-plot watermark, timeouts, generation inspection tool, stricter `submission_check.py`, plot axis labels), Part B web-UI demo features (AI Learning Timeline, Counterfactual Rewind, Retention Explainer, Judge Mode), Part C Colab notebook upgrade (intro, "how this works", quick demo, before/after, client usage, takeaways). `submission_check.py` separates BLOCKING failures from warnings and checks README sections (The Problem, What We Built, Reward Functions, Anti-Gaming, Results).

## 13. How to run

```bash
# server (from repo root)
python -m uvicorn app:app --host 0.0.0.0 --port 7860
python scripts/smoke_test_remote.py --url http://localhost:7860

# episode / demo / checks
python scripts/run_dummy_episode.py --difficulty easy --steps 3 --verbose
python demo/run_demo.py --script S03 --compare
python scripts/submission_check.py
python scripts/train_retention_model.py        # required once for R10 (model.joblib is in the repo)
pytest                                         # from repo root

# UI
cd web-ui && npm install && npm run dev        # http://localhost:3000
```
Env vars: `HF_TOKEN` (default backend), `GROQ_API_KEY` if `backend="groq"`, `WANDB_API_KEY` optional. There is a `viral_script_engine/.env` (gitignored).

## 14. Known issues and gotchas

- **Windows:** the `pyarrow` DLL is blocked by App Control, so `sklearn` / `sentence_transformers` / TRL imports fail locally. Training and R2/R5/R10 must run on Linux/Colab. Dry-run and tests monkeypatch R2/R5 before importing the env. In env tests, always mock `CoherenceReward.score` and `DefenderPreservationReward.score`.
- Agents hard-code `backend="hf"` in `ViralScriptEnv.__init__`, so any live episode needs HF inference access. `RewriterAgent` still defaults to `backend="anthropic"` and `claude-haiku-4-5-20251001` in its own signature, but env overrides it.
- The Defender LLM sometimes returns multiple JSON objects ("Extra data" decode error). Greedy `\{.*\}` regex over-captures; use the balanced-brace walker.
- Initialise `terminated`/`truncated` before try/except blocks in loops.
- `GRPOConfig` test: use try/except import, not `pytest.importorskip` (trl is installed but fails to import on Windows).
- HF `/reset` creates a fresh env (and reloads all reward models) per call, and sessions live in memory only, so a Space restart drops them.
- Curriculum JSONL contains mojibake (`â€”`) from a UTF-8/cp1252 mix-up in script text.
- **Security:** the `hf` git remote URL has an HF access token embedded (`https://vajeeda:hf_…@huggingface.co/...`), stored in `.git/config`. Rotate that token and switch the remote to a credential helper before sharing the repo or the folder. (Deliberately not reproduced here.)
- Real GRPO training results are still pending; do not present the results table as measured.
- `README.md` "Mini-blog" and "Video Demo" links are `#` placeholders.

## 15. Where to look for what

| Question | File |
|---|---|
| Step logic and termination | `viral_script_engine/environment/env.py` |
| Reward weights and total | `environment/observations.py` |
| Anti-gaming rules | `rewards/reward_aggregator.py` |
| Agent prompts | `agents/critic.py`, `defender.py`, `rewriter.py`, `training/rollout_function.py` (Arbitrator) |
| Escalation | `escalation/difficulty_tracker.py`, `critic_escalation_engine.py` |
| A/B mode | `environment/ab_env.py`, `trajectory.py`, `rewards/contrastive_reward.py` |
| Platform and persona rules | `platforms/platform_kb.json`, `data/persona_advice_kb.json` |
| Cultural regions | `data/cultural_kb.json` (mumbai_gen_z, tier2_hindi_belt, pan_india_english, hinglish) |
| Phase history | `session/phase-log.md`, `session/summary.md`, `session/context.md` |
| Original build prompts | `prompts/phase-*.md` |
| Submission requirements | `fixes.md`, `scripts/submission_check.py` |
