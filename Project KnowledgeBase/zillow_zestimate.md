# Zillow Zestimate — Project Context

**Location:** `C:\Users\kittu\OneDrive\Desktop\My Projects\zillow_zestimate`
**GitHub origin:** `https://github.com/vajeedashaik/zillow_zestimate.git`
**Type:** Personal/portfolio project (built for a student project expo), not a production system.

## 1. What this project is

An interactive Next.js showcase site built around a stacked-ensemble ML model that predicts `logerror = log(Zestimate) − log(SalePrice)` for the classic **Zillow Prize** dataset (~2.9M Southern California properties, ~90K labeled transactions across LA, Orange, and Ventura counties). Positive `logerror` means Zillow overestimated the sale price (seller advantage); negative means Zillow underestimated (buyer opportunity).

The project has three loosely-coupled layers:

1. **ML layer** (real, but not deployed in this repo) — a Colab-notebook-derived training pipeline plus a well-engineered inference/serving/batch-scoring toolkit around it. The trained model artifacts are never committed, so this layer isn't runnable end-to-end without re-running the notebook first.
2. **Live demo/cloud layer** (real, modest-scale, synthetic-data-backed) — 20 real AWS + GCP services wired up and seeded with generated data, fronting a separately pre-deployed AWS Lambda that serves "live" predictions via a hand-tuned approximation formula (not the real ensemble).
3. **Frontend showcase layer** (the bulk of the visible code) — an 8-page Next.js/React/TypeScript site telling the ML story and offering interactive tools, built almost entirely from a sequence of AI-coding-assistant prompt-spec files checked into the repo.

## 2. Model results

5-fold CV RMSE on `logerror`, out-of-fold stacking (no leakage):

| Model | CV RMSE |
|---|---|
| Ridge (baseline) | 0.0791 |
| ExtraTrees | 0.0763 |
| XGBoost | 0.0754 |
| CatBoost | 0.0751 |
| LightGBM (Bayesian-tuned, Optuna 20 trials) | 0.0749 |
| **Ensemble Stack (XGB+LGB+CatBoost → Ridge meta-model)** | **0.0742** (best, −6.2% vs baseline) |

County error analysis: LA +0.0142 (overestimated), Orange −0.0089 (slightly underestimated), Ventura +0.0031 (near-accurate). Older homes (60+ yrs) trend negative; newer trend positive.

**12 engineered features** (mirrored in both Python and `lib/featureEngineering.ts`): `property_age`, `tax_ratio`, `structure_land_ratio`, `living_area_ratio`, `month_sin`/`month_cos` (cyclical transaction month), `geo_cluster` (KMeans k=25 on lat/lon), `geo_cluster_density`, `cluster_mean_tax`, `geo_cluster_te` / `county_te` (5-fold CV target encoding, leakage-safe), `age_cluster_interaction`.

## 3. Frontend (Next.js) — the showcase site

**Stack:** Next.js 16 (App Router), React 19, TypeScript 5, Tailwind CSS 3 (dark theme), Recharts 2 (all charts), Leaflet + react-leaflet + leaflet.heat (geo heatmap), Firebase JS SDK v12 (Firestore client), Lucide icons. No test framework, no state-management library, no UI component library — all custom Tailwind components. `package.json` name is `zillow-zestimate-heatmap`, suggesting the project started as just the heatmap feature and grew into the full site.

Scripts: `npm run dev` / `build` / `start` / `lint`.

**Routes:**

| Route | Purpose | Live cloud services used |
|---|---|---|
| `/` | Main showcase (Hero, ProblemStatement, Pipeline, FeatureEngineering, CountyAnalysis, ModelComparison, StackingSection, SHAP) | BigQuery (county chart) |
| `/demo` | Live prediction, full ML output, SHAP bar chart, CI, dollar mispricing | Lambda · Firestore · SNS · SES |
| `/risk-checker` | Consumer risk tool (buyer/seller) | Lambda · Firestore · SES |
| `/explainer` | SHAP waterfall + importance, "Surprise me" | Firestore write |
| `/heatmap` | Leaflet heatmap, ~12,500 simulated properties, 25 KMeans cluster overlays | none (client-side mock data) |
| `/monitoring` | Live prediction counter, Firestore feed, RMSE tracking, feature drift, alert log, drift simulation | Lambda (count) · Firestore · CloudWatch |
| `/pipeline` | Batch scoring architecture, Step Functions diagram, animated progress, CSV download, Cloud Run scorer | CloudFront · Cloud Run |
| `/services` | Index of all 20 cloud services | — |

**Key directories:**
- `app/` — App Router pages, one folder per route above.
- `components/` — ~20 components (`CountyAnalysis.tsx`, `GeoHeatmap.tsx`/`HeatmapSidebar.tsx`, `ModelComparison.tsx`/`StackingSection.tsx`/`SHAPSection.tsx`, `PredictionCard.tsx`/`PredictionGauge.tsx`/`PriceRangeCard.tsx`/`RiskBadge.tsx`/`NeighbourhoodContext.tsx`, `WaterfallChart.tsx`/`ShapBarChart.tsx`, `Navbar.tsx`/`Hero.tsx`/`Footer.tsx`, plus a `monitoring/` subfolder with `RmseChart.tsx`, `FeatureDriftPanel.tsx`, `AlertLog.tsx`).
- `lib/` — `firebase.ts` (Firestore init from `NEXT_PUBLIC_FIREBASE_*` env vars), `featureEngineering.ts` (JS mirror of the Python feature pipeline), `mockPredictor.ts` / `riskPredictor.ts` (client-side fallback prediction formulas), `shapMock.ts` (mock SHAP generator, largest lib file).
- `data/` — `mockData.ts` (county stats, model journey, waterfall example), `mockGeoData.ts` (~12,500 simulated heatmap points), `mockMonitoringData.ts` (RMSE history, drift, alert log, prediction volume simulation).
- `types/` — only ambient TypeScript declarations for the untyped `leaflet.heat` plugin; no shared domain-model types file exists (domain shapes live as untyped JS objects in `data/`/`lib/`).

**Design philosophy:** every cloud call is fallback-first / optimistic — hardcoded fallback data always renders first, live fetches upgrade it if they succeed, and try/catch ensures a down service never breaks the UI.

## 4. ML / Python layer

- **`model.py`** — a verbatim export of the original Google Colab training notebook (`MCDS Review1.ipynb`), including Kaggle download magic commands. This is the canonical training pipeline: loads `train_2016_v2.csv` + `properties_2016.csv`, memory-optimizes dtypes, imputes/clips outliers, engineers the 12 features (incl. `KMeans(n_clusters=25)` spatial clustering and 5-fold CV target encoding), drops multicollinear features (corr > 0.95, VIF > 20), trains Ridge/ExtraTrees baselines then XGBoost/LightGBM/CatBoost (LightGBM Bayesian-tuned via Optuna), builds the OOF stacking ensemble with a Ridge meta-model, runs SHAP analysis, and produces `final_submission.csv`. Not a clean standalone script — it's notebook history and the "spec" that the inference code below must replicate exactly.
- **`save_models.py`** — meant to be run inside the same notebook session right after training; dumps 13 joblib artifacts into `./models/` (geo_scaler, kmeans, xgb/lgb/cat models, meta_model, feature_columns, geo_cluster_means, county_means, cluster_density, cluster_agg, train_medians, global_mean). **`models/` is empty except `.gitkeep` in this repo** — no trained artifacts are committed.
- **`feature_engineering.py`** — `ModelArtifacts` dataclass (shared artifact container) + `engineer_features()` / `engineer_features_batch()`, replicating `model.py`'s feature logic at inference time for a single property dict.
- **`model_loader.py`** — `load_artifacts()` loads the 13 joblib files into a `ModelArtifacts` instance; raises clear `FileNotFoundError`/`RuntimeError` so the API can report "degraded" instead of crashing.
- **`main.py`** — FastAPI app ("Zillow Zestimate Residual Predictor" v1.0.0). Loads artifacts at startup via a `lifespan` context manager. Endpoints: `GET /`, `GET /health`, `GET /model/info`, `POST /predict`, `POST /predict/batch` (up to 1000 properties), and a mocked `POST`/`GET /score` (seeded random batch-status response). Since no model artifacts are committed, this service starts in degraded mode unless someone re-runs the notebook and populates `models/`.
- **`schemas.py`** — Pydantic v2 models: `PropertyInput` (tax fields, physical attributes, lat/lon in real decimal degrees, `transactiondate`, `fips` restricted to {6037 LA, 6059 Orange, 6111 Ventura}), `BatchPropertyInput`, `PredictionResponse` (`logerror`, `ci_lower`, `ci_upper`, `interpretation`), `BatchPredictionResponse`, `HealthResponse`, `ModelInfoResponse`.
- **`batch_scorer.py`** — production-grade chunked batch scoring pipeline for the full 2.9M-row `properties_2016.csv`: vectorized per-chunk feature engineering + ensemble prediction, outputs one Parquet file per county (`fips_6037.parquet`, etc.) with schema `parcelid, fips, geo_cluster, predicted_logerror, percentile_rank`, plus `run_metadata.json` and a log file. CLI: `--input --output-dir --chunk-size --models-dir`.
- **`lookup.py`** — fast in-memory lookup service over the Parquet output (`load_predictions(parcelid)`, `get_cluster_stats()`, `lookup_batch()`), lazily cached.
- **`benchmark.py`** — benchmarks both the batch-scoring pipeline (across chunk sizes) and lookup latency (cold load, single/batch lookup percentiles); writes `benchmark_results.json`.
- **`generate_sample_csv.py`** — generates a small synthetic `sample_predictions.csv` (25 rows) using a hand-tuned pseudo-model formula, with an optional `--upload` to S3/CloudFront.
- **Seed scripts** (`seed_all.py`, `seed_aws_services.py`, `seed_dynamodb.py`, `seed_firestore.py`, `seed_gcp_services.py`, `setup_bigquery.py`) — populate the 20 live cloud services with synthetic data so the site's dashboards and feeds look active. `seed_all.py` orchestrates the others with `--aws-only`/`--gcp-only`/`--skip-deploy` flags.

**Python dependencies:** `fastapi`, `uvicorn`, `pydantic`, `joblib`, `numpy`, `pandas`, `scikit-learn`, `xgboost`, `lightgbm`, `catboost` (serving); plus `pyarrow`, `tqdm`, `psutil` for the batch pipeline. Real SHAP (`shap` package) is used only in the training notebook — the live site uses a JS-side mock SHAP approximation, not real SHAP at inference.

## 5. Cloud architecture — 20 live services

**AWS (10, us-east-1):**
Lambda (Prediction — stacked-ensemble-style inference via a hardcoded approximation), Lambda (SES) — HTML email reports, Lambda (Count) — reads DynamoDB total, DynamoDB (`zillow-predictions`, 500+ historical records), S3 (model artifacts/CSV), CloudWatch (dashboard + 3 alarms), SNS (`zillow-price-alerts`, fires when logerror > 0.08), SQS (`zillow-prediction-queue`), CloudFront (serves `sample_predictions.csv`), Step Functions (`ZillowZestimate-MLPipeline`, 5-stage state machine), SES (`ZillowPredictionReport` template), Amplify (hosts the Next.js app).

**GCP (10, us-central1, project `cap-expt-492003`):**
Firestore (project `zillow-f19c7`, real-time prediction feed via `onSnapshot`), BigQuery (`zillow_data.county_stats` + `predictions`), Cloud Run (`zillow-scorer` — see note below), Cloud Functions (`model-health`), Cloud Storage (`zillow-demo-vajeeda` bucket), Cloud Monitoring (2 uptime checks + alerting), Cloud Logging, Cloud Scheduler (`zillow-daily-batch-scorer`, 2 AM daily, + weekly health report), Cloud Build (CI/CD via `cloudbuild.yaml`), Secret Manager (10 secrets).

**Request flow:** form submit → AWS Lambda (Prediction) → DynamoDB write + GCP Firestore write (live feed) + SNS alert (if logerror > 0.08) + CloudWatch logging → optional Lambda (SES) → email.

**Important caveat:** the "Cloud Run scorer" (`zillow-scorer/main.py`) is a **minimal Flask stub** (not the real ML pipeline) that returns the same mocked, seeded-random JSON as `main.py`'s `/score` route (`rows_processed: 2_900_000`, `rmse: 0.0742`, etc.). `cloudbuild.yaml` only builds/deploys this stub, not `batch_scorer.py`. Similarly, the "model-health" Cloud Function (`cloud_function_health/main.py`) returns a mostly-static JSON payload. The actual heavy 2.9M-row batch scoring pipeline (`batch_scorer.py`) exists as real, well-engineered code but is not what's continuously deployed to the cloud.

**Deployment/build files:**
- `Dockerfile` — multi-stage build for the FastAPI service (`python:3.11-slim`), runs `uvicorn main:app --workers 2` on port 8080; explicitly designed to start in degraded mode if `models/` is empty.
- `cloudbuild.yaml` — builds/pushes/deploys the `zillow-scorer` Flask stub to Cloud Run on push to main.
- `curl_examples.sh` — example requests against the local FastAPI app (`/health`, `/model/info`, `/predict`, `/predict/batch`), documents the authoritative request/response contract.
- `verify-cloud.ps1` — PowerShell smoke-test script exercising all 20 live cloud services (Lambda, DynamoDB, S3/CloudFront, CloudWatch, SNS, SQS, Step Functions, SES, Amplify, and the 10 GCP services).
- `.claude/settings.local.json` — dev-tooling permission allowlist confirming this project was built/iterated on with Claude Code, with AWS/GCP CLI access during development.

## 6. How the project was actually built

The root-level `.txt`/`.md` files (`batch_scoring_pipeline.txt`, `buyerseller_risk_tool.txt`, `heatmap.txt`, `model_monitoring_dashboard.txt`, `property_eval_api.txt`, `shapexplainablility.txt`, `integration.txt`, `cloud_integration_prompt.md`) are **not architecture docs** — they are the literal AI-coding-assistant prompts used to generate each feature/page, one spec per feature. `integration.txt` was the original "build the whole mocked showcase site" master prompt; `cloud_integration_prompt.md` was the later pass that wired real AWS/GCP calls into the previously fully-mocked site. The app/components/lib/data code is close to a 1:1 realization of these specs.

Git history (11 commits) confirms this chronology: `mvp created` → `implemented aws and gcp services` → `Heatmap issue fixed` → UI/testing polish → `CSP Services tested and verified` → `final mvp completed` → `Service guide page added` → three README refinement passes.

## 7. Data

- No raw `properties_2016.csv` / `train_2016.csv` (the real ~2.9M-row Zillow Prize dataset) is committed — the README explicitly discloses this is a portfolio/demo project running on **seeded synthetic data**, not production Zillow data or the full real dataset.
- `sample_predictions.csv` (repo root, 25 rows) — synthetic, generated by `generate_sample_csv.py` via a hand-tuned formula, weighted to match real county distributions (LA 72% / Orange 20% / Ventura 8%) and county target-encoding values.
- `data/*.ts` — TypeScript mock data for the heatmap (~12,500 points), monitoring dashboard (RMSE/drift/alerts), and general showcase stats — all simulated.

## 8. Notable gaps / things to know before touching this repo

- **No trained model artifacts are committed** (`models/` only has `.gitkeep`). The FastAPI service (`main.py`) and batch scorer will not produce real predictions until someone re-runs the Colab notebook (`model.py`) and `save_models.py` to regenerate the 13 joblib artifacts.
- **The live site's "real" predictions are not the real ensemble** — they come from a pre-deployed AWS Lambda using a hand-tuned linear approximation formula, mirrored in `generate_sample_csv.py`/`seed_dynamodb.py`/`seed_firestore.py`, not from `batch_scorer.py` or the notebook's trained models.
- **Real, live credentials are checked into the repo** across `.env.local`, `cloud_integration_prompt.md`, `seed_aws_services.py`, `seed_gcp_services.py`, and `verify-cloud.ps1`: AWS account ID, Lambda Function URLs, SNS ARN, SQS URL, CloudFront domain, GCP project ID, Firebase project config, and a BigQuery API key. This is already committed to the public GitHub repo history — worth being aware of if doing any further work here (e.g., rotating keys, or being careful not to further publicize them).
- Repo owner/origin is GitHub user `vajeedashaik` — the user's own personal/portfolio project, living in their local `My Projects` folder.
