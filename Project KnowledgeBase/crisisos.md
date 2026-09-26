# CrisisOS — Project Context

**Location:** `C:\Users\kittu\OneDrive\Desktop\My Projects\CrisisOS`
**Git:** yes. Remote `origin` = `https://github.com/vajeedashaik/CrisisOS.git`. ~10 commits (latest: "updated readme"). Last build log entry is dated 2026-04-28.
**Firebase project:** `hackathon-e5b7a` (`.firebaserc`). Also has a `.replit` / `replit.md` from an earlier Replit-hosted stage.
**Type:** Hackathon demo (built for a national hackathon, "needs to look cinematic and professional"). Everything is simulated client-side: no real hotel, sensors, cameras or guests.
**Built across:** 17 spec'd phases (`Prompts/Phase-1.md` … `Phase-17.md`, prompts fed to Claude one by one). Only Phases 1–3 are written up in `progress.md`; later phases are documented in `replit.md` and `README.md`.

## 1. What this project is

**CrisisOS** is an AI-powered hotel emergency-response command platform. The pitch: *"from detection to full coordinated response in under 90 seconds"*, versus about 6 minutes for a manual response.

A scripted crisis (default: **Kitchen Fire, Floor 3 East Wing**) plays out on a simulated clock at the fictional **Horizon Grand Hotel** (address used in the responder brief: 1 Horizon Plaza, Grand Avenue). The command dashboard shows:

- 24 virtual sensors changing over time
- a chain of **three LLM "agents"** that reason about the incident (Detection → Coordination → Emergency Bridge)
- CCTV and thermal-camera simulations
- a predictive risk monitor that raises a pre-alert before the fire is detected
- voice announcements
- guest and staff mobile companion pages that sync live with the dashboard in other browser tabs

Problems it claims to address: delayed detection, radio bottlenecks across 12 staff and 6 floors, no real-time guest accountability for 251 guests, and slow first-responder briefing (solved by a machine-generated JSON brief).

## 2. Headline numbers (from README, checked against code where noted)

| Item | Value |
|---|---|
| Floors | 6 (Lobby, Floor 1–4, Roof) |
| Sensors | 24 (smoke, motion, temperature, CO2) — `src/data/hotelData.js` |
| Guests | 251 (generated programmatically) |
| Staff | 12 (Security, Medical, Manager, Concierge, Housekeeping) |
| Emergency exits | 8, plus primary/secondary evacuation routes per floor |
| LLM agents | 3, run in sequence |
| Thermal cameras | 4, each a 64×48 `Float32Array` heat grid |
| CCTV feeds | 4 canvas feeds at 15 fps |
| Speed modes | 1×, 2×, 4×, 8× |
| Surfaces | 3 (Command Center, Guest, Staff) |
| Size | 33 JSX components, 3 hooks, 7 services; ~4k lines of JS in hooks/services/data (`useCrisisSimulation.js` is 724 lines) |

## 3. Tech stack

**Frontend** (`package.json`, name `crisios`, ESM)
- React 18.3, Vite 5.4, react-router-dom 7 (**HashRouter**, so static hosting works)
- Tailwind CSS 3.4 + PostCSS/autoprefixer, custom dark theme
- Framer Motion 12, Three.js **0.128** (r128, pinned for the 3D view)
- HTML5 Canvas for CCTV and thermal, Web Speech API for voice, BroadcastChannel + localStorage for cross-tab sync
- No Redux/Zustand/UI library. State is `useState`/`useRef` in one hook plus props.
- No tests, no linter, no TypeScript. Plain JavaScript/JSX.

**Backend** (`functions/`)
- Firebase Cloud Functions v2 (`firebase-functions` ^6.5, Node 20, ESM)
- `@google/genai` ^1.13 calling **Gemini 2.0 Flash** (model overridable with `GEMINI_MODEL`)
- Firebase Hosting serves `dist/`

## 4. How to run

```bash
cd CrisisOS
npm install
npm run dev        # http://localhost:5000 (host 0.0.0.0, allowedHosts true — Replit-friendly)
npm run build      # outputs dist/
```

- Guest view: `http://localhost:5000/#/guest`
- Staff view: `http://localhost:5000/#/staff`
- Open those in extra tabs, start the Kitchen Fire scenario from the dashboard, and all three tabs sync.

**Live Gemini (optional):**
- Copy `functions/.env.example` to `functions/.env` and set `GEMINI_API_KEY` (and optionally `GEMINI_MODEL`). A `functions/.env` already exists locally.
- Frontend `.env.example` only has `VITE_GEMINI_ENDPOINT=/api/gemini-agent`.
- **Do not commit or expose `functions/.env`.** Check `.gitignore` before pushing.
- Under `vite dev` there is no `/api/gemini-agent` route, so the call fails and the UI uses the built-in fallback. Live calls only work behind Firebase Hosting (or the emulator) with the rewrite below.

**Deploy:** `firebase deploy` (hosting + functions). `firebase.json` rewrites `/api/gemini-agent` → function `geminiAgent`, and `**` → `/index.html`.

## 5. Routes / user surfaces

| Route | Audience | What it shows |
|---|---|---|
| `/` | Command centre staff | Full dashboard (`src/App.jsx`) |
| `/#/guest` | Guests (mobile) | SOS button, "I'm Safe", floor status, instructions (`src/pages/GuestView.jsx`) |
| `/#/staff` | Field staff (mobile) | Task assignments, floor status, status toggle En Route / On Scene / Resolved, escalation (`src/pages/StaffView.jsx`) |

Cross-tab flow: the dashboard publishes a state snapshot each tick through `simBus`; guest and staff tabs subscribe and send actions back (SOS, I'm Safe, staff status). The sidebar QR (`QRBadge`) points at `/#/guest` for live demos.

## 6. Dashboard layout (`src/App.jsx`)

- **Header:** CrisisOS logo (red), "Horizon Grand Hotel", live clock, status badge (green "ALL SYSTEMS NOMINAL" → red pulsing on alert), risk chip, connectivity chip.
- **Left sidebar (280px):** 4 stat tiles, Sensors/Staff tabs, `SensorGrid` (24 cards with sparklines), QR badge.
- **Central stage tabs:** Floor map 2D / 3D-ISO, CCTV, Thermal, Multi-Sensor (CCTV + Thermal split). Auto-switches to Surveillance for ~6 s when evacuation is first declared.
- **Right panel (320px):** severity gauge, agent status, voice panel, alert feed, scenario picker.
- **Agent panel:** Terminal / Flow toggle; auto-switches to Flow at severity ≥ 5.
- **Bottom panel:** Responder Brief, Dispatch Board, Accountability Tracker, Zone Communications (PA text).
- **Overlays / modals:** IntroSplash (once per session via `sessionStorage`), CountdownOverlay (3-2-1), NarrationBar, StatsOverlay, DemoController (floating, bottom-right), RiskMonitor, ConnectivityPanel, IncidentReview, BeforeAfterComparison, CounterfactualSimulator, PostIncidentReport.

Color tokens: bg `#0a0a0f` / `#111118`, red `#ef4444`, amber `#f59e0b`, green `#22c55e`, blue `#3b82f6`, text `#f1f5f9` / `#94a3b8`. Inter font. Custom keyframes: `crisis-pulse`, `fade-in`, `scanning`, plus a drifting grid background.

## 7. Scenarios (`src/data/crisisScenarios.js`)

1. **Kitchen Fire — Floor 3 East Wing** (`scenario-001`): 9 events, T+0 → T+90s. Temperature 187 °C then 312 °C, smoke 0.78 g/m³, motion surge (14 people), smoke reaches rooms 308–315, guest SOS from Room 312, CO₂ 4200 ppm, CCTV sees a crowd bottleneck at the north stairwell, heat migrates to Floor 4 via ventilation.
2. **Crowd Surge — Lobby**: 7 events, T+0 → T+60s. Crowd density at reception, 74 people at the entrance, CO₂ 1800 ppm in the bar, crush risk, collapsed guest, surge toward the Floor 1 stairwell, congestion confirmed.

Both are exported in `ALL_SCENARIOS`. The scenario picker defaults to the first if a name isn't found.

## 8. Source map

```
CrisisOS/
├── index.html, vite.config.js, tailwind.config.js, postcss.config.js
├── firebase.json, .firebaserc, .replit, replit.md
├── README.md            # long marketing-style README
├── progress.md          # build log, Phases 1–3 only
├── Prompts/Phase-1..17.md   # the original build prompts
├── dist/                # built output
├── functions/           # Firebase function (index.js), package.json, .env(.example)
└── src/
    ├── main.jsx         # HashRouter: /, /guest, /staff
    ├── App.jsx          # dashboard + state orchestration
    ├── index.css
    ├── data/            # hotelData.js, crisisScenarios.js
    ├── hooks/           # useCrisisSimulation.js, useDerivedTimeline.js, useCountUp.js
    ├── services/        # agents, simBus, voiceAnnouncer, riskPredictor, thermalSim,
    │                    #   connectivityManager, edgeFallback
    ├── pages/           # GuestView.jsx, StaffView.jsx
    └── components/      # 26 folders, 33 files (list below)
```

**Components by folder:** AccountabilityTracker, AgentFlow (AgentFlowVisualizer), AgentPanel (AgentOrchestrator, AgentPanel), AlertFeed, AuditLog, CCTVPanel (CCTVGrid, CCTVFeed), Comparison (BeforeAfterComparison), Countdown, Counterfactual (CounterfactualSimulator), Demo (DemoController, IntroSplash, NarrationBar, StatsOverlay), DispatchBoard, EdgeMode (ConnectivityPanel), FloorMap, FloorMap3D, IncidentReview (IncidentReviewPanel), Metrics (IncidentMetrics), PostIncident (PostIncidentReport), QR (QRBadge), ResponderBrief, RiskMonitor (RiskMonitor, FloorRiskHeatmap), SensorGrid, SeverityGauge, Thermal (ThermalCamera, ThermalGrid), Timeline (IncidentTimeline), VoicePanel, ZoneCommunications.

## 9. Core engine: `useCrisisSimulation.js`

The single source of truth for the demo.

- **State:** `simulationStatus` (`idle | running | crisis`), `elapsedSeconds`, `severityScore` (0–10), `evacuationActive`, `accountedGuests`, live `sensors` (value + 10-point history + status), `crisisEvents`, `agentLogs`, `incidentTimeline`, `dispatchedStaff`, `flashRed`, plus pre-stage state (`preStageSeconds` 0–60, `preStageActive`) and thermal-anomaly lead time.
- **Tick:** 1 s `setInterval` (speed multiplier adjusts the interval). Fires scenario events due at the current T. Mutable bookkeeping is in `useRef` to avoid stale closures. Non-triggered sensors get ±8% noise.
- **Severity:** `(highest contribution / 10) × compound`, where compound adds ~12% per additional distinct signal type active within the last 30 s. A full-screen red flash fires the first time severity crosses 7.
- **Evacuation:** auto-triggers at severity ≥ 4. `accountedGuests` drops 247 → ~180, climbs back over ~55 s, and at T+75 the last 4 unaccounted are found → 251. Voice milestones fire at fixed offsets (fire detected → evacuation initiated → all accounted).
- **Pre-stage (Phase 13/14):** a 60 s pre-crisis window in which the risk score builds and the thermal cameras show an IR anomaly before smoke triggers; a "lead time" banner reports how many seconds the IR beat the smoke sensor.
- **Exposed API:** `startSimulation(scenarioName)`, `resetSimulation()`, `triggerManualSOS(floor, room)`, `acknowledgeAlert(alertId)`, plus the state above.
- Publishes a snapshot to `simBus` after each tick; feeds thermal sources to `thermalSim`, context to `riskPredictor`, and milestone lines to `voiceAnnouncer`.

## 10. Services

| File | Role |
|---|---|
| `agents.js` | The 3 LLM agents. `runDetectionAgent`, `runCoordinationAgent`, `runEmergencyBridgeAgent`. Each builds a prompt from live state, POSTs `{agent, payload:{system,userMessage}}` to `/api/gemini-agent` (`VITE_GEMINI_ENDPOINT`), then replays the returned text via `simulateStream` (3 chars / 18 ms) for a typing effect. On any failure (except AbortError) it streams a **hard-coded fallback** for that agent. |
| `simBus.js` | Cross-tab pub/sub. Channel `crisisos-bus`, localStorage keys `crisisos:bus:lastEvent` and `crisisos:bus:state` as a fallback and for late-joining tabs. |
| `voiceAnnouncer.js` | Web Speech queue (FIFO, no overlap), 3 priority profiles (normal / calm / critical), best-English-voice selection on `voiceschanged`, persistent mute in localStorage, observer callbacks for speaking state. |
| `riskPredictor.js` | `computeRiskSnapshot(ctx)` → composite 0–100 score from environmental, occupancy, behavioral and external factors, per-floor. `PRE_ALERT_THRESHOLD = 71`; exports `RISK_LEVEL_COLOR`, `PRE_ACTIONS`. |
| `thermalSim.js` | 4 cameras (F3 Kitchen, F3 East Wing, F2 Common Area, Lobby), 64×48 grids, `diffuse()` heat spread, fire cells up to ~312 °C, `colorForTemp()` false-color map (blue ~15 °C → red 80 °C+), `hottestCell()` for the readout. |
| `connectivityManager.js` | Pub/sub state machine: cloud / edge / degraded. `simulateDisconnect/Reconnect/Degraded`, sync-queue (`queueEvent`), `SYSTEM_CHECKLIST`. |
| `edgeFallback.js` | Pre-cached crisis protocols per scenario (`EDGE_PROTOCOLS`, `pickProtocol`) used when "offline". |

## 11. The three-agent pipeline

Shared settings: Gemini 2.0 Flash, temperature 0.25, system prompt + user message concatenated into one `generateContent` call in `functions/index.js`.

1. **Detection & Analysis** — input: last 10 events + up to 12 anomalous sensor readings. Output sections: INCIDENT TYPE, LOCATION, SPREAD PATTERN, IMMEDIATE THREATS, CONFIDENCE LEVEL.
2. **Coordination & Response** — input: detection result, staff list, guest count, evacuation routes. Output: EVACUATION PLAN (per floor), STAFF ASSIGNMENTS, ZONE COMMUNICATIONS, PRIORITY ACTIONS, ESTIMATED COORDINATION TIME.
3. **Emergency Bridge** — input: both prior outputs + hotel metadata. Output: **JSON first-responder brief** (incident type, severity 1–10, known hazards, blocked/clear routes, recommended entry point, on-scene staff contacts, unaccounted persons, special considerations, hotel address, floor-plan note). Rendered by `ResponderBrief.jsx`.

`AgentOrchestrator.jsx` sequences the three calls (with `delay` pacing) and drives `AgentPanel` and `AgentFlowVisualizer` (3-node graph with packet animation, payload inspector and Gantt).

Cloud function (`functions/index.js`, export `geminiAgent`): POST only, CORS `*`, requires `GEMINI_API_KEY` env, validates `agent`, `payload.system`, `payload.userMessage`, returns `{agent, text}`; 400 / 405 / 500 / 502 for errors. The API key never reaches the browser.

## 12. Feature modules (phase → what it added)

| Phase | Feature |
|---|---|
| 1 | Foundation: Tailwind theme, global CSS, layout, `hotelData`, `crisisScenarios` |
| 2 | SVG floor map: 800×480 viewBox, guest dots, sensor diamonds, staff dots, exit arrows, animated evac routes, hazard zone overlays, floor status bar |
| 3 | Sensor grid + simulation engine (`useCrisisSimulation`) |
| 4 | Multi-agent LLM pipeline (`agents.js`, agent panel) |
| 5 | Responder brief + dispatch board |
| 6 | Incident timeline + audit log |
| 7 | Before/After comparison + counterfactual ("what if no AI") |
| 8 | Polish and final layout assembly |
| 9 | Guest/Staff mobile views, cross-tab sync, QR badge |
| 10 | Demo controller, presentation mode, narration bar, stats overlay, intro splash, README |
| 11 | Voice announcer + VoicePanel + speaking indicators on Zone Communications |
| 12 | Canvas CCTV grid (4 feeds, expand-on-click, vision-agent sweep at severity ≥ 4) |
| 13 | Predictive risk monitor, per-floor heatmap, header chip, pre-alert modal |
| 14 | Thermal cameras + Multi-Sensor tab + lead-time banner |
| 15 | Agent flow visualizer (Terminal/Flow toggle) |
| 16 | Three.js isometric 6-floor 3D map (fire particles, evac tubes, drag-orbit), 2D / 3D-ISO toggle |
| 17 | Edge / offline mode: connectivity chip + modal + banner, "Simulate Network Failure / Restore" in Demo Controller, edge-fallback badge in agent panel |

Also present: Drill Mode ("Training Simulation" badge), Presentation Mode (collapses sidebars), speed control, Incident Review (timeline, metrics, audit log, before/after), Post-Incident Report (with System Resilience section).

## 13. Suggested demo script

1. Open the dashboard; the intro splash plays once per session.
2. Open `/#/guest` and `/#/staff` in other tabs.
3. Start Kitchen Fire from the scenario picker. Watch the countdown, risk pre-alert and thermal anomaly precede the smoke sensor.
4. Severity climbs, evacuation is declared, the CCTV tab auto-opens, voice announcements play.
5. Agents stream Detection → Coordination → Responder Brief.
6. Press SOS on the guest tab; the alert appears on the command view instantly.
7. Toggle Drill Mode and Presentation Mode; try "Simulate Network Failure" to show edge mode.
8. After resolution, open Incident Review, Comparison and Counterfactual.

## 14. Things to know / gotchas

- **Docs disagree about live LLM calls.** `replit.md` says live calls were removed and responses are pre-scripted. That is out of date: `agents.js` does call `/api/gemini-agent`, which goes to the Firebase function, and falls back to scripted text on failure. Trust the code.
- **In practice the demo usually runs on fallback text.** `npm run dev` has no `/api` route, so live Gemini only works when deployed to Firebase Hosting (or via the Functions emulator with a proxy) with a valid `GEMINI_API_KEY`.
- Fallback responses are hard-coded for the Kitchen Fire scenario (Marcus Reid, Sarah Chen, James Okafor, Maria Santos, Room 312/315, etc.), so the Crowd Surge scenario shows fire-flavored fallback text if the API is down.
- README claims of "6 min → 90 s", "≈60% guest accountability" for manual response, etc. are scripted comparison numbers, not measurements.
- The `README.md` says "React 33 components / 3 hooks / 7 services" and lists a "Kitchen Fire" trigger in the right panel; `progress.md` stops at Phase 3 even though Phases 4–17 exist. The git log shows the actual history.
- Vite dev port is 5000 (Replit legacy), not the default 5173. The Vite config's `progress.md` still mentions 5173 from Phase 1.
- Three.js is pinned at 0.128 because the Phase 16 prompt targeted r128.
- `.claude/settings.local.json` only allows a few scaffolding commands (npm create, create-vite, vite).
- License note in README: "Internal demo project — Horizon Grand Hotel, 2026."
