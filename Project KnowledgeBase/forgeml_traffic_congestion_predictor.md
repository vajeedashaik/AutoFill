# ForgeML Traffic Congestion Predictor — Project Context

**Location:** `C:\Users\kittu\OneDrive\Desktop\My Projects\ForgeML Traffic Congestion Predictor`
(outer folder holds docs; the actual code repo is the nested `Urban-Traffic-Congestion-Prediction/`)
**GitHub origin:** `https://github.com/vajeedashaik/Urban-Traffic-Congestion-Prediction.git`
**Type:** Personal/portfolio project, listed as **Project 1 on the resume** (alongside CrisisOS and MetaModEnv). Git author: `vajeedashaik`.
**Status:** Complete and working; last commit 2026-05-10 ("docker and readme changes"). Local working copy has small uncommitted changes (a whitespace-only edit in `agents.py`, line-ending noise in `requirements.txt` / `supabase_setup.sql`) and two untracked Streamlit log files.

## 1. What this project is

An end-to-end ML system that predicts **traffic congestion on the Metro I-94 interstate corridor (Minneapolis–Saint Paul, MN)** as one of four levels — Low / Medium / High / Severe — from time of day, weather, and recent traffic volume. Each prediction is wrapped with a plain-English explanation and a recommended action plan for traffic operators.

It has four loosely-coupled layers:

1. **ML layer** — data cleaning, 27-feature engineering, 3-model comparison, permutation feature importance, serialized `model.pkl`.
2. **Agent/explanation layer** — a 4-agent orchestration pipeline (Data → Prediction → Insight → Recommendation) plus a rule-based explanation engine (no LLM, no external AI API; fully offline).
3. **Presentation layer** — an 11-page dark-themed Streamlit dashboard with 18 Plotly/Folium chart types and an interactive I-94 corridor map.
4. **Integration layer** — 4 FastMCP servers (22 tools, 11 resources) that expose the model to Claude Desktop, plus optional Supabase cloud data backend and Docker/Heroku-style deployment files.

Headline numbers: **93.69% accuracy / 0.9368 weighted F1**, ~5 s CPU training, 1–3 ms inference, ~6,200 lines of Python (README claims "~2,500+", which is stale).

## 2. Dataset

| Property | Value |
|---|---|
| Source | UCI "Metro Interstate Traffic Volume" (MnDOT ATR station 301, westbound I-94 near Minneapolis; weather from OpenWeatherMap) |
| File | `Metro_Interstate_Traffic_Volume.csv` |
| Span | Oct 2012 – Sep 2018, hourly |
| Rows | 48,204 raw → 48,187 after dedup (17 duplicates dropped) |
| Raw columns (9) | `holiday`, `temp` (Kelvin), `rain_1h` (mm), `snow_1h` (cm), `clouds_all` (%), `weather_main`, `weather_description`, `date_time`, `traffic_volume` (target basis) |
| Split | 85/15 stratified random split (~41,059 train / 7,128 test), `random_state=42` |
| Volume range | 0 – 7,280 vehicles/hour |

**Data source switch:** `load_raw()` in `data_processing.py` reads from **Supabase** if `SUPABASE_URL` and `SUPABASE_KEY` env vars are set, otherwise falls back to the local CSV automatically.

### Cleaning steps (`clean()`)
- Drop ~17 duplicate rows.
- `temp == 0` K (impossible sensor error) → imputed with the **median temperature for that hour-of-day**.
- `rain_1h` clipped to max 50 mm/hr (one raw outlier is ~9,831 mm/hr).
- `holiday` NaN → string `"None"`.

## 3. Target definition

`traffic_volume` is binned (`CONGESTION_BINS = [0, 1000, 2500, 4500, 7300]`) into:

| Class | Code | Vehicles/hr | Meaning |
|---|---|---|---|
| Low | 0 | 0–1,000 | Free flow |
| Medium | 1 | 1,001–2,500 | Some slowdowns, +10–20% travel time |
| High | 2 | 2,501–4,500 | Heavy, +30–50% travel time |
| Severe | 3 | 4,501–7,280 | Gridlock possible, travel time may double |

Framed as classification rather than regression because tiers map directly to operator actions.

## 4. Feature engineering — 27 features (`FEATURE_COLS` in `data_processing.py`)

`FEATURE_COLS` is the single authoritative column order used everywhere (app, agents, MCP servers, map).

| Group | Features | # |
|---|---|---|
| Time | `hour`, `day_of_week`, `month`, `year`, `day_of_year` | 5 |
| Rush hour | `is_morning_rush` (7–9), `is_evening_rush` (16–18), `is_rush_hour`, `rush_type` (0/1/2) | 4 |
| Cyclical | `hour_sin`, `hour_cos`, `month_sin`, `month_cos` | 4 |
| Calendar | `is_weekend`, `is_holiday` | 2 |
| Weather | `temp_celsius`, `rain_1h`, `snow_1h`, `clouds_all`, `is_raining`, `is_snowing`, `weather_severity`, `weather_code` | 8 |
| Lag / rolling | `traffic_lag_1h`, `traffic_lag_24h`, `traffic_rolling_3h`, `traffic_rolling_6h` | 4 |

Notes:
- `weather_severity` = rain bucket via `pd.cut([0, 2.5, 7.6, 55])` + snow flag (0–4 ordinal).
- `weather_code` is an ordinal map of `weather_main` (Clear=0, Clouds/Haze=1, Smoke/Drizzle=2, Rain/Mist=3, Fog/Thunderstorm/Snow=4, Squall=5). The MCP core uses a slightly richer map than the training pipeline.
- Lag/rolling features are built after sorting by `date_time` using `.shift(1)` before rolling windows (no same-hour leakage); first rows filled with the median.
- Sin/cos encoding (`hour_cos = cos(2π·hour/24)`) makes 23:00 adjacent to 00:00.

## 5. Model training & results (`model_training.py`)

Three models trained on the same split; the winner is chosen by **weighted F1**.

| Model | Key hyperparameters | Accuracy | Weighted F1 | Train time |
|---|---|---|---|---|
| **HistGradientBoostingClassifier** (winner) | max_iter=300, lr=0.08, max_depth=8, min_samples_leaf=20, l2=0.1, early stopping (val 10%, patience 20) | **93.69%** | **0.9368** | 5.0 s |
| RandomForest | 300 trees, depth 15, min_leaf 10, balanced weights | 92.54% | 0.9259 | 4.4 s |
| ExtraTrees | 300 trees, depth 15, min_leaf 10, balanced weights | 91.06% | 0.9111 | 2.1 s |

**Per-class (best model, 7,128 test rows):** Low P/R/F1 97.6/98.4/98.0 · Medium 89.2/87.8/88.5 · High 90.7/90.4/90.5 · Severe 95.3/95.6/95.4.

**Confusion matrix (rows = actual):**
```
        Low   Med  High  Sev
Low    1622    27     0    0
Med      39   859    80    0
High      1    76  1862  121
Sev       0     1   111 2430
```
Zero Low↔Severe confusion; nearly all errors are adjacent-class (Medium↔High, High↔Severe).

**Feature importance** — permutation importance (5 repeats, up to 5,000 test rows, `f1_weighted` scoring):

| Rank | Feature | Share |
|---|---|---|
| 1 | `traffic_lag_1h` | 36.4% |
| 2 | `hour_cos` | 21.9% |
| 3 | `hour` | 15.3% |
| 4 | `day_of_week` | 6.6% |
| 5 | `hour_sin` | 2.5% |
| 6–7 | `traffic_rolling_3h` / `_6h` | 1.7% / 1.4% |

Top 3 = 73.6%; lag + time-of-day ≈ 76%; **all 8 weather features combined < 1%**.

**Artifacts written:** `model.pkl` (joblib), `feature_importance.csv`, `model_results.json` (accuracy/F1/time), `model_comparison.json` (adds confusion matrices + per-class reports, feeds the Model Comparison page).

## 6. Agent pipeline (`agents.py`, ~200 lines)

Orchestrated by `AgentOrchestrator.run_prediction_pipeline()`, with a shared timestamped `AgentLog` shown in the UI's "Agent Execution Trace" expander:

1. **DataAgent** — profiles dataset: record count, date range, class distribution, zero-traffic anomalies, % of severe congestion during rush hours.
2. **PredictionAgent** — builds the 27-key feature array, `predict()` + `predict_proba()`, measures inference ms, returns label/confidence/all class probabilities.
3. **InsightAgent** — calls `explain_prediction()` in `insights.py` for contributing factors.
4. **RecommendationAgent** — base actions per level + contextual add-ons (rush + High/Severe → push mobile alert; rain > 2 mm/hr → wet-weather speed signs; hour < 6 → maintenance window; confidence < 60% → verify with live sensors). Assigns urgency: LOW / NORMAL / HIGH / CRITICAL.

## 7. Explanation engine (`insights.py`)

Pure rule-based (if/else), instant and fully auditable. `explain_prediction()` evaluates rule blocks: rush hour (HIGH), weekend (REDUCING), weekday (MODERATE), holiday (REDUCING), heavy rain >5 mm/hr (HIGH), light rain 0.5–5 (MODERATE), snow (HIGH), poor visibility `weather_code ≥ 4` (HIGH), prior hour >4,500 (HIGH), prior hour <1,000 (REDUCING), late night hour <5 or ≥23 (REDUCING). It then assembles a narrative sentence. README cites "9 rule blocks"; the code has ~11 conditions including weekday and late-night.

`generate_system_insights()` produces 6 data-driven insight cards (rush-hour dominance, weekday vs weekend, holiday relief, weather effects, seasonal peak, temporal momentum). `get_recommendations_for_level()` returns hardcoded action lists per level.

## 8. Streamlit dashboard (`app.py`, 1,357 lines)

Dark theme (bg `#080c18`, purple accent `#7c3aed`, CSS in `styles.py`, config in `.streamlit/config.toml`). Uses `st.cache_data` / `st.cache_resource` for data, model, and the MCP bridge. Sidebar has page nav, model-status indicator, and a **Train / Retrain** button that runs the full training pipeline and clears cache.

**11 pages:** Overview · Traffic Patterns (hourly/daily/weather/seasonal tabs) · Predict (form → confidence, factors, action plan, agent trace) · AI Insights · Model Report · Model Comparison (3-model accuracy/F1, confusion matrices, per-class F1, radar chart) · Traffic Map · Heatmaps · Weather Analysis · Risk Scoring (24-h risk forecast) · MCP Live Intelligence (Live Now / 24h Forecast / Corridor Status / MCP Servers tabs).

`analytics.py` (674 lines) holds ~18 Plotly chart builders plus `compute_24h_risk()`.

## 9. I-94 corridor map (`map_view.py`)

Folium + OpenStreetMap tiles (no API key). **16 hand-defined segments** from Plymouth/I-494 west to East Saint Paul/I-494 east, each with coordinates, exit label and a **spatial congestion factor 0.70–1.50** applied to the model's base prediction (this is how one sensor is extrapolated to a corridor). Worst bottleneck: Downtown Minneapolis – I-35W Interchange (1.50); next Downtown Minneapolis West (1.40), Downtown St. Paul Core (1.30). Segments are color-coded green/yellow/orange/red; Severe segments get animated `AntPath` dashed lines; popups show exit, level, factor. **The factors are hand-set heuristics, not learned from data** — only one physical sensor exists.

## 10. MCP server suite (`mcp_servers/`)

FastMCP, stdio transport, registered in `mcp_config.json` for Claude Desktop. All servers share `core.py` (lazy `lru_cache` singletons for model/DataFrame/feature importance, plus `build_feature_vector()` and `run_prediction()` as the single source of truth).

| Server | Tools | Resources | Focus |
|---|---|---|---|
| `prediction_server` | 5: `predict_congestion`, `predict_batch_hours`, `compute_risk_forecast`, `get_feature_vector`, `predict_now` | 3 | Predictions and risk forecasting |
| `analytics_server` | 8 (hourly profile, distribution, weather impact, rush stats, weekday vs weekend, segment risk, bottlenecks, dataset overview) | 3 | Dataset statistics |
| `insights_server` | 5 (explain, action plan, system insights, level description, narrative) | 3 | Explanations and recommendations |
| `map_server` | 4 (corridor map, bottlenecks, segment history, list segments) | 2 | Spatial intelligence |
| **Total** | **22** | **11** | |

`bridge.py` (`MCPBridge`) lets Streamlit call the same core functions **directly and synchronously** instead of running a real MCP client/subprocess (Streamlit reruns the whole script on each interaction). Resource URIs look like `traffic://model/info`, `traffic://map/segments`.

## 11. Deployment & infrastructure

- **Docker:** `Dockerfile` (python:3.12-slim, port 8501, `/healthz` healthcheck) and `docker-compose.yml` with 5 services (dashboard + 4 MCP servers), mounting model artifacts so retraining persists.
- **Heroku-style:** `Procfile`, `runtime.txt` (python-3.12.0).
- **Supabase (optional):** `supabase_setup.sql` (table `traffic_volume`, indexes on `date_time` and hour, RLS read-only for anon key) and `supabase_upload.py` (batched one-time CSV upload using the service-role key; supports `--csv`, `--batch-size`, `--dry-run`). Env in `.env.example` (`SUPABASE_URL`, `SUPABASE_KEY`, optional `SUPABASE_SERVICE_KEY`).
- **Dependencies (`requirements.txt`):** pandas, numpy, scikit-learn==1.7.2 (pinned to match the pickle), joblib, streamlit≥1.32, plotly≥5.18, folium≥0.15, streamlit-folium≥0.18, mcp≥1.0, supabase≥2.3, python-dotenv. No GPU needed.
- The local `.venv` is Python 3.13 on Windows.

## 12. Repo layout

```
ForgeML Traffic Congestion Predictor/
├── README.md                              # Polished overview (outer copy)
├── Interview_QA_Traffic_Congestion_Predictor.md   # Long-form interview Q&A
├── Resume_Projects_QA_Cheatsheet.md       # One-liner Q&A for 3 resume projects
└── Urban-Traffic-Congestion-Prediction/   # the git repo
    ├── app.py  agents.py  data_processing.py  model_training.py
    ├── insights.py  analytics.py  map_view.py  styles.py
    ├── mcp_servers/ (core, bridge, prediction/analytics/insights/map servers)
    ├── model.pkl  feature_importance.csv  model_results.json  model_comparison.json
    ├── Metro_Interstate_Traffic_Volume.csv
    ├── mcp_config.json  requirements.txt  runtime.txt  Procfile
    ├── Dockerfile  docker-compose.yml  .env.example
    ├── supabase_setup.sql  supabase_upload.py
    └── README.md  PROJECT_EXPLAINED.md  SETUP_AND_GUIDES.md
```

## 13. How to run

```bash
cd Urban-Traffic-Congestion-Prediction
pip install -r requirements.txt
streamlit run app.py                 # http://localhost:8501
python model_training.py             # optional retrain (~5–60 s)
python -m mcp_servers.prediction_server   # and analytics / insights / map
docker compose up --build            # dashboard + 4 MCP servers
```
For Claude Desktop, copy the `mcpServers` block from `mcp_config.json` into `claude_desktop_config.json` and set `cwd` to the repo root.

## 14. Key findings (talking points)

1. Previous-hour volume (`traffic_lag_1h`) is the #1 predictor — temporal momentum.
2. Time-of-day features + lag ≈ 76% of predictive power.
3. Weather is nearly irrelevant (< 1%); I-94 congestion is time-driven.
4. Weekday avg 3,812.7 vs weekend 2,745.3 vehicles/hr (~27% gap); rush hours avg 5,114.2 (~34% premium).
5. Rush windows (7–9 AM, 4–6 PM) dominate High/Severe classes; August is the peak month.

## 15. Known limitations & inconsistencies (be upfront in interviews)

- **Random stratified split, not a temporal split** — random rows adjacent in time leak between train/test, so 93.7% is likely optimistic for true forecasting.
- **Lag features at inference:** the dashboard/MCP predictors require the user to supply lag/rolling values (or defaults), so "forecasting" ahead relies on assumed recent traffic. The 24-hour risk forecast is therefore an approximation.
- **Single sensor** — the 16-segment map uses hand-tuned multipliers, not measured per-segment data.
- **Stale data (2012–2018)** — would need retraining before use on current traffic (post-remote-work commute shifts).
- **`year` feature** is set to the current year at inference, which is outside the training range (2012–2018) — a mild extrapolation risk.
- **Rule-based explanations** describe general drivers; they are not model-derived (no SHAP/LIME) and can diverge from what the model actually weighed.
- **Doc inconsistencies:** `PROJECT_EXPLAINED.md` is out of date (5 pages, 93.64% / 14.8 s numbers); `SETUP_AND_GUIDES.md` says "10 pages" and "Random Forest" for the trained model and links a different GitHub path; `mcp_config.json` says "93.6% F1"; README says "~2,500+ lines" (actual ~6,200) and "9 rule blocks". Authoritative numbers come from `model_results.json` (93.69% / 0.9368).
- Some tool docs reference `Python 3.9+`, while the Docker/Heroku setup uses 3.12 and the local venv 3.13.

## 16. Related interview material (already in the project folder)

- `Interview_QA_Traffic_Congestion_Predictor.md` — long-form Q&A grounded in the codebase (overview, data, features, model selection, agents, MCP, limitations).
- `Resume_Projects_QA_Cheatsheet.md` — 30-second pitch and one-liners for this project plus CrisisOS and MetaModEnv.

**30-second pitch:** "ML system that predicts I-94 congestion (Low/Med/High/Severe) from 6 years of sensor data at 93.7% accuracy, with a 4-agent explanation pipeline, an 11-page Streamlit dashboard with an interactive corridor map, and 4 MCP servers exposing 22 tools so Claude Desktop can query it in natural language."
