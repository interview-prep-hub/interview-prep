# Study Roadmap

> How this knowledge base is organized, what to study first, and day-by-day plans for 6 weeks, 2 weeks and 3 days. Start here, then keep the [README](../README.md) checklist up to date as you finish files.

## How the knowledge base is organized

- **Folders are in study order** (`00` to `18`). Files inside a folder are numbered in reading order.
- **Every topic file has the same shape:**
  1. `Quick revision`: the facts to re-read 10 minutes before an interview.
  2. Concept sections that build from zero to interview depth, with code.
  3. `Common pitfalls`: the mistakes interviewers like to probe.
  4. `How to talk about this in an interview`: ties the topic to your resume.
  5. `Interview questions`: model answers plus the likely follow-up.
- **`99-interview-questions.md` files** are rapid-fire banks for active recall: cover the answer, say yours out loud, then check.
- **`> **Fill in:**` prompts** mark places where only you know the answer (real numbers, decisions, reasons for a job change). Fill them in during week 1: your project answers depend on them. Search the folder for `Fill in:` to find them all.
- **Tiers** tell you how deep to go:

| Tier | Meaning | You are done when |
|---|---|---|
| **T1** | Know cold | You can explain it from memory, write the code, and handle two levels of follow-up |
| **T2** | Working knowledge | You can explain concepts and trade-offs and read code confidently |
| **T3** | Essentials, honest framing | You can define the key terms and say truthfully how much you have used it |

## Ground rules that make studying stick

1. **Recall beats rereading.** After reading a file, close it and write its `Quick revision` from memory. Then compare.
2. **Say answers out loud.** You taught labs as a TA: use that. An answer you have never spoken will come out messy in an interview.
3. **Write DSA code without autocomplete,** in C++, on paper or in a plain editor, then run it on an online judge.
4. **Space your reviews.** Revisit each `99-` bank 1, 3 and 7 days after you first study the folder.
5. **Keep a log of real interview questions.** After every interview, write down each question and how well you answered. Add the weak ones to your next week's plan.
6. **Never bluff.** For T3 skills, "I used it in my final-year project, here is what I did, and here is how I would ramp up" beats a confident wrong answer.

## Priority order

You are targeting four role types (Rails full-stack, JS/TS full-stack, React Native, Python/Django). Every interview, whatever the stack, touches the same core, so study the core first:

| Priority | Area | Why | Start with |
|---|---|---|---|
| 1 | Your story and projects | Asked in every single interview, and only you can prepare it | [Tell me about yourself](../18-behavioral-and-hr/01-tell-me-about-yourself.md), [resume defense](../18-behavioral-and-hr/06-resume-line-by-line-defense.md), [project deep-dives](../17-projects-deep-dive/01-crystalcommerce.md) |
| 2 | OOP, SOLID, DSA (C++) | Screening tests and first technical rounds at local companies | [OOP](../01-cs-fundamentals/02-oop.md), [SOLID](../01-cs-fundamentals/03-solid-and-design-principles.md), [problem-solving framework](../02-dsa/00-big-o-and-problem-solving.md) |
| 3 | SQL and databases | Query-writing rounds are common, and Postgres runs both client projects | [SQL from zero](../12-databases/02-sql-from-zero.md), [SQL practice](../12-databases/03-sql-practice-problems.md) |
| 4 | Web fundamentals | HTTP, REST, auth and security come up in every full-stack round | [How the web works](../04-web-fundamentals/01-how-the-web-works.md), [REST design](../04-web-fundamentals/04-rest-api-design.md) |
| 5 | Primary stack | Ruby/Rails and JS/TS/React are your daily work | [Ruby basics](../03-languages/ruby/01-basics.md), [Rails fundamentals](../05-ruby-on-rails/01-rails-fundamentals.md), [JavaScript](../03-languages/javascript/01-basics-types-coercion.md), [React](../06-react/01-fundamentals.md) |
| 6 | Role stacks | React Native, Node/Next.js/TanStack, Python/Django | [RN fundamentals](../08-react-native-expo/01-react-native-fundamentals.md), [Node fundamentals](../09-nodejs-backend/01-nodejs-fundamentals.md), [Django fundamentals](../10-python-django/01-django-fundamentals.md) |
| 7 | Breadth | Architecture, system design, DevOps, .NET, Java, C | [Multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md), [system design](../15-system-design/01-fundamentals.md), [Git](../14-devops-and-tools/01-git.md) |

## When an interview is booked: role tracks

Once you know the role, spend the last 5-7 days on its track. "Must review" means read fully and answer every question out loud; "skim" means `Quick revision` plus the questions.

| Role | Must review | Skim |
|---|---|---|
| **Rails full-stack** | [Ruby](../03-languages/ruby/01-basics.md) (all files + [99](../03-languages/ruby/99-interview-questions.md)), [Rails](../05-ruby-on-rails/01-rails-fundamentals.md) (all files + [99](../05-ruby-on-rails/99-interview-questions.md)), [Postgres](../12-databases/04-postgresql.md), [indexing](../12-databases/05-indexing-and-query-optimization.md), [transactions](../12-databases/06-transactions-and-concurrency.md), [CrystalCommerce](../17-projects-deep-dive/01-crystalcommerce.md), [Colochousing](../17-projects-deep-dive/02-colochousing-web-and-api.md) | [React fundamentals](../06-react/01-fundamentals.md), [hooks](../06-react/02-hooks.md), [background processing](../13-architecture-and-practices/06-background-processing-patterns.md), [API integration](../13-architecture-and-practices/05-third-party-api-integration.md) |
| **JS/TS full-stack** | [JavaScript](../03-languages/javascript/99-interview-questions.md), [TypeScript](../03-languages/typescript/99-interview-questions.md), [React](../06-react/99-interview-questions.md), [Next.js and TanStack](../07-nextjs-and-tanstack/01-nextjs-fundamentals.md), [Node](../09-nodejs-backend/01-nodejs-fundamentals.md), [oRPC](../09-nodejs-backend/05-orpc-and-typesafe-apis.md), [Drizzle](../12-databases/13-drizzle-orm.md), [MongoDB](../12-databases/08-mongodb-fundamentals.md), [machine coding](../06-react/11-machine-coding-rounds.md) | [Multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md), [platform migration](../17-projects-deep-dive/04-multi-tenant-platform-migration.md), [RemoteDev](../17-projects-deep-dive/06-remotedev-nextjs-mongodb.md) |
| **React Native** | [RN and Expo](../08-react-native-expo/01-react-native-fundamentals.md) (all files + [99](../08-react-native-expo/99-interview-questions.md)), [React hooks](../06-react/02-hooks.md), [React performance](../06-react/06-performance.md), [TypeScript in practice](../03-languages/typescript/04-ts-in-practice.md), [Coloc app](../17-projects-deep-dive/03-coloc-resident-app.md) | [Rails APIs](../05-ruby-on-rails/10-api-development.md), [payments](../13-architecture-and-practices/08-payments-and-subscriptions.md), [auth concepts](../04-web-fundamentals/06-authentication-authorization.md) |
| **Python/Django** | [Python](../03-languages/python/01-fundamentals.md) (all files + [99](../03-languages/python/99-interview-questions.md)), [Django and DRF](../10-python-django/01-django-fundamentals.md) (all files + [99](../10-python-django/99-interview-questions.md)), [SQL](../12-databases/02-sql-from-zero.md), [transactions](../12-databases/06-transactions-and-concurrency.md), [LMS project](../17-projects-deep-dive/05-lms-coursera-clone.md) | [REST design](../04-web-fundamentals/04-rest-api-design.md), [auth concepts](../04-web-fundamentals/06-authentication-authorization.md), [payments](../13-architecture-and-practices/08-payments-and-subscriptions.md) |

## The 6-week plan

Sized for about 20 hours a week next to a full-time job (2-3 hours on weekdays, 5-6 on weekend days). If you have less time, keep the order and stretch it to 8-10 weeks. In this plan, T1 files are read fully; T2 and T3 files get their `Quick revision` and questions unless the week says otherwise.

| Week | Topic study | DSA in C++ (daily) | Recall and story |
|---|---|---|---|
| **1** | [CS fundamentals](../01-cs-fundamentals/01-programming-fundamentals.md) (all), [C++ fundamentals](../03-languages/c-cpp/02-cpp-fundamentals.md), [C++ OOP](../03-languages/c-cpp/03-cpp-oop.md), [STL for DSA](../03-languages/c-cpp/04-stl-for-dsa.md), [Ruby 01-03](../03-languages/ruby/01-basics.md) | [Big-O](../02-dsa/00-big-o-and-problem-solving.md), [arrays and hashing](../02-dsa/01-arrays-strings-hashing.md), [two pointers](../02-dsa/02-two-pointers-sliding-window.md) | Draft your 90-second [intro](../18-behavioral-and-hr/01-tell-me-about-yourself.md); answer every Fill-in in [17-projects-deep-dive](../17-projects-deep-dive/01-crystalcommerce.md) |
| **2** | [Ruby 04-06](../03-languages/ruby/04-blocks-procs-lambdas.md), [Rails 01-09](../05-ruby-on-rails/01-rails-fundamentals.md), [DBMS](../12-databases/01-dbms-fundamentals.md), [SQL from zero](../12-databases/02-sql-from-zero.md) | [Linked lists](../02-dsa/03-linked-lists.md), [stacks and queues](../02-dsa/04-stacks-queues.md) | [Ruby 99](../03-languages/ruby/99-interview-questions.md), [CS 99](../01-cs-fundamentals/99-interview-questions.md); write 4 [STAR stories](../18-behavioral-and-hr/02-star-stories-bank.md) |
| **3** | [Rails 10-18](../05-ruby-on-rails/10-api-development.md), [SQL practice](../12-databases/03-sql-practice-problems.md), [Postgres](../12-databases/04-postgresql.md), [indexing](../12-databases/05-indexing-and-query-optimization.md), [transactions](../12-databases/06-transactions-and-concurrency.md), [JS 01-04](../03-languages/javascript/01-basics-types-coercion.md) | [Recursion and backtracking](../02-dsa/05-recursion-backtracking.md), [sorting and binary search](../02-dsa/06-sorting-binary-search.md) | [Rails 99](../05-ruby-on-rails/99-interview-questions.md), [DB 99](../12-databases/99-interview-questions.md) |
| **4** | [JS 05-07](../03-languages/javascript/05-es6-plus.md), [TypeScript](../03-languages/typescript/01-fundamentals.md) (all), [React 01-06](../06-react/01-fundamentals.md), [web fundamentals](../04-web-fundamentals/01-how-the-web-works.md) (01, 04, 06, 07 fully) | [Trees and BST](../02-dsa/07-trees-bst.md), [heaps](../02-dsa/08-heaps.md) | [JS 99](../03-languages/javascript/99-interview-questions.md) output puzzles, [TS 99](../03-languages/typescript/99-interview-questions.md), [web 99](../04-web-fundamentals/99-interview-questions.md) |
| **5** | [React 07-11](../06-react/07-patterns-and-architecture.md), [Next.js and TanStack](../07-nextjs-and-tanstack/01-nextjs-fundamentals.md), [Node](../09-nodejs-backend/01-nodejs-fundamentals.md) (all), [Drizzle](../12-databases/13-drizzle-orm.md), [MongoDB](../12-databases/08-mongodb-fundamentals.md) (08-09), [React Native](../08-react-native-expo/01-react-native-fundamentals.md) 01-05 | [Graphs](../02-dsa/09-graphs.md) | [React 99](../06-react/99-interview-questions.md) live coding, one [machine-coding](../06-react/11-machine-coding-rounds.md) problem timed |
| **6** | [React Native](../08-react-native-expo/06-payments-and-split-bills.md) 06-09, [Python](../03-languages/python/01-fundamentals.md), [Django and DRF](../10-python-django/01-django-fundamentals.md), [architecture 03-06](../13-architecture-and-practices/03-multi-tenancy.md), [system design](../15-system-design/01-fundamentals.md), [Git](../14-devops-and-tools/01-git.md), [Docker](../14-devops-and-tools/03-docker.md), T3 skims ([C#](../03-languages/csharp/01-fundamentals.md), [.NET](../11-dotnet/01-aspnet-core-mvc.md), [Java](../03-languages/java/01-java-essentials.md)) | [Dynamic programming](../02-dsa/10-dynamic-programming.md), [greedy, intervals, bits](../02-dsa/11-greedy-intervals-bits.md) | Two full mock interviews, every remaining `99-` bank, [last-minute revision](04-last-minute-revision.md) |

**DSA rhythm for all six weeks:** one new problem and one re-solve every weekday, three new problems each weekend day, all from the [practice list](../02-dsa/12-practice-problem-list.md) ("first 30" before anything else).

## The 2-week crash plan

Use this when interviews are close. Two DSA problems a day throughout (from the "first 30" list), plus 15 minutes of story practice.

| Day | Focus |
|---|---|
| 1 | [Resume defense](../18-behavioral-and-hr/06-resume-line-by-line-defense.md), [intro](../18-behavioral-and-hr/01-tell-me-about-yourself.md), all six [project files](../17-projects-deep-dive/01-crystalcommerce.md) and their Fill-ins |
| 2 | [OOP](../01-cs-fundamentals/02-oop.md), [SOLID](../01-cs-fundamentals/03-solid-and-design-principles.md), [C++ OOP](../03-languages/c-cpp/03-cpp-oop.md) quick revisions; [CS 99](../01-cs-fundamentals/99-interview-questions.md) |
| 3 | Ruby quick revisions; [Ruby 99](../03-languages/ruby/99-interview-questions.md) |
| 4 | Rails 01-09 quick revisions and questions |
| 5 | Rails 10-18 quick revisions; [Rails 99](../05-ruby-on-rails/99-interview-questions.md) |
| 6 | [SQL from zero](../12-databases/02-sql-from-zero.md), 15 [SQL practice problems](../12-databases/03-sql-practice-problems.md), [indexing](../12-databases/05-indexing-and-query-optimization.md), [transactions](../12-databases/06-transactions-and-concurrency.md) |
| 7 | JavaScript quick revisions; [JS 99](../03-languages/javascript/99-interview-questions.md) output puzzles; implement debounce and Promise.all from memory |
| 8 | TypeScript quick revisions and [TS 99](../03-languages/typescript/99-interview-questions.md); React 01-04 |
| 9 | React 05-11; [React 99](../06-react/99-interview-questions.md) |
| 10 | React Native and Expo quick revisions; [RN 99](../08-react-native-expo/99-interview-questions.md) |
| 11 | Node, Next.js, TanStack and Drizzle quick revisions; their `99-` banks |
| 12 | Python and Django quick revisions; their `99-` banks |
| 13 | [Web 99](../04-web-fundamentals/99-interview-questions.md), [multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md), [API integration](../13-architecture-and-practices/05-third-party-api-integration.md), [system design from your projects](../15-system-design/03-case-studies-from-your-projects.md); one mock interview |
| 14 | [Last-minute revision](04-last-minute-revision.md), behavioral rehearsal, light DSA, early night |

## The 3-day emergency plan

For an interview at very short notice. Pick the row for your role from the role tracks table above.

| Day | Morning | Afternoon | Evening |
|---|---|---|---|
| 1 | Intro, resume defense, the two projects closest to the role | Role track: `Quick revision` of every must-review file | 4 DSA problems from the "first 30" |
| 2 | Role `99-` bank out loud (mark misses) | [SQL practice](../12-databases/03-sql-practice-problems.md) (10 problems), [web 99](../04-web-fundamentals/99-interview-questions.md) | Re-read the files behind every missed question |
| 3 | [Last-minute revision](04-last-minute-revision.md), [HR questions](../18-behavioral-and-hr/03-common-hr-questions.md), [questions to ask](../18-behavioral-and-hr/05-questions-to-ask.md) | One mock interview (ask a friend, or record yourself) | Rest. Sleep matters more than one more file |

## A daily routine that works

| Block | Time | What to do |
|---|---|---|
| DSA | 45-60 min | One new problem (timed: 25 min to solve, then study the solution) and one re-solve of an older problem |
| Topic study | 60-90 min | One T1 file: read it, close it, write its `Quick revision` from memory, then answer its questions aloud |
| Recall | 20-30 min | 15 questions from a `99-` bank, answered out loud before looking |
| Story | 10 min | Tell one STAR story or your intro out loud, ideally recorded |

## Mock interview checklist

- [ ] 90-second intro without notes
- [ ] One project explained end to end: problem, architecture, your part, a hard bug, what you would change
- [ ] One DSA problem solved aloud in C++ in under 30 minutes, with complexity analysis
- [ ] One SQL query written live (top N per group, or second highest salary)
- [ ] Five rapid-fire questions from the role's `99-` bank
- [ ] One behavioral question in STAR format
- [ ] Two thoughtful questions for the interviewer
- [ ] Afterwards: write down every question you fumbled and schedule its file for tomorrow
