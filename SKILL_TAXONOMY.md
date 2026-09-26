# SKILL TAXONOMY

Normalizes the many ways a job posting names a skill to one canonical skill.

**Scope rule:** every skill here is evidenced in at least one project file (short names from `KNOWLEDGE_BASE_INDEX.md`). "Evidenced in a project" is not "I used it": the agent may only claim a skill once I confirm it in `PERSONAL_PROFILE.md` section 5 or in the project's "My contribution". Skills not in this file are NOT claimable; if a posting asks for one, mark it UNKNOWN - USER INPUT REQUIRED. Extend by adding rows, only from real evidence.

Columns: Canonical | Synonyms | Related | Parent category | Projects

## Languages
| Canonical | Synonyms | Related | Parent | Projects |
|---|---|---|---|---|
| Python | Python 3, py, Python3.x | pandas, FastAPI, Flask | Programming Languages | Zillow, ConvoyMind, MetaDebate, ForgeML, CSPM, CrisisCore, MetaModEnv, BovineID |
| TypeScript | TS | JavaScript, React | Programming Languages | Zillow, ConvoyMind, ReNotify, FlamLayout, CSPM, GrowwPulse |
| JavaScript | JS, ES6, ECMAScript, vanilla JS | TypeScript | Programming Languages | CrisisOS, CrisisCore, BovineID, MetaDebate |
| SQL | Structured Query Language, PL/pgSQL, Postgres SQL | PostgreSQL, Supabase | Programming Languages / Databases | ReNotify, GrowwPulse, CSPM, ForgeML, Zillow (BigQuery) |

## Backend / APIs
| Canonical | Synonyms | Related | Parent | Projects |
|---|---|---|---|---|
| FastAPI | Fast API | Pydantic, Uvicorn, REST | Backend Frameworks | Zillow, ConvoyMind, MetaDebate, CSPM, CrisisCore, MetaModEnv |
| Flask | Flask 3 | Gunicorn | Backend Frameworks | BovineID |
| REST API Design | RESTful APIs, HTTP APIs, web services, API development, endpoints | OpenAPI/Swagger | APIs | Zillow, MetaDebate, CSPM, CrisisCore, MetaModEnv, BovineID |
| Serverless / Edge Functions | Lambda functions, Cloud Functions, Supabase Edge Functions, FaaS | AWS Lambda, Firebase Functions | Backend / Cloud | ReNotify, Zillow, CSPM, CrisisOS |
| Background Jobs / Scheduling | cron jobs, task queue, scheduled jobs, job scheduler | Inngest, SQS, Cloud Scheduler | Backend | GrowwPulse, ReNotify, CSPM, Zillow |
| OAuth 2.0 | OAuth, Google OAuth, auth code flow | Gmail API, Clerk | Security / Auth | ReNotify, CSPM (Cognito/Entra) |
| Authentication & Authorization | Auth, AuthN/AuthZ, SSO, JWT, identity management, RBAC (row-level) | Clerk, Cognito, Entra ID, RLS | Security / Auth | ReNotify, GrowwPulse, CSPM |
| Model Context Protocol | MCP, MCP server, tool server | FastMCP, agents | AI / Integration | ForgeML, MetaModEnv |

## Frontend
| Canonical | Synonyms | Related | Parent | Projects |
|---|---|---|---|---|
| React | React.js, ReactJS, React 18/19 | Next.js, Vite | Frontend | Zillow, ConvoyMind, ReNotify, FlamLayout, CSPM, CrisisOS, GrowwPulse |
| Next.js | NextJS, Next, App Router | React, SSR | Frontend | Zillow, MetaDebate, CSPM, GrowwPulse |
| Tailwind CSS | Tailwind, utility-first CSS | CSS | Frontend | Zillow, ConvoyMind, ReNotify, CSPM, CrisisOS, GrowwPulse |
| Data Visualization | dashboards, charts, Recharts, Plotly, Chart.js, Folium | Streamlit | Frontend / Data | Zillow, ForgeML, MetaDebate, BovineID, GrowwPulse |
| Streamlit | - | Plotly, Python dashboards | Frontend / Data | ForgeML |
| 3D / Canvas Graphics | Three.js, HTML5 Canvas, WebGL | - | Frontend | CrisisOS, ConvoyMind (SVG), BovineID (canvas) |

## Databases
| Canonical | Synonyms | Related | Parent | Projects |
|---|---|---|---|---|
| PostgreSQL | Postgres, PostgresSQL, PSQL, pg | Supabase, RDS, SQL | Databases | ReNotify, GrowwPulse, CSPM, ForgeML |
| Supabase | - | PostgreSQL, RLS, Realtime | Databases / BaaS | ReNotify, GrowwPulse, ForgeML, BovineID |
| NoSQL | Firestore, DynamoDB, document DB, key-value store | - | Databases | Zillow, CrisisOS (Firebase) |
| BigQuery | Google BigQuery, data warehouse | SQL | Databases / Data | Zillow |
| Database Migrations | schema migrations, Alembic | SQLAlchemy | Databases | CSPM, ReNotify, GrowwPulse |
| ORM | SQLAlchemy | Alembic | Databases | CSPM |

## Data Engineering / Data Science
| Canonical | Synonyms | Related | Parent | Projects |
|---|---|---|---|---|
| Data Engineering / ETL / Data Pipelines | ETL pipeline, ELT, data pipeline, data ingestion pipeline, batch processing, data processing | Parquet, scheduled jobs | Data Engineering | Zillow (batch_scorer), GrowwPulse (snapshot ingest), ReNotify (Gmail->OCR pipeline), ForgeML (cleaning), CSPM (CV pipeline) |
| pandas / NumPy | Pandas, data wrangling, data manipulation | scikit-learn | Data Science | Zillow, ForgeML, BovineID, MetaDebate |
| Feature Engineering | feature extraction, feature creation, target encoding | scikit-learn | Data Science / ML | Zillow, ForgeML, ConvoyMind, MetaDebate |
| Exploratory Data Analysis | EDA, data analysis, analytics | Plotly | Data Science | ForgeML, Zillow |
| Model Explainability | XAI, SHAP, permutation importance, interpretability | - | Data Science / ML | Zillow, ForgeML |
| Hyperparameter Tuning | Optuna, Bayesian optimization, model tuning | - | ML | Zillow |

## Machine Learning / AI
| Canonical | Synonyms | Related | Parent | Projects |
|---|---|---|---|---|
| scikit-learn | sklearn | pandas | Machine Learning | Zillow, ForgeML, MetaDebate, ConvoyMind, BovineID (eval) |
| Gradient Boosting | XGBoost, LightGBM, CatBoost, HistGradientBoosting, GBM, ensemble/stacking | scikit-learn | Machine Learning | Zillow, ConvoyMind, ForgeML, MetaDebate (GBR) |
| Deep Learning / Transfer Learning | TensorFlow, Keras, VGG16, CNN, fine-tuning | Computer Vision | Machine Learning | BovineID |
| Computer Vision | CV, image processing, image classification, OpenCV, OCR | Hough transform, Vision API | AI / CV | CSPM, BovineID, ReNotify (OCR) |
| OCR | optical character recognition, text extraction, Google Vision, Tesseract | Computer Vision | AI / CV | ReNotify, CSPM |
| Reinforcement Learning | RL, GRPO, RLHF-style training, reward design, RL environments, OpenEnv, Gym-style | TRL, Unsloth | AI / ML | MetaDebate, CrisisCore, MetaModEnv (environment only) |
| LLM Application Development | GenAI, generative AI, LLM integration, prompt engineering, LLM APIs | Claude, Gemini, OpenAI | AI / GenAI | ReNotify, MetaDebate, CrisisOS, GrowwPulse, CSPM, MetaModEnv |
| Multi-Agent Systems | agents, agentic AI, agent orchestration, multi-agent LLM, arbitration | LLM Application Development | AI / GenAI | ConvoyMind, MetaDebate, CrisisOS, CrisisCore, ForgeML |
| LLM Fine-tuning | LoRA, QLoRA, Unsloth, TRL | RL | AI / GenAI | MetaDebate, CrisisCore (scripts; training not completed - see caveats) |
| Model Serving / Inference | model deployment, ONNX, inference API, MLOps (serving) | FastAPI, ONNX Runtime | ML Engineering | ConvoyMind, Zillow, BovineID, ForgeML |
| Embeddings / Semantic Similarity | sentence-transformers, cosine similarity, TF-IDF | NLP | AI / NLP | MetaDebate |
| NLP | natural language processing, text classification, rule-based text processing | LLM | AI / NLP | MetaDebate, MetaModEnv, ReNotify, CSPM (command parsing) |

## Cloud / DevOps
| Canonical | Synonyms | Related | Parent | Projects |
|---|---|---|---|---|
| AWS | Amazon Web Services, Amazon AWS | Lambda, S3, SQS, RDS, Cognito | Cloud | Zillow, CSPM |
| GCP | Google Cloud Platform, Google Cloud | Cloud Run, Firestore, BigQuery | Cloud | Zillow, ReNotify (Vision), CrisisOS (Gemini/Firebase) |
| Firebase | Firebase Hosting, Cloud Functions for Firebase | GCP | Cloud | CrisisOS, ReNotify (hosting) |
| Docker | containers, containerization, Dockerfile, docker-compose | Cloud Run, Render | DevOps | Zillow, ForgeML, CSPM, MetaDebate, MetaModEnv, BovineID, CrisisCore |
| CI/CD | continuous integration, Cloud Build, pipelines | Docker | DevOps | Zillow (Cloud Build); GrowwPulse CI requested but NOT present |
| Observability / Monitoring | logging, metrics, CloudWatch, alerting, monitoring dashboards | - | DevOps / SRE | CSPM, Zillow |
| Resilience Patterns | retry, circuit breaker, graceful degradation, fallback | - | Backend / SRE | CSPM, CrisisOS, ConvoyMind, ReNotify |
| Cloud Storage | S3, GCS, object storage | - | Cloud | CSPM, Zillow |
| Message Queues | SQS, queue-based processing | Lambda | Cloud | CSPM, Zillow |
| Infrastructure as Code | IaC, Terraform, CDK | - | DevOps | NOT evidenced (CSPM infra folders are README-only). Do not claim. |

## Testing / QA
| Canonical | Synonyms | Related | Parent | Projects |
|---|---|---|---|---|
| Unit Testing | pytest, Vitest, test-driven development, automated tests | - | Testing | CSPM, CrisisCore, FlamLayout, ConvoyMind, MetaDebate |
| End-to-End Testing | E2E, Playwright, browser automation, UI testing | Chrome DevTools MCP | Testing / Automation | ConvoyMind, CSPM (live E2E via DevTools MCP), FlamLayout (verified via MCP) |
| Stress / Adversarial Testing | fuzzing, randomized testing, load testing (limited) | - | Testing | FlamLayout (Stress Lab); GrowwPulse perf test NOT run |
| Static Typing / Linting | mypy, ruff, black, ESLint, tsc strict | - | Code Quality | CSPM, FlamLayout, ConvoyMind |

## Software Engineering Practices
| Canonical | Synonyms | Related | Parent | Projects |
|---|---|---|---|---|
| Git / GitHub | version control | - | Tools | all |
| AI-Assisted Development | Claude Code, AI coding assistants, prompt-driven development | - | Tools | Zillow, ConvoyMind, MetaDebate, FlamLayout, CrisisOS, CrisisCore, MetaModEnv, GrowwPulse, CSPM |
| Software Architecture / System Design | modular monolith, design decisions, ADRs | - | Engineering | CSPM (D1-D30), GrowwPulse, FlamLayout |
| Security Practices | RLS, secret redaction, input validation, rate limiting, IDOR testing | - | Security | CSPM, GrowwPulse |
| Documentation | README, architecture docs | - | Engineering | all |
