# STAR Stories Bank

> **Tier:** T1 (know cold) · **Related:** [Tell me about yourself](01-tell-me-about-yourself.md), [Common HR questions](03-common-hr-questions.md), [Resume line-by-line defense](06-resume-line-by-line-defense.md), [CrystalCommerce](../17-projects-deep-dive/01-crystalcommerce.md), [Coloc resident app](../17-projects-deep-dive/03-coloc-resident-app.md), [Platform migration](../17-projects-deep-dive/04-multi-tenant-platform-migration.md), [LMS](../17-projects-deep-dive/05-lms-coursera-clone.md)
> **On your resume:** every role. Behavioral questions ("Tell me about a time...") are answered with stories from your real experience, and this file maps your resume to the stories you should prepare.

## Quick revision
- **STAR:** Situation, Task, Action, Result. **CAR:** Context, Action, Result (shorter, for quick rounds).
- **Proportions:** Situation and Task about 20%, Action about 60%, Result about 20%. Most weak answers are all situation and no action.
- **Say "I" in the Action.** "We" is fine for context; the interviewer is hiring you.
- **Results must be real.** If you have no metric, use a concrete, true outcome (it shipped, the client accepted it, the bug stopped recurring) plus what you learned. Never estimate numbers you did not measure.
- **Prepare 6 to 8 core stories** and map each one to several prompts; you cannot prepare 50 separate answers.
- **Every skeleton below needs your real details.** Nothing in this file is a finished story; the prompts tell you what to fill in.
- **Length:** 1.5 to 2.5 minutes per story, then stop and let them ask.
- **Expect follow-ups:** "What exactly did you do?", "What would you do differently?", "How did you know it worked?", "What did the others think?"

## 1. What behavioral questions test
Interviewers assume past behavior predicts future behavior. Each prompt probes one or two competencies:

| Competency | Typical prompts |
|---|---|
| Problem solving and technical depth | Hardest technical challenge, a production bug, most challenging project |
| Learning | Learning something fast, working in an unfamiliar codebase or stack |
| Ownership and initiative | Going beyond your role, proposing an idea, proudest achievement |
| Collaboration | Conflict, disagreeing with a decision, working with a difficult person |
| Communication | Explaining something technical, teaching, working with clients |
| Handling pressure | Tight deadlines, competing priorities |
| Self-awareness | Failure, mistakes, critical feedback |

Know which competency a prompt targets, and make sure your story shows it explicitly.

## 2. STAR and CAR

| Letter | What to include | Keep it to |
|---|---|---|
| **S**ituation | Where, when, what was going on, why it mattered | Two or three sentences |
| **T**ask | Your specific responsibility or goal | One or two sentences |
| **A**ction | What you did, step by step, and why; the obstacle; the decision you made | The bulk of the answer |
| **R**esult | The outcome (measured if you truly have numbers), plus what you learned or changed afterward | Two or three sentences |

CAR merges situation and task into **Context**, then Action and Result. Use it when the interviewer is rushing or asks several behavioral questions in a row.

## 3. Weak vs good: one example
> The scenario below is generic and only shows the shape of an answer. It is not from your resume, so never reuse it; build yours from the skeletons in section 8.

**Weak:**
> "We had a slow page in one of our apps. I checked it and added caching, and after that it was fast. The client was happy."

Problems: no context, no reasoning, "we" vs "I" unclear, a vague result, and a fix (caching) that hides the real cause.

**Good (STAR):**
> **Situation:** "An internal admin page that lists orders had become slow; staff said it took several seconds to open, and it got worse as data grew."
> **Task:** "I was asked to find the cause and fix it without changing how the page worked for staff."
> **Action:** "I reproduced it locally with production-like data and read the SQL log. The page ran one query for the orders and then one more per order to load the customer: an N+1 query. I added eager loading for the customer association, added an index on the column the page sorted by, paginated the list, and checked the query plan to confirm the index was used. I also added a test that fails if the number of queries grows with the number of rows."
> **Result:** "The page loaded in under a second afterwards, and staff stopped reporting it. The lesson I kept: read the query log before reaching for caching."

**Same story as CAR (about 40 seconds):**
> "An admin orders page had become slow as data grew. I traced it to an N+1 query in the SQL log, added eager loading, an index for the sort column and pagination, and a test that guards the query count. It went back to loading in under a second."

## 4. Rules for strong stories
- **Specific beats impressive.** One real detail ("the SQL log showed one query per order") is more convincing than big claims.
- **Show your reasoning:** the options you considered and why you chose one.
- **Include the obstacle.** A story where everything went smoothly shows nothing.
- **Separate "I" from "we":** "The team decided to ...; my part was ..."
- **Numbers only when real.** If you know them, use them; if you do not, say what you observed.
- **End with learning:** what you do differently since.
- **Be ready to go one level deeper** on any sentence, technically and personally.
- **Never invent.** Interviewers probe details; a made-up story collapses on the second follow-up, and it is dishonest.

## 5. Results when you have no metrics
Juniors often lack measured outcomes. True, concrete results still work:
- It shipped (to production, to the App Store and Google Play, to the client's staging).
- The client or reviewer accepted it, or asked for it to be extended.
- A recurring bug or complaint stopped.
- Manual work disappeared (a task now runs as a background job).
- A teammate reused your approach or code.
- You received specific feedback from a lead, mentor or client.

If you have none of these, describe what you would measure now ("I'd track sync failures per day") and what you learned. That is honest and still shows judgment.

## 6. Your core stories
Candidate stories from your resume. Pick 6 to 8 and fill in the details before any interview.

| ID | Story | Source on your resume |
|---|---|---|
| A | Integrating PriceCharting as a sales channel | CrystalCommerce at Devntech |
| B | Working safely in a legacy Rails codebase | CrystalCommerce at Devntech |
| C | Shipping features and releases of the Coloc app | Colochousing at Devntech |
| D | The multi-tenant migration: sync APIs, marketplace page or org-scoped access | Colochousing at Devntech |
| E | One month of Rails training, then professional Rails work | Visnext, then Devntech |
| F | Proposing and building the LMS | PureLogics internship |
| G | Running two internships in the same months | PureLogics and RemoteDev |
| H | Teaching labs as a TA | PUCIT, 2023 to 2025 |
| I | Collaborating as a remote intern | RemoteDev |
| J | The final-year project | PUCIT FYP |

> **Fill in:** next to each ID, one line on the real event you would use (or "none" if nothing real fits). Drop the ones with nothing real behind them.

## 7. Story bank: prompts mapped to your experience

| Prompt | Best-fit stories | Why it fits |
|---|---|---|
| Hardest technical challenge / most challenging project | A, D | Third-party sync with consistency problems; tenant isolation in a new stack |
| Conflict with a teammate or stakeholder | C, D, I, J | Team work under deadlines; design discussions in a migration; group FYP |
| A failure or mistake | A, C, F | Integration bugs, release issues, scoping a project yourself |
| Learning something fast | E, D, C | Rails in a month; oRPC, TanStack Start and Drizzle; React Native |
| Tight deadline | C, G, F | Store releases, two internships at once, a three-month internship project |
| Working with legacy code | B | A legacy Rails platform serving 700+ stores |
| Handling ambiguity | F, A, D | Defining your own scope; an unfamiliar API; a moving migration plan |
| Teaching or mentoring | H | Two years helping students in labs |
| Working with international clients | A, C, D | Devntech's international clients, including a Belgian platform |
| Disagreeing with a decision | D, C | Technology and design choices in the migration or the app |
| A production bug | C, A | A live app with 400+ residents; a sync touching real store data |
| Proudest achievement | C, A, F | An app live on both stores; a new sales channel; a project you proposed |

## 8. STAR skeletons
Each skeleton gives you a starting sentence built only from resume facts, then prompts for the parts only you know.

### 8.1 Hardest technical challenge (also "your most challenging project")
**Best fit:** A (PriceCharting); alternative: D (org-scoped access control).

- **Situation (starter):** "On CrystalCommerce, a legacy Rails e-commerce platform serving 700+ game stores and 8M+ customers, the work was to integrate PriceCharting as a sales channel, with background jobs syncing listings, prices, stock and orders."
- **Task:** your exact responsibility within that.
- **Action:** the hardest specific problem (for example duplicate orders, stock drifting between channels, rate limits, matching products, a legacy code path you had to change), how you investigated it, the options you weighed, what you built, how you tested and rolled it out.
- **Result:** what shipped, who uses it, any outcome you actually know, and the lesson.

> **Fill in:** the one hardest problem, told in four to six concrete actions. Prepare the technical depth from [the CrystalCommerce deep-dive](../17-projects-deep-dive/01-crystalcommerce.md), because the follow-ups will be technical.

### 8.2 Conflict with a teammate or stakeholder
**Best fit:** whichever real disagreement you had at Devntech, in an internship, in the FYP team or in a lab.

- **Situation:** who, what project, what the disagreement was about (an approach, a deadline, a code review comment, a priority).
- **Task:** what you needed (agreement, a decision, a working relationship).
- **Action:** how you listened first, restated their view, brought evidence (a prototype, numbers, docs), proposed a compromise or escalated respectfully.
- **Result:** the decision, the relationship afterwards, what you learned about working with that person.

> **Fill in:** a real disagreement. Keep it about the work, never about personalities, and make sure the other person comes across as reasonable.

### 8.3 A failure or mistake
**Best fit:** a bug you introduced, a release that went wrong, an estimate you missed, or scope you underestimated (for example in the LMS).

- **Situation:** what you were doing and what went wrong.
- **Task:** what you were responsible for.
- **Action:** how you noticed, how you told the right people quickly, how you fixed it, and what you changed so it would not happen again (a test, a checklist, a review step).
- **Result:** the impact contained, the fix, and the habit you kept.

> **Fill in:** a real mistake with real stakes but no disaster. Own it in the first sentence ("I shipped a bug that..."); never blame others.

### 8.4 Learning something fast
**Best fit:** E (Rails in a month at Visnext); alternatives: the TypeScript platform stack, React Native and Expo.

- **Situation (starter):** "In October 2025 I joined Visnext Solutions and spent a month in Ruby on Rails training covering MVC, ActiveRecord and Hotwire, after internships in Django and Next.js."
- **Task:** what you had to be able to do by the end.
- **Action:** how you learned (official guides, building small apps, reading real code, asking for reviews), how you mapped Rails to what you already knew from Django, and what was hardest.
- **Result:** what you could do afterwards, and how you have used Rails since at Devntech.

> **Fill in:** your actual learning method and one concrete thing you built during the training. Do not claim the training caused the Devntech offer unless that is true.

### 8.5 Tight deadline
**Best fit:** C (a store release or a client deadline), G (two internships), F (the LMS in about three months).

- **Situation:** the deadline, why it was fixed, what was at stake.
- **Task:** your part.
- **Action:** how you broke the work down, what you cut or deferred and with whose agreement, how you communicated progress and risks early, any extra steps you took to protect quality (tests, a checklist, a preview build).
- **Result:** delivered or not, with what scope, and what you would plan differently.

> **Fill in:** a real deadline story. If you missed it, the story still works if you show how you handled it.

### 8.6 Working with legacy code
**Best fit:** B (CrystalCommerce).

- **Situation (starter):** "CrystalCommerce is a legacy Rails e-commerce platform serving 700+ game stores, and I had to change it safely."
- **Task:** the change you had to make.
- **Action:** how you understood the existing code (tracing the request path, `git log` and `git blame`, asking people), how you protected behavior (characterization tests, feature flags, small pull requests), how you kept your new code isolated.
- **Result:** the change shipped without breaking stores, plus what you would improve in that codebase.

> **Fill in:** the specific legacy challenge and your concrete steps. Be respectful about the codebase and the people who built it.

### 8.7 Handling ambiguity
**Best fit:** F (you defined the LMS scope yourself), A (an unfamiliar third-party API), D (a migration whose plan evolved).

- **Situation:** what was unclear (requirements, API behavior, ownership, priorities).
- **Task:** what you had to deliver despite it.
- **Action:** how you reduced the ambiguity: questions you asked, assumptions you wrote down and confirmed, a small prototype, a phased plan.
- **Result:** what got delivered and how the clarity you created helped others.

> **Fill in:** the real ambiguous situation and the questions you asked.

### 8.8 Teaching or mentoring
**Best fit:** H (TA at PUCIT, 2023 to 2025).

- **Situation (starter):** "As a teaching assistant at PUCIT I helped run labs for Programming Fundamentals, OOP, Data Structures, Databases and Web Development."
- **Task:** a specific student, group or concept that was hard.
- **Action:** how you diagnosed the misunderstanding, changed your explanation (an analogy, a diagram, a smaller example), and checked that it landed.
- **Result:** what changed for the student or the lab, and what teaching taught you about explaining technical ideas, which you now use in code reviews and client calls.

> **Fill in:** one concrete teaching moment (a concept such as pointers, recursion, joins or normalization, and the explanation that worked).

### 8.9 Working with international clients
**Best fit:** A, C or D (Devntech builds features for international clients; Colochousing is a Belgian platform).

- **Situation (starter):** "At Devntech I build features for international clients, including a Belgian co-living platform."
- **Task:** a moment where communication mattered (unclear requirements, feedback on a feature, a time-zone constraint, an urgent issue).
- **Action:** how you communicated (written summaries, clarifying questions, demos, expectations about timing), and how you handled the time difference.
- **Result:** what the client got, and the communication habit you kept.

> **Fill in:** a real client interaction. Keep client-confidential details out.

### 8.10 Disagreeing with a decision
**Best fit:** D (a technical choice in the migration), C (a product or UX decision in the app), or a code review.

- **Situation:** the decision and who made it.
- **Task:** why you thought it was wrong or risky.
- **Action:** how you raised it (privately, with evidence, offering alternatives), how you listened to the reasons, and what you did after the decision ("disagree and commit").
- **Result:** what happened, whether you changed your mind, and what you learned.

> **Fill in:** a real disagreement. A strong ending is either "they convinced me, because..." or "we went with their decision and I made it work by...".

### 8.11 A production bug
**Best fit:** C (the live Coloc app), A (the sync touching real store data).

- **Situation:** how the bug surfaced (crash report, client message, failed jobs) and who was affected.
- **Task:** your role in responding.
- **Action:** triage (how bad, how many users), mitigation first (rollback, feature flag, hotfix, an OTA update if JavaScript-only), root cause (logs, reproduction), the fix, and prevention (a test, an alert, a checklist).
- **Result:** time to recovery if you know it, user impact, what changed afterwards.

> **Fill in:** a real production bug you handled or helped handle. If you have none, say so and describe how you would respond, using an internship or staging incident instead.

### 8.12 Proudest achievement
**Best fit:** C (an app live on both stores for 400+ residents), A (a new sales channel for a large platform), F (a project you proposed and built).

- **Situation:** what it was and why it mattered.
- **Task:** your role.
- **Action:** the two or three things you did that made the difference.
- **Result:** what exists now because of your work, and why you are proud of it (the people it helps, the difficulty, the initiative).

> **Fill in:** your choice and the reason it matters to you personally. Pride that sounds personal is more convincing than pride in size.

## 9. The "most challenging project" answer in full
This prompt is half behavioral, half technical: after your story, expect 10 to 15 minutes of technical drilling.

**Choosing the project:** pick the one that scores on all three: real technical difficulty, clear ownership by you, and an honest lesson. For your resume the strongest candidates are the PriceCharting integration (A) and the migration work (D); the Coloc app (C) works if you owned a hard feature end to end.

**Structure (2 to 3 minutes):**
1. Context in one or two sentences (platform, scale with attribution, your role).
2. Why it was hard: two or three concrete reasons, for example consistency between two systems, rate limits, idempotent jobs, legacy code, tenant isolation, running two systems during a migration. Use only the reasons that were true.
3. Your approach: three or four actions, including one decision with its trade-off.
4. Result: what exists now, plus what you would do differently.

**Drill-down to prepare:**

| If you pick | Prepare answers to | Study |
|---|---|---|
| PriceCharting | Overselling, rate limits, idempotency, polling vs webhooks, testing, legacy code | [CrystalCommerce](../17-projects-deep-dive/01-crystalcommerce.md) |
| The migration | Why migrate, org scoping and leak tests, sync design, SEO parity | [Platform migration](../17-projects-deep-dive/04-multi-tenant-platform-migration.md) |
| The Coloc app | Push notifications, payments, bill splitting, EAS releases, rollback | [Coloc resident app](../17-projects-deep-dive/03-coloc-resident-app.md) |

> **Fill in:** your chosen project, the two or three reasons it was hard, and the decision you would highlight.

## 10. Story x competency matrix
Mark which competencies each story can show. An "x" is a natural fit; a "?" fits only if your real details support it.

| Story | Technical depth | Learning | Ownership | Teamwork | Communication | Pressure | Failure / lesson |
|---|---|---|---|---|---|---|---|
| A PriceCharting | x | x | x | ? | ? | ? | ? |
| B Legacy Rails | x | x | ? | ? | | | ? |
| C Coloc releases | x | ? | x | x | ? | x | ? |
| D Migration | x | x | ? | x | x | ? | ? |
| E Rails training | | x | ? | | | x | |
| F LMS proposal | x | x | x | | x | x | ? |
| G Two internships | | ? | x | | x | x | ? |
| H TA labs | | | ? | x | x | | ? |
| I Remote internship | ? | x | ? | x | x | | |
| J Final-year project | x | x | ? | ? | ? | x | ? |

> **Fill in:** after filling in your stories, turn every "?" into "x" or blank. You want every column covered by at least two stories.

## 11. More prompts, reusing the same stories

| Prompt | Reuse |
|---|---|
| A time you went beyond your role | F (proposing the LMS) |
| A time you received critical feedback | A code review at Devntech or feedback during an internship |
| A time you had to prioritize | G (two internships), C (release scope) |
| A time you helped a teammate | H (teaching), or helping a colleague at Devntech |
| A time you improved a process | Anything you automated or standardized (tests, release checklists) |
| A time you worked with a difficult person | 8.2, told generously |
| A time you made a decision without all the information | 8.7 |
| A time you had to say no or push back | 8.10, or a scope negotiation in 8.5 |
| Why should we trust you with production systems? | B and 8.11: how you change things safely |

> **Fill in:** for each row, which real story you would use.

## 12. Handling follow-up questions

| Follow-up | What they want | How to answer |
|---|---|---|
| "What exactly did you do?" | Your contribution vs the team's | Two or three concrete actions with "I" |
| "Why did you choose that approach?" | Reasoning | The options and the trade-off that decided it |
| "What would you do differently?" | Self-awareness | One real improvement, not "nothing" |
| "How did you know it worked?" | Verification habits | Tests, monitoring, feedback you actually had |
| "What did your lead or client think?" | Collaboration | Real feedback, or "I didn't get explicit feedback, but ..." |
| "What was the hardest part?" | Depth | The obstacle in your Action, with one more level of detail |
| "Give me another example" | Breadth | Switch to a second core story covering the same competency |

## 13. Situational questions ("What would you do if...?")
These ask about a hypothetical, so a framework answer is fine; add "for example, at Devntech I..." only if a real case exists. The pattern that works for most of them: clarify, assess impact, communicate early, propose options, follow through.

| Question | A strong answer covers |
|---|---|
| A client keeps changing requirements mid-sprint | Clarify the change and the reason; show the impact on the timeline; involve the project manager; offer options (swap scope, move it to the next sprint); confirm the decision in writing |
| You cannot finish a task by the deadline | Raise it as soon as you know, with what is done and what is left; propose options (cut scope, more time, help); never surprise people on the due date |
| You disagree with your lead's technical approach | Ask for the reasoning first; share concerns privately with evidence; accept the decision and make it work if it stands |
| You find a serious production bug late on a Friday | Assess impact; follow the team's incident process; mitigate first (rollback, feature flag); tell the right people; fix forward with a review rather than a rushed solo change |
| A teammate is not doing their share | Talk to them privately and kindly; offer help; involve the lead only if it continues and affects delivery |
| You are asked to use a technology you do not know | Say so honestly; estimate the learning time; learn from official docs and the existing code; ask for an early review |
| You are often asked to stay late | Occasional crunch is normal; if it becomes constant, raise it as a planning problem, respectfully and with suggestions |
| You made a mistake nobody noticed | Disclose it and fix it; trust is worth more than looking perfect |

## 14. Common mistakes

| Mistake | Fix |
|---|---|
| Long situation, short action | Cut the setup to three sentences |
| "We" all the way through | Name the team's part once, then use "I" |
| A hypothetical answer ("I would...") to "tell me about a time" | Use a real story; say "I haven't faced exactly that, but the closest was ..." |
| Invented metrics | Use observed outcomes or none |
| A failure story where you were not really at fault | Choose a real mistake you owned |
| A conflict story where the other person is a villain | Show their reasonable side |
| No lesson | End with what you do differently now |
| The same story for every prompt | Prepare 6 to 8 and rotate |

## 15. Practice routine
1. Fill in section 6 and drop stories with nothing real behind them.
2. For each remaining story, write five bullet points (S, T, three actions, R). Not a script.
3. Tell each story out loud with a two-minute timer; then tell it as CAR in 45 seconds.
4. Ask a friend to interrupt with the follow-ups in section 12.
5. Before an interview, read the job description, pick the competencies it stresses, and choose which stories to lead with.

A worksheet to copy for each story:

```text
Story ID and name:
Prompts it answers:
Situation (2 to 3 sentences):
Task (1 sentence, "my job was ..."):
Action 1:
Action 2:
Action 3 (the obstacle and how I handled it):
Result (true outcome only):
Lesson (what I do differently now):
Likely follow-ups and my short answers:
What I must NOT claim (team work, numbers I don't know):
```

> **Fill in:** one worksheet per core story, kept in your notes and reread the night before each interview.
