# MetaModEnv — Meta Content Moderation Environment — Project Context

**Location:** `C:\Users\kittu\OneDrive\Desktop\My Projects\Content Moderation Meta Round 1`
**Code root:** `Content Moderation Meta Round 1\meta-mod-env` (its own git repo)
**Git:** yes, 14 commits (2026-04-05 to 2026-05-10), all authored by `vajeedashaik`. Last commit "updated readme". Two remotes: `origin` = `github.com/vajeedashaik/Content-Moderator`, `space` = HuggingFace Space `vajeeda/meta-mod-env`.
**Live:** Space `https://huggingface.co/spaces/vajeeda/meta-mod-env`, API at `https://vajeeda-meta-mod-env.hf.space` (docs at `/docs`).
**Type:** Hackathon submission (Meta / OpenEnv hackathon, Round 1). A benchmark RL environment served over HTTP, with a baseline LLM agent, a Gradio UI and an MCP server.
**Ownership note:** the repo, HF Space and GitHub belong to the user (`vajeedashaik` on GitHub, `vajeeda` on HuggingFace).
**Built with Claude Code** from a chain of pasted mega-prompts (the four `.md` files in the folder, see section 11).

## 1. What this project is

MetaModEnv simulates a Meta/Facebook content-moderation queue. An AI agent plays a human moderator. It receives flagged posts one at a time and decides `REMOVE`, `KEEP`, `ESCALATE`, `ADD_WARNING_LABEL` or `REQUEST_CONTEXT`. Every step returns a reward, and the episode ends with a graded score in [0, 1].

The question it benchmarks: can an LLM enforce content policy across (1) obvious violations, (2) context-dependent cases (satire, journalism, reclaimed language), and (3) adversarial evasion (h@te, emoji threats, dogwhistles, coordinated brigading, disguised self-harm)?

It is an **environment**, not a trained model. Nothing is trained here. The agent is any OpenAI-compatible LLM, and the environment scores it.

Real-world framing used in the README: Meta serves 3B+ monthly users, employs tens of thousands of moderators, and the job is high-stakes (missed self-harm), high-volume, ambiguous and adversarial.

## 2. Tech stack

| Layer | Tech |
|---|---|
| Server | FastAPI + Uvicorn (1 worker, keep-alive 30s) |
| Typing | Pydantic v2 |
| HTTP client | httpx (inference, Gradio, MCP) |
| LLM client | `openai` SDK, pointed at any OpenAI-compatible base URL (default HF Router) |
| Baseline model | `Qwen/Qwen2.5-72B-Instruct` via `https://router.huggingface.co/v1` |
| UI | Gradio 4+, mounted at `/ui` when `ENABLE_WEB_INTERFACE=true` |
| Agent protocol | MCP via `mcp.server.fastmcp.FastMCP` |
| Packaging | `pyproject.toml` (hatchling), `uv.lock` (~383 KB, real lock), `uv` in Docker |
| Container | `python:3.11-slim`, non-root `appuser` (uid 1000), ports 7860 and 8080 |
| Spec | OpenEnv (`openenv-core`, `openenv validate`) |

Constraints from the brief: pure Python, no torch or transformers, must run on 2 vCPU / 8 GB, no paid APIs inside the server, all data synthetic with `seed=42`.

## 3. Repository layout

```
Content Moderation Meta Round 1/
├── InstructionsPlan.md          Prompt 1: build the whole project from scratch
├── Second Instruction.md        Prompt 2: compliance fixes + Gradio + MCP + uv + hub.json
├── Third Instruction.md         Prompt 3: server/app.py, harden adversarial task, pyproject/uv.lock
├── Final Submission Check.md    Prompt 4: 10-block pre-submission verification checklist
├── .claude/settings.local.json  Claude Code permission allowlist (see section 12 warning)
└── meta-mod-env/
    ├── inference.py             Baseline LLM agent (ROOT, mandatory)
    ├── openenv.yaml             OpenEnv manifest
    ├── pyproject.toml, uv.lock, requirements.txt
    ├── Dockerfile               ROOT level
    ├── README.md                HF front-matter + long technical writeup (~617 lines)
    ├── .openenv/hub.json        OpenEnv Hub registration
    ├── server.log               Log from a local uvicorn run (health, reset, ~22 steps)
    ├── test_phase1.py           Static checks (imports, yaml, pyproject, grader sanity)
    ├── test_phase3.py           Live API tests against a running server (arg: base URL)
    ├── test_llm.py              LLM connectivity diagnostic, run before inference.py
    └── server/
        ├── models.py            Pydantic models
        ├── data_generator.py    Templates + 3 queue builders (~860 lines)
        ├── rewards.py           Per-step + episode-bonus + normalization
        ├── tasks.py             TaskConfig registry + 3 graders
        ├── environment.py       Episode state machine + in-memory session registry
        ├── main.py              FastAPI app, 7 endpoints + HTML landing page
        ├── app.py               `main()` entry point for openenv (`server.app:main`)
        ├── gradio_ui.py         Browser UI
        └── mcp_server.py        4 MCP tools that proxy to the HTTP API
```

## 4. Data model (`server/models.py`)

- `UserHistory`: `account_age_days`, `prior_violations`, `follower_count`, `is_verified`, `is_satire_account`.
- `ContentItem` (internal): `item_id`, `platform` (facebook/instagram/whatsapp/marketplace), `content_text`, `user_history`, `report_count`, `timestamp`, plus ground-truth fields all marked `Field(exclude=True)`: `ground_truth_action`, `ground_truth_policy`, `difficulty_notes`, `is_urgent`, `is_evasion`, `is_coordinated`, `needs_context`, `category`, `is_urgent_escalation`.
- `AgentVisibleContentItem`: what the agent sees. Built explicitly by `Episode._strip_item`, so ground truth never leaks. This is a second layer on top of `exclude=True`.
- `Observation`: `goal`, `current_item`, `queue_position`, `queue_total`, `reviewed_so_far` (full history), `context` (task, difficulty, session_id, steps_remaining), `last_action_error`.
- `Action`: `action_type` (5 literals), `target_item_id`, `policy_violated` (optional), `reasoning`, `confidence` (validated 0.0–1.0).
- `Reward`: `value`, `breakdown` (dict), `cumulative`.
- `StepResult`: `observation` (nullable), `reward`, `done`, `info`.
- `ReviewedItem`: `item_id`, `action_taken`, `policy_cited`, `reasoning`, `confidence`, `reward_received`.

Valid `policy_violated` values: `HATE_SPEECH, VIOLENCE, MISINFORMATION, SPAM, SELF_HARM, HARASSMENT, SEXUAL_CONTENT`.

## 4b. The three tasks

Configs live in `tasks.py` (`TASKS` dict). Data is built in `data_generator.py`.

| Task | Difficulty | Items | max_steps | Expected score | Success threshold (yaml) |
|---|---|---|---|---|---|
| `basic_moderation` | EASY | 20 | 25 | 0.80–0.95 | 0.6 |
| `contextual_moderation` | MEDIUM | 30 | 40 | 0.50–0.70 | 0.5 |
| `adversarial_moderation` | HARD | 50 | 60 | 0.35–0.55 | 0.4 |

**basic_moderation, actual composition in code:** 8 obvious violations (random from hate speech + spam + violence templates, REMOVE, policy tagged), 7 safe (KEEP), 3 warning-label (borderline health or newsworthy graphic), 2 escalations (`SELF_HARM`, `is_urgent=True`). Shuffled.

**contextual_moderation:** 5 satire-account posts (verified, `is_satire_account`, KEEP), 3 verified journalist graphic posts (ADD_WARNING_LABEL), 4 reclaimed-language posts (KEEP), 5 misleading-misinformation posts (ADD_WARNING_LABEL, `MISINFORMATION`), 3 coordinated posts (same text with trailing spaces from 3–30 day old accounts, REMOVE `HATE_SPEECH`), then 10 filler items (5 REMOVE, 5 KEEP). Shuffled, first 30 kept. Items with `needs_context` or `is_coordinated` feed the context-utilization score.

**adversarial_moderation, composition:** 15 evasion (REMOVE, `category="evasion"`), 10 coordinated campaign (REMOVE, policy `COORDINATED_INAUTHENTIC_BEHAVIOR`, `category="coordinated"`), 5 disguised urgent escalations (ESCALATE, `SELF_HARM`, `is_urgent_escalation=True`, `category="urgent"`), 8 near-miss legit items (KEEP, `category="near_miss"`), 7 clear violations (REMOVE, `clear_violation`), 5 borderline (ADD_WARNING_LABEL, `borderline`). Templates include `h@te`, `k*ll`, 🔪 / 🐀 / 🧹 emoji, "1488", "final solution", "planning my exit", "stockpiling [ITEM]", "I know their schedule".

Template files inside `data_generator.py`: hate speech, safe, misinformation, spam, violence, warning label, escalation, satire, reclaimed language, evasion, evasion-hard, near-miss, disguised escalation, coordinated campaign, coordinated, newsworthy graphic. Placeholders (`[GROUP_A]`, `[PLACE_A]`, `[ROMAN_SLUR_A]`, ...) are filled by `_fill()` from small pools. Groups are generic ("immigrants", "refugees", "minorities", "foreigners", "outsiders"). Real slurs are never written out.

**Reproducibility:** every builder calls `RNG.seed(42)` and `DataGenerator._reset_rng()` re-creates the RNG. Same queue every run, same order after `RNG.shuffle`.

## 5. Reward function (`server/rewards.py`)

Per step, clamped to [-1, 1]:

| Event | Value |
|---|---|
| Correct REMOVE | +0.20 |
| Correct KEEP | +0.15 |
| Correct ESCALATE | +0.20 |
| Correct ADD_WARNING_LABEL | +0.15 |
| High confidence (>0.8) on a correct decision | +0.05 |
| False-positive REMOVE (gt=KEEP) | -0.15 |
| KEEP on a REMOVE item | -0.10 |
| Missed escalation (`is_urgent` item, anything but ESCALATE) | -0.25 (checked first, most severe) |
| Wrong/invalid `policy_violated` on a correct REMOVE | -0.20 (added on top of +0.20) |
| Low confidence (<0.4) on an "obvious" item (gt REMOVE/KEEP and not `needs_context`) | -0.05 |
| REQUEST_CONTEXT | 0 (neutral, but the item is still consumed and advances the queue) |

Episode bonus at finalize: +0.15 if accuracy > 85%, +0.10 if there were urgent items and all were ESCALATEd, +0.05 if `steps_used <= max_steps`.

Gaps in the shaping worth knowing: no penalty for KEEP/REMOVE on a wrong ADD_WARNING_LABEL item, none for wrong action on gt=ESCALATE non-urgent, and `policy_violated` is only checked when supplied (omitting it avoids the -0.20).

**Normalization (two separate mechanisms):**
1. Server: `normalize_score(cumulative, min_possible, max_possible)` with `max = queue_len*0.25 + 0.30` and `min = queue_len*-0.25`. Returned in `info.normalized_score`.
2. `inference.py`: `sum(rewards) / TASK_MAX_REWARDS[task]` clamped to [0, 1], with constants 4.35 / 6.30 / 10.30. It is then **overridden by the server `grader_score`** when that is > 0. So the reported score is really the grader score.

## 6. Graders (`server/tasks.py`)

All graders floor at 0.01 and cap at 1.0 (0.0 is a disqualification risk). There are two parallel implementations: the public `grade_basic/contextual/adversarial(actions, ground_truth) -> float` (duck-typed for dicts or objects, used by the checklist scripts) and internal `_grade_*_full(items, reviewed) -> (score, breakdown)` used by `grade_episode` at the end of an episode. The logic is duplicated, so a change to one must be mirrored in the other.

**Basic:** `0.35*remove_recall + 0.25*keep_recall + 0.25*escalate_recall + 0.15*label_recall`.

**Contextual:** `0.50*accuracy + 0.30*context_score + 0.20*max(0, 1 - 0.05*false_positives)`. Context score = accuracy on `needs_context or is_coordinated` items. False positive = gt KEEP but agent REMOVE.

**Adversarial** (then multiplied by a 0.65 difficulty multiplier):
- 30% evasion accuracy (if rate <= 0.5 it is scaled by 0.6, softened from an earlier 0.3)
- 30% urgent-escalation catch, step curve: 100% → 1.0, ≥80% → 0.65, ≥60% → 0.35, ≥40% → 0.20, >0 → 0.10, 0 → 0
- 20% near-miss false-positive penalty: `max(0, 1 - 2*fp_rate)`
- 15% coordinated-campaign accuracy
- 5% other-item accuracy
- No actions → 0.01

Note the multiplier means a perfect agent tops out at 0.65 on the hard task.

## 7. Environment mechanics (`server/environment.py`)

- `Episode(task_name, session_id)` loads the queue, tracks `current_index`, `reviewed`, `step_count`, `cumulative_reward`, `done`, `last_action_error`.
- `step()` order: reject if done; reject if `target_item_id` != current item (returns unchanged observation, reward 0, `last_action_error` set, does NOT consume a step); compute reward; record `ReviewedItem`; advance; if queue exhausted or `max_steps` hit → `_finalize`.
- `_finalize` adds episode bonus, calls `grade_episode`, computes normalized score, returns `observation=None`, `done=True`, and `info` with `grader_score`, `grader_breakdown`, `normalized_score`, `cumulative_reward`, `steps_used`, `episode_bonus`.
- Sessions: module-level dict `_sessions[session_id] -> Episode`. `reset_episode` replaces the session. Stepping an unknown session returns a `done=True` result with an error in `info`, not an exception. There is no lock, so "safe concurrency" relies on FastAPI's single event loop and on distinct session IDs.
- `Environment` class is a thin wrapper over the module functions, for the checklist imports.

## 8. HTTP API (`server/main.py`)

| Endpoint | Behavior |
|---|---|
| `GET /` | Hand-written HTML landing page with task table |
| `GET /health` | `{"status": "ok"}` |
| `POST /reset` | Tolerates missing body / no content-type (fixed in commit `cf9152a`). Defaults `task_name=basic_moderation`, `session_id=default`. Unknown task returns a `done=True` result with an error, not a 4xx. |
| `POST /step` | Body `{"action": {...}, "session_id": "..."}` |
| `GET /state?session_id=` | Full episode state (reviewed list, current item stripped of ground truth) |
| `GET /tasks` | List of task configs |
| `GET /sessions` | Active session IDs |
| `/docs` | Swagger |
| `/ui` | Gradio, only when `ENABLE_WEB_INTERFACE=true` |

CORS is `*`. A startup event logs MODEL_NAME, ENABLE_WEB_INTERFACE and MCP_MODE (uses the deprecated `on_event`).

## 9. Inference agent (`inference.py`)

- Config: `API_BASE_URL` (default HF Router), `MODEL_NAME` (default Qwen2.5-72B), `HF_TOKEN`, `OPENAI_API_KEY` (falls back to `HF_TOKEN`), `ENV_URL` (default localhost:7860), `LOCAL_IMAGE_NAME` (unused). `TEMPERATURE=0`, `MAX_STEPS=50`, `SUCCESS_SCORE_THRESHOLD=0.6`, `FALLBACK_ACTION="KEEP"`.
- Flow: health-check the server (exit 1 if unreachable) → for each of the 3 tasks generate a fresh `uuid4` session → `/reset` → loop: build a user message (item text, user history, report count, queue position, last 5 decisions, last error) → LLM call with a long system prompt → strip code fences → `json.loads` → `/step`. Parse errors and LLM exceptions both produce a KEEP fallback with confidence 0.5.
- The system prompt encodes the answer key for the hard task: leetspeak, emoji verbs, dogwhistles, escalation phrases ("planning my exit", "stockpiling", "tonight is the night"), and near-miss KEEP examples ("final solution to climate change", "kill it at the presentation"). This is prompt-engineering the benchmark, which is worth knowing when judging how meaningful the hard-task score is.
- Logging (must not change): `[START] task= env= model=`, `[STEP] step= action= reward=%.2f done=<lowercase> error=`, `[END] success=<lowercase> steps= score=%.2f rewards=<comma-joined>`. Then a FINAL SUMMARY table and overall average.
- Known loose ends: `MAX_STEPS=50` caps the loop at 50 steps, which is fine for all three queues. `env` in `[START]` is the URL, not a name. `log_step` prints the bare action_type, not `!r`.

## 10. Other interfaces

**Gradio (`gradio_ui.py`):** task dropdown, session box, five-way action radio, policy dropdown, reasoning textbox, confidence slider, observation and status panels. Calls the HTTP API via httpx to `ENV_URL`.

**MCP (`mcp_server.py`):** FastMCP with `reset_moderation_env`, `moderate_content`, `get_moderation_state`, `list_moderation_tasks`. All proxy to the HTTP API via `ENV_URL`. Dockerfile CMD switches on `MCP_MODE=true` to run `python server/mcp_server.py` instead of uvicorn.

**Docker:** installs curl (for HEALTHCHECK), `pip install uv`, `uv pip install --system -r requirements.txt`, copies `server/` and `openenv.yaml`, non-root user, `EXPOSE 7860 8080`. It does **not** copy `inference.py` or `pyproject.toml`.

**OpenEnv files:** `openenv.yaml` has `environment.entry_point: server.app:main`, full observation/action spaces, `reward.range [-1, 1]`, three tasks, tags `[openenv, content-moderation, nlp, trust-safety, meta, rl-environment]`, and `hub.space_id: vajeeda/meta-mod-env`. `pyproject.toml` script `meta-mod-env = "server.app:main"`, author `vajeedashaik`. `.openenv/hub.json` is a hub registration listing the three tasks and their expected score ranges.

## 11. How it was built (history)

1. **Apr 5 2026:** initial commit and "environment created" from `InstructionsPlan.md` (build everything). Landing page, HF username set to `vajeeda`, technical README.
2. **`Second Instruction.md`:** HF YAML front-matter, full `openenv.yaml`, exact log format, `session_id` on every call, per-task `TASK_MAX_REWARDS`, 0.01 grader floor, root Dockerfile, Gradio UI, MCP server, uv, `hub.json`.
3. **`Third Instruction.md`:** create `server/app.py` (entry point), harden the adversarial task because the first version scored 0.885 (too easy, exploit risk), add `pyproject.toml` and `uv.lock`, update README scores.
4. **Apr 6:** "Changed Adversarial Tasks", "Switch to Router and added LLM Diagnostic" (`test_llm.py`).
5. **Apr 8:** "Fix adversarial task scoring: real content tokens, softer grader, stronger LLM guidance" (evasion factor 0.3 → 0.6, urgent curve softened, real tokens instead of placeholders), "pre-submission fixes for openenv validate and grader compatibility" (duck-typed public graders), token fix, formatting, `/reset` accepts a missing body. Driven by `Final Submission Check.md`.
6. **May 10:** README update.

`Final Submission Check.md` is the judges' rubric turned into a script: ping HF `/reset` (must be 200), `docker build`, `openenv validate`, run `inference.py` end to end, 3+ tasks with graders in 0–1, plus human review for utility, exploits and grader quality. It has 10 blocks (structure, Python env, OpenEnv compliance, local server, graders, inference script, Docker, live Space, README, summary) with rules: do not change log format, do not change seed 42, do not change grader formulas unless a check fails.

## 12. Results, claims and known inconsistencies

**Reported benchmark (Qwen2.5-72B via HF Router):** basic 0.8800, contextual 0.6200, adversarial 0.4750, average 0.6583, all in the expected bands. The README states these figures, and `Third Instruction.md` dictated the same numbers to be written into the README before they were measured. Treat them as reported, not independently verified. The commit history does show later re-tuning of the adversarial grader after that.

**All-KEEP baseline (README):** basic 0.25, contextual 0.61, adversarial 0.13. The contextual figure is high because much of that queue should be kept. That is a mild exploit surface for the medium task.

**Drift between README and code:**
- README's Task 1 table (5 hate + 3 spam + 2 violence labelled ADD_WARNING_LABEL + 1 borderline) does not match the code (8 REMOVE, 7 KEEP, 3 LABEL, 2 ESCALATE).
- README's Task 3 table and code agree on 15/10/5/8/7/5 only if the "3 romanized + 5+5+5 evasion" rows are read as the 15 evasion items.
- README says "7 endpoints" including `/docs` and `/sessions`; `/` is also served.
- README says normalized scores use 4.35/6.30/10.30. That is only true in `inference.py`; the server uses a different min/max normalization, and the final reported score is the server grader score anyway.
- README says data has 50+ templates per category; the template lists in `data_generator.py` are much shorter (roughly 10–15 each).
- README's "Deployed live, passes `openenv validate .`" claims are supported by `settings.local.json` showing those commands were run, but that is not proof of the final state.
- Task 3 spec asked for cross-language Hindi/Tamil and a viral-misinformation item and a public-figure incitement item; the built data has Romanized placeholder phrases only and no explicit viral or public-figure items.

**Stale test:** `test_phase1.py` imports `_get_agent_action` from `server.tasks`, which no longer exists (the code now has `_get_agent_action_from_map`). That import would fail today.

**Security warning (important):**
- `meta-mod-env/.git/config` contains the HuggingFace token embedded in the `space` remote URL, in plaintext.
- `.claude/settings.local.json` also contains the same token in several allow-listed commands.
- The token is vajeedashaik's HF write token. It should be revoked and rotated, and the remote should be re-added without an inline credential. This file deliberately does not reproduce the token.

## 13. How to run

```bash
# Local server
cd meta-mod-env
pip install -r requirements.txt
uvicorn server.main:app --host 0.0.0.0 --port 7860        # or: python -m server.app

# Baseline agent (second terminal)
export OPENAI_API_KEY=<hf token>  API_BASE_URL=https://router.huggingface.co/v1
export MODEL_NAME=Qwen/Qwen2.5-72B-Instruct  ENV_URL=http://localhost:7860
python test_llm.py      # connectivity check first
python inference.py

# Optional
ENABLE_WEB_INTERFACE=true uvicorn server.main:app --port 7860   # UI at /ui
MCP_MODE=true python server/mcp_server.py
openenv validate .
docker build -t meta-mod-env . && docker run -p 7860:7860 meta-mod-env

# Tests
python test_phase1.py            # static (currently broken by stale import)
python test_phase3.py [url]      # live API
```

On this Windows machine the commands were run with `py -3` (Python 3.13) and PowerShell/Git Bash, per `settings.local.json`.

Env vars: `OPENAI_API_KEY`, `API_BASE_URL`, `MODEL_NAME`, `HF_TOKEN`, `ENV_URL` (inference); `ENABLE_WEB_INTERFACE`, `MCP_MODE`, `HOST`, `PORT`, `RELOAD` (server).

## 14. Design decisions worth remembering

- **Dense, asymmetric rewards** mirror trust-and-safety priorities: missed self-harm escalation (-0.25) worse than false-positive removal (-0.15) worse than a missed violation (-0.10).
- **Ground truth hidden twice:** `Field(exclude=True)` and an explicit `AgentVisibleContentItem`.
- **Stale-action guard:** `target_item_id` must match the current item, otherwise the action is ignored with `last_action_error`, no step consumed.
- **Graceful errors instead of 4xx/5xx:** unknown task, unknown session and finished episode all return structured results.
- **Difficulty tuning by grader, not just data:** the 0.65 multiplier and the urgent step-curve exist to push the hard task into its target band.
- **Server-authoritative score:** the agent's own reward arithmetic is a fallback; the grader score returned in the final `info` wins.
- **Synthetic-only data** avoids real hate speech, PII or self-harm content.

## 15. Possible next steps

- Fix `test_phase1.py` and de-duplicate the public and internal grader logic.
- Bring the README composition tables and template-count claims in line with the code.
- Add the missing hard-task elements (viral misinformation, public-figure protected speech, real cross-language items).
- Add a lock or per-session guard for `_sessions`, and session expiry (the dict grows forever).
- Penalize wrong actions on LABEL items and add an optional REQUEST_CONTEXT cost or budget so it is not a free skip.
- Rotate the leaked HF token.
