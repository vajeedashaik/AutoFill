# ANSWER STRATEGIES

How to build each kind of answer. Types come from `APPLICATION_QUESTION_TYPES.md`. Every strategy is bound by `KNOWLEDGE_BASE_RULES.md`.

## Application Context (build fresh per application, Master Prompt §12-13)
```
APPLICATION CONTEXT
Company / Role / Job family (primary + secondary) / Location
Required skills / Preferred skills / Technologies / Responsibilities
Experience level / Degree requirement / Industry-domain / Keywords
Company context (with sources)
Relevant projects (ranked, with justification)
Relevant experience / skills / certifications
Questions (each with type + source)
Unknown fields
```
Never carry any of this to another application.

## Project selection (Master Prompt §10)
Never pick projects at random. For each application:
1. Determine job family (`JOB_FAMILIES.md`) and extract required skills, preferred skills, technologies, responsibilities, domain, industry.
2. Normalize skills with `SKILL_TAXONOMY.md` (e.g. "Postgres" -> PostgreSQL).
3. Score each project in `KNOWLEDGE_BASE_INDEX.md`, opening its canonical file to confirm:
   - **Skill overlap** with required skills (highest weight)
   - **Technology overlap**
   - **Domain/industry fit**
   - **Responsibility fit** (does the project show the kind of work described?)
   - **Preferred-skill overlap**
   - **Depth of evidence** (real, tested, deployed > simulated demo)
   - **Ownership**: "My contribution" filled? If blank, the project is usable only as "what the project is" and I must confirm before first-person use.
   - **Caveat penalty**: unfinished training, unverified results, etc. (does not exclude, but constrains wording)
4. Rank, then choose 1-3. **Write the justification** ("Selected X because the posting needs A and B; the project file shows A at <section> and B at <section>").
5. Different questions on the same application may use different projects; avoid repeating one project for every answer unless it is clearly the best.

Worked example (structure only, no personal claims): Data Engineer posting needing SQL, Python, ETL, AWS, PostgreSQL, data pipelines -> candidates GrowwPulse (scheduled ingestion, Postgres), ReNotify (Gmail->OCR pipeline, Postgres), Zillow (batch scoring, AWS), CSPM (queued pipeline on AWS, Postgres). Choose by which file gives the strongest concrete evidence for the specific question.

## Strategy by question type

### FACTUAL FIELDS (PERSONAL_INFORMATION, EDUCATION, LINKS, CERTIFICATIONS)
Copy the exact value from `PERSONAL_PROFILE.md`. Do not rephrase. If the field is a dropdown, choose the option that exactly matches; if none match, leave it and list for review.

### COMPANY_MOTIVATION
Inputs: company research (products, domain, engineering focus, terminology) + role requirements + my relevant skills + one relevant project + relevant experience.
Structure (2-4 sentences, or as limits allow):
1. A specific, sourced fact about the company (product, domain, engineering problem).
2. The overlap with a specific project/skill of mine (cite the project truthfully).
3. What I want to work on there, in terms the posting supports.
Forbidden without evidence: "I admire your innovative culture", "industry leader", "passionate about your mission". If research gives no specific hook, say so to me and ask instead of padding.

### ROLE_MOTIVATION
Inputs: role responsibilities + required skills + relevant experience + most relevant project.
Structure: what the role does -> which of those responsibilities my project evidence matches -> what I want to grow into (from profile career goals). No generic filler.

### PROJECT_EXPERIENCE / PROJECT_QUESTION
Inputs: question requirements + best-matching project + problem + **my** contribution + technology + a challenge + a result.
Structure (STAR-like): Problem -> my contribution -> approach/tech -> challenge -> result. Use only the file's figures. Apply that project's caveats (e.g. "simulated data", "not deployed"). If "My contribution" is blank: ask me before writing in first person.

### TECHNICAL_SKILLS / TECHNICAL_EXPERIENCE
Inputs: the technology + project evidence + my exact contribution + relevant outcome.
Structure: "Used <tech> in <project> to <what>, <outcome>." Only for technologies I confirmed. For proficiency ratings, use profile section 5; if blank, ask.

### OPEN_ENDED_TECHNICAL
Answer from concepts demonstrated in the KB (e.g. why a design choice was made, tradeoffs the file documents). Design decisions and "known limitations" sections in project files are good evidence of reasoning. Do not claim knowledge beyond what the KB supports; say what I would do differently only if I've told you.

### PROBLEM_SOLVING
Use the project files' bug/challenge/decision sections (e.g. CSPM "Bugs found & fixed", ReNotify "Known issues", FlamLayout "Phase 8 audit"). Format: situation -> diagnosis -> fix -> outcome. Only if my contribution to that fix is confirmed.

### STRENGTHS / CAREER_GOALS / WEAKNESSES
Use the wording and themes I supplied in profile sections 8, 13, 14, 15. If blank, ask. Do not invent a weakness.

### LEADERSHIP / TEAMWORK
High fabrication risk. Use only stories I have provided (profile sections 4, 7). If none, ask; do not turn "used AI assistants to build" into leadership.

### WORK_AUTHORIZATION / RELOCATION / SALARY / AVAILABILITY / DEMOGRAPHIC_LEGAL
Copy the profile value verbatim. If missing: stop, list the field, ask me.

## Writing constraints
- Apply `PERSONAL_PROFILE.md` section 13 (tone, length, banned words). If blank, default to plain, concrete, first person, no buzzword filler.
- Numbers exactly as in the source; never round up.
- Respect character/word limits; leave a small margin.
- Prefer a single clear example over a list of many.
- Check each answer with the grounding test below.

## Grounding test (run on every generated answer)
For each factual clause: (a) which file/section supports it? (b) is it about me or about the project team? (c) does it contradict a listed caveat? Delete any clause that fails.

## Before filling (Master Prompt §21)
```
Company:
Role:
Job family:
Relevant projects: 1. 2. 3.  (with one-line justification each)
Important skills: 1. 2. 3.
Application questions requiring generated responses: 1. 2.
Fields requiring manual input: 1. 2.
```

## After filling (Master Prompt §22)
```
APPLICATION PREPARED
Company / Role
Filled: personal, education, skills, links, resume, generated answers (tick what applies)
Projects used:
Fields requiring review: (work authorization, salary, ...)
Submission: NOT SUBMITTED
```

## Approved answers
When I approve an answer, save it to `APPLICATION/APPROVED_ANSWERS.md` with the question type and the company it was written for. Reuse it only for the same question type, adapted with the new application's own context (Rule 7), never pasted blindly.
