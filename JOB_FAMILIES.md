# JOB FAMILIES

Maps job titles to families, and families to the evidence in the KB. Extensible: add titles, aliases or families by appending; never delete.

"KB evidence" = projects whose files support the family (short names from `KNOWLEDGE_BASE_INDEX.md`). It is a draft for ranking, not a claim. **Confirm ordering with me in `PERSONAL_PROFILE.md` section 8/14.**

## Family definitions

### Software Engineering
| Title | Aliases | KB evidence | Skills to look for |
|---|---|---|---|
| Software Engineer | SWE, Software Developer, Programmer Analyst, Member of Technical Staff | FlamLayout, CSPM, GrowwPulse, ReNotify, Zillow | Python/TypeScript, system design, testing, APIs |
| SDE | SDE-1/SDE I, Software Development Engineer, Associate Software Engineer | same as above | DSA (NOT evidenced in KB, ask me), APIs, cloud |
| Backend Engineer | Backend Developer, Server-side Engineer, API Engineer | CSPM, GrowwPulse, ReNotify, MetaModEnv, CrisisCore | FastAPI/Flask, PostgreSQL, queues, auth, resilience |
| Frontend Engineer | Frontend Developer, UI Engineer, React Developer, Web Developer | FlamLayout, CrisisOS, ConvoyMind, GrowwPulse, Zillow | React, TypeScript, Tailwind, visualization, a11y |
| Full Stack Engineer | Full Stack Developer, Web Engineer | ReNotify, GrowwPulse, CSPM, Zillow, BovineID | React/Next.js + Python/Node + SQL |
| Python Developer | Python Engineer | ForgeML, MetaModEnv, CrisisCore, BovineID, CSPM | Python, FastAPI/Flask, pytest |
| Java Developer | Java Engineer | **none** | NOT evidenced anywhere in KB. Do not claim; mark UNKNOWN and ask me. |

### Data
| Title | Aliases | KB evidence | Skills to look for |
|---|---|---|---|
| Data Engineer | Analytics Engineer, ETL Developer, Big Data Engineer | Zillow (batch pipeline), GrowwPulse (ingestion), ReNotify (email->OCR pipeline), ForgeML | SQL, Python, ETL, PostgreSQL, AWS/GCP, scheduling |
| Data Analyst | Business Analyst (data), Reporting Analyst | ForgeML, Zillow, GrowwPulse | SQL, pandas, dashboards, storytelling with data |
| Data Scientist | Applied Scientist (junior), Decision Scientist | Zillow, ForgeML, ConvoyMind, BovineID | modelling, feature engineering, evaluation, explainability |
| BI Analyst | BI Developer, Reporting Engineer | ForgeML (Streamlit/Plotly), Zillow (dashboards) | SQL, dashboards. (Power BI/Tableau NOT evidenced) |

### AI / ML
| Title | Aliases | KB evidence | Skills to look for |
|---|---|---|---|
| ML Engineer | Machine Learning Engineer, MLE, ML Developer | Zillow, ForgeML, ConvoyMind, BovineID, MetaDebate | training, serving (ONNX/FastAPI), pipelines, evaluation |
| AI Engineer | AI Developer, Applied AI Engineer, LLM Engineer, Agent Engineer | MetaDebate, MetaModEnv, ReNotify, CrisisOS, CrisisCore, ConvoyMind, ForgeML (MCP) | LLM APIs, agents, prompts, evals, MCP |
| GenAI Engineer | Generative AI Developer, Prompt Engineer, LLM Application Developer | ReNotify, CrisisOS, MetaDebate, GrowwPulse, MetaModEnv | Claude/Gemini APIs, structured output, degradation, RAG (RAG NOT evidenced) |
| Computer Vision Engineer | CV Engineer, Vision Engineer, Image Processing Engineer | BovineID, CSPM, ReNotify (OCR) | OpenCV, CNN/transfer learning, OCR |
| NLP Engineer | NLP Developer, Language Engineer | MetaDebate, MetaModEnv, CSPM (command parsing), ReNotify (extraction) | text pipelines, embeddings, LLMs |

### Cloud / Infrastructure
| Title | Aliases | KB evidence | Skills to look for |
|---|---|---|---|
| Cloud Engineer | Cloud Developer, AWS Engineer, GCP Engineer | CSPM, Zillow | AWS/GCP services, IAM, serverless, cost awareness |
| Cloud Architect | Solutions Architect (cloud) | CSPM (multi-cloud design docs, D1-D30) | design docs; senior role, likely above my level: confirm before applying |
| Cloud Support | Cloud Support Associate, Cloud Support Engineer | CSPM, Zillow | troubleshooting AWS services, Linux (Linux NOT evidenced) |
| DevOps Engineer | Release Engineer, Build Engineer | Zillow (Cloud Build), CSPM (Docker, Lambda image), MetaModEnv | Docker, CI/CD, scripting. (Kubernetes, Terraform NOT evidenced) |
| Platform Engineer | Developer Platform Engineer | CSPM | Docker, cloud services, observability |
| SRE | Site Reliability Engineer, Production Engineer | CSPM (metrics, retries, circuit breakers) | observability, resilience. Limited evidence |

### QA
| Title | Aliases | KB evidence | Skills to look for |
|---|---|---|---|
| QA Engineer | Test Engineer, Quality Engineer | CSPM, CrisisCore, ConvoyMind, FlamLayout | pytest/Vitest, test design |
| Automation Engineer | Test Automation Engineer | ConvoyMind, CSPM (Playwright, DevTools MCP) | Playwright, scripting |
| SDET | Software Development Engineer in Test | ConvoyMind, FlamLayout, CSPM | test frameworks, CI (CI limited) |

### Other Technical
| Title | Aliases | KB evidence | Skills to look for |
|---|---|---|---|
| Technical Analyst | Systems Analyst, Application Analyst, Technology Analyst | any project | requirements, SQL, documentation |
| Solutions Engineer | Sales Engineer, Customer Engineer, Implementation Engineer | ReNotify, CSPM, MetaModEnv | APIs, demos, integrations; communication evidence needed from me |
| Technical Support Engineer | Application Support, Support Engineer | CSPM, ReNotify | debugging, APIs, logs |
| Graduate Engineer Trainee | GET, Management Trainee (tech), Campus Hire | all | fundamentals, learning agility; degree/CGPA from profile |
| Associate Engineer | Junior Engineer, Trainee Engineer, Engineer I | all | general |

## Detection procedure
1. Read the posting title and responsibilities. Match title/aliases first; if ambiguous, match on responsibilities.
2. If it fits several families (e.g. "Software Engineer, ML Platform"), record a **primary** and a **secondary** family.
3. If it fits none, add a new row here (append) and tell me.
4. Family determines the project ranking start point, not the final choice (see `ANSWER_STRATEGIES.md`).

## Extending
Append rows under a family or add a new `###` heading. Record the date and reason in a comment below.

<!-- Change log: initial version created from Master Prompt section 7. -->
