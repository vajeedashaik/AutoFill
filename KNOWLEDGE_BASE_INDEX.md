# KNOWLEDGE BASE INDEX

Map of the whole Career KB. Detailed project knowledge lives in `Project KnowledgeBase/` and is **never duplicated or rewritten** (Master Prompt §4, §23). Traverse: this index -> project file.

Short names below are used across `SKILL_TAXONOMY.md` and `JOB_FAMILIES.md`.

## Map

```
PERSONAL
  -> PERSONAL_PROFILE.md               source of truth for personal facts (fill manually)
  -> PERSONAL_PROFILE_QUESTIONNAIRE.md what I still have to provide
  -> PROFILE_INFORMATION_GAPS.md       known / partial / missing / needs verification

EXPERIENCE     -> PERSONAL_PROFILE.md section 4  (no separate EXPERIENCE.md until there is content)
SKILLS         -> PERSONAL_PROFILE.md section 5  + SKILL_TAXONOMY.md
CERTIFICATIONS -> PERSONAL_PROFILE.md section 6
ACHIEVEMENTS   -> PERSONAL_PROFILE.md section 7

PROJECTS       -> Project KnowledgeBase/*.md   (12 files, listed below)

REASONING LAYER
  -> SKILL_TAXONOMY.md
  -> JOB_FAMILIES.md
  -> APPLICATION_QUESTION_TYPES.md
  -> ANSWER_STRATEGIES.md
  -> KNOWLEDGE_BASE_RULES.md

APPLICATION     -> APPLICATION/APPROVED_ANSWERS.md   (answers I have approved; reusable)
```

## Projects

**Identity (confirmed by user):** GitHub `vajeedashaik`, HuggingFace `vajeeda`, email `vajeeda.mastan@gmail.com`. All repos in the project files are treated as the user's. Even so, "My contribution" (what *I* specifically built, versus AI-assisted or teammate work) stays blank until I fill it in below; until then the agent must not write specifics of "I built X".

### 1. Zillow Zestimate
- Short name: **Zillow**
- File: `Project KnowledgeBase/zillow_zestimate.md`
- Domains: real estate, ML, cloud showcase
- Main technologies: Python, XGBoost, LightGBM, CatBoost, scikit-learn, Optuna, SHAP, FastAPI, Next.js 16, React 19, TypeScript, Tailwind, AWS (Lambda, DynamoDB, S3, SNS, SQS, CloudWatch, Step Functions, SES, Amplify), GCP (Firestore, BigQuery, Cloud Run, Cloud Build)
- Relevant job families: ML Engineer, Data Scientist, Full Stack, Cloud Engineer
- Major skills: stacked-ensemble modelling (CV RMSE 0.0742), feature engineering, batch scoring pipeline, multi-cloud service wiring
- Caveats (from file): live site uses an approximation formula, not the ensemble; no model artifacts committed; synthetic seeded data
- My contribution: [FILL]

### 2. ConvoyMind (TATA Fleet)
- Short name: **ConvoyMind**
- File: `Project KnowledgeBase/convoymind.md`
- Domains: logistics, fleet safety, multi-agent simulation
- Main technologies: React 19, TypeScript, Vite, Tailwind 4, Vitest, Playwright, XGBoost -> ONNX, FastAPI, onnxruntime
- Relevant job families: AI Engineer, ML Engineer, Frontend, Full Stack, QA/Automation
- Major skills: simulation engine design, agent arbitration, ONNX model serving with fallback, e2e testing
- Caveats: fully simulated; synthetic training data; MAE 5.14 vs 5.0 target
- My contribution: [FILL]

### 3. ReNotify V1
- Short name: **ReNotify**
- File: `Project KnowledgeBase/renotify_v1.md`
- Domains: consumer productivity, document AI, reminders
- Main technologies: React 18, TypeScript, Vite, Tailwind, shadcn/ui, Clerk, Supabase (Postgres, Realtime, Edge Functions/Deno), Google Cloud Vision, Gmail API/OAuth, Claude Haiku 4.5, Web Push, Firebase Hosting
- Relevant job families: Full Stack, Backend, AI Engineer (GenAI), Data Engineer (pipeline)
- Major skills: OCR + regex/LLM extraction pipeline, OAuth integration, scheduled sync jobs, realtime UI
- Caveats: RLS effectively open, tokens base64 only, PDFs unsupported, push likely broken
- My contribution: [FILL]

### 4. MetaDebate (Viral Script Debugging Engine)
- Short name: **MetaDebate**
- File: `Project KnowledgeBase/meta_viral_script_engine.md`
- Domains: creator economy, RL environments, multi-agent LLM
- Main technologies: Python, Pydantic v2, FastAPI, TRL GRPO, Unsloth, Qwen2.5-7B, sentence-transformers, scikit-learn, Next.js 14, Framer Motion, Docker, HuggingFace Spaces, OpenEnv
- Relevant job families: AI Engineer, GenAI Engineer, ML Engineer, NLP Engineer
- Major skills: RL environment/reward design (10 reward components, anti-gaming), multi-agent debate, curriculum/self-improvement loop
- Caveats: real GRPO run NOT completed; results are placeholder/target
- My contribution: [FILL]

### 5. ForgeML Traffic Congestion Predictor
- Short name: **ForgeML**
- File: `Project KnowledgeBase/forgeml_traffic_congestion_predictor.md`
- Domains: transportation, ML, decision support
- Main technologies: Python, pandas, scikit-learn (HistGradientBoosting 93.69% acc, F1 0.9368), Streamlit, Plotly, Folium, FastMCP (4 servers, 22 tools), Supabase, Docker
- Relevant job families: ML Engineer, Data Scientist, Data Analyst, AI Engineer
- Major skills: feature engineering (27 features), model comparison, dashboards, MCP tool servers
- Caveats: random split (optimistic), hand-set corridor factors, rule-based explanations
- Note: file says this is Project 1 on my resume
- My contribution: [FILL]

### 6. Adaptive Layout Engine (flam_vajeeda)
- Short name: **FlamLayout**
- File: `Project KnowledgeBase/flam_vajeeda.md`
- Domains: ad-tech, layout/constraint solving
- Main technologies: TypeScript (strict), React 18, Vite, Vitest (144 tests)
- Relevant job families: Software Engineer, Frontend, Full Stack
- Major skills: algorithm/pipeline design, deterministic scoring, explainability, property/adversarial testing
- Caveats: not deployed; hand-tuned weights
- My contribution: [FILL]

### 7. CSPM: AEC AI Platform
- Short name: **CSPM**
- File: `Project KnowledgeBase/cspm_aec_ai_platform.md`
- Domains: architecture/engineering/construction, computer vision, SaaS
- Main technologies: Python, FastAPI, SQLAlchemy/Alembic, PostgreSQL, OpenCV, Shapely, Next.js 16, React 19, Tailwind 4, AWS (Cognito, S3, SQS, Lambda, RDS, CloudWatch, Secrets Manager, SNS), Docker, pytest (270), Vitest (65), LLM providers (Claude/OpenAI/Gemini, optional)
- Relevant job families: Backend, Full Stack, Cloud Engineer, Computer Vision Engineer, Platform Engineer
- Major skills: async CV pipeline, typed command pipeline with validation, versioned audit log, cloud service integration, resilience/observability
- Caveats: only synthetic plans tested; nothing marked VERIFIED
- My contribution: [FILL]

### 8. CrisisOS
- Short name: **CrisisOS**
- File: `Project KnowledgeBase/crisisos.md`
- Domains: hospitality safety, emergency response, LLM agents
- Main technologies: React 18, Vite, Tailwind, Framer Motion, Three.js, Canvas, Web Speech API, BroadcastChannel, Firebase Functions, Gemini 2.0 Flash
- Relevant job families: Frontend, Full Stack, AI Engineer
- Major skills: real-time dashboards, 3-agent LLM pipeline with fallback, cross-tab sync, simulation
- Caveats: simulated; scripted comparison numbers
- My contribution: [FILL]

### 9. Crisis Simulator (CrisisCore)
- Short name: **CrisisCore**
- File: `Project KnowledgeBase/crisis_simulator.md`
- Domains: emergency response, RL environments
- Main technologies: Python, FastAPI, dataclasses, Docker, Unsloth/TRL (Qwen2.5-3B), pytest (258), vanilla JS dashboard
- Relevant job families: AI Engineer, ML Engineer, Backend
- Major skills: RL environment design, agent hierarchy, curriculum + incident log, unit testing
- Caveats: `train.py` never run; two conflicting reward systems; no results exist
- My contribution: [FILL]

### 10. MetaModEnv (Content Moderation)
- Short name: **MetaModEnv**
- File: `Project KnowledgeBase/content_moderation_metamodenv.md`
- Domains: trust & safety, LLM evaluation
- Main technologies: Python, FastAPI, Pydantic v2, Gradio, MCP (FastMCP), OpenEnv, Docker, uv, HuggingFace Spaces, OpenAI-compatible LLM client
- Relevant job families: AI Engineer, GenAI Engineer, Backend, Python Developer
- Major skills: benchmark/grader design, reward shaping, API design, MCP server
- Caveats: reported scores unverified; nothing trained; file says resume project
- My contribution: [FILL]

### 11. Groww Pulse (code-by-groww)
- Short name: **GrowwPulse**
- File: `Project KnowledgeBase/code_by_groww.md`
- Domains: fintech, market data, LLM-assisted analysis
- Main technologies: Next.js 15, React 19, TypeScript, Tailwind 4, Supabase (Postgres + RLS), Clerk, Inngest, Gemini, yahoo-finance2, Finnhub, Resend, lightweight-charts
- Relevant job families: Full Stack, Backend, Data Engineer (scheduled ingestion), Software Engineer
- Major skills: background jobs, RLS design, volatility-normalized scoring, data-source reconciliation, email alerts
- Caveats: not deployed; no test runner; CI missing
- My contribution: [FILL]

### 12. BovineID (Cattle Breed Classifier)
- Short name: **BovineID**
- File: `Project KnowledgeBase/cattle_breed_classifier.md`
- Domains: agriculture, computer vision
- Main technologies: Python, Flask, TensorFlow/Keras VGG16, Pillow, Gunicorn, Supabase, HuggingFace Hub, Docker, Render, vanilla JS, Chart.js
- Relevant job families: Computer Vision Engineer, ML Engineer, Python Developer, Full Stack
- Major skills: transfer learning (78.37% top-1 / 92.19% top-5, 41 classes), test-time augmentation, model serving, deployment
- Caveats: validation set = test set; model page text partly describes another training script
- My contribution: [FILL]

## Use by the agent
1. Pick projects via the ranking method in `ANSWER_STRATEGIES.md` (section "Project selection").
2. Open the canonical file for any detail. Do not answer project-specific questions from this index alone.
3. Respect the caveats above in every generated answer.
