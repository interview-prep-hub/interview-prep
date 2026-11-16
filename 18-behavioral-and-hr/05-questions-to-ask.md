# Questions to Ask the Interviewer

> **Tier:** T2 (working knowledge) · **Related:** [Common HR questions](03-common-hr-questions.md), [Salary and offers](04-salary-and-offers.md), [Tell me about yourself](01-tell-me-about-yourself.md), [Interview formats](../00-start-here/03-interview-formats.md)
> **On your resume:** your experience tells you what to ask: you have worked in legacy Rails code, a React Native app with store releases, a platform migration and client-facing services work, so you know which answers matter.

## Quick revision
- **Always have questions.** "No, I think you covered everything" sounds like low interest.
- **Ask two or three per round,** chosen for the person in front of you: engineers about daily work, leads about architecture and growth, managers about expectations, HR about process and benefits.
- **Do not ask what their website answers,** and do not open a technical round with salary or leave.
- **Listen to the answers:** vague or defensive replies are data. Section 9 lists red flags.
- **Close well:** thank them, restate your interest and fit in one sentence, ask about next steps and the timeline.
- **Follow up** with a short thank-you message within a day.

## 1. Why your questions matter
Your questions do two jobs. They show the interviewer that you think like an engineer who cares about how work gets done, and they give you information to decide whether you want the job. A role can look good on paper and still have no code review, constant overtime or an unclear path for juniors; the questions below are designed to surface those things politely.

## 2. Rules
- **Prepare five, ask two or three.** Some will be answered during the interview.
- **Make them open:** "How do you..." and "What does... look like?" rather than yes/no questions.
- **Connect them to your experience:** "On my current project we release the mobile app through EAS; how do you handle releases?" shows real experience and invites a real answer.
- **One follow-up is good:** it shows you listened.
- **Watch the time:** if the interviewer is rushing, ask your single best question.
- **Write the answers down** right after the interview, while you remember them.

## 3. Questions for engineers (your future peers)

| Question | What you learn |
|---|---|
| What does a typical week look like for you? | Meetings vs focused time, how work arrives |
| How does code review work here? Who reviews, and how long does a review usually take? | Quality culture and speed |
| How do you test, and what runs in CI before a merge? | Whether you will break things without a safety net |
| How often do you deploy, and how? | Release maturity; manual vs automated |
| What does on-call or production support look like? | Hidden hours and stress |
| What is the oldest or messiest part of the codebase, and how do you deal with it? | Honesty about tech debt and whether it is managed |
| What do you like most about working here, and what would you change? | Unfiltered culture signals |
| How did your onboarding go? How long until you shipped something? | What your first weeks will look like |

## 4. Questions for tech leads and architects

| Question | What you learn |
|---|---|
| What are the biggest technical challenges for the team in the next six months? | Where you would contribute and what you would learn |
| How are technical decisions made and recorded? | Whether juniors have a voice; whether decisions are written down |
| How do you decide between paying down tech debt and building features? | Long-term thinking |
| Are there migrations or upgrades in progress (framework versions, a new platform)? | Messy but valuable learning; your migration experience is relevant here |
| How do you help junior engineers grow? Pairing, reviews, mentorship? | Real support vs "figure it out" |
| What would make someone in this role successful in the first three and six months? | Concrete expectations to aim for |
| How do you handle production incidents? Is there a post-incident review? | Blameless culture or blame culture |

## 5. Questions for engineering managers and project managers

| Question | What you learn |
|---|---|
| How is performance reviewed, and how often? | Whether growth is measured fairly |
| What does promotion to the next level require? | A visible path, or none |
| How is work assigned across projects and clients? | Ownership vs being moved around |
| How do you handle scope changes from clients? | Whether deadlines are protected or overtime absorbs everything |
| How much overtime is normal, especially near releases? | Work-life balance in practice |
| How does the team communicate with clients: through a manager, or do engineers join calls? | Client exposure and expectations |
| What happens between projects? | In services companies, idle time between client projects can mean training, internal work or uncertainty |
| What is the team's biggest challenge right now? | Honest context |

## 6. Questions for HR
Save salary questions for HR rounds, and ask about process and benefits here.

| Question | What you learn |
|---|---|
| What are the next steps, and what is the expected timeline? | When to follow up |
| How long is probation, and what does confirmation depend on? | Terms that affect your security |
| What is the notice period during and after probation? | Whether it could trap you later |
| What does the benefits package include (health cover, provident fund, allowances, leave)? | The real value of the package |
| When does the first salary review happen, and how often after that? | Growth in pay |
| What are the working hours, and is there remote or hybrid flexibility? | Daily life |
| Is there a training or learning budget? | Investment in your growth |
| Are there any bond or penalty clauses in the contract? | Hidden costs of leaving |

## 7. Questions for founders and CTOs at small companies

| Question | What you learn |
|---|---|
| What is the company's runway or main source of revenue? | Stability (ask politely; many founders appreciate the question) |
| Who are the main customers, and how do you get new ones? | Business health |
| What would you want this hire to own within six months? | Scope and ownership |
| How big will the engineering team be in a year? | Growth, and your chance to grow with it |
| How do you make product decisions? | Whether engineers influence the product |

## 8. Role-specific questions
Pick the ones that match the job; they show depth because they come from real experience.

**Rails teams:**
- Which Rails and Ruby versions are you on, and how do you approach upgrades?
- What do you use for background jobs, and how do you monitor failures and retries?
- Is the app a monolith, and how do you keep its boundaries clean?

**React Native teams:**
- How do you release: EAS Build and Submit, OTA updates, or something else? How often?
- What is your policy for over-the-air updates vs store releases?
- How do you monitor crashes, and what is your crash-free rate goal?
- How do you test on real devices?

**TypeScript platform teams:**
- How do you share types between the API and the clients?
- How is multi-tenancy or access control enforced, and how is it tested?
- Which parts of the stack are you most and least happy with?

**Django teams:**
- Do you use Django REST Framework? How are permissions structured?
- How do you run background tasks?
- How do you handle database migrations on large tables?

**Services companies with international clients:**
- Will I work directly with clients, and in which time zones?
- How many projects does an engineer usually handle at once?
- How long do engineers typically stay on one client project?

**Remote teams:**
- What are the core overlap hours?
- How does the team communicate decisions: written documents, calls, chat?
- How do you onboard and mentor remote engineers?

> **Fill in:** for each upcoming interview, the two questions you will ask, chosen from this file or written from your research of that company.

## 9. Questions to avoid, and better versions

| Avoid | Why | Ask instead |
|---|---|---|
| "What does your company do?" | Shows no research | "I saw you build [product]; which part would this role work on?" |
| "How many leaves do I get?" as your first question | Signals priorities you may not intend | Ask HR about the benefits package as a whole, later in the process |
| "When will I be promoted?" | Sounds entitled before you have started | "What does growth to the next level look like here?" |
| "Did I get the job?" | Puts the interviewer on the spot | "What are the next steps?" |
| "Can I work from home from day one?" in a technical round | Wrong audience and timing | Ask HR about remote or hybrid policy |
| "Do you check what employees do online?" | Sounds like you plan to slack | "How do you measure performance?" |
| Questions answered five minutes earlier | Shows you were not listening | Build a follow-up on what they said |

## 10. Which questions for which stage

| Stage | Best questions |
|---|---|
| Recruiter or HR screen | Process and timeline, the team the role sits in, working hours and location, benefits overview |
| Technical round with engineers | Daily work, code review, testing, deployment, codebase health |
| Technical round with a lead | Architecture, upcoming challenges, how juniors grow, expectations for the first months |
| Manager or final round | Performance reviews, promotion path, how work is assigned, team challenges |
| A client interviewing you for their project | Their product and users, the team you would join, overlap hours, how they like updates delivered |

For a client interview, two good questions are "What does a great first month on this project look like for you?" and "How do you prefer to receive progress updates?" Both show you care about their workflow, and both answers help you succeed if you are assigned.

## 11. Red flags to listen for

| What you hear | What it may mean | A polite follow-up |
|---|---|---|
| "We don't really do code reviews" or "no time for tests" | Quality is left to luck | "How do you catch bugs before production?" |
| "Everyone stays late before releases" said proudly | Chronic overtime | "How often does that happen in a typical month?" |
| Vague answers about the role or the team | The role is not defined, or the team is in flux | "Who would I work with day to day?" |
| "We are like a family" plus unclear policies | Boundaries may be blurry | "How are working hours and leave handled?" |
| Several people left recently, and no clear reason | Possible turnover problem | "How long has the team been together?" |
| Pressure to accept immediately | A tactic, or desperation | "I'd like a couple of days to review the offer." |
| Unpaid "trial" work beyond a reasonable take-home task | Free labor | "How long should this task take, and how is it evaluated?" |
| No written offer, or payment details that change | Contract and payment risk | "Could you share the offer in writing?" |
| Interviewers who are dismissive or late without apology | A preview of daily treatment | Note it; one bad interviewer is not the whole company, but patterns matter |
| Salary delays mentioned by employees | Cash-flow problems | Ask former employees if you can reach them |

One red flag is not a verdict; several together usually are.

## 12. Closing the interview well
In the last two minutes:
1. **Thank them** for their time and for the specific thing you enjoyed discussing.
2. **Restate your interest and fit in one sentence:** "After hearing about your mobile release process and the API work, I'm even more interested; it's close to what I do on the Coloc app at Devntech." Adjust to what you really discussed.
3. **Optionally invite concerns:** "Is there anything about my background that you would like me to clarify?" It gives you one last chance to address a doubt.
4. **Ask about next steps:** "What are the next steps, and when should I expect to hear back?"

Thank-you message (send within a day, short):

```text
Subject: Thank you - [Role] interview

Hi [Name],

Thank you for taking the time to speak with me today about the [Role] position.
I enjoyed learning about [specific topic you discussed], and it made me even more
interested in the role. I look forward to hearing about the next steps.

Best regards,
[Your Name]
```

## 13. After the interview
- **Write notes immediately:** questions you were asked, what went well, what you would answer differently, and the answers to your own questions.
- **Update your story bank** and your project notes with anything that came up unprepared.
- **Follow up once** if the promised timeline passes: a short, polite message asking about the status.
- **Keep interviewing** until you have a signed offer; one good interview is not a job.

> **Fill in:** a simple tracker for your applications: company, role, stage, date, contact person, next step, and notes on red or green flags.

## 14. Your default shortlist
If your mind goes blank, these five work in almost any round:
1. "What would success look like for this role in the first three months?"
2. "What is the biggest challenge the team is facing right now?"
3. "How does code get from a pull request to production here?"
4. "How do you support engineers at my level to grow?"
5. "What are the next steps in the process?"

Follow-up phrases that show you listened:
- "You mentioned [X]; how did the team decide on that?"
- "That sounds similar to [something from your experience]; did you run into [a related problem]?"
- "What would you do differently if you were starting that project today?"

Keep follow-ups to one per answer, and only when time allows.
