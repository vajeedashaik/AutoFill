# APPLICATION QUESTION TYPES

Classify every application question into exactly one primary type (optionally a secondary). Then look up the strategy in `ANSWER_STRATEGIES.md`. Matching is by intent, not exact wording.

Answer source legend: **PROFILE** = `PERSONAL_PROFILE.md` (factual, exact); **KB** = project files via `KNOWLEDGE_BASE_INDEX.md`; **RESEARCH** = company research; **ASK** = stop and ask me if not present.

| Type | Intent | Example phrasings / synonyms | Source | Risk |
|---|---|---|---|---|
| PERSONAL_INFORMATION | name, contact, address, links | First/last name, email, phone, city, LinkedIn, GitHub, portfolio, current location | PROFILE | Low (exact copy) |
| EDUCATION | degree, institution, grades | University, degree, major, CGPA/GPA, graduation year, 10th/12th %, relevant coursework | PROFILE | Low |
| EXPERIENCE | jobs, internships, years | Work history, total experience, current employer, roles held | PROFILE | Medium (never invent) |
| TECHNICAL_SKILLS | tools/languages known | "Rate yourself in Python", skills list, proficiency, technologies you have used | PROFILE + KB | Medium |
| PROJECT_EXPERIENCE | describe a project | "Tell us about a project you are proud of", "Describe a technical project", "Link to a project" | KB | Medium |
| ACHIEVEMENTS | awards, hackathons | Accomplishments, awards, publications, rankings | PROFILE | Medium |
| COMPANY_MOTIVATION | why this company | "Why do you want to work here?", "Why this company?", "What interests you about our organization?", "What attracts you to our company?", "Why us?" | RESEARCH + KB + PROFILE | Medium |
| ROLE_MOTIVATION | why this role | "Why are you interested in this position?", "Why this role?", "What excites you about this job?" | JD + KB | Medium |
| CAREER_GOALS | future direction | "Where do you see yourself in 5 years?", "Career aspirations", "Long-term goals" | PROFILE section 8 | Medium |
| STRENGTHS | what I am good at | "What are your strengths?", "What makes you a good fit?", "What do you bring?" | PROFILE + KB | Medium |
| WEAKNESSES | growth areas | "Greatest weakness?", "Area you are improving?" | PROFILE section 15 (ASK if empty) | Medium |
| LEADERSHIP | led people/initiatives | "Describe a time you led", "Leadership experience", "Took initiative" | PROFILE section 7 / KB (ASK if none) | High (easy to fabricate) |
| TEAMWORK | collaboration | "Describe working in a team", "Conflict with a teammate", "Collaboration example" | PROFILE / KB (ASK if none) | High |
| PROBLEM_SOLVING | debugging / hard problem | "Tell us about a challenge you solved", "Hardest bug", "Time you overcame an obstacle" | KB (challenges/bugs sections) | Medium |
| WORK_AUTHORIZATION | legal right to work, sponsorship | "Are you authorized to work in...?", "Will you require sponsorship?", "Visa status" | PROFILE section 9 only | **HIGH** |
| RELOCATION | willingness to move | "Willing to relocate?", "Comfortable working onsite in...?", "Preferred office" | PROFILE sections 8, 12 only | **HIGH** |
| SALARY | pay expectations | "Expected CTC/salary/stipend", "Current compensation", "Desired pay" | PROFILE section 11 only | **HIGH** |
| AVAILABILITY | start date, notice | "Earliest start date", "Notice period", "Internship duration available" | PROFILE section 10 only | **HIGH** |
| OPEN_ENDED_TECHNICAL | explain / design / how-would-you | "How would you design...?", "Explain a concept you know well", "Describe your experience with X", "Anything else we should know (technical)?" | KB + PROFILE | Medium |

## Additional types (fields, not essays)
| Type | Examples | Source | Risk |
|---|---|---|---|
| LINKS | LinkedIn, GitHub, portfolio, personal site | PROFILE section 2 | Low |
| CERTIFICATIONS | certifications list | PROFILE section 6 | Medium |
| DEMOGRAPHIC_LEGAL | gender, ethnicity, veteran, disability, criminal record, background-check consent, legal declarations, data-processing consent | PROFILE section 15 only | **HIGH** (never infer) |
| REFERRAL_SOURCE | how did you hear about us | PROFILE section 15 | Low |
| OTHER | anything unclassified | ASK | - |

## Classification procedure
1. Strip greeting/fluff; find the verb ("why", "describe", "rate", "are you authorized").
2. Match against synonyms above; if several match, pick the type whose Source is most restrictive (HIGH > PROFILE > KB).
3. If nothing matches, classify as OTHER and ask me.
4. Record `question -> type -> source used` in the plan before filling (`ANSWER_STRATEGIES.md`, "Before filling").

## HIGH-risk rule (Master Prompt §19)
WORK_AUTHORIZATION, RELOCATION, SALARY, AVAILABILITY, DEMOGRAPHIC_LEGAL: use only explicit values from `PERSONAL_PROFILE.md`. If the value is `[FILL: ...]`, stop and ask. Never infer from location, education, or projects.
