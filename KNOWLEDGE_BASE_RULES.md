# KNOWLEDGE BASE RULES

Hard rules for the application agent. Where two rules conflict, the more conservative one wins.

## Source of truth
| Kind of fact | Only allowed source |
|---|---|
| Personal facts (name, contact, education, dates, compensation, authorization) | `PERSONAL_PROFILE.md` |
| Project facts (what it does, tech, results, challenges) | the project's file in `Project KnowledgeBase/` |
| What *I* did in a project | `KNOWLEDGE_BASE_INDEX.md` -> "My contribution" (or profile section 4). Blank = unknown |
| Company / role facts | the job posting + official company sources researched for this application |
| Approved reusable wording | `APPLICATION/APPROVED_ANSWERS.md` |

## Rules
1. **Never invent facts.** No experience, skills, metrics, certifications, titles, employers, achievements, legal info.
2. **Project-specific claims come from the project file.** Quote numbers exactly as the file states them.
3. **Personal facts come from `PERSONAL_PROFILE.md`.** Copy factual values verbatim; do not reformat unless the field requires it.
4. **Company-specific claims come from company research** done for this application, with the source noted in the plan.
5. **Missing information -> write `UNKNOWN - USER INPUT REQUIRED` and ask me.** Never guess. A `[FILL: ...]` placeholder counts as missing.
6. **Never submit.** Do not click Submit, Submit Application, Apply, Final Submit, Complete Application, Confirm Application, or any equivalent finalizing control. Stop, report, and let me submit.
7. **No cross-application contamination.** Do not reuse one company's context, motivations or research in another. Rebuild the Application Context each time.
8. **Answers stay grounded.** Every factual sentence in a generated answer must trace to a KB source. Remove anything that doesn't.
9. **Respect limits.** Count characters/words for each field; stay under the limit; never truncate mid-sentence.
10. **When uncertain, prefer manual review.** Fill nothing rather than something doubtful; list the field under "Fields requiring review".

## Additional rules specific to this KB
11. **Ownership gate.** A project's "My contribution" must be filled before I claim work in first person. Until then, describe the project neutrally ("the project does...") or ask me.
12. **Carry the caveats.** If a project file lists limitations (synthetic data, simulated, training not completed, not deployed, unverified scores), the answer must not contradict them. Examples: never say "trained/achieved" for MetaDebate or CrisisCore results; never present Zillow live predictions as the real ensemble; say "simulated"/"synthetic" where the file says so.
13. **Skills gate.** Only claim skills listed in `SKILL_TAXONOMY.md` that I have confirmed in `PERSONAL_PROFILE.md` section 5. Skills marked "NOT evidenced" (Java, Kubernetes, Terraform, Power BI, Tableau, Linux, RAG, DSA) require my explicit confirmation.
14. **Existing files are protected.** Do not delete or rewrite anything in `Project KnowledgeBase/`; only add indexes and references.
15. **Secrets.** Never reproduce credentials, tokens or keys mentioned in any file (Zillow, MetaDebate, MetaModEnv contain exposed ones). Never put them in an application.
16. **Privacy.** Only fill fields with my data when the field is on the application I asked to prepare. Do not send my data anywhere else. Uploading my resume is allowed only where the form asks for it.
17. **High-risk questions** (work authorization, sponsorship, criminal history, disability, veteran, demographics, legal declarations, salary, relocation, availability): explicit values from the profile only; otherwise stop and ask.
18. **Tests use no fiction.** Test runs must not write invented facts into the KB.
19. **No spending, no accounts.** Never create external accounts or spend money.
20. **Irreversible decisions.** If an architectural choice is unclear, explain options first.

## Pre-fill checklist (agent must satisfy before touching the page)
- [ ] Application Context built for this application only
- [ ] Every field classified (`APPLICATION_QUESTION_TYPES.md`)
- [ ] Every value mapped to an allowed source, or marked UNKNOWN
- [ ] Projects chosen and justified (`ANSWER_STRATEGIES.md`)
- [ ] Plan shown to me (Master Prompt §21)

## Post-fill checklist
- [ ] No submit control clicked
- [ ] Completion report shown (Master Prompt §22) with "Submission: NOT SUBMITTED"
- [ ] All unknown / high-risk fields listed for my review
