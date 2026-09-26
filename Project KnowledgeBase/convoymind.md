# ConvoyMind (TATA Fleet) — Project Context

**Location:** `C:\Users\kittu\OneDrive\Desktop\My Projects\TATA Fleet` (frontend in `convoymind/`, ML in `model/`)
**Git:** not a git repo, no remote, not deployed. Runs on localhost only.
**Type:** Demo / hackathon simulation. Everything is simulated client-side: no real fleet, sensors or trucks.
**Built across:** 9 spec'd phases (`Phase1.md`–`Phase9.md`), logged in `Progress.md` (~83 KB build log). Later polish pass described in `design changes.md`.

## 1. What this project is

An AI-coordinated **truck convoy** simulation. Four trucks drive a night highway run, each with its own driver. Four onboard "agents" (fatigue, health, coordination, ADAS) propose speed and spacing changes every tick. They talk over a simulated **V2X** (vehicle-to-everything) layer. An **arbitration hub** resolves conflicts between them. A trained ML model predicts each driver's fatigue live, and the app falls back to a synthetic formula if the model server isn't running.

The UI is meant to feel like a logistics operations room watching 4 assets move through the night, not a car dashboard or a SaaS product.

It is a single fixed scenario: 4 trucks, 4 named drivers, one route. The only inputs are distance, deadline, time of day and a scenario preset.

The repo holds two separate projects:

| Folder | What it is |
|---|---|
| `convoymind/` | React + TypeScript + Vite frontend. Contains the whole UI and the simulation engine. |
| `model/` | Standalone Python project: trains an XGBoost fatigue model, exports it to ONNX, serves it via FastAPI on `localhost:8001`. |

Other files at the root: `README.md`, `APP_GUIDE.md` (full "current state" reference), `Progress.md`, `Phase1–9.md` (original specs), `design changes.md`.

## 2. Tech stack

**Frontend** (`convoymind/package.json`)
- React 19, react-router-dom 7, TypeScript 6, Vite 8
- Tailwind CSS 4 via `@tailwindcss/vite`, but most styling is inline styles plus CSS variables
- Vitest 4 (unit tests), Playwright 1.61 (e2e, uses system Chrome via `channel: 'chrome'`)
- oxlint for linting
- No state library and no UI library. State is React state plus one context.

**Model** (`model/requirements.txt`, pinned, in its own `.venv`)
- numpy, pandas, scikit-learn, xgboost 3.3, onnx, onnxruntime, onnxmltools, skl2onnx, seaborn, matplotlib
- fastapi 0.139 and uvicorn

## 3. How to run

```bash
# Frontend (required)
cd convoymind
npm install
npm run dev            # http://localhost:5173 -> redirects to /setup

# Model server (optional but recommended)
cd model
.venv/Scripts/python.exe server.py    # http://127.0.0.1:8001
```

The bare system `python` lacks the ONNX packages, so use the `.venv` interpreter.

Scripts (from `convoymind/`):

| Script | Action |
|---|---|
| `npm run dev` | Vite dev server |
| `npm run build` | `tsc -b && vite build` |
| `npm test` | Vitest: 8 engine unit tests |
| `npm run test:e2e` | Playwright full demo flow (~3.5 min). Run it with the model server **stopped**. |
| `npm run lint` | oxlint |

Rebuild the model (rarely needed; artifacts are committed): `generate_dataset.py` → `train_model.py` → `export_onnx.py`.

## 4. Frontend structure (`convoymind/src/`)

| File | Role |
|---|---|
| `main.tsx`, `App.tsx` | Entry. Routes: `/` redirects to `/setup`, `/setup` is MissionSetup, `/convoy` is FleetOperations. Wrapped in `ConvoyProvider`, with a fixed `Header`. |
| `context/ConvoyContext.tsx` | Holds the single `SimulationEngine` in state. `deploy(config)` creates a new engine. `useConvoy()` reads it. This is how `/setup` and `/convoy` share one run. |
| `simulation/engine.ts` | **The core.** `SimulationEngine` class (~810 lines). Read this first. |
| `simulation/engine.test.ts` | 8 Vitest tests: init with 4 distinct drivers, position advance, fatigue growth, V2X beacons each tick, fatigue warning threshold, arbitration on conflict, metrics accumulate, reset. Stress-tested 20 consecutive runs with 0 failures. |
| `types/convoy.ts` | All shared types: `TruckState`, `DriverProfile`, `AgentRecommendation`, `V2XBeacon`, `NegotiationLogEntry`, `FleetMetrics`, `SimulationConfig`, `HazardEvent`, `SimulationTick`. |
| `hooks/useModelServerHealth.ts` | Polls `GET :8001/health` every 10 s. Returns status `checking`/`online`/`offline` plus model stats. |
| `routes/MissionSetup.tsx` | `/setup` page. |
| `routes/FleetOperations.tsx` | `/convoy` page. Subscribes to engine ticks, auto-plays on mount, auto-pauses when all trucks reach 100%, runs the entrance animation. |
| `components/RouteMap.tsx` (+css) | Largest component (~670 lines). SVG top-down night highway: trucks, lane markings, stars, V2X pulses (tiered by severity), packet-drop ✗ markers, hazard marker, LEAD badge and handoff animation, hover tooltips, legend. |
| `components/AgentPanel.tsx` | Container for 4 `TruckAgentCard`s and the `ArbitrationFeed`, plus the "model server offline" banner. |
| `components/TruckAgentCard.tsx` | Per-truck card: ID, driver, ADAS badge, colour-coded fatigue bar, speed/fuel/health. |
| `components/ArbitrationFeed.tsx` (+`AgentPanel.css`) | Live negotiation log. Shows the 8 most recent entries, colour-coded by agent, with `[×N repeated]` collapsing. |
| `components/FleetIntelligenceBar.tsx` (+css) | Bottom bar: 6 live metrics. Expands to ~320 px at run completion with AI vs. without-AI comparison bars, badges and "run another mission →". |
| `components/PlaybackControls.tsx` | Play/pause, 1×/2×/4× speed, scenario tag. |
| `components/ParticleBackground.tsx` | Canvas drifting-dot background (36 particles). Respects `prefers-reduced-motion`. |
| `Header`, `ConvoyIcon`, `Slider`, `Sparkline`, `TruckGlyph` | Small UI pieces. |
| `e2e/convoy_flow.test.ts` | Permanent Playwright test of the full worst-case demo. Uses an 800 km route so fatigue warnings appear before completion. Filters the expected `ERR_CONNECTION_REFUSED` errors from the offline model server. |

## 5. UI / pages

- **Header (56 px, fixed):** logo, nav links (`mission setup`, `fleet operations`), and an empty reserved slot on the right for status pills.
- **`/setup` (Mission Setup):**
  - Sliders: Route Distance 100–800 km (default 420), Delivery Deadline 60–720 min (default 380).
  - Night/Dawn toggle.
  - Read-only driver roster (2×2 grid) with fatigue sparklines.
  - Scenario pills: `smooth run`, `fatigue event`, `road hazard`, `tight deadline`, `worst case`. Worst case is intentionally red.
  - Sticky deploy bar: truck preview, settings summary, `load worst case →` (680 km / 340 min / worst_case / night) and `deploy convoy →`.
- **`/convoy` (Fleet Operations):**
  - Left 60%: playback controls and the route map.
  - Right 40%: agent panel with 4 truck cards and the negotiation feed.
  - Bottom: Fleet Intelligence Bar.
  - With no deployed engine it shows a "no mission configured" empty state.

**Design tokens** (from `Phase1.md`, defined in `index.css`):

| Token | Value |
|---|---|
| Backgrounds | `#05080F` base, `#0D1520` panel, `#0A1628` deep |
| Blue (healthy) | `#3D8EF0` |
| Teal (V2X/live) | `#00E5B0` |
| Amber (warning) | `#F0A500` |
| Red (critical) | `#FF4D5E` |
| Violet (arbitration only) | `#7B61FF` |
| Text | `#E8EFF9` primary, `#8DA3BC` secondary |
| Muted text | spec `#4A6080`, **lightened to `#6884AC`** in Phase 9 to pass WCAG 4.5:1 contrast |
| Fonts | JetBrains Mono for numbers/data/IDs, Inter for labels/body |

Transitions run 200–400 ms and respect `prefers-reduced-motion` (a global CSS override, plus explicit handling in the JS `useSmoothedNumber` tween and the canvas particles).

`design changes.md` is a follow-up prompt asking for a full dark-theme pass on `/convoy`: no light surfaces, warning/critical tints on cards, truck-ID badges instead of emoji, and so on.

## 6. Simulation engine: mental model

**Time.** 1 tick = 1 simulated minute. `play()` runs `tick()` every `1000 / speedMultiplier` ms (1×/2×/4×). The session starts at 23:00 sim time.

**Trucks and drivers** (fixed):

| Truck | Driver | Fatigue baseline | Night-adapted | Notes |
|---|---|---|---|---|
| T1 | Arjun | 5.5 h | yes | Lead truck; most resistant |
| T2 | Priya | 3.2 h | no | Fatigues fastest, 1.8× rate in fatigue scenarios |
| T3 | Ravi | 4.8 h | yes | Moderate |
| T4 | Meera | 4.0 h | no | Moderate |

**`tick()` pipeline**

- **A0.** One-shot hazard trigger (only in `road_hazard` and `worst_case`).
- **A.** `applyPhysics` per truck. Speed converges to the pending target (max ±8 km/h per tick, clamped to 20–90). Position and fuel update. Brake temp and engine stress are stochastic. Fatigue comes from the model or the synthetic fallback. Spacing converges by 3 m per tick.
- **A2.** Every 5 ticks, POST features to the model server for each truck. This is fire-and-forget with a 1500 ms timeout. Stale responses are dropped via `resetGeneration`.
- **B.** Generate V2X beacons: every truck to every other truck, each tick, with a 5% drop rate (`delivered: false`).
- **C–F.** Per truck, run four agents in order: fatigue, health, coordination, ADAS. Each pushes `AgentRecommendation`s. Agents never mutate speed or spacing directly.
- **F2.** Leader-handoff check.
- **G.** `runArbitration`. Recommendations are grouped by `(truck, metric)`, where metric is speed or spacing. Any recommendation with priority ≥ 8 from the fatigue agent, or with positive safety impact, overrides. Otherwise the winner is picked by weighted score: `safety×3 + fatigue×2.5 + schedule×1 + fuel×1.5`. Groups with ≥ 2 distinct actions are logged as conflicts. **Arbitration is the sole authority** that applies effects, queued for the next tick's physics through a `WeakMap` side-table.

**Agent rules**

| Agent | Rule |
|---|---|
| Fatigue | Score > 55 → warning, `increase_spacing_22` (priority 6). Score > 75 → critical, `reduce_speed_25` (priority 9), ADAS status `critical`. Also computes predicted minutes to reach 80. |
| Health | `healthScore = 0.3·fuel + 0.3·(100−brake%) + 0.4·(100−engineStress)`. Below 40 → `reduce_speed_10` (priority 7). |
| Coordination | If a delivered beacon shows another truck's fatigue > 55 → `reduce_speed_10` plus 22 m spacing (priority 6). |
| ADAS | Platooning (spacing < 15 m) → status `assisting`. Behind schedule → `increase_speed_8` (priority 4). Otherwise `maintain_speed` (priority 2). |

**Hazard event.** At tick 30, at 35% of the route, in `road_hazard` and `worst_case` only. The closest truck is the detector. Directly (bypassing arbitration) it sets every truck's speed −20 km/h, spacing target 35 m and ADAS `alert`, so all 4 react the same tick. The detecting truck re-broadcasts hazard beacons for 8 ticks.

**Leader handoff.** T1 leads by default. If T1's fatigue ≥ 75 or health < 30, leadership passes to T2, once.

**Key constants**

| Constant | Value |
|---|---|
| Tank | 400 L |
| Baseline consumption | 0.35 L/km |
| Platoon threshold / fuel saving | < 15 m / 12% |
| Spacing targets | normal 12 m, fatigue 22 m, hazard 35 m |
| CO₂ per litre diesel | 2.67 kg |
| Base speed | 65 km/h |
| Fatigue warning / critical | 55 / 75 (predicted-critical threshold is 80) |

**Synthetic fatigue fallback:** rate/hour = `(10 / baselineHours) × 0.8 if night-adapted × 1.2 during 03–04h × 1.8 for T2 in fatigue scenarios`, plus ±2 noise applied in the same per-hour unit before scaling to per-tick. Scaling matters: raw per-tick noise swamped the driver differences and made tests flaky.

**Scenarios:**

| Preset | Behaviour |
|---|---|
| `smooth` | Mostly uneventful. |
| `fatigue_event` | T2 fatigue 1.8×. |
| `road_hazard` | Hazard at tick 30. |
| `tight_deadline` | Short deadline, ADAS pushes speed up. |
| `worst_case` | Hazard plus T2 fatigue. |

**Fleet metrics:** fuel saved, CO₂ avoided, fatigue events caught (edge-triggered per warning episode), incidents prevented (= events × 0.23, illustrative), platooning efficiency (share of fleet platooning × 12%), ETA vs. remaining deadline.

## 7. ML model layer (`model/`)

**Pipeline:** `generate_dataset.py` → `train_model.py` → `export_onnx.py` → `server.py`.

- **Data:** 50,000 rows of **synthetic** data (seed 42), generated from published PERCLOS/drowsiness literature, not real sensor data. The UTA RLDD dataset wasn't obtainable at feature level. There are 4 driver-sensitivity profiles (multipliers 0.85–1.30).
- **8 features:** `perclos`, `blink_rate`, `head_nod_freq`, `steering_entropy`, `session_hours`, `time_of_night_factor`, `speed_consistency`, `is_night_adapted`. Target: `fatigue_score` 0–100.
- **Model:** XGBoost regressor (`objective="reg:absoluteerror"`), exported to `fatigue_model.onnx`.
- **`personalize.py`:** `PersonalizedFatiguePredictor` multiplies the raw score by `5.5 / driver.fatigueBaselineHours`, clips to 0–100, and estimates minutes to critical (score 80).
- **`server.py`:** FastAPI with open CORS.
  - `POST /predict` takes the 8 features, plus optional `fatigueBaselineHours` and `fatigue_rate_per_hour`. It returns `{fatigue_score, predicted_critical_in_minutes}`.
  - `GET /health` returns `model_stats.json` plus `status: ok`.

**Model stats:**

| Metric | Value |
|---|---|
| MAE | 5.14 (target 5.0, narrowly missed) |
| RMSE | 7.80 |
| R² | 0.919 (target 0.88) |
| Size | 378 KB |
| Latency | ~0.018 ms |

**Runtime proxy features.** The engine has no real eye or steering telemetry. It derives perclos, blink rate, head-nod frequency, steering entropy and speed consistency from `hoursDriven / fatigueBaselineHours` plus noise, deliberately **not** from `fatigueScore`, to avoid a feedback loop. `session_hours`, `time_of_night_factor` (unwrapped hour 20–28, matching the training domain) and `is_night_adapted` are real.

**Interpolation.** The model target is smoothly interpolated over the 5 ticks between polls. On any failure or timeout, `modelFatigueTarget` is cleared and the synthetic formula takes over. The UI shows "running in simulation mode (model server offline)".

## 8. Known issues and trade-offs

1. **MAE 5.14, just above the 5.0 target.** About 38% of training targets saturate at exactly 100, and the residual error sits at that clipping boundary. Tuning alone won't fix it.
2. **Leader handoff is practically unreachable.** It needs T1 to hit critical, which takes 40+ simulated hours. It was verified with a direct engine-level test.
3. **The feed shows only the 8 most recent entries.** This is deliberate.
4. **`negotiationLog` grows unbounded** per engine instance. That is fine for a demo.
5. **The "peak driver fatigue" without-AI figure is an illustrative estimate.** It is +15 over the actual peak (`WITHOUT_AI_FATIGUE_BUMP`). There is no real counterfactual. Fuel and range comparisons are derived from real running totals. The "at risk" ETA margin (20 min) is also a guess.
6. **With the model server on, long routes can leave a truck near minimum speed** once predicted fatigue saturates near 100. The critical response correctly keeps firing. This is why e2e runs without the server.
7. **The training data is synthetic**, and the runtime features are proxies.
8. **Field-name typo:** it is `fleetMetrics.platoningEfficiency`, not `platooningEfficiency`. It is kept everywhere.
9. **Small deliberate deviations from spec text.** These include the red worst-case pill, the `[arbitration · fleet]` log format, and the lightened muted-text colour. All are documented in `Progress.md`.
10. **Driver sparklines look smooth** because the pattern generator produces a smooth ramp. This is cosmetic.
11. **Coordination reacts to fatigue beacons only.** Hazard reception was moved out of it into `triggerHazardIfDue` to guarantee a simultaneous fleet response regardless of packet drops.

## 9. What to expect when running

- The default config is quiet cruising.
- Worst case at 680 km, played at 4×, takes about 2 min real time. At 800 km it takes 3+ min. A hazard fires around minute 30, and T2 (Priya) turns amber and then possibly red first.
- The AI-vs-without-AI panel and badges appear only when all 4 trucks reach 100%. Badges are `zero fatigue incidents`, `green convoy` (platooning > 8%) and `on-time delivery`.

## 10. Where to look for more

- `APP_GUIDE.md`: complete current-state reference, with a per-component table.
- `Progress.md`: phase-by-phase log with every design decision, bug fix and measured result.
- `Phase1–9.md`: original build specs.
  1. Shell and design tokens
  2. Simulation engine
  3. Mission Setup
  4. Route map
  5. Agent panel
  6. Fleet Intelligence Bar
  7. ML model
  8. Hazard, handoff, beacon tiers
  9. Polish and e2e
  
  Phase names are inferred from the code, not read from the spec files.
