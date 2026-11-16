# Code Quality, Code Review and Agile Delivery

> **Tier:** T2 (working knowledge) · **Related:** [SOLID and design principles](../01-cs-fundamentals/03-solid-and-design-principles.md), [Design patterns](../01-cs-fundamentals/04-design-patterns.md), [Testing fundamentals](09-testing-fundamentals.md), [Legacy Rails codebases](../05-ruby-on-rails/17-legacy-codebases-and-upgrades.md), [Git](../14-devops-and-tools/01-git.md), [GitHub and workflows](../14-devops-and-tools/02-github-and-workflows.md), [STAR stories bank](../18-behavioral-and-hr/02-star-stories-bank.md), [Common HR questions](../18-behavioral-and-hr/03-common-hr-questions.md)
> **On your resume:** building features for international clients at Devntech across legacy Rails monoliths, React, TypeScript and React Native; remote collaboration at RemoteDev; explaining code to students as a teaching assistant at PUCIT.

## Quick revision
- **Clean code:** intention-revealing names in the domain's language, small functions at one level of abstraction, guard clauses, no magic numbers, comments that explain **why**, consistency with the codebase.
- **Code smells** are hints, not bugs: long method, god class, duplication, long parameter lists, primitive obsession, feature envy, shotgun surgery, callback chains. Each has a standard refactoring.
- **Refactor safely:** tests first, small steps, behavior unchanged, refactoring separate from feature changes.
- **Formatter** (Prettier, `ruff format`, RuboCop's layout cops) settles layout automatically; **linter** (RuboCop, ESLint, Ruff, Biome) flags suspicious code. Enforce both in the editor, pre-commit hooks and CI.
- **Code review:** correctness, security (authorization, tenant scoping, injection, secrets), data and migrations, performance (N+1, indexes), tests, readability, backward compatibility.
- **Feedback:** specific, kind, about the code, with the reason; label severity (`nit:`, suggestion, blocking). Receiving: assume good intent, clarify, explain, resolve every comment.
- **Technical debt:** the future cost of shortcuts or outdated design; deliberate or accidental, prudent or reckless. Make it visible, pay it down near feature work, explain it in business terms.
- **Docs:** a README that gets a new developer running, ADRs for significant decisions, runbooks for operations, comments for the why.
- **Scrum:** Product Owner, Scrum Master, Developers; Sprint, Sprint Planning, Daily Scrum, Sprint Review, Sprint Retrospective; Product Backlog, Sprint Backlog, Increment. Refinement is ongoing, not a formal event.
- **User stories:** "As a..., I want..., so that..." with testable acceptance criteria; INVEST.
- **Story points** measure relative size (effort, complexity, uncertainty), not hours; velocity forecasts one team's capacity and is not a performance metric.
- **Kanban:** visualize flow, limit work in progress, pull work, measure lead and cycle time.
- **International clients:** async written updates, plan around time zones, confirm requirements in writing, demo often, handle scope changes explicitly.
- **Definition of Done** is the quality bar for every item; acceptance criteria are specific to one story.

## 1. Clean code basics

```ruby
# Before: cryptic names, magic numbers, floats, a boolean flag and mixed concerns
def calc(o, f)
  t = 0
  o.items.each { |i| t += i.p * i.q }
  t = t - (t * 0.1) if f
  t = t + (t * 0.21) if o.country == "BE"
  t
end

# After: names from the domain, integer cents, explicit parameters, one idea per line
VAT_PERCENT = { "BE" => 21 }.freeze          # real systems get rates from a tax service

def order_total_cents(order, discount_percent: 0)
  subtotal = order.items.sum { |item| item.unit_price_cents * item.quantity }
  subtotal -= percent_of(subtotal, discount_percent)
  subtotal + percent_of(subtotal, VAT_PERCENT.fetch(order.country, 0))   # VAT applies after discounts
end

def percent_of(cents, percent) = (cents * percent + 50) / 100            # rounded half up
```

- **Names** reveal intent and use the business's words (`listing`, `rent_payment`, `overdue?`); booleans read as questions; name length grows with scope.
- **Functions** do one thing at one level of abstraction; early returns (guard clauses) beat deep nesting; few parameters, using keyword arguments or parameter objects; avoid boolean flags that switch behavior.
- **Comments** explain why (a business rule, a workaround with a ticket link, a non-obvious constraint), not what the code already says. Delete commented-out code; Git remembers.
- **Consistency** beats personal taste: follow the codebase's conventions and the framework's way (the Rails way in a Rails app), even where you would choose differently on a new project.

## 2. Code smells and refactorings

| Smell | What you see | Refactoring |
|---|---|---|
| Long method | Scrolling to understand one method | Extract Method with intention-revealing names |
| Large class, god model | `User` or `Order` with thousands of lines | Extract Class, service objects, value objects |
| Duplicated code | The same logic in three places | Extract a method or module, after the third copy (rule of three) |
| Long parameter list | `create(a, b, c, d, e, f)` | Keyword arguments or a parameter object |
| Primitive obsession | Money as floats, statuses as loose strings | Value objects (`Money`), enums |
| Feature envy | A method mostly uses another object's data | Move Method to that object |
| Data clumps | Street, city and postcode always travel together | Extract Class (`Address`) |
| Type switches | `case channel_type` in many places | Polymorphism or Strategy ([design patterns](../01-cs-fundamentals/04-design-patterns.md)) |
| Shotgun surgery | One change touches ten files | Move related behavior together |
| Speculative generality | Hooks and abstractions nobody uses | Inline and delete (YAGNI) |
| Callback chains | Saving a model sends emails and calls APIs | Explicit service objects and after-commit jobs |
| Dead code | Unused methods, flags, endpoints | Delete it |

Refactor with a safety net: tests first (characterization tests for legacy code, see [testing fundamentals](09-testing-fundamentals.md)), small steps, behavior unchanged, and refactoring kept in separate commits or pull requests from behavior changes so reviews stay readable.

## 3. Linters and formatters

| Language | Linting | Formatting | Also useful |
|---|---|---|---|
| Ruby | RuboCop with rubocop-rails and rubocop-rspec (Rails 7.2+ apps ship rubocop-rails-omakase) | RuboCop layout cops, or Standard | Brakeman for security (also in Rails 7.2+ defaults), bundler-audit |
| JavaScript, TypeScript | ESLint with typescript-eslint (flat config files since ESLint 9) | Prettier | Biome: a fast all-in-one linter and formatter |
| Python | Ruff (very fast; replaces flake8 and isort for most teams) | `ruff format` or Black | mypy for type checking |
| C# | Roslyn analyzers | `dotnet format`, `.editorconfig` | Nullable reference types as warnings |

```bash
bundle exec rubocop -a                  # autocorrect safe offenses
bundle exec rubocop --auto-gen-config   # legacy app: park existing offenses in .rubocop_todo.yml, fix gradually
npx eslint . --fix && npx prettier --check .
ruff check . && ruff format .
```

A formatter removes style debates by rewriting layout automatically; a linter catches likely bugs and enforces conventions. Run them on save in the editor, in pre-commit hooks (lefthook, husky with lint-staged, pre-commit for Python) and as a required CI check. In legacy code, adopt them gradually so a lint rollout does not become a 5,000-file pull request.

## 4. Code reviews

**What to look for**, roughly in priority order:
1. **Correctness:** does it do what the ticket asks, including edge cases (empty lists, nil, zero, time zones)?
2. **Security:** authorization on every action, tenant scoping, injection, mass assignment, secrets in code or logs.
3. **Data:** safe migrations (locks, backfills, indexes built concurrently), constraints, reversibility.
4. **Performance:** N+1 queries, missing indexes, unbounded queries, heavy work in requests instead of jobs.
5. **Tests:** present, meaningful, covering the risky paths.
6. **Design and readability:** names, size, fit with existing conventions.
7. **Operations:** error handling, logging, metrics; backward compatibility for APIs used by mobile apps still on older versions.

**As the author:** keep pull requests small and focused, review your own diff first, write a description with what, why, how to test, screenshots and risks, and link the ticket.

| Instead of | Write |
|---|---|
| "This is wrong." | "If `items` is empty this raises on `.first`. Could we guard for it?" |
| "Why would you do it like this?" | "I'd expect this lookup to be scoped to the current store, otherwise any store's listing can be edited. Am I missing something?" |
| "Use a service." | "nit: this controller action has grown; extracting `Listings::Reprice` would make it testable on its own. Not blocking." |

**Giving feedback:** comment on the code, not the person; explain the reason; ask questions when unsure; label severity (`nit:`, suggestion, blocking); suggest concrete code; mention what is good; review promptly. **Receiving feedback:** assume good intent, ask for clarification, explain your reasoning when you disagree, take long debates to a short call and record the outcome, and resolve every comment with a change or a reply.

## 5. Technical debt

Technical debt is the extra cost future changes pay because of shortcuts or outdated design; the "interest" is slower delivery, more bugs and fragile releases.

| | Prudent | Reckless |
|---|---|---|
| Deliberate | "We ship now and refactor after the launch" (recorded, planned) | "No time for tests" (habitually) |
| Inadvertent | "Now we know how this should have been designed" | "What's layering?" |

Kinds you will meet: code debt (duplication, god classes), design debt (a single-tenant data model when the business needs multi-tenancy), test debt, dependency debt (old Ruby, Rails or Node versions nearing end of life), infrastructure debt, and knowledge debt (only one person understands the sync).

**Managing it:** make it visible as tickets with impact and a rough cost; follow the boy scout rule within the scope of your task; reserve some capacity each sprint; pay down debt in areas you are already changing; prioritize by interest (how often the area changes times how much it hurts); upgrade dependencies regularly in small steps; watch lead time, bug rates and incidents.

**Explaining it to a client:** use business language and concrete effects ("every new sales channel takes about twice as long because channel logic is copied in several places"), mention risks ("this framework version stops receiving security patches"), offer options with costs, and propose an incremental plan alongside features. Never blame previous developers; they made decisions under constraints you cannot see.

## 6. Documentation

- **README:** what the project is, prerequisites, setup and environment variables, how to run, test and deploy, common problems. The test: a new developer runs the app on day one.
- **ADRs (architecture decision records):** short, numbered files in the repository capturing a significant decision so future readers know why.
- **API docs:** OpenAPI specs or generated docs; keep them in sync through generation where possible.
- **Runbooks:** how to re-run a failed sync, rotate a key or handle a stuck queue.
- **Code comments and changelogs:** the why near the code; release notes for clients.

```text
docs/adr/0007-enforce-tenant-isolation-with-rls.md   (an illustrative example)

Status: Accepted (2026-03-02)
Context: All tenant tables share a schema; a missed org filter would leak data across organizations.
Decision: Keep app-level scoping and add Postgres row-level security on tenant tables,
          with the org set per transaction.
Consequences: Every query must run inside a transaction with the org set; migrations use a
              separate owner role; tests cover cross-org access.
```

## 7. Scrum

| Element | What it is |
|---|---|
| Product Owner | Maximizes product value; owns and orders the Product Backlog |
| Scrum Master | Coaches the team in Scrum, facilitates, removes impediments |
| Developers | The people building the Increment; cross-functional |
| Sprint | A fixed timebox of one month or less (two weeks is common) containing all the other events |
| Sprint Planning | Decide why the sprint matters (Sprint Goal), what to take and how to do it; at most 8 hours for a one-month sprint |
| Daily Scrum | 15 minutes for Developers to inspect progress toward the Sprint Goal and adapt; not a status report to a manager |
| Sprint Review | Inspect the Increment with stakeholders and adapt the backlog; at most 4 hours for a one-month sprint |
| Sprint Retrospective | Improve how the team works; at most 3 hours for a one-month sprint |
| Artifacts | Product Backlog (with the Product Goal), Sprint Backlog (with the Sprint Goal), Increment (meets the Definition of Done) |

Backlog refinement (clarifying, splitting and estimating upcoming items) is an ongoing activity rather than a formal event. These names follow the 2020 Scrum Guide, which talks about "Developers" rather than a "Development Team". Many agencies run a lighter version: two-week sprints, a daily standup, planning and a demo.

## 8. User stories and estimation

```text
Story:  As a resident, I want to split a house bill between housemates
        so that everyone pays their share without manual calculations.
Acceptance criteria:
  Given a 100.00 EUR bill and three residents in the house
  When I split it equally
  Then each resident owes 33.33 EUR and one share absorbs the remaining cent
  And every resident gets a push notification with their amount
  And residents who already left the house are not included
```

- **INVEST:** Independent, Negotiable, Valuable, Estimable, Small, Testable.
- **Splitting** big stories: by workflow step, by data variation, happy path first and edge cases next, or a time-boxed **spike** to remove uncertainty before estimating.
- **Story points** express relative size (effort, complexity and uncertainty) on a Fibonacci-like scale (1, 2, 3, 5, 8, 13). Relative sizing is easier and more consistent than hours, and it absorbs differences in individual speed.
- **Planning poker:** everyone reveals at once to avoid anchoring, then the highest and lowest explain their reasoning; the discussion is the real value.
- **Velocity** (points completed per sprint) forecasts one team's capacity. Comparing teams or judging individuals by it makes the numbers meaningless.
- **Client work in hours:** break the work into tasks, estimate ranges, state assumptions and unknowns, add a buffer for integration and review, and re-estimate when scope changes.

## 9. Kanban and Jira

**Kanban** visualizes the workflow on a board (To do, In progress, Review, QA, Done), sets **WIP limits** per column so work finishes before new work starts, and lets people pull work when capacity frees up. There are no fixed sprints; flow is measured with **lead time** (request to done), **cycle time** (work started to done), throughput and cumulative flow diagrams. Scrum suits planned increments with a stable team; Kanban suits continuous, interrupt-driven work such as support and maintenance; many teams mix them.

**Jira** organizes issues (epics, stories, tasks, bugs, sub-tasks) on Scrum or Kanban boards with workflows and sprints. Put the issue key in branch names, commits and pull request titles so they link automatically, and use JQL filters:

```text
project = SHOP AND status = "In Progress" AND assignee = currentUser() ORDER BY priority DESC
```

## 10. Working with international clients

- **Async written updates:** a short end-of-day message beats silence.
- **Time zones:** Pakistan (UTC+5) is four hours ahead of Central Europe in winter and three in summer, so your afternoon overlaps their morning; save questions for the overlap and send them early. With US clients the overlap is small, so written clarity matters even more.
- **Clarifying requirements:** restate the request in your own words, ask about edge cases, attach a mockup or screenshot, and when the client is offline propose a default ("I'll assume X unless you prefer otherwise") so you are not blocked.
- **Demos:** show working software on staging often; a short screen recording works when there is no live slot.
- **Scope changes:** acknowledge, estimate the impact (time, cost, what moves out), get approval in writing and update the ticket. Silent scope creep hurts everyone.
- **Habits:** raise risks early, give explicit dates, confirm decisions in writing after calls, and keep messages short and structured.

```text
Update, Thu 6 Nov
Done: rent reminder job is idempotent (spec added); fixed the N+1 on the bills page (from X ms to Y ms on staging).
In progress: marketplace page metadata; PR open, needs review.
Blocked: need the production webhook secret for staging tests (who can share it securely?).
Question: should reminders go out on weekends? I'll keep them on unless you say otherwise.
```

(A template: replace the items and numbers with your own.)

## 11. Definition of Done

A shared checklist every backlog item must meet before it counts as done, for example: code reviewed and approved; tests added and passing in CI; lint clean; acceptance criteria verified; deployed to staging (or production behind a flag); migrations safe; documentation and changelog updated; no known critical bugs; monitoring in place for risky changes. The Definition of Done is the quality bar for **every** item; acceptance criteria describe what **this** story must do.

## Common pitfalls
- Clever, compressed code that saves lines and costs every reader minutes.
- Comments that restate the code, or lie because the code changed.
- Big-bang refactors mixed into feature pull requests.
- Turning on every lint rule in a legacy app at once.
- Pull requests with thousands of lines and no description.
- Reviews that only nitpick style while missing authorization, migrations or N+1 queries.
- Treating story points as hours, or velocity as a productivity score.
- Daily standups that turn into problem-solving sessions or status reports to a manager.
- Saying "yes" to scope changes without stating the impact.
- Waiting silently when blocked instead of asking early with a proposed default.

## How to talk about this in an interview
- You work onsite at Devntech on features for international clients, so you can speak to async updates, time zones, clarifying requirements and demos from real practice; your remote internship at RemoteDev adds remote collaboration.
- For code quality, use legacy code examples: adding tests before refactoring, improving names and extracting services within the scope of a ticket, and leaving the code better than you found it.
- For code review, show the checklist mindset (security, tenant scoping, migrations, N+1) and the human side (kind, specific, severity labels).
- Teaching lab sessions at PUCIT trained you to explain code clearly and give feedback without discouraging people, which maps directly onto reviews.
- Do not claim a process you did not use; describe what your team actually does.

> **Fill in:** the process your Devntech team uses (Scrum sprints, Kanban, or a mix), the tool (Jira or another), how standups and demos work, and how estimates are given to clients.

> **Fill in:** one code review comment you gave or received that changed your approach, and one piece of technical debt you paid down or flagged.

> **Fill in:** how you communicate with the clients' teams (time zones, channels, update cadence) and one requirement you clarified before building.

## Interview questions

**Q1. What makes code clean?**
It is easy to read and safe to change: names that reveal intent in the domain's language, small functions at one level of abstraction, no magic numbers, explicit dependencies, comments that explain why, and consistency with the codebase. Tests back it up so changes are safe.
*Follow-up:* When is a comment necessary? To explain a business rule, a workaround or a non-obvious constraint that the code cannot express.

**Q2. What are code smells? Give examples and their refactorings.**
Surface signs of deeper design problems: a long method (extract method), a god class (extract class or service objects), duplication (extract shared code after the third copy), primitive obsession (value objects such as `Money`), feature envy (move the method), type switches (polymorphism or strategy), callback chains (explicit services).
*Follow-up:* How do you refactor safely? Tests first, small steps, unchanged behavior, separate pull requests from features.

**Q3. What is the difference between a linter and a formatter?**
A formatter rewrites layout automatically (indentation, quotes, line breaks), ending style debates. A linter analyzes code for likely bugs and convention violations, some autocorrectable. Examples: Prettier and ESLint, `ruff format` and `ruff check`, RuboCop for both in Ruby; Biome combines both for JavaScript and TypeScript.
*Follow-up:* How do you introduce RuboCop to a legacy app? Generate a todo file of existing offenses, enforce rules on new code, and fix old offenses gradually.

**Q4. What do you look for in a code review?**
Correctness and edge cases first, then security (authorization, tenant scoping, injection, secrets), data safety (migrations, constraints), performance (N+1, indexes, unbounded queries), tests, readability and fit with conventions, and operational concerns such as logging and backward compatibility.
*Follow-up:* How do you comment? Specifically and kindly, explaining why, labeling nits versus blocking issues.

**Q5. How do you handle disagreement in a code review?**
Assume good intent, make sure I understand the reviewer's concern, and explain my reasoning with evidence (a test, a benchmark, the docs). If it stays unresolved after a round or two, move to a quick call, agree, and record the decision in the pull request; I am happy to be convinced.
*Follow-up:* What if a senior insists on something you think is wrong? Raise the concern clearly once with reasons, then follow the team's decision unless it is a security or data risk, which I escalate.

**Q6. What is technical debt, and how do you manage it?**
The extra future cost of shortcuts or outdated design, paid as slower changes and more bugs. It can be deliberate or accidental, prudent or reckless. Manage it by making it visible with impact, reserving some capacity, paying it down where you are already working, prioritizing high-interest areas, and keeping dependencies current.
*Follow-up:* Is all technical debt bad? No; deliberate, recorded debt to hit a deadline can be a good trade if it is repaid.

**Q7. How would you explain technical debt to a non-technical client?**
In business terms: what it slows down ("each new integration takes twice as long"), what risk it carries ("this version stops getting security fixes"), and options with costs, ideally an incremental plan delivered alongside features. No jargon and no blame.
*Follow-up:* How do you justify the time? Tie it to something the client wants soon that the debt is blocking or slowing.

**Q8. Explain the Scrum roles, events and artifacts.**
Roles: Product Owner (value and backlog order), Scrum Master (coaching and impediments), Developers (build the increment). Events: the Sprint containing Sprint Planning, the 15-minute Daily Scrum, the Sprint Review with stakeholders and the Retrospective. Artifacts: Product Backlog, Sprint Backlog and the Increment, each with a commitment: Product Goal, Sprint Goal and Definition of Done.
*Follow-up:* What is the Daily Scrum for? Developers inspecting progress toward the Sprint Goal and adapting the plan, not reporting status to a manager.

**Q9. What are story points, and why not estimate in hours?**
A relative measure of size combining effort, complexity and uncertainty. People compare items more reliably than they predict hours, and points absorb individual speed differences. Velocity then forecasts how much the team usually completes per sprint.
*Follow-up:* Why not compare velocity across teams? Each team's scale is relative to itself, so comparisons are meaningless and invite inflation.

**Q10. Scrum or Kanban?**
Scrum fits planned work in fixed sprints with a stable team and regular stakeholder reviews. Kanban fits continuous, interrupt-driven work (support, maintenance) with WIP limits and flow metrics instead of sprints. Many agency teams mix them: a Kanban board with regular demos and retrospectives.
*Follow-up:* What do WIP limits achieve? They force finishing over starting, exposing bottlenecks and shortening cycle time.

**Q11. What makes a good user story and acceptance criteria?**
A story states who wants what and why, is small enough for a few days, and follows INVEST. Acceptance criteria are concrete and testable, often Given, When, Then, covering the main path and important edge cases, and agreed before work starts.
*Follow-up:* Acceptance criteria vs Definition of Done? Criteria are specific to one story; the Definition of Done is the quality bar for every item.

**Q12. How do you work with a client in a different time zone when requirements are unclear?**
Write questions early and precisely, restate my understanding, attach mockups, and propose a default so I can keep moving while waiting. Use the overlap hours for calls, send end-of-day written updates, confirm decisions in writing, and demo often so misunderstandings surface early.
*Follow-up:* And when the client changes scope mid-sprint? Acknowledge it, estimate the impact, offer trade-offs, and get written approval before switching.

## Further reading
- The Scrum Guide: https://scrumguides.org/scrum-guide.html
- RuboCop docs: https://docs.rubocop.org/rubocop/
- ESLint configuration files: https://eslint.org/docs/latest/use/configure/configuration-files
