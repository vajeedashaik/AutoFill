# BovineID (Cattle Breed Classifier) — Project Context

**Location:** `C:\Users\kittu\OneDrive\Desktop\My Projects\cattle-breed-classifier`
**GitHub origin:** `https://github.com/vajeedashaik/cattle-breed-classifier.git`
**Type:** Full-stack ML web app (Flask + TensorFlow/Keras VGG16 + vanilla JS frontend), built as a **Software Engineering & Project Management (SEPM)** course capstone. Product name in the UI/README is **BovineID**.
**Git history (4 commits):** Initial commit → Add Supabase integration, TTA inference, full-stack deployment setup → Dashboard Changes → Increased the Model's Accuracy.
**License:** MIT (per README).

## 1. What this project is

A web app that identifies **41 Indian bovine breeds** (cattle + buffalo) from a single photo. A user uploads (or captures with camera) an image; the server runs a VGG16 transfer-learning classifier with Test-Time Augmentation, and the UI shows:

- an AI-style **scanning animation** (canvas HUD, scan line, bounding box, keypoint skeleton, phase text) while the request is in flight,
- the **top-1 breed + confidence** and **Top-5** predictions with animated bars,
- a **breed info card** (origin, purpose, milk yield, description),
- a filterable **gallery of all 41 breeds** with a detail modal,
- a **Recent Identifications** history panel (from Supabase),
- a separate **Model Analytics page** (`/model`) with learning curves, per-breed F1, confusion matrix, architecture and training config.

Target audience per the About section: "the Indian farmer" — breed identification for livestock management/dairy/breeding decisions.

## 2. Tech stack

| Layer | Tech |
|---|---|
| ML | TensorFlow/Keras, VGG16 (ImageNet weights, `include_top=False`), all base layers frozen |
| Backend | Flask 3.x, Gunicorn (prod), Pillow, NumPy, python-dotenv |
| DB | Supabase (Postgres) via `supabase` Python client — prediction logging/history (optional) |
| Model hosting | Hugging Face Hub (`huggingface_hub`) — optional auto-download of the `.h5` on cold start |
| Frontend | Jinja2 templates, hand-written CSS (~1,290 lines, dark premium theme, amber/emerald accents), vanilla JS (`main.js`, ~690 lines), HTML5 Canvas, Chart.js 4.4 (CDN, model page only) |
| Fonts | Google Fonts: Plus Jakarta Sans, Inter, Instrument Serif, JetBrains Mono |
| Deploy | Docker (python:3.12-slim), Render (`render.yaml`), Heroku legacy (`Procfile`, `runtime.txt` = python-3.12.4) |
| Dataset | Kaggle "Indian Bovine Breeds" — `lukex9442/indian-bovine-breeds` (≈5,800 images total, inferred: 1,165 val images = 20% split) |

`requirements.txt` is loosely pinned (`flask>=3.0`, `tensorflow>=2.13`, `Pillow>=10`, `numpy>=1.24`, `supabase>=2.0`, `huggingface_hub>=0.20`, `gunicorn>=21`, `python-dotenv>=1.0`). README claims TF 2.21 / Python 3.13 locally, Docker/Render use Python 3.12.

## 3. Repository layout

```
cattle-breed-classifier/
├── app.py                        Flask app: routes, BREED_INFO, LABELS, model loading, TTA, Supabase logging
├── train_model.py                "Modern" 2-phase training script (224×224, preprocess_input, GAP head)
├── sepm_proj (1).py              Original Colab export — the script that actually produced the saved model
├── upload_model.py               One-off script: pushes the .h5 to a Hugging Face model repo
├── bovine_breed_classifier_vgg16.h5   Trained model (~111 MB on disk; git-ignored via *.h5)
├── model_metrics.json            Real eval results: top1/top5, macro/weighted F1, per-class F1, 15×15 confusion matrix
├── supabase_schema.sql           predictions + breeds tables, RLS policies
├── requirements.txt / Dockerfile / Procfile / render.yaml / runtime.txt
├── .env.example                  SUPABASE_URL, SUPABASE_ANON_KEY, SUPABASE_SERVICE_ROLE_KEY, FLASK_*, MODEL_PATH
├── .gitignore                    ignores *.h5, *.keras, .env*, venvs, caches
├── .claude/settings.local.json   Claude Code permission allowlist (py app.py, curl /health, kagglehub path checks…)
├── templates/
│   ├── index.html   (352 lines)  Navbar, classifier, How It Works, breeds gallery, history, About, breed modal
│   └── model.html   (822 lines)  Model analytics page (Chart.js), inline styles + inline JS with the metric data
└── static/
    ├── css/style.css             Design system + animations
    └── js/main.js                Upload, scan animation, /predict call, results rendering, gallery, modal, history
```

## 4. ML model

### 4.1 The saved model (what production actually serves)
Produced by `sepm_proj (1).py` (Colab notebook export):

- **Data loading:** all images read with OpenCV, BGR→RGB, resized to **150×150**, held in memory as a float32 array `/255.0`.
- **Split:** `train_test_split(test_size=0.2, random_state=42, stratify=labels)` → 80/20. The "test" set doubles as the validation set for early stopping (no separate held-out test set).
- **Labels:** `sorted()` folder names from the dataset (`Indian_bovine_breeds/Indian_bovine_breeds/<Breed>/`), 41 classes.
- **Architecture:** VGG16 base (frozen) → `Flatten` (4×4×512 = 8192) → `Dense(512, relu)` → `Dropout(0.5)` → `Dense(256, relu)` → `Dropout(0.5)` → `Dense(41, softmax)`. ~19.05 M params, ~4.46 M trainable (head only).
- **Training:** Adam lr 1e-4, categorical cross-entropy, metrics accuracy + AUC, batch 32, up to 50 epochs, `EarlyStopping(val_loss, patience=10, restore_best_weights)`, `ReduceLROnPlateau(val_loss, factor 0.2, patience 5, min_lr 1e-6)`. **No data augmentation.**
- Also contains a `predict_from_url` demo (Gir cow image) and a 12-sample random prediction grid.

### 4.2 `train_model.py` (newer, NOT what the saved model used)
- 224×224 input, `vgg16.preprocess_input` (caffe-style mean subtraction, **not** `/255`).
- `ImageDataGenerator` augmentation: h-flip, rotation 20°, width/height shift 0.15, zoom 0.2, brightness 0.75–1.25, shear 0.1.
- Head uses **`GlobalAveragePooling2D`** → Dense(512) → Dropout(0.5) → Dense(256) → Dropout(0.4) → softmax.
- **Phase 1:** frozen base, lr 1e-4, ≤30 epochs. **Phase 2:** unfreeze `block5*` layers, lr 1e-5, ≤20 epochs. EarlyStopping (val_accuracy, patience 8), ReduceLROnPlateau, ModelCheckpoint (writes `bovine_breed_classifier_vgg16.h5`), TensorBoard.
- Config says `VAL_SPLIT=0.15`, `TEST_SPLIT=0.15`, but both are added into a single `validation_split` of 0.30 — there is no real test split.
- Prints class order and reminds you to sync it with `LABELS` in `app.py`.
- **Warning:** if you retrain with this script and overwrite the `.h5`, `app.py`'s `preprocess_image` (`/255`) and `LABELS` order must be updated to match, otherwise predictions silently degrade. The README states the two scripts "are not interchangeable".

### 4.3 Evaluation results (`model_metrics.json`)
On 1,165 held-out validation images:

| Metric | Value |
|---|---|
| Top-1 accuracy | **78.37%** |
| Top-5 accuracy | **92.19%** |
| Macro F1 | 0.7493 |
| Weighted F1 | 0.7791 |

- **Best per-class F1:** Kangayam 0.909, Brown Swiss 0.870, Jersey 0.861, Sahiwal 0.859, Holstein Friesian 0.853, Vechur 0.846, Ayrshire 0.837/Gir 0.837.
- **Worst:** Kherigarh 0.25, Bhadawari 0.48, Krishna Valley 0.522, Malnad Gidda 0.588, Kenkatha 0.60.
- **Confusion matrix** in the JSON covers only the **top 15 breeds by sample count** (row-normalised, 15×15): Sahiwal, Gir, Holstein Friesian, Ayrshire, Brown Swiss, Tharparkar, Jersey, Ongole, Hallikar, Nagpuri, Kankrej, Murrah, Red Dane, Red Sindhi, Rathi. Notable confusions: Ongole→Tharparkar (14.7%), Red Sindhi→Sahiwal/Gir (10%/13%), Rathi→Gir (10.7%); Red Dane is weakest of the 15 (59.4%); Kankrej and Murrah are 100%.
- The JSON is **not read by the backend** — the same numbers are hard-coded inside `templates/model.html`.
- Learning-curve arrays on the analytics page (e.g. `VAL_LOSS`, ~35 epochs) are hard-coded in the template; the per-class F1 and confusion matrix are labelled "real"; the curves appear to be transcribed from the training run (treat them as approximate).

## 5. Backend (`app.py`)

### 5.1 Config & startup
- `MAX_CONTENT_LENGTH` = 16 MB; `FLASK_SECRET_KEY` (default `"dev-secret"`); `load_dotenv()`; Windows stdout UTF-8 reconfigure.
- Global JSON error handlers for 404, 405, 413, 500 and a catch-all `Exception` handler (returns JSON, logs traceback).
- `LOW_CONFIDENCE_THRESHOLD = 0.40`.
- `BREED_INFO` dict: 41 breeds with `origin`, `purpose` (Dairy / Draft / Dual / "Dairy (Buffalo)"), `milk_yield`, `description`, `color` (hex), `emoji` (🐄/🐂/🐃). Keys use underscores, e.g. `Brown_Swiss`, `Holstein_Friesian`, `Krishna_Valley`, `Malnad_gidda` (lowercase g), `Nili_Ravi`, `Red_Dane`, `Red_Sindhi`.
- `LABELS`: 41 breed names in **alphabetical order**, which must match the training class index order.

### 5.2 Model loading (`load_model_once`)
Lazy, thread-safe (lock + `_model_load_attempted` flag; tries once only), triggered on the first `/predict`, not at import. If the `.h5` is missing it tries `_download_model_from_hf()` (needs `HF_MODEL_REPO`, optional `HF_TOKEN`). On success sets `IMG_SIZE` from `model.input_shape[1]`. On any failure the app runs in **demo mode**: random pseudo-predictions (one class boosted ×6) flagged `demo_mode: true`.

### 5.3 Inference
- `preprocess_image`: resize to `IMG_SIZE`, `/255.0`, add batch dim.
- `predict_with_tta`: 4 views → original, horizontal flip, ~90% centre crop (5% margin each side) resized back with LANCZOS, brightness ×1.15 → one batched `model.predict` → mean of softmax vectors.
- Top-5 via `argsort`; `low_confidence` flag if top-1 < 0.40.
- Response includes a 400px JPEG thumbnail as a base64 data URI (`preview`).

### 5.4 Routes

| Method | Path | Purpose |
|---|---|---|
| GET | `/` | Main UI (`index.html`, passes `breeds`, `labels`) |
| POST | `/predict` | Multipart field `image`; validates presence, filename, MIME (jpeg/png/webp/gif/bmp → else 415), non-empty (400), decodable (422); returns `{success, predictions[{breed, confidence, info}], demo_mode, tta_enabled, low_confidence, preview}` |
| GET | `/breeds` | Full `BREED_INFO` JSON |
| GET | `/history` | 20 most recent predictions from Supabase (`{available, predictions}`) |
| GET | `/health` | `{status, model: loaded/demo/not_loaded, supabase, breeds}` |
| GET | `/model` | Model analytics page |

`MemoryError` → 507. Running directly: `app.run(debug=FLASK_DEBUG=="1", port=5000)`.

### 5.5 Supabase
`get_supabase()` lazily creates a client if `SUPABASE_URL`/`SUPABASE_ANON_KEY` are set and not the placeholder. `save_prediction` inserts `{breed, confidence (4 dp), top5 [{breed, confidence}], demo_mode}`; failures are logged and swallowed. Uploaded images themselves are **not stored** — only the prediction metadata.

**Schema (`supabase_schema.sql`):**
- `predictions`: `id bigserial PK`, `created_at timestamptz default now()`, `breed text`, `confidence numeric(6,4)`, `top5 jsonb`, `demo_mode boolean`; index on `created_at desc`; RLS enabled with policies allowing **anyone** to select and insert.
- `breeds` (optional, unused by the app): `id serial`, `name unique`, `origin`, `purpose`, `milk_yield`, `description`, `color`, `emoji`; public read policy.
- Note: the README's SQL snippet shows `id uuid` while the real schema file uses `bigserial` — the SQL file is authoritative.

## 6. Frontend

### `templates/index.html` (main page)
Sections: sticky navbar (mobile toggle) → hero → **Upload & Identify** (drop zone, camera capture input with `capture="environment"`, Classify / Clear buttons, demo-mode and low-confidence notices, results panel) → **How It Works** → **Indian Bovine Breeds** gallery (filterable, opens breed modal) → **Recent Identifications** (history grid) → **About / "Built for the Indian Farmer"** → breed detail modal.

### `static/js/main.js` (IIFE, vanilla)
- `setFile` (validation + preview), drag-and-drop, camera input.
- Canvas scanner: `startScanAnimation` / `drawScanFrame` with `drawBBox`, `drawSkeleton`, `drawKeypoints`, `drawHUD`, `drawPhaseText` (purely cosmetic — it does not reflect real detection).
- `fetch("/predict", POST FormData)` → `renderResults` (top result, confidence bars, breed card), `showError`, loading state.
- `loadHistory` → `fetch("/history")`.
- `window.openBreedModal` / `closeBreedModal`, `escHtml` helper, scroll-reveal animations.

### `templates/model.html` (analytics page)
Self-contained page with inline CSS/JS and Chart.js from jsDelivr CDN. Sections: hero stats, **Learning Curves** (accuracy + loss, with a "Phase 2 →" annotation plugin), **F1 Score by Breed** (all 41), **Confusion Matrix** (15 breeds, rendered from `CM_DATA`), **VGG16 + Custom Head** architecture explainer, training configuration, data augmentation notes, and a "How Predictions Are Made" TTA walkthrough.

## 7. Deployment

- **Docker:** `python:3.12-slim` + `libgl1-mesa-glx`, `libglib2.0-0`; installs requirements; `gunicorn app:app --bind 0.0.0.0:5000 --workers 1 --timeout 120`; sets `FLASK_ENV=production`, `FLASK_DEBUG=0`. Model is expected via Hugging Face download (`*.h5` is git-ignored).
- **Render:** `render.yaml` — web service `bovineid`, python runtime, plan **starter ($7/mo, needed for TensorFlow's ~1 GB RAM)**, env vars `PYTHON_VERSION=3.12.4`, `FLASK_ENV`, `FLASK_DEBUG=0`, `FLASK_SECRET_KEY` (auto-generated), and manual `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `HF_MODEL_REPO`, `HF_TOKEN`. Single Gunicorn worker, 120 s timeout (cold model load + TF import is slow).
- **Heroku (legacy):** `Procfile` → `web: gunicorn app:app --bind 0.0.0.0:$PORT --workers 1 --timeout 120`.
- **Model distribution workflow:** run `upload_model.py` once locally (needs a Hugging Face **write** token) → creates `<username>/bovine-breed-classifier` model repo, uploads the `.h5`, prints `HF_MODEL_REPO`/`HF_TOKEN` to set on Render.

### Environment variables
| Variable | Required | Purpose |
|---|---|---|
| `FLASK_SECRET_KEY` | prod | Flask session secret |
| `SUPABASE_URL`, `SUPABASE_ANON_KEY` | optional | Enables history logging |
| `SUPABASE_SERVICE_ROLE_KEY` | listed in `.env.example` | Not used by `app.py` |
| `HF_MODEL_REPO`, `HF_TOKEN` | optional | Auto-download model |
| `MODEL_PATH` | optional | Override model file path |
| `FLASK_DEBUG` | optional | `1` = debug (**default is 1** if unset) |

## 8. How to run locally

```bash
python -m venv .venv && .venv\Scripts\activate      # Windows
pip install -r requirements.txt
cp .env.example .env                                  # fill in optional keys
# place bovine_breed_classifier_vgg16.h5 in the project root (or set HF_MODEL_REPO)
python app.py                                         # http://127.0.0.1:5000
```
Without the `.h5` (and no HF repo) the app still runs in demo mode. Health check: `curl http://127.0.0.1:5000/health`. Retraining: `python train_model.py --data_dir <path>/Indian_bovine_breeds` (see §4.2 caveats), or re-run the Colab notebook script (downloads the dataset via `kagglehub`).

## 9. The 41 breeds

- **Indigenous cattle (26 in code):** Alambadi, Amritmahal, Bargur, Dangi, Deoni, Gir, Hallikar, Hariana, Kangayam, Kankrej, Kasargod, Kenkatha, Kherigarh, Khillari, Krishna Valley, Malnad Gidda, Nagori, Nimari, Ongole, Pulikulam, Rathi, Red Sindhi, Sahiwal, Tharparkar, Umblachery, Vechur (the README's list of 27 also includes Toda — see §10)
- **Exotic / crossbred cattle (6):** Ayrshire, Brown Swiss, Guernsey, Holstein Friesian, Jersey, Red Dane
- **Buffalo (9 in code; README lists 8):** Banni, Bhadawari, Jaffrabadi, Mehsana, Murrah, Nagpuri, Nili Ravi, Surti, Toda

## 10. Design decisions, gotchas and known issues

**Deliberate decisions (from README):** VGG16 for reliability and cheap CPU inference; Flatten head to match the original training; `/255` normalisation to match training; 4-view TTA for a free accuracy boost (~4× inference compute); vanilla JS/no bundler for zero build tooling; Supabase for zero-setup logging.

**Things to be aware of:**
1. **Two incompatible training pipelines.** Saved model = 150×150, `/255`, Flatten head, no augmentation. `train_model.py` = 224×224, `preprocess_input`, GAP head, augmentation, fine-tuning. Swapping the model requires editing `preprocess_image` and re-verifying `LABELS`.
2. **The model page describes the newer pipeline.** `model.html` shows a "Phase 2" fine-tuning annotation, augmentation details and a 2-phase config, which describe `train_model.py`-style training, while the deployed weights came from the single-phase, un-augmented Colab script. The headline metrics in `model_metrics.json` are the evaluated numbers; the descriptive text on the page may not match the saved model exactly. Verify before presenting it as fact (e.g. in a viva/report).
3. **Metrics are static.** `model_metrics.json` is not loaded by Flask; numbers are duplicated in `model.html`. Updating the model means editing both by hand.
4. **Validation = test.** The 20% split is used for early stopping/LR scheduling and for reporting, so the reported accuracy is slightly optimistic.
5. **Hard-coded dependence on class order.** `LABELS` is alphabetical to match `sorted(os.listdir(...))`. Note the `Malnad_gidda` lowercase spelling is the actual folder name and is used consistently as a key.
6. **`Toda` categorisation:** the README lists it under indigenous cattle, but `BREED_INFO` labels it "Dairy (Buffalo)" (Toda buffalo is the real breed). In code the split is 26 indigenous cattle + 6 exotic + 9 buffalo = 41.
7. **Debug default:** `FLASK_DEBUG` defaults to `"1"` when unset in `app.py`'s `__main__`; production uses Gunicorn (so debug isn't used there) and Docker/Render set `FLASK_DEBUG=0`.
8. **Public Supabase RLS:** anon key can insert and read every prediction; anyone can spam the table via the public anon key. Fine for a demo, not for production. No rate limiting on `/predict`.
9. **Memory/cold start:** TensorFlow + VGG16 needs ~1 GB RAM; first `/predict` after a cold start is slow because the model loads lazily (and may first download ~111 MB from Hugging Face).
10. **`/predict` MIME check is client-supplied** (`Content-Type` header); real validation happens when Pillow decodes the bytes.
11. **Unused/extra bits:** `breeds` Supabase table, `SUPABASE_SERVICE_ROLE_KEY`, and `requirements.txt` includes TensorFlow even though the README notes demo mode needs none.
12. **Filename oddity:** `sepm_proj (1).py` (space and parentheses in the filename) is the original Colab export and is the only place the true training recipe for the deployed model is recorded.
13. `.claude/settings.local.json` is tracked in the project folder — it only contains local permission allowlist entries, no secrets.

## 11. Quick reference — key numbers

| Item | Value |
|---|---|
| Classes | 41 |
| Input size (deployed model) | 150 × 150 × 3, pixels / 255 |
| Params | ~19.05 M (≈4.46 M trainable) |
| Model file | ~111 MB `.h5` |
| Top-1 / Top-5 | 78.37% / 92.19% |
| Macro / Weighted F1 | 0.7493 / 0.7791 |
| Val images | 1,165 (80/20 stratified, seed 42) |
| TTA views | 4 (orig, h-flip, 90% crop, +15% brightness) |
| Low-confidence threshold | 0.40 |
| Upload limit | 16 MB |
| Gunicorn | 1 worker, 120 s timeout |
| Local port | 5000 |
