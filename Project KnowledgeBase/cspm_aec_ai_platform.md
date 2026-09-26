# CSPM (AEC AI Platform) — Project Context

**Location:** `C:\Users\kittu\OneDrive\Desktop\My Projects\CSPM`
**Type:** Academic / portfolio project built as a production-oriented SaaS (multi-cloud coursework), not a disposable demo.
**Naming note:** the folder is `CSPM`, but the product is the **AEC AI Platform** — an AI-native SaaS for Architecture / Engineering / Construction.
**Git:** local repo on `main`, 7 commits (2026-08-28 → 2026-09-08), working tree clean at time of writing. Latest commit: `b33df52 UI Polish of MVP`.
**Written:** 2026-09-26, from reading the repo's own docs (`PROJECT_OVERVIEW.md`, `context.md`, `claude.md`, `SETUP_PROGRESS.md`, `Review-1.md`, `AWS_MIGRATION_PLAN.md`, `docs/*`), `docker-compose.yml`, the code layout and git history. **Test counts below are the numbers recorded in those docs — I did not re-run the suites.**

---

## 1. What this project is

A user uploads a floor-plan image (PNG / JPG / PDF). Computer vision converts it into a **typed, semantic Building Model** (walls, rooms, doors, windows, columns, dimensions…). The user then edits that model with **typed or spoken natural-language commands** ("move the wall between the kitchen and living room 500 mm left", "delete the door"). Every edit produces a new immutable version with undo/redo and an audit log.

### The one rule that governs everything (`claude.md` §3, §4, §35)

> **AI interprets → structured command describes intent → validation checks intent → geometry engine computes → Building Model stores truth → renderer displays truth.**

- A floor plan is a **structured model, not an image**. The model is canonical; the image is never re-edited.
- The **LLM never touches geometry**. It only turns language into a structured command (or, in the CV pipeline, labels text as a `RoomType`). Its output is treated as **untrusted** and fully re-validated (schema → entity resolution → semantic → constraints) before the deterministic Shapely engine runs.
- **Ambiguity → clarification, never a guess** (e.g. "delete the door" with 4 doors returns candidate ids).
- **No fake scale:** geometry is stored in image pixels; `mm_per_pixel` is `null` until calibrated from dimension OCR. Pixels are never assumed to be mm; `mm` commands are rejected without a scale.

### Pipeline

```
Floor plan image → Computer Vision → Semantic Building Model
  → AI (interpret language) → Structured Command → Schema validation
  → Entity resolution → Semantic validation → Geometry engine
  → Constraint validation → new Building Model version → Renderer
```

### Status in one paragraph

Phases 0–6 are implemented. The app runs **fully offline by default** (local auth, local storage, in-process worker, OpenCV, no OCR/LLM). A subsequent AWS setup session wired real AWS services (Cognito, S3, SQS, Lambda, RDS, CloudWatch, Secrets Manager, SNS) and verified them end-to-end; GCP and Azure were **skipped on purpose**. Nothing in `docs/FEATURE_STATUS.md` is marked `VERIFIED` (policy: only a human/real-system run earns that), though the live AWS runs and browser E2E in `context.md` / `SETUP_PROGRESS.md` cover a lot of that ground.

---

## 2. Tech stack

| Layer | Choice |
|---|---|
| Frontend | Next.js **16.3** (App Router), React **19.2**, TypeScript, Tailwind CSS **4**, custom SVG renderer (no canvas lib) |
| Backend | Python 3.11+ (dev box uses 3.13), FastAPI 0.115, Pydantic v2, SQLAlchemy 2.0, Alembic |
| DB | PostgreSQL 16 via `psycopg` v3 (SQLite in-memory for tests) |
| CV | OpenCV (headless), NumPy, Pillow, PyMuPDF (PDF raster); optional Tesseract, Google Vision, YOLO |
| Geometry | Shapely 2.0 / GEOS |
| AI (all optional, lazy-imported) | Anthropic Claude, OpenAI, Google Gemini |
| Auth | Local stub (Argon2id + HS256) · AWS Cognito (JWKS, RS256) · Azure Entra ID SSO (additive) |
| Cloud SDK | boto3 1.35.90 |
| Tests | pytest (+ `moto` for S3), ruff, black, mypy; Vitest + Testing Library, tsc, eslint |
| Dev/deploy | Docker Compose (Postgres + API + web); `Dockerfile.lambda` for the Lambda worker image |

---

## 3. Repository layout

```
CSPM/
├─ apps/
│  ├─ api/            FastAPI backend (~127 py files) + alembic/ + tests/ + Dockerfile(.lambda)
│  └─ web/            Next.js frontend
├─ docs/              ARCHITECTURE, DECISIONS (D1–D30), FEATURE_STATUS, HUMAN_TASKS, TESTING,
│                     DEMO_RUNBOOK, COMMAND_SYSTEM, GEOMETRY_ENGINE, MULTI_CLOUD_ARCHITECTURE,
│                     CLOUD_SERVICES, COST_OPTIMIZATION, AWS/GCP/AZURE/AI_SETUP
├─ services/          README-only placeholders (vision-worker, geometry-engine, rendering-engine) — intentional (D1)
├─ packages/schemas/  placeholder
├─ infrastructure/    aws/ azure/ gcp/ — README-only placeholders (no IaC yet)
├─ tests/             README only (cross-service tests, future)
├─ docker-compose.yml
├─ claude.md          the 35-section project charter / instructions for Claude Code (also CLAUDE.md context)
├─ phase0.md … phase6.md   the original per-phase prompts
├─ PROJECT_OVERVIEW.md     single reference for phases 0–6 + remaining manual work
├─ context.md              session handoff notes (2026-08-31 review + live E2E + bugs fixed)
├─ final.md                from-scratch, click-by-click setup incl. every cloud
├─ SETUP_GUIDE.md          AWS-only setup guide (Parts 1–13)
├─ SETUP_PROGRESS.md       live checklist of what is actually set up on AWS + teardown commands
├─ Review-1.md             module inventory & which module to present at the review
├─ DEMO_SCRIPT.md          AWS demo, step by step
├─ VIDEO_SCRIPT.md         demo video script
├─ AWS_MIGRATION_PLAN.md   PLAN ONLY — consolidate GCP+Azure into AWS (not executed; see §12)
└─ README.md               STALE (says "Phase 3", "68 tests")
```

### Backend (`apps/api/app`)

| Path | Role |
|---|---|
| `main.py` | FastAPI app, CORS, catch-all safe-500 handler, rate-limit middleware |
| `core/config.py` | All env config (pydantic-settings); production guardrails; blank→None coercion; Secrets Manager resolver for `DATABASE_URL_SECRET` |
| `api/deps.py` | `get_current_user` — user id **always** from verified token `sub` |
| `api/routes/` | `auth`, `projects`, `floorplans`, `processing`, `editing`, `storage`, `health` |
| `api/rate_limit.py` | in-process fixed-window limiter (auth 20/min, write 40/min, read 120/min) |
| `db/models/` | `user`, `project`, `floorplan`, `processing_job`, `processing_artifact`, `building_model`, `building_edit`, `local_credential` |
| `repositories/` | owner-scoped data access per model |
| `domain/building/` | `model.py` (Building, FloorPlanModel, Wall, Room, Door, Window, Column, Stair, Furniture, Dimension), `coordinates.py`, `enums.py`, `relationships.py` (wall↔room linking), `insights.py` |
| `domain/commands/models.py` | 8 typed commands (discriminated union, `extra="forbid"`) |
| `services/auth/` | `base`, `local`, `cognito`, `entra`, `composite`, `factory` |
| `services/storage/` | `base`, `local` (HMAC-signed URLs), `s3` (SSE-S3, presigned GET) |
| `services/queue/` | `inline`, `local` (thread), `sqs` |
| `services/processing/` | `pipeline.py` (`run_pipeline`), `stages`, `assemble`, `calibrate`, `overlay`, `deps` |
| `services/vision/` | `preprocess`, `walls` (Hough), `rooms`, `objects`, `objects_factory`, `images` |
| `services/ocr/` | `stub`, `tesseract`, `google_vision`, `parsing` (dimension parser + room lexicon) |
| `services/semantic/` | rules lexicon + optional LLM for `UNKNOWN` labels only |
| `services/editing/` | `pipeline.py` (`EditPipeline`), `rules.py` (offline grammar), `ai_providers.py`, `entity_resolution.py`, `semantic_validation.py`, `context.py`, `mock.py` |
| `services/geometry/` | `engine`, `operations`, `constraints`, `errors` (Shapely, deterministic) |
| `services/speech/` | `stub`, `google_speech` |
| `services/resilience.py` | `call_with_retry` (bounded backoff) + `CircuitBreaker` |
| `observability/` | JSON logging + secret-redaction filter; CloudWatch EMF metrics sink |
| `worker.py` / `lambda_handler.py` | two entrypoints, one image, both call `run_pipeline` (SQS long-poll consumer / AWS Lambda SQS event handler with `batchItemFailures`) |

### Frontend (`apps/web/src`)

| Path | Role |
|---|---|
| `app/` | routes: `/login`, `/register`, `/auth/sso/callback`, `/` (dashboard), `/projects`, `/projects/[id]`, `/projects/[id]/floorplans/[fpId]` |
| `lib/api.ts` | typed fetch client · `lib/auth-context.tsx` (token in `localStorage` key `aec.access_token`) · `lib/entra.ts` |
| `lib/render/*` | pure deterministic renderer: `buildScene(model)` → `Scene` → SVG string **or** React layer |
| `lib/model/*` | geometry / units / derived-data helpers |
| `components/viewer/*` | `<FloorPlanViewer>` (pan/zoom/fit/hover/select/property sidebar) |
| `components/editor/ai-command-panel.tsx` | Preview · Apply · Undo/Redo · history · clarification · 🎤 voice |
| `components/insights/building-insights-panel.tsx` | deterministic analytics |
| `components/states.tsx` | Spinner / Skeleton / EmptyState / ErrorState / ConfirmDialog |
| `components/auth-shell.tsx`, `nav-bar.tsx`, `ui.tsx` | added in the "UI Polish of MVP" commit (large `globals.css` rewrite, redesigned login/register/dashboard/project/floor-plan pages) |

---

## 4. Architecture

```
Browser (Next.js :3000)
   │ bearer token
   ▼
API (FastAPI :8000) ── PostgreSQL (users, projects, floorplans, jobs,
   │                              building_models, edits, artifacts)
   ├─ AuthService      local | Cognito   (+ Entra SSO via CompositeAuthService, dispatch by token `iss`)
   ├─ StorageService   local signed-URL FS | S3
   ├─ QueueService     inline | local thread | SQS
   │     └─ run_pipeline (8 CV stages) — also as AWS Lambda / GCP Cloud Run
   ├─ OCRService       stub | Tesseract | Google Vision
   ├─ SemanticInterp.  rules | Gemini/OpenAI/Claude   (text → RoomType only)
   ├─ CommandParser    rules | mock | Gemini/OpenAI/Claude
   ├─ SpeechService    stub | Google Speech-to-Text
   ├─ GeometryEngine   deterministic Shapely — the ONLY thing that edits geometry
   ├─ MetricsSink      none | CloudWatch EMF (via log stream)
   └─ RateLimiter      in-process fixed window per IP
```

Every provider sits behind an interface; an env var selects the concrete one; a missing SDK/key falls back to the offline default without crashing. `ENVIRONMENT=production` **refuses to start** with `AUTH_PROVIDER=local` or `STORAGE_PROVIDER=local`.

### Intended multi-cloud ownership (charter, `claude.md` §5–6)

- **AWS** — core infra: S3, SQS, RDS, Lambda, CloudWatch (API Gateway planned).
- **GCP** — AI/processing: Cloud Run, Gemini/Vertex, Vision OCR, Speech-to-Text.
- **Azure** — enterprise identity: Entra ID SSO.
- Principle: each provider only where it has a meaningful capability; avoid cross-cloud hops per request.

### Actual current state
Single-cloud **AWS** (see §8). GCP/Azure code paths exist and are unit-tested but were never connected to real accounts.

---

## 5. Feature detail by phase

### Phase 0 — Foundation
Monorepo, Next shell, FastAPI `GET /health`, CORS, structured logging, SQLAlchemy + Alembic (no tables yet — D2), Docker Compose, lint/format/test tooling.

### Phase 1 — Auth, projects, upload
- `AuthService` (local Argon2id/HS256 · Cognito register/login/logout via boto3, **local JWKS verification** — RS256 + issuer + `token_use` + `client_id`).
- Owner-scoped project CRUD (cross-user → 404, no existence leak).
- Upload (`POST /projects/{id}/floorplans`): sanitized extension + declared MIME + **magic bytes** must all agree, non-empty, ≤ `MAX_UPLOAD_BYTES` (25 MB). Storage key `projects/{id}/floorplans/{uuid}.{ext}` — client filename never a key. Each upload = new immutable `FloorPlan.version`. Orphan object deleted on DB failure.
- Storage: S3 (SSE-S3, presigned GET, never public) or local (HMAC-signed, time-limited, path-traversal guarded).
- Migration `0001`: `users`, `projects`, `floorplans`, `processing_jobs`, `local_credentials`, `processing_status` enum.

### Phase 2 — CV pipeline (async)
`processing_jobs` row is source of truth (`status`, `attempts`, per-stage `stages` JSON, timestamps, `building_model_id`). 8 stages, each isolated:

| # | Stage | Critical? | Notes |
|---|---|---|---|
| 1 | preprocess | ✔ | decode PNG/PDF → normalize → grayscale → denoise → CLAHE → adaptive threshold → deskew |
| 2 | wall_detection | ✔ | morphology → `HoughLinesP` → collinear merge → gap bridging |
| 3 | ocr | | stub / Tesseract / Google Vision; dimension parser + room lexicon |
| 4 | object_detection | | none / heuristic (door arcs, columns) / YOLO |
| 5 | room_detection | | rasterize walls → flood-fill exterior → enclosed components → polygons → label assignment |
| 6 | calibration | | dimension text ↔ nearest wall length → `mm_per_pixel`; none found ⇒ `null` |
| 7 | semantic_assembly | ✔ | rules lexicon; LLM only for `UNKNOWN` labels; validated to a fixed `RoomType` enum |
| 8 | artifacts | | preprocessed PNG, walls-overlay PNG, `detected_elements.json`, `semantic_model.json` |

Non-critical stage failure → `degraded`, job still completes. Critical failure → job `FAILED` with a user-safe message; nothing persisted; the original upload is never modified. Every element has UUID `id`, optional `confidence`, and `source` (`DETECTED` / `INFERRED` / `USER_CONFIRMED`). Migration `0002`: `building_models`, `processing_artifacts`.

### Phase 3 — Interactive viewer
Pure-TS deterministic renderer (`buildScene` → `Scene`; byte-identical SVG string for snapshot tests, or interactive React layer). Three coordinate spaces never mixed: Model (px), World (mm/m², only if scaled), Screen (`ViewTransform`). Pan/zoom/fit/hover/select/Esc, labels, dimensions, legend, property sidebar. Gestures update one `<g transform>` imperatively (zero React re-renders while navigating). Wall↔room linking (`relationships.py`) populates `Room.wall_ids`. Selection only, no editing in this phase.

### Phase 4 — AI editing
- 8 operations: `MOVE_WALL`, `ADD_WALL`, `DELETE_WALL`, `RESIZE_ROOM`, `ADD_DOOR`, `DELETE_DOOR`, `ADD_WINDOW`, `DELETE_WINDOW`. Distances carry a unit `{value, unit: mm|px}`.
- `EditPipeline`: **parse → schema validate → entity resolution → semantic validation → geometry engine → constraint validation → atomic versioned write → audit row.** Each stage has its own error type and a `stage` field in the response.
- Constraints: simple polygons, non-self-intersecting, openings stay on walls, in-bounds, positive dimensions, unique ids; coincident wall endpoints move together; deleting a wall cascades to its openings.
- Failure ⇒ session rolled back, an audit row records the rejection, stored model unchanged.
- Versioning: append-only `building_models` + movable **HEAD pointer** (`floorplans.current_model_version`). Undo/redo only move the pointer (O(1)). A fresh edit while HEAD is behind marks the stale redo branch `discarded` (kept for audit).
- Audit table `building_model_edits`: user, time, instruction, resolved command, status (`APPLIED`/`REJECTED`/`CLARIFICATION`/`UNDO`/`REDO`), stage, error, version lineage.
- Command parser default is an **offline regex grammar (`rules`)**; LLM providers are opt-in.
- Migration `0003`.

### Phase 5 — Multi-cloud, voice, resilience, observability
- **Azure Entra SSO** (additive): `EntraAuthService` (offline JWKS verify) + `CompositeAuthService` routing by token `iss`; frontend `lib/entra.ts`, `/auth/sso/callback`, "Sign in with Microsoft" (shown only when `NEXT_PUBLIC_ENTRA_*` set). Uses the **implicit** flow — MSAL+PKCE recommended for prod.
- **Voice**: `POST /floorplans/{id}/commands/voice` (+ `/voice/interpret`) → validate audio (allow-list, ≤ 10 MB) → transcript (text only) → **the same** `EditPipeline`. Provider outage → 502, model unchanged. Frontend 🎤 button via MediaRecorder.
- **Resilience**: `call_with_retry` + `CircuitBreaker` around SQS publish, Speech `recognize`, LLM `_complete`. Failure matrix: SQS publish fails → job `FAILED` + 503 (re-runnable); Vision/semantic AI fails → stage `degraded`, job still completes; LLM parser fails → edit `REJECTED` at `parse`; Speech fails → 502.
- **Observability**: JSON logs with a secret-redaction filter (API keys, `Authorization`, JWTs, `sk-…`, `ghp_…`); CloudWatch **Embedded Metric Format** metrics: `pipeline.duration_ms`, `pipeline.stage.duration_ms`, `pipeline.stage.degraded`, `pipeline.failed`, `ai.latency_ms`, `ai.error`, `edit.command{status}`, `geometry.command.failed{operation,stage}`, `queue.enqueue.failed`, `external.error{service}`, `external.circuit_open`.
- **Compute parity**: `python -m app.worker` (Cloud Run) and `app.lambda_handler.handler` (Lambda) share `run_pipeline`.

### Phase 6 — Reliability, polish, insights, security
- **Building insights** (`compute_insights`, pure + deterministic, no LLM): counts per element type, total/avg floor area (shoelace), largest/smallest room, total & exterior wall length (heuristic: "bounds ≤ 1 room"), rooms-by-type. Metric units only when calibrated. Embedded in `GET /model` and at `GET /model/insights`.
- `PROCESSING_MAX_ATTEMPTS` (3) now enforced — poison-job guard.
- Rate limiting (429 + `Retry-After`; `/health`, `/docs`, `OPTIONS` exempt; off in tests).
- **Security review** — no critical findings. Hardening: CRLF/quote stripping in `Content-Disposition`, CORS methods/headers narrowed, catch-all 500 returns a generic message, SQLi N/A (ORM only), IDOR sweep test over 10 routes, uploads served as `application/octet-stream; attachment`.
- UI states kit + modal confirm replacing `window.confirm()`.

---

## 6. Database

Migrations: `0001_initial` · `0002_phase2` · `0003_phase4`. Relationships:
`User 1─< Project 1─< FloorPlan 1─< ProcessingJob`; `FloorPlan 1─< BuildingModel` (versioned, append-only, HEAD = `current_model_version`); `FloorPlan 1─< BuildingModelEdit`. 9 tables total once migrated. Migration `0001` was **fixed to work on real Postgres** (generic `sa.Enum(create_type=False)` flag is lost on the PG dialect → duplicate `CREATE TYPE`; now uses `postgresql.ENUM` and guards create/drop by dialect). Verified `upgrade head` / `downgrade base` / `upgrade head` on Postgres 16, and applied to RDS.

## 7. API surface

`GET /health` · `POST /auth/{register,confirm,login,logout}` · `GET /auth/me` · project CRUD · `POST/GET /projects/{id}/floorplans[/{fpId}]` · `POST /floorplans/{id}/process` (202) · `GET …/processing-status` · `GET …/model` · `GET …/model/insights` · `GET …/artifacts` · `POST …/commands/interpret` (dry run) · `POST …/commands` · `POST …/commands/voice[/interpret]` · `POST …/model/undo|redo` · `GET …/model/history` · `GET /storage/local/{key}` (local provider only).

## 8. Current AWS deployment state (from `SETUP_PROGRESS.md`)

Region **`ap-south-1`** (Mumbai), free-tier-only account, IAM user `aec-dev`. **GCP and Azure skipped.** Only the API and web run locally; everything else is on AWS.

| Service | Resource | State |
|---|---|---|
| Cognito | pool `aec-platform-dev` (email = username, public app client, `ALLOW_USER_PASSWORD_AUTH`) | ✅ verified E2E. No code-entry screen in the UI — confirm users via console/CLI |
| S3 | `aec-floorplans-vajeeda-dev` (Block Public Access, SSE-S3, 30-day lifecycle) | ✅ verified |
| SQS | `aec-processing` + DLQ (`maxReceiveCount=5`, visibility 720 s) | ✅ verified |
| Lambda | `aec-processing` — container image from `Dockerfile.lambda`, 2048 MB / 120 s, SQS event source (batch 5, partial-batch failures) | ✅ verified (6.5 s cold / 4.5 s warm, 247 MB peak). Image built on a throwaway t3.micro EC2 (CodeBuild unavailable) |
| RDS | `aec-db`, PostgreSQL 16, `db.t4g.micro`, 20 GB gp3, publicly accessible | ✅ **only standing cost: free 12 months, then ≈ $13/mo** |
| CloudWatch | `/aws/lambda/aec-processing` (14-day retention), EMF → `aec` namespace, dashboard `aec-review` (7 widgets), alarm `aec-pipeline-failed` | ✅ genuinely verified after a bug fix (below) |
| SNS | `aec-alerts` wired to the alarm | ⚠ email subscription must be added + confirmed by the user |
| Secrets Manager | `aec/database-url` → `DATABASE_URL_SECRET`; no DB password in env/Lambda config | ✅ |

Offline providers still in use: `OCR_PROVIDER=stub`, `SEMANTIC_AI_PROVIDER=rules`, `COMMAND_AI_PROVIDER=rules`, `SPEECH_PROVIDER=stub`.

Teardown commands are in `SETUP_PROGRESS.md` (delete RDS first — it is the only ongoing cost). A leftover "Cloud Verification Test" project belongs to the demo user and can be deleted in-app.

## 9. Bugs found & fixed (history worth remembering)

| # | Bug | Fix |
|---|---|---|
| 1 | Frontend: "Start processing" disabled for a fresh upload (`PENDING` treated as running) — blocked everything past Phase 2 in the browser | `isRunning()` = `PROCESSING` or (`PENDING` and `started_at != null`) |
| 2 | `FloorPlan.status` never updated by the pipeline | mirror job state onto floor plan in `pipeline.py` |
| 3 | Alembic `0001` crashed on real Postgres (enum re-created) | `postgresql.ENUM` + dialect guard |
| 4 | `S3_ENDPOINT_URL=` blank in `.env` → `boto3` `Invalid endpoint` → S3 dead | `field_validator` maps blank → `None` for optional endpoint/id/key/secret fields |
| 5 | Cognito pool made via console had email as *alias* → `sign_up(Username=email)` failed | recreated with `UsernameAttributes: ["email"]` (CLI recipe in `SETUP_GUIDE.md` 3.1) |
| 6 | Login flaky: local clock ~9 s behind AWS; PyJWT `iat` zero leeway → 401 right after login | `leeway=30` in Cognito + Entra `jwt.decode`; tests widened; `DATABASE_URL_SECRET` force-disabled in `conftest.py` |
| 7 | CloudWatch metrics **not** populating — EMF JSON was re-wrapped by the root `JsonFormatter`, pushing `_aws` a level too deep (an earlier "verified" claim was a false positive based on a hand-made log line) | dedicated pass-through handler + `propagate=False` for `app.metrics`; Lambda image rebuilt; a real datapoint then confirmed |

## 10. Testing

Recorded results (last documented runs; not re-run for this file):

- Backend `pytest`: **270 passed** (~29 s) · `ruff` / `black --check` / `mypy app` clean (127 source files).
- Frontend `vitest run`: **65 passed** (11 files) · `eslint` clean · `tsc --noEmit` clean\* · `next dev` boots ~12 s.
- \* `tsc` fails on a **cold clone** (`Cannot find name 'LayoutProps'`, a Next 16 generated global) until `next dev`/`next build` has run once.
- Everything runs offline — no cloud account. Full workflow test `test_workflow.py` (register → project → upload → process → model → insights → NL command → v2 → undo/redo → history) plus a failure-path variant.
- Live E2E (SQLite + Chrome DevTools MCP): 900×600 synthetic PNG → 8/8 stages in ~0.4 s → 10 walls, 2 rooms, 4 doors → "delete the door" ⇒ CLARIFICATION → specific id ⇒ APPLIED v2 → undo → redo.
- **Only synthetic plans have been exercised** — CV accuracy on real drawings is unmeasured.

Test files cover: auth (local/Cognito/Entra/composite), projects + isolation, upload validation, storage (moto), pipeline + each stage + retry cap, domain model, command models/grammar, entity resolution, geometry engine, edit pipeline (rollback/versioning/undo-redo), insights, resilience, observability, Lambda handler, cloud-failure degradation, rate limit, security sweep, speech, voice API.

## 11. How to run

**Offline, no cloud** (`apps/api`, Windows: use `py -3.13`, `python` is a Store stub):
```
cd apps/api
./.venv/Scripts/python.exe -m pytest -q          # + ruff check ., black --check ., mypy app
cd ../web && node_modules/.bin/vitest run         # + tsc --noEmit, eslint
node_modules/.bin/next dev -p 3000
```
SQLite smoke run: create schema with `Base.metadata.create_all`, then `uvicorn app.main:app` with `DATABASE_URL=sqlite+pysqlite:///./dev_smoke.db`, `AUTH_PROVIDER=local`, `STORAGE_PROVIDER=local`, `QUEUE_PROVIDER=local`, `RATE_LIMIT_ENABLED=false`. Synthetic plan generator: `tests/fixtures/synthetic_plan.make_two_room_plan`.

**Docker:** `docker compose up --build` (Postgres 16 + API running `alembic upgrade head` + web). Postgres credentials in compose are dev-only (`aec/aec`).

**Against AWS (current setup):** start API (`uvicorn app.main:app --port 8000` from `apps/api`) and web (`npm run dev` from `apps/web`); `aws sts get-caller-identity` must work; register a demo user and confirm it via CLI/console; follow `DEMO_SCRIPT.md`.

**Environment gotchas:** pip install of the backend deps took ~40 min on a slow link (`--retries 10 --timeout 120`); OpenCV headless wheel is `cp37-abi3`, so Python 3.13 is fine; Chrome DevTools MCP hard navigation can wipe `localStorage` (drive via clicks); `next build` can hit a locked `.next` under OneDrive.

## 12. Key configuration

All env-driven (`pydantic-settings`); `.env.example` files document keys; defaults are offline. Never commit `.env`.

`ENVIRONMENT`, `AUTH_PROVIDER` (local|cognito), `ENTRA_SSO_ENABLED`(+`ENTRA_TENANT_ID`/`ENTRA_CLIENT_ID`), `STORAGE_PROVIDER` (local|s3), `QUEUE_PROVIDER` (inline|local|sqs), `OCR_PROVIDER` (stub|tesseract|google_vision), `OBJECT_DETECTOR` (none|heuristic|yolo), `SEMANTIC_AI_PROVIDER` / `COMMAND_AI_PROVIDER` (rules|mock|gemini|openai|claude), `SPEECH_PROVIDER` (stub|google), `METRICS_PROVIDER`, `LOG_FORMAT`, `RATE_LIMIT_ENABLED`, `PROCESSING_MAX_ATTEMPTS`, `EXTERNAL_CALL_MAX_ATTEMPTS`/`_BACKOFF_SECONDS`, `MAX_UPLOAD_BYTES`/`MAX_AUDIO_BYTES`, `DATABASE_URL`, `DATABASE_URL_SECRET`, `CORS_ORIGINS`, `API_PUBLIC_BASE_URL`, plus `AWS_REGION`, `COGNITO_*`, `S3_BUCKET_NAME`, `SQS_QUEUE_URL`, LLM keys. Frontend: `NEXT_PUBLIC_API_URL`, `NEXT_PUBLIC_ENTRA_*`.

## 13. Open items, known gaps & risks

**Documented but unfixed (from `context.md` §5):**
1. `README.md` is stale (says Phase 3 / 68 tests).
2. `apps/web/.env.example` is missing though other docs reference it (needs `NEXT_PUBLIC_API_URL`).
3. **Entra label bug:** `get_current_user` provisions users with the *primary* `auth_provider`, even for Entra tokens; `CompositeAuthService.provider_for(token)` exists but is unused, so the HUMAN_TASKS check for `auth_provider = entra` fails as written.
4. `docs/HUMAN_TASKS.md` #9 is stale ("master, no commits").
5. Compose nits: `web` `depends_on: api` has no health condition; `api` bind-mounts source but runs uvicorn without `--reload`.
6. `npm install` churns `@emnapi/*` entries in `apps/web/package-lock.json`.

**Product limitations:**
- Geometry engine enforces simple invariants only — **not** architectural/engineering correctness.
- Entity resolution on real plans is weak: wall↔room linking is approximate, so "the kitchen wall" often yields a clarification; open doorways drawn as wall gaps merge rooms.
- "Exterior wall" insight is a heuristic.
- OCR and object detection are weak by default (`stub` / `heuristic`).
- Rate limiter is per-process and trusts `X-Forwarded-For`; edge limiting needed at scale.
- Entra SSO uses implicit flow (swap for MSAL + PKCE).
- Cognito revocation isn't instant (JWKS local verification, D6).

**Not started (out of scope so far):** quantity takeoff, cost estimation, building-code compliance, custom-trained CV models, server-side SVG/PNG/PDF export, MSAL PKCE, SES email, **API Gateway**, IaC (Terraform/CDK), DWG/DXF/IFC/Revit import, real-time collaboration, manual scale-calibration UI.

**Production requirements still to do:** `ENVIRONMENT=production` with Cognito + S3, RDS + migrations, SQS + DLQ, HTTPS/LB, real `CORS_ORIGINS`, edge rate limiting, secrets in a manager, IAM-role credentials (not keys), and the deployed API (currently only local).

## 14. Pending decision: AWS consolidation (`AWS_MIGRATION_PLAN.md`)

A written-but-**not executed** plan (no `bedrock` anywhere in `app/`) to make the project single-cloud AWS: Cloud Vision → **Textract**, Gemini → **Bedrock** (`converse`), Google Speech → **Transcribe** (batch + poll), Cloud Run → Lambda (already exists), Entra → **Cognito SAML/OIDC federation**; would delete `entra.py` / `composite.py` / frontend Entra code and add `D31` to `DECISIONS.md`. Flagged risks: it contradicts the charter's deliberate multi-cloud principle (§6), Transcribe batch latency (~10–30 s), Textract's weakness on CAD-style drawings, and a one-time Bedrock model-access request. Offline providers and LLM providers Claude/OpenAI would stay.

## 15. Review / demo plan

- `Review-1.md` scores 16 modules on cloud footprint × demo-readiness; recommends presenting **M1 — the CV processing pipeline** as the headline (S3 → SQS → Lambda → CloudWatch → RDS), with M2 (AI editing), M9 (viewer) and M13 (insights) as the payoff. Azure-centric M8 was excluded by the review's own rule.
- `DEMO_SCRIPT.md` (AWS-only, ~step-by-step) and `VIDEO_SCRIPT.md` (demo video) follow that flow: sign in → upload plan → Start processing (8 stages go green) → viewer + insights → type "delete the door" → clarification → pick id → Apply → Undo/Redo.
- The offline path is the safe fallback if Wi-Fi fails on the day.

## 16. Key design decisions (`docs/DECISIONS.md`, D1–D30 — highlights)

- **D1** modular monolith, not microservices (`services/*` are placeholders for later extraction). **D2** no DB tables in Phase 0.
- **D3** `psycopg` v3. **D4** Next.js App Router.
- **D5** auth behind an interface — Cognito for prod, Argon2id local stub for dev/test (prod guard blocks the stub). **D6** Cognito verified locally via JWKS, not per-request `get_user`. **D7** storage behind an interface (S3 vs signed-URL local FS).
- Later decisions cover the Building Model schema, pixel-first coordinates with optional calibration, deterministic geometry, command union + entity resolution, HEAD-pointer versioning, additive Entra SSO, EMF metrics via logs, and in-process rate limiting (read the file for D8–D30).

## 17. Working conventions for this repo (from `claude.md`)

Phase-by-phase implementation; never mark a feature `VERIFIED` automatically (statuses: `NOT_STARTED` / `IN_PROGRESS` / `IMPLEMENTED` / `TESTED` / `VERIFIED`); label things `IMPLEMENTED` / `PARTIALLY IMPLEMENTED` / `MOCKED` / `PLACEHOLDER` / `NOT IMPLEMENTED`; don't fake functionality or claim success without executed tests; manual/credential tasks go in `docs/HUMAN_TASKS.md` (REQUIRED NOW / LATER / OPTIONAL) — Claude can't create accounts or credentials; keep cost near zero (student developer: serverless, scale-to-zero, free tiers, budget alerts); no secrets in code or logs; keep docs matching the actual implementation; end each phase with Implementation / Files Changed / Architecture / Tests / Manual Tasks / Known Limitations / Next Step.

## 18. Where to read more

| Need | File |
|---|---|
| Whole-project summary | `PROJECT_OVERVIEW.md` |
| Latest session findings & bugs | `context.md`, `SETUP_PROGRESS.md` |
| Charter / principles | `claude.md` |
| Deep architecture | `docs/ARCHITECTURE.md` |
| Per-feature status | `docs/FEATURE_STATUS.md` |
| Decisions | `docs/DECISIONS.md` |
| Manual tasks | `docs/HUMAN_TASKS.md` |
| Command grammar / geometry | `docs/COMMAND_SYSTEM.md`, `docs/GEOMETRY_ENGINE.md` |
| Cloud setup & cost | `docs/AWS_SETUP.md`, `GCP_SETUP.md`, `AZURE_SETUP.md`, `CLOUD_SERVICES.md`, `COST_OPTIMIZATION.md`, `SETUP_GUIDE.md` |
| Running tests / demo | `docs/TESTING.md`, `docs/DEMO_RUNBOOK.md`, `DEMO_SCRIPT.md`, `VIDEO_SCRIPT.md` |
