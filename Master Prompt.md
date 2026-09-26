# PERSONAL JOB APPLICATION AGENT — MASTER BUILD PROMPT

## ROLE

You are building a personal job-application intelligence and browser automation system for me.

The goal is NOT to build a generic AI job application bot.

The goal is to build a system that deeply understands MY professional background and can use that knowledge to prepare job applications.

I already have a folder containing detailed `.md` files for my projects.

These project files are the CORE PROJECT KNOWLEDGE BASE.

Your job is to:

1. Inspect and understand all existing project-context files.
2. Determine what additional information about ME is required.
3. Create a structured personal knowledge base with placeholders that I will fill manually.
4. Build an architecture that combines:
   - My personal knowledge
   - My project knowledge
   - My experience
   - My skills
   - My education
   - My certifications
   - My achievements
   - The target job description
   - Company research
   - Application form questions
5. Use this information to prepare job applications.
6. Use Chrome DevTools MCP to interact with the browser and fill forms.
7. NEVER submit an application.

The first priority is KNOWLEDGE QUALITY.
The second priority is APPLICATION REASONING.
The third priority is BROWSER AUTOMATION.

Do not reverse these priorities.

---

# 1. FIRST: INSPECT THE EXISTING WORKSPACE

Before creating anything:

- Inspect the entire workspace.
- Find the folder containing the project `.md` files.
- Read every project-context file.
- Understand the structure and terminology used in those files.
- Do NOT recreate information that already exists in the project files.
- Do NOT overwrite existing project files unless explicitly instructed.
- Treat the existing project files as authoritative project knowledge.

Create a mental/indexed understanding of:

- Project names
- Project domains
- Technologies
- Languages
- Frameworks
- Databases
- Cloud platforms
- Architecture
- Technical contributions
- Challenges
- Outcomes
- Skills demonstrated
- Relevant job families
- Relevant industries
- Important technical terminology

The project files should remain the primary source of truth for project-specific information.

---

# 2. CREATE A PERSONAL KNOWLEDGE BASE

After inspecting the project files, determine what is missing about ME.

Create:

`PERSONAL_PROFILE.md`

This file must contain placeholders rather than invented information.

DO NOT guess any personal information.

Use clear placeholders such as:

`[FILL: FULL NAME]`

`[FILL: PHONE NUMBER]`

`[FILL: EXPECTED GRADUATION DATE]`

etc.

The purpose is for me to open the file and fill it manually.

---

# 3. PERSONAL_PROFILE.md STRUCTURE

Create a comprehensive structure covering everything that could reasonably be required during a job application.

Include:

## Identity

- Full name
- Preferred name
- First name
- Last name
- Email
- Phone
- City
- State
- Country
- Current location

## Professional Links

- LinkedIn
- GitHub
- Portfolio
- Personal website
- Other relevant links

## Education

For every degree:

- Institution
- Degree
- Major
- Specialization
- Start date
- Expected graduation
- CGPA
- Percentage if applicable
- Relevant coursework
- Academic achievements

## Experience

For every internship/work experience:

- Company
- Role
- Location
- Start date
- End date
- Employment type
- Project
- Responsibilities
- Exact technical contributions
- Technologies used
- Achievements
- Impact
- Challenges
- Lessons learned

IMPORTANT:

Do not duplicate project information unnecessarily if it already exists in the project `.md` files.

Instead, reference the relevant project.

## Skills

Create a structured skills inventory.

Categories should include:

- Programming languages
- Backend
- Frontend
- Databases
- Data engineering
- Data science
- Machine learning
- AI
- Cloud
- DevOps
- Testing
- APIs
- Tools
- Frameworks
- Other technical skills

For each important skill, provide placeholders for:

- Skill
- Experience level
- Where I used it
- Relevant project(s)
- Relevant experience
- Confidence / depth

Do not invent proficiency levels.

## Certifications

For each:

- Certification
- Provider
- Date
- Credential ID if applicable
- Verification URL if applicable
- Skills represented

## Achievements

Include:

- Hackathons
- Publications
- Research
- Awards
- Competitions
- Presentations
- Leadership
- Academic achievements
- Other accomplishments

## Career Preferences

Include:

- Preferred job families
- Preferred industries
- Preferred locations
- Remote/hybrid/onsite preference
- Relocation preference
- Internship/full-time preference
- Desired role types
- Technologies I want to work with
- Technologies I do not want to emphasize
- Career goals

## Work Authorization

Create explicit placeholders for:

- Current work authorization
- Countries where I can work
- Visa requirements
- Sponsorship requirements

DO NOT fill these automatically.

## Availability

Include:

- Earliest joining date
- Notice period
- Current status
- Internship availability
- Full-time availability

## Compensation

Include placeholders for:

- Current compensation if relevant
- Expected compensation
- Minimum acceptable compensation
- Preferred compensation format

Do not make assumptions.

## Relocation

Include:

- Willingness to relocate
- Preferred cities
- Countries willing to relocate to
- Remote preference

## Personal Writing Style

Create placeholders for:

- Preferred tone
- Preferred answer length
- Words/phrases I like
- Words/phrases I dislike
- How technical answers should be
- How I describe myself
- Claims I do not want to make
- Common writing preferences

## Application Preferences

Include:

- How I want to describe myself
- Preferred project emphasis
- Preferred technical strengths
- Topics I want emphasized
- Topics I do not want unnecessarily emphasized
- How I want achievements presented

---

# 4. DO NOT DUPLICATE PROJECT KNOWLEDGE

The existing project `.md` files are the authoritative source for project information.

For example, if:

`PROJECTS/ReNotify.md`

already contains:

- Architecture
- Technologies
- Challenges
- Technical decisions
- Contributions
- Results

do NOT copy all of that into `PERSONAL_PROFILE.md`.

Instead, use references such as:

```text
Project:
ReNotify

Canonical source:
PROJECTS/ReNotify.md

The system should be designed so Claude can traverse from the personal profile to the detailed project context.

5. CREATE A KNOWLEDGE BASE INDEX

Create:

KNOWLEDGE_BASE_INDEX.md

This should act as the map of the entire knowledge base.

Example:

PERSONAL
→ PERSONAL_PROFILE.md

PROJECTS
→ PROJECTS/ReNotify.md
→ PROJECTS/TensAI.md
→ ...

EXPERIENCE
→ EXPERIENCE.md

SKILLS
→ SKILLS.md

CERTIFICATIONS
→ CERTIFICATIONS.md

ACHIEVEMENTS
→ ACHIEVEMENTS.md

For every project, include:

Name
File
Domains
Main technologies
Relevant job families
Major skills demonstrated

This allows fast retrieval.

6. CREATE A SKILL TAXONOMY

Create:

SKILL_TAXONOMY.md

The purpose is to normalize different ways companies describe the same skill.

Examples:

Postgres
PostgreSQL
PostgresSQL

→ PostgreSQL
Amazon Web Services
AWS

→ AWS
ETL pipeline
Data pipeline
Data ingestion pipeline

→ Data Engineering / ETL / Data Pipelines

Include:

Canonical skill
Synonyms
Related skills
Parent category
Relevant projects

Do not invent skills that do not exist in my actual knowledge base.

7. CREATE A JOB FAMILY TAXONOMY

Create:

JOB_FAMILIES.md

Support common job families such as:

Software Engineering
Software Engineer
SDE
Backend Engineer
Frontend Engineer
Full Stack Engineer
Java Developer
Python Developer
Data
Data Engineer
Data Analyst
Data Scientist
BI Analyst
AI/ML
ML Engineer
AI Engineer
GenAI Engineer
Computer Vision Engineer
NLP Engineer
Cloud / Infrastructure
Cloud Engineer
Cloud Architect
Cloud Support
DevOps Engineer
Platform Engineer
SRE
QA
QA Engineer
Automation Engineer
SDET
Other Technical
Technical Analyst
Solutions Engineer
Technical Support Engineer
Graduate Engineer Trainee
Associate Engineer

Make this extensible.

8. CREATE A QUESTION TAXONOMY

Create:

APPLICATION_QUESTION_TYPES.md

Classify application questions into categories such as:

PERSONAL_INFORMATION
EDUCATION
EXPERIENCE
TECHNICAL_SKILLS
PROJECT_EXPERIENCE
ACHIEVEMENTS
COMPANY_MOTIVATION
ROLE_MOTIVATION
CAREER_GOALS
STRENGTHS
WEAKNESSES
LEADERSHIP
TEAMWORK
PROBLEM_SOLVING
WORK_AUTHORIZATION
RELOCATION
SALARY
AVAILABILITY
OPEN_ENDED_TECHNICAL

Also include common synonyms.

For example:

"Why do you want to work here?"
"Why this company?"
"What interests you about our organization?"
"What attracts you to our company?"

→ COMPANY_MOTIVATION
9. CREATE ANSWER STRATEGIES

Create:

ANSWER_STRATEGIES.md

Define how the system should answer different question types.

COMPANY_MOTIVATION

Use:

Company context
+
Role requirements
+
Relevant user skills
+
Relevant project
+
Relevant experience

Do NOT produce generic answers like:

"I admire your company's innovative culture..."

unless there is actual evidence supporting the statement.

ROLE_MOTIVATION

Use:

Role responsibilities
+
Required skills
+
Relevant experience
+
Most relevant project
PROJECT_QUESTION

Use:

Question requirements
+
Best matching project
+
Problem
+
User's contribution
+
Technology
+
Challenge
+
Result
TECHNICAL_EXPERIENCE

Use:

Technology
+
Project evidence
+
Exact contribution
+
Relevant outcome
10. PROJECT SELECTION SYSTEM

Claude must NOT randomly choose projects.

For every job/application, determine:

Job family
Required skills
Preferred skills
Technologies
Responsibilities
Domain
Industry
Relevant experience

Then inspect the project knowledge base.

Rank projects based on relevance.

Example:

JOB:
Data Engineer

Requirements:
SQL
Python
ETL
AWS
PostgreSQL
Data pipelines

Potential projects:

ReNotify
PriceLens
TensAI

Determine which project provides the strongest evidence for the actual question.

The selected project must be justified.

11. COMPANY RESEARCH

Before answering company-specific questions, research the company when necessary.

Prioritize:

Official job description
Official company website
Official careers page
Official engineering/product pages
Reliable public sources

Extract:

Company
Industry
Products
Business domain
Technologies
Engineering focus
Role requirements
Relevant company terminology

The purpose is to answer:

"Why this company?"

with actual company-specific reasoning connected to my real experience.

Do not fabricate company information.

12. JOB DESCRIPTION ANALYSIS

For every application, identify:

Company
Role
Location
Job family
Required skills
Preferred skills
Technologies
Responsibilities
Experience level
Degree requirements
Industry/domain
Keywords

Then compare those requirements against my Career KB.

13. APPLICATION CONTEXT

For every job application, create an internal context like:

APPLICATION CONTEXT

Company:
[detected]

Role:
[detected]

Job Family:
[detected]

Location:
[detected]

Required Skills:
[...]

Preferred Skills:
[...]

Company Context:
[...]

Relevant Projects:
[...]

Relevant Experience:
[...]

Relevant Skills:
[...]

Relevant Certifications:
[...]

Questions:
[...]

Unknown Fields:
[...]

This context must be specific to the current application.

Do not carry assumptions from one company/application to another.

14. BROWSER AUTOMATION

The current browser execution layer is:

Chrome DevTools MCP

Do not build a Chrome extension yet.

The intended workflow is:

User gives job/application URL
        ↓
Claude opens/inspects page using Chrome DevTools MCP
        ↓
Identify company and role
        ↓
Read job/application context
        ↓
Research company if needed
        ↓
Read Career KB
        ↓
Determine relevant projects/skills
        ↓
Answer application questions
        ↓
Use Chrome DevTools MCP to fill fields
        ↓
STOP
        ↓
User reviews
        ↓
User manually submits
15. FORM FIELD HANDLING

Claude should inspect:

Labels
Input names
IDs
Placeholders
ARIA labels
Nearby text
Select options
Radio buttons
Checkboxes
Textareas
Form section headings

Classify fields into:

PERSONAL
EDUCATION
EXPERIENCE
SKILLS
CERTIFICATIONS
LINKS
WORK_AUTHORIZATION
RELOCATION
SALARY
AVAILABILITY
OPEN_ENDED

Then map them to the appropriate source in the Career KB.

16. FACTUAL AUTOFILL

For deterministic information:

First Name
Last Name
Email
Phone
University
Degree
Major
Graduation Year
CGPA
LinkedIn
GitHub
Portfolio

Use the exact information in the personal profile.

Do not rewrite factual values unless required by the form.

17. OPEN-ENDED ANSWERS

For open-ended questions:

Classify the question.
Understand the job.
Understand the company if relevant.
Search the Career KB.
Identify relevant evidence.
Select the best project/experience.
Construct a personalized answer.
Check that every factual claim is supported.
Respect word/character limits.
Fill the field.
18. NEVER HALLUCINATE

This is a hard requirement.

Claude must NEVER invent:

Experience
Technologies
Responsibilities
Achievements
Metrics
Certifications
Job titles
Company relationships
Skills
Work authorization
Salary information
Legal information

If information is unavailable:

UNKNOWN — USER INPUT REQUIRED

Do not guess.

19. HIGH-RISK QUESTIONS

For questions involving:

Work authorization
Visa sponsorship
Criminal history
Disability
Veteran status
Demographic information
Legal declarations
Salary
Relocation
Availability

Use explicit user-provided information only.

If missing, stop and ask me.

Do not infer.

20. NEVER SUBMIT

This is one of the most important rules.

You may fill the application.

You may navigate the form.

You may select options when the answer is explicitly known.

You may upload my resume when appropriate.

But:

NEVER CLICK:
Submit
Submit Application
Apply
Final Submit
Complete Application
Confirm Application
Any equivalent finalization control

The application must always stop before final submission.

I will manually review and submit it.

21. BEFORE FILLING

Before modifying the page, provide a concise plan:

Company:
Role:

Job family:

Relevant projects:
1.
2.
3.

Important skills:
1.
2.
3.

Application questions requiring generated responses:
1.
2.

Fields requiring manual input:
1.
2.

Then proceed with filling.

22. AFTER FILLING

Provide a completion report:

APPLICATION PREPARED

Company:
Role:

Filled:
✓ Personal information
✓ Education
✓ Skills
✓ Links
✓ Resume
✓ Generated answers

Projects used:
- Project A
- Project B

Fields requiring review:
⚠ Work authorization
⚠ Salary expectation

Submission:
NOT SUBMITTED
23. DO NOT OVERWRITE EXISTING KNOWLEDGE

Existing project files are valuable.

Do not rewrite or simplify them.

Do not replace detailed project context with short summaries.

Instead, create indexes and references around them.

The architecture should make it easy for Claude to locate detailed information when needed.

24. INFORMATION GATHERING MODE

After creating the initial PERSONAL_PROFILE.md, do NOT immediately fill it with guesses.

Instead, inspect what is already available in the workspace.

Then create:

PROFILE_INFORMATION_GAPS.md

List:

Information already known
Information partially known
Information missing
Information that needs verification

Then ask me for the missing information.

The goal is to avoid asking me for information that already exists somewhere in the workspace.

25. BUILD A QUESTIONNAIRE

Create:

PERSONAL_PROFILE_QUESTIONNAIRE.md

This should contain every piece of information I still need to provide.

Organize it into logical sections:

Identity
Contact
Education
Experience
Skills
Certifications
Achievements
Career preferences
Work authorization
Relocation
Availability
Compensation
Writing style
Application preferences

Use clear placeholders.

Example:

### Preferred Name

[FILL: What name should be used in applications?]

### Work Authorization

[FILL: Countries where you are authorized to work]

### Earliest Joining Date

[FILL: Date]

Make this questionnaire comprehensive.

Do not ask me for information that is already clearly available in the workspace.

26. VALIDATION SYSTEM

Create:

KNOWLEDGE_BASE_RULES.md

Define rules such as:

RULE 1:
Never invent facts.

RULE 2:
Project-specific claims must come from the corresponding project file.

RULE 3:
Personal facts must come from PERSONAL_PROFILE.md.

RULE 4:
Company-specific claims should come from company research.

RULE 5:
If information is missing, ask the user.

RULE 6:
Never submit an application.

RULE 7:
Do not use one company's context for another company.

RULE 8:
Generated answers must remain grounded in the Career KB.

RULE 9:
Respect application character/word limits.

RULE 10:
When uncertain, prefer manual review.
27. DO NOT BUILD A COMPLEX SOFTWARE SYSTEM YET

At this stage, prioritize:

Markdown knowledge
+
Clear structure
+
Good indexing
+
Strong reasoning instructions
+
Claude
+
Chrome DevTools MCP

Do NOT prematurely build:

A Chrome extension
A custom AI backend
An API gateway
A vector database
A complex frontend
Authentication
Cloud infrastructure

Those can come later if the workflow proves useful.

28. TESTING THE SYSTEM

After creating the architecture, test it against several job types.

At minimum:

Software Engineer
Backend Engineer
Data Engineer
Data Analyst
ML Engineer
AI Engineer
Cloud Engineer
DevOps Engineer
QA / Automation Engineer
Technical Analyst

For each test:

Determine job family.
Identify important skills.
Select relevant projects.
Explain why.
Identify relevant experience.
Generate a sample answer.
Check factual grounding.

Do not modify the Career KB with fictional information from these tests.

29. IMPORTANT: PROTECT EXISTING PROJECT KNOWLEDGE

You have permission to:

Inspect files
Create new knowledge-base files
Create templates
Create indexes
Create taxonomies
Create scripts/configuration needed for the workflow

Do NOT:

Delete project files
Rewrite project context
Replace the existing KB
Add fake information
Submit applications
Create external accounts
Spend money
Expose private information

If a major architectural decision is unclear, explain the options before making an irreversible decision.

30. FINAL EXPECTED WORKSPACE

After completing the initial setup, the workspace should contain something similar to:

CAREER_KB/
│
├── PERSONAL_PROFILE.md
├── PERSONAL_PROFILE_QUESTIONNAIRE.md
├── PROFILE_INFORMATION_GAPS.md
├── KNOWLEDGE_BASE_INDEX.md
├── SKILL_TAXONOMY.md
├── JOB_FAMILIES.md
├── APPLICATION_QUESTION_TYPES.md
├── ANSWER_STRATEGIES.md
├── KNOWLEDGE_BASE_RULES.md
│
├── PROJECTS/
│   ├── [existing project files]
│   └── ...
│
└── APPLICATION/
    ├── APPROVED_ANSWERS.md
    └── ...

Do not duplicate the existing project files.

31. NORMAL OPERATING WORKFLOW

Once the knowledge base is populated, the normal workflow should be as simple as:

1. I give you the job application URL.

2. You inspect the page using Chrome DevTools MCP.

3. You identify the company and role.

4. You research the company and job where appropriate.

5. You inspect my Career KB.

6. You determine the most relevant:
   - Projects
   - Skills
   - Experience
   - Certifications
   - Achievements

7. You answer application questions using only truthful information from my KB.

8. You fill the form using Chrome DevTools MCP.

9. You stop before submission.

10. I review everything.

11. I manually click Submit.

The ultimate goal is:

I should only have to provide the application link and review what you prepared.

Everything else should be handled by the knowledge base + Claude + Chrome DevTools MCP.

32. START NOW

Begin immediately.

Step 1

Inspect the entire workspace and locate all existing project .md files.

Step 2

Read and understand the project knowledge.

Step 3

Determine what information about me is already available and what is missing.

Step 4

Create:

PERSONAL_PROFILE.md
PERSONAL_PROFILE_QUESTIONNAIRE.md
PROFILE_INFORMATION_GAPS.md
KNOWLEDGE_BASE_INDEX.md
SKILL_TAXONOMY.md
JOB_FAMILIES.md
APPLICATION_QUESTION_TYPES.md
ANSWER_STRATEGIES.md
KNOWLEDGE_BASE_RULES.md
Step 5

Do not invent any missing information.

Step 6

Show me the information questionnaire/gaps that I need to fill.

Step 7

After I fill the personal information, validate the complete Career KB.

Step 8

Then test the application workflow using Chrome DevTools MCP on a real job application.

Remember:

The existing project .md files are the source of truth for project knowledge.

PERSONAL_PROFILE.md is the source of truth for personal facts.

Claude is the reasoning layer.

Company research provides external context.

Chrome DevTools MCP is the browser execution layer.

The user always performs the final submission.

NEVER SUBMIT AN APPLICATION.