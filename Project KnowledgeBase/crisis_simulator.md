# Crisis Simulator (CrisisCore) — Project Context

**Location:** `C:\Users\kittu\OneDrive\Desktop\My Projects\Crisis Simulator`
**Git:** yes. Remote `origin` = `https://github.com/vajeedashaik/Crisis-Simulator.git`. 15 commits on one line of history (first "Base Structure Established", latest "mvp version1 completed"). Working tree has one uncommitted change: `train.py` (batch size 4→2, `num_generations` 8→4, `max_new_tokens` arg removed, most likely to fit Colab GPU memory).
**Type:** Reinforcement-learning environment plus a small multi-agent framework, a REST server, a training script and a browser dashboard. Built for a hackathon-style submission (the README front-matter carries the `openenv` tag and targets a HuggingFace Space).
**Names:** folder is "Crisis Simulator"; the code, README and Space call it **CrisisCore** (`CrisisCoreEnv`, `CrisisCoreClient`, dashboard title "Crisis Core Dashboard").
**Language / stack:** Python 3 (files were tested on 3.12 per the `.pyc` names; Docker image uses 3.11), FastAPI, plain dataclasses, one self-contained HTML dashboard. Training uses Unsloth + TRL + Qwen2.5-3B.
**Built:** 2026-04-24 → 2026-04-25, in 8 phases, each driven by a prompt file in `prompts/phase-N.md` fed to Claude Code one at a time.
**Status:** all 8 phases marked complete. **258/258 unit tests pass** (re-run and confirmed while writing this file: `py -m pytest tests/unit -q`, 1.2 s, 3 deprecation warnings about FastAPI `on_event`). `train.py` has never been run locally because it needs a GPU.

---

## 1. What this project is

**CrisisCore** is a small simulated building-evacuation environment in which AI agents must coordinate an emergency response. Each tick the agent gets a partial, noisy view of the building (sensor readings, room occupancy, which exits are open) and emits one action: route a zone's occupants toward an exit, dispatch an emergency service, broadcast a PA message, or set an incident severity label. A reward signal scores the choice.

The pitch in the README: *"training emergency-response AI systems requires realistic, dynamic environments where agents must coordinate evacuation routing, service dispatch, and communications under time pressure with noisy sensor data."* CrisisCore is meant to be a lightweight environment that plugs into normal RL loops over a REST API.

Three layers sit on top of the environment:

1. **A hierarchy of agents** (Orchestrator → Evacuation / Dispatch / Comms) that turn observations into JSON prompts for an LLM and parse the reply into actions.
2. **A curriculum** of three difficulty levels plus an "incident log" of past mistakes that is fed back into prompts.
3. **A GRPO fine-tuning script** (Qwen2.5-3B via Unsloth) meant to run on Colab, and **a cinematic dashboard** to watch episodes live.

## 2. File map

| Path | Size | What it does |
|---|---|---|
| `schema.py` | ~3 KB | Single source of truth for data types: 4 enums, 8 dataclasses (see §4). |
| `environment.py` | ~18 KB | `CrisisCoreEnv`: `reset()`, `step()`, `get_info()`, BFS routing, hazard spread, sensor noise, and a **built-in inline reward**. Has a 5-episode random smoke test under `__main__`. |
| `rewards.py` | ~8 KB | Stand-alone reward functions and `compute_reward()` with weights. Used **only by `train.py`**, not by the env itself (see §7). Has an "anti-hacking" self-check under `__main__`. |
| `agents.py` | ~9 KB | `EvacuationAgent`, `DispatchAgent`, `CommsAgent`, `OrchestratorAgent`, `build_agent_prompt`, `_parse_action`. |
| `curriculum.py` | ~6 KB | `CURRICULUM_LEVELS`, `CurriculumManager`, `IncidentLog`, `run_curriculum_episode`. |
| `train.py` | ~10 KB | Colab GRPO script (imports Unsloth at module top, so it cannot even be imported without it). |
| `app.py` | ~7.5 KB | FastAPI server: `/reset`, `/step`, `/state`, `/trigger-crisis`, `/health`. CORS open to all origins. |
| `client.py` | ~3 KB | `CrisisCoreClient` (requests wrapper) plus a 3-episode random demo. Default URL is a placeholder: `https://yourusername-crisiscore.hf.space`. |
| `dashboard.html` | 612 lines | Dark, dependency-free live dashboard that talks to the API. |
| `Dockerfile` | | `python:3.11-slim`, copies only `schema, environment, rewards, agents, curriculum, app`, exposes 7860. |
| `requirements.txt` | | `fastapi==0.115.5`, `uvicorn==0.32.1`, `pydantic==2.10.3`, `requests==2.32.3`, `httpx==0.28.1`. |
| `README.md` | | HuggingFace Space front-matter (`sdk: docker`, tags multi-agent / RL / crisis-response / openenv) plus API and reward docs. |
| `tests/unit/` | 6 files | 258 tests (counts in §10). |
| `prompts/phase-1.md … phase-8.md` | 17–41 lines each | The exact prompts each phase was built from. Useful as the original spec. |
| `docs/superpowers/specs/2026-04-24-dashboard-design.md` | | Full dashboard design spec (layout, panels, colours, data flow). |
| `docs/superpowers/plans/2026-04-25-phase8-dashboard.md` | 848 lines | Implementation plan for the dashboard. |
| `docs/progress.md`, `session/*.md` | | Progress ledger, phase log, last-session summary. Accurate and worth trusting. |
| `docs/prd.md`, `architecture.md`, `deployment.md`, `testing.md`, `debugging.md`, `learnings.md` | | **Unfilled boilerplate** from a generic Supabase/Next.js/Jest template (see §12). They do not describe this project. |
| `claude.md` / `CLAUDE.md` | | Workflow rules for Claude sessions (see §11). |
| `.claude/skills/*.md`, `.claude/settings.local.json` | | Generic Supabase/debugging/testing skill notes; settings only allow `python -m pytest tests/ --tb=no -q`. |
| `scripts/dev.sh`, `test.sh`, `deploy.sh` | 0 bytes | **Empty placeholder files.** |
| `logs/errors.log`, `logs/test.log` | | UTF-16 encoded (they look spaced-out in a plain viewer). Last logged test run said 255 passed, which predates the 3 trigger-crisis tests. |
| `src/` | empty | Unused. |

## 3. The environment (`environment.py`)

### Config (dict passed to `CrisisCoreEnv(config)`)
`num_floors` (1), `num_zones_per_floor` (4), `num_people` (10), `num_hazards` (1), `max_ticks` (30), `sensor_noise_rate` (0.0). Defaults are in parentheses.

### `reset()` builds
- **Zones:** for each floor, a near-square grid (`cols = ceil(sqrt(n))`). Zone IDs look like `f0_r1_c0`. Each zone has capacity 20 and 4-neighbour adjacency **within its own floor only**. Floors are never connected to each other.
- **Exits:** two per floor, on the first and last zone of the grid, named `exit_f{floor}_{1|2}`. An "exit" is a zone with `has_exit=True`; `available_exits` and `route_to_exit` both use the **zone ID**, not the exit ID.
- **People:** `person_0 … person_{n-1}`, each placed in a uniformly random zone.
- **Hazards:** each placed in a random non-exit zone, random type (fire / smoke / structural), intensity `uniform(0.4, 1.0)`, spread rate 0.3.
- Then sensors are refreshed and the first observation is returned.

### `step(action)` order of operations
1. Snapshot evacuated count and hazard zones.
2. Apply the action (a missing required field just sets `format_ok=False`; it does not raise).
3. Spread hazards, refresh sensors, `tick += 1`.
4. Check termination: everyone evacuated **or** `tick >= max_ticks`.
5. Compute the reward (inline, see §7) and return `(observation, RewardBreakdown, done, info)`.

`info` = `total_people, evacuated_count, remaining_count, active_hazards, current_tick, hazard_zones`.

### Actions
- **`route_zone`**: every un-evacuated person in `zone_id` takes **one step** along the BFS shortest path toward `route_to_exit`. Anyone who lands on (or already stands on) an exit zone is evacuated. Each person who steps into a hazard zone costs -0.5. If either ID is unknown, nothing happens and there is no penalty.
- **`dispatch_service`**: recorded, then scored (`fire→fire_brigade`, `smoke→ems`, `structural→police`).
- **`broadcast_pa`**: message stored in `_pa_messages`. **No effect on the simulation.**
- **`update_severity`**: stored in `_current_severity` (echoed back in observations) and scored against the highest hazard intensity.

### Partial observability (`_get_observation`)
- `zone_occupancy`: exact head-count of un-evacuated people per zone (so the agent is **not** blind to positions at zone level).
- `known_hazard_zones`: only true hazard zones whose sensor shows `smoke > 0.3` or `sound > 0.5`. Noise can therefore **hide** a hazard but can never invent a false one.
- `available_exits`: exit zones not in `blocked_exits`.
- `sos_signals`: persons with `has_sos` and not evacuated.
- `sensor_readings`: per-zone `smoke_level, motion_detected, sound_level, is_noisy`.

### Sensor model
Normal reading: hazard zone → `smoke = intensity` (fire/smoke types) or `0.1` (structural), `sound = uniform(0.3, 1.0)`; safe zone → smoke 0, `sound = uniform(0, 0.15)`; `motion = zone occupied`. With probability `sensor_noise_rate` a zone's reading is replaced by uniform-random values. Structural hazards are effectively detected by sound only.

### Simplifications worth knowing
- **People never get hurt.** Hazards affect only rewards, never casualty counts.
- **Hazards spread but never intensify or die out.** Each tick, with probability 0.3, one random neighbouring zone joins `affected_zones`.
- **`has_sos` is never set to True and `blocked_exits` is never populated anywhere**, so `sos_signals` is always empty and every exit is always open. These features exist in the schema and observation but are effectively dead.
- Only one person-step per `route_zone` call, and only one zone per tick, so evacuation is throttled by the single-action-per-tick design.
- **Multi-floor caveat:** because floors aren't connected, routing a zone on floor 1 to an exit on floor 0 finds no path and silently does nothing. Random agents at Levels 2 and 3 waste many ticks this way.

## 4. Data model (`schema.py`)

- **Enums:** `ActionType` (route_zone, dispatch_service, broadcast_pa, update_severity), `ServiceType` (fire_brigade, ems, police), `HazardType` (fire, smoke, structural), `SeverityLevel` (low, medium, high, critical). All values are **lowercase strings**, and the API only accepts those.
- **Dataclasses:** `Zone`, `Person`, `Hazard`, `SensorReading`, `BuildingState`, `AgentAction`, `AgentObservation`, `RewardBreakdown`.
- `AgentAction` is one flat record with optional fields; which fields are required depends on `action_type`.
- `RewardBreakdown` has `evacuation_speed, route_safety, dispatch_accuracy, severity_accuracy, format_compliance, timeout_penalty, total`.

## 5. Agent hierarchy (`agents.py`)

Each sub-agent has `observe()` (narrow the full observation to what that role should see) and `act(narrowed_obs, model_fn)` where `model_fn: str -> str` is any callable that maps a prompt to a JSON string. Swapping `model_fn` is how you plug in a real LLM, a rule-based stub or a random policy.

| Agent | Sees | Emits | Safe default if parsing fails |
|---|---|---|---|
| `EvacuationAgent` | zone_occupancy, available_exits, known_hazard_zones | `route_zone` | route toward `exits[0]` (note: `zone_id` left `None`) |
| `DispatchAgent` | sos_signals, current_severity, active hazard types | `dispatch_service` | `fire_brigade` if any hazard, else `ems` |
| `CommsAgent` | evacuation agent's routing decision + occupancy | `broadcast_pa` | generic "please evacuate calmly" message |
| `OrchestratorAgent` | full obs + state | list of the 3 actions | n/a |

- **Orchestrator flow:** Evacuation and Dispatch run in parallel in a `ThreadPoolExecutor`; Comms is submitted as soon as the evacuation action exists, so it can reference the routing decision. Returns `[evac, dispatch, comms]`.
- `build_agent_prompt(role, observation, incident_log)` creates `SYSTEM: <role paragraph>\n\nUSER: Current observation:\n<json>` and appends `Last episode mistakes: a; b; c` if the log is non-empty.
- `_parse_action` uses `json.loads`, builds an `AgentAction`, and on **any** exception logs a warning and returns the default. It does not enforce that the parsed `action_type` matches the agent's role.
- **Important:** everywhere the orchestrator is used for real (`run_curriculum_episode`, `train.rollout`) **only `actions[0]` (the evacuation action) is sent to `env.step`.** The dispatch and comms actions are computed and discarded. The env accepts one action per tick, so the "three agents" are currently three prompts of which one matters.

## 6. Curriculum and incident log (`curriculum.py`)

| Level | Floors × zones | People | Hazards | Max ticks | Sensor noise |
|---|---|---|---|---|---|
| 1 | 1 × 4 | 5 | 1 | 20 | 0.00 |
| 2 | 2 × 4 | 15 | 2 | 30 | 0.10 |
| 3 | 3 × 6 | 30 | 3 | 40 | 0.15 |

- `CurriculumManager(promotion_threshold=0.7, window=20)`: `record_episode(reward_total, max_possible)` appends `reward/max_possible`; once the window is full and its mean exceeds the threshold, the level goes up by one (max 3) and the window clears. `max_possible` is supplied by the caller (the demo hard-codes 50.0).
- `IncidentLog(max_entries=10)`: deque of human-readable mistakes generated from a reward: routed through a hazard (`route_safety < -1.0`), wrong service (`dispatch_accuracy < 0`), severity misclassified (`severity_accuracy < 0`), timeout with N people unrescued. It is fed back into prompts as "Last episode mistakes" (a lightweight self-correction loop).
- `run_curriculum_episode(env, orchestrator, model_fn, incident_log)` runs one full episode and returns `(total_reward, info)`.
- **Not wired into the server or trainer:** `app.py` hard-codes Level 1 (`/health` always reports `level: 1`) and `train.py` only trains on Level 1 and never instantiates `CurriculumManager`.

## 7. Rewards — two separate systems

This is the single most confusing part of the codebase. There are **two reward implementations that disagree**:

### A. Inline reward in `CrisisCoreEnv.step()` (what the API, dashboard and rollouts actually receive)
| Component | Rule |
|---|---|
| `evacuation_speed` | `newly_evacuated × (1 − tick/max_ticks)` (tick already incremented) |
| `route_safety` | −0.5 per person who steps into a hazard zone |
| `dispatch_accuracy` | +1.0 if the service matches any active hazard type (fire→fire_brigade, smoke→ems, structural→police), else 0 |
| `severity_accuracy` | +0.5 if the label's intensity bracket contains the max hazard intensity (low <0.3, medium 0.3–0.6, high 0.6–0.8, critical ≥0.8), else 0 |
| `format_compliance` | +0.1 if all required fields present |
| `timeout_penalty` | −1.0 per person left inside when the episode ends |
| `total` | plain sum |

### B. `rewards.py` (`compute_reward`, used **only** by `train.py` for GRPO)
| Function | Rule |
|---|---|
| `evacuation_speed_reward` | `new × (1 + (max−tick)/max)`, capped at 5.0 |
| `route_safety_reward` | −3.0 if the source zone is a hazard zone; else −2.0 if the exit is a hazard zone; else 0 |
| `dispatch_accuracy_reward` | fire_brigade +2.0 if fire/smoke present; ems +1.0 if any hazard; police −0.5 only when *no* crowd-risk hazard exists (which is never, so police is effectively 0) |
| `severity_accuracy_reward` | ground truth from the share of people still in hazard zones (<10% low, <30% medium, <60% high, else critical); exact +1.5, off by one +0.5, worse −1.0 |
| `format_compliance_reward` | +0.2 if valid, −1.0 if a required field is missing |
| `timeout_penalty` | −5.0 flat if done and anyone remains |
| `compute_reward` weights | evac 1.0, route 1.0, dispatch 0.8, severity 0.6, format 0.4, timeout 1.0 |

**Consequences to keep in mind**
- The README reward table is a loose description; neither system matches its wording exactly ("+points per evacuee scaled by time" and "bonus within one bracket").
- In GRPO (`compute_rewards_for_grpo`) each completion is scored by calling `env.reset()` and then `compute_reward(env.state, action, prev_evacuated=0, done=False)` on a **fresh random state that has nothing to do with the prompt the model saw**. Evacuation, timeout and severity terms are therefore near-constant noise there, and the effective learning signal is mostly format validity (unparseable JSON = −2.0) and dispatch/route heuristics against a random building. Treat any "trained beats baseline" number from this script sceptically.
- Both systems allow **reward farming by repetition** (e.g. dispatching the right service every tick keeps paying). `rewards.py` includes an "anti-hacking" check that only proves the reward doesn't *grow* with repetition, not that it's zero.

## 8. REST API (`app.py`)

Single global environment (Level 1 config), single global episode. Fine for a demo or one Space, **not** safe for concurrent clients.

| Method | Path | Behaviour |
|---|---|---|
| POST | `/reset` | New episode; returns `AgentObservation` JSON. |
| POST | `/step` | Body = `ActionRequest` (all-string fields). Invalid enum value → **422**; step before reset → 400; other → 500. Returns `{observation, reward_breakdown, done, info}`. |
| GET | `/state` | Full `BuildingState` (all zones, people, hazards, sensors). 400 if no episode. |
| POST | `/trigger-crisis` | Adds a new hazard (`crisis_<8 hex>`, random type, intensity 0.8, spread 0.3) in a random zone not already hazardous. 400 if no episode or all zones affected. Returns the `BuildingState`. |
| GET | `/health` | `{status: "ok", tick, level}`. |

Note `app.py` creates an `OrchestratorAgent` and defines a rule-based `_default_model_fn`, but **no endpoint calls either** (progress.md says "OrchestratorAgent wired", which is optimistic). Startup uses the deprecated `@app.on_event("startup")` (source of the 3 test warnings).

## 9. Dashboard (`dashboard.html`)

Pure HTML/CSS/JS, no CDN, works offline against a running server. Theme: `#0a0a0f` background, monospace, green/red/orange/blue accents.

- **Floor map (left, ~60%)**: CSS-grid zones colour-coded (safe / near hazard / hazard / evacuated-empty / exit), person dots, pulsing red border on hazards, floor tabs when >1 floor.
- **Agent decision log (top right)**: one line per tick, labelled by inferred sub-agent (route→Evacuation, dispatch→Dispatch, PA/severity→Comms), green/red/grey by reward sign.
- **Metrics (2×2)**: evacuated/total, tick/max, severity, running episode reward.
- **Responder payload (bottom right)**: syntax-highlighted JSON incident brief (`incident_type, location_coordinates, affected_persons, hazards_present, access_routes`) that slides in whenever a dispatch happens.
- **Controls**: Start, Pause, Reset, speed slider 0.5×–3×, Random/Trained toggle, ⚡ Trigger Crisis. A pulsing LIVE dot shows while running; the episode auto-pauses when done.
- **How it runs:** the browser drives the loop (`setTimeout(1000/speed)`), generating an action, POSTing `/step`, then GETting `/state`. "Trained" mode is **not a model**; it is a client-side heuristic (route a hazard zone first, dispatch fire_brigade if severity is high/critical, else broadcast).

### Known dashboard bugs (found while reading the code)
1. **Wrong port:** `const API = 'http://localhost:8000'`, but the README, client, Dockerfile and docs all use **7860**. Run `uvicorn app:app --port 8000` or edit that constant, otherwise every call fails.
2. **Severity casing:** the dashboard sends `'LOW' | 'MODERATE' | 'HIGH' | 'CRITICAL'`, but the API only accepts lowercase `low | medium | high | critical`. Every random-mode `update_severity` call therefore gets a **422**, and `MODERATE` isn't a valid level at all.
3. **Trained-mode severity check never fires:** it compares `obs.current_severity` (returned lowercase) to `'HIGH'`/`'CRITICAL'`, so the dispatch branch is dead. Combined with the "route the first hazard zone" rule taking priority, trained mode mostly just routes hazard zones.
4. `Dockerfile` doesn't copy `dashboard.html` (fine, it's opened locally), and the dashboard only sees the Level 1 environment.

## 10. Tests

`py -m pytest tests/unit -q` → **258 passed**.

| File | Tests | Covers |
|---|---|---|
| `test_schema.py` | 61 | enums, dataclass construction and defaults |
| `test_environment.py` | 86 | reset layout, adjacency, exits, BFS, step for all actions, spread, sensors, termination, info |
| `test_rewards.py` | 46 | each reward function and `compute_reward` weighting |
| `test_agents.py` | 31 | observe/act, defaults, parsing, prompt builder, orchestrator |
| `test_curriculum.py` | 31 | levels, promotion, sliding window, incident log, episode runner |
| `test_trigger_crisis.py` | 3 | 400 before reset, one hazard added, response shape (FastAPI `TestClient`) |

Not covered: `train.py` (GPU), `client.py`, the dashboard, and the API endpoints other than `/trigger-crisis` (the phase-7 endpoints were smoke-tested by hand with curl).

## 11. How to run

```bash
pip install fastapi uvicorn pydantic requests httpx pytest
uvicorn app:app --host 0.0.0.0 --port 7860      # use --port 8000 if you want dashboard.html to work unchanged
python client.py                                # 3 random episodes against localhost:7860
python environment.py                           # 5-episode smoke test
python rewards.py                               # reward self-check
python agents.py                                # 3-tick orchestrator demo
python curriculum.py                            # 30-episode curriculum demo
py -m pytest tests/unit -q                      # tests (on this machine the launcher is `py`, plain `python` isn't on PATH)
```
Then open `dashboard.html` directly in a browser.

**Training (Colab only):** install `unsloth[colab-new]`, `trl>=0.8.0`, `transformers`, `torch`, `datasets`, `matplotlib`, then `python train.py`. It loads `unsloth/Qwen2.5-3B-Instruct` in 4-bit (max seq 2048), wraps it with LoRA (r=16, alpha=16, `q_proj`/`v_proj`), builds 200 Level-1 reset prompts, runs a 20-episode baseline, trains 1 epoch with GRPO (currently batch 2, 4 generations), runs a 20-episode post-eval, prints `Baseline → Trained | Improvement`, saves `reward_curve.png` and the merged 16-bit model to `crisiscore_model/`.

**Deploy:** push to a HuggingFace Space (Docker SDK, port 7860). Caveat: the Dockerfile `pip install`s `openenv`, which the last session noted **is not on PyPI**. The build may fail; nothing imports it, so removing it from that line is the likely fix. The env's docstring says "OpenEnv-compliant" but it doesn't subclass any OpenEnv base class.

## 12. Development workflow (how this repo was built)

`claude.md` defines a strict, phase-gated workflow:
- On session start read only `docs/index.md`, `session/phase-log.md`, `session/context.md`, then the current `prompts/phase-N.md`. Load other docs on demand.
- Work on one phase at a time; never move on without explicit user confirmation and passing tests.
- On session end: give a one-line git commit message per change, **do not push**, update `session/summary.md`, append one line to `session/phase-log.md`, update `docs/progress.md`, add any workaround to `docs/learnings.md`.
- Error log format: `[ERROR] [timestamp] [file:function] — reason — fix hint`. Only `environment.py` implements this helper; `agents.py` logs warnings in a similar style.
- Supabase rules in `claude.md`/`.claude/skills/supabase.md`/`docs/deployment.md` are **inherited template text. This project does not use Supabase, Next.js or Jest.** The same goes for `docs/prd.md`, `docs/architecture.md`, `docs/testing.md` and `docs/debugging.md`, which are still template placeholders. The real spec lives in `prompts/` and `docs/progress.md`.

### Phase timeline
| Phase | Deliverable | Date | Tests |
|---|---|---|---|
| 1 | `schema.py` | 2026-04-24 | 61 |
| 2 | `environment.py` | 2026-04-24 | 86 |
| 3 | `rewards.py` | 2026-04-24 | 46 |
| 4 | `agents.py` | 2026-04-24 | 31 |
| 5 | `curriculum.py` | 2026-04-24 | 31 |
| 6 | `train.py` (GRPO) | 2026-04-24 | none (GPU) |
| 7 | `app.py`, Dockerfile, README, client, requirements | 2026-04-24 | manual curl |
| 8 | `/trigger-crisis`, `dashboard.html`, CORS | 2026-04-25 | 3 |

## 13. Open issues and loose ends (prioritised)

1. **Dashboard is broken as shipped:** port 8000 vs 7860, and uppercase severity values give 422s (§9). Easiest fix: change `API` to 7860 and use lowercase `low/medium/high/critical`.
2. **Two reward systems** (env inline vs `rewards.py`) with different scales; the GRPO reward is computed on a fresh random state unrelated to the prompt (§7). Real training would need the reward tied to the actual observation/state the completion was generated for.
3. **Only one of the three orchestrator actions reaches the environment** (§5). Either the env should accept multi-action steps or the agents should be merged.
4. **Curriculum not used** by server or trainer; Level 2/3 exist but nothing runs them except the demos.
5. **Dead features:** `has_sos`/`sos_signals`, `blocked_exits`, PA messages and multi-floor connectivity have no effect.
6. **README and repo drift:** README embeds `reward_curve.png`, which doesn't exist (it's generated by `train.py`); `client.py` default URL is a placeholder; README calls the reward "route_safety penalty for routing through or toward hazard zones" though the two implementations differ.
7. **Deployment risk:** `openenv` in the Dockerfile; no `dashboard.html` in the image; single global episode is not concurrency-safe.
8. **Housekeeping:** empty `scripts/*.sh` and `src/`; `__pycache__` files are committed to git; template docs (`prd`, `architecture`, etc.) never filled in; `on_event` deprecation; uncommitted `train.py` edit.
9. **No result numbers exist:** no baseline/trained comparison or reward curve has ever been produced in this repo.

## 14. Quick-start cheat sheet for a future session

- Core loop: `obs = env.reset()` → build `AgentAction` → `obs, reward, done, info = env.step(action)`. Everything is plain dataclasses; `app._to_dict` handles enum-safe JSON.
- To plug in an LLM, pass any `model_fn(prompt: str) -> str` returning JSON like `{"action_type": "route_zone", "zone_id": "f0_r0_c1", "route_to_exit": "f0_r0_c0"}` into `OrchestratorAgent.act`.
- Valid string values: actions `route_zone|dispatch_service|broadcast_pa|update_severity`; services `fire_brigade|ems|police`; severities `low|medium|high|critical`; hazards `fire|smoke|structural`.
- Zone IDs: `f{floor}_r{row}_c{col}`. Exits are the first and last zone of each floor and are referenced by zone ID.
- If asked to "improve" the project, the highest-value work is items 1–3 in §13, then producing an actual training run and reward curve.
