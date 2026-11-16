# Resume Line-by-Line Defense

> **Tier:** T1 (know cold) · **Related:** [Resume-to-topics map](../00-start-here/02-resume-to-topics-map.md), [Tell me about yourself](01-tell-me-about-yourself.md), [STAR stories bank](02-star-stories-bank.md), [Common HR questions](03-common-hr-questions.md), [Projects: CrystalCommerce](../17-projects-deep-dive/01-crystalcommerce.md), [Final-year project](../16-ai-and-final-year-project/02-human-activity-recognition-project.md)
> **On your resume:** every line. Interviewers read the resume with a pen: they circle what matches the job and what looks inflated, then ask about both.

## Quick revision
- **Every word is a question waiting to happen.** If you cannot talk about a line for five minutes, reword it or remove it before you apply.
- **Verbs carry claims:** "built" and "integrated" mean you did it; "worked on" means you contributed to something existing; "contributed to" means a team effort with specific pieces yours; "enhanced" means you improved an existing system. Defend each at its real level.
- **Scale numbers belong to platforms:** 700+ stores, 8M+ customers, 16K+ users, 65 houses in 8 cities, 24K+ applications and 400+ residents describe the products you worked on. Say "the platform serves..." and know where each number comes from.
- **Skills without a project** (GraphQL, Firebase, Blazor, Dapper, Java, MySQL, C#, C, C++): state the real exposure, show the fundamentals, connect to a related production skill. Never imply production use you do not have.
- **Confidentiality:** share the stack, your role, general patterns and public facts; never code, credentials, internal metrics or customer data.
- **Education:** BS Information Technology, PUCIT (Old Campus), 2021 to 2025; final-year project; TA from 2023 to 2025. Expect fundamentals questions from the courses you assisted in.

## 1. How interviewers read your resume
A typical reviewer spends a minute or two on the first pass: current title and company, the stack, the projects, then the skills list. In the interview they come back to:
- **Matches:** lines that fit the job, to check depth ("you used Sidekiq; how do retries work?").
- **Inflation signals:** big numbers, long skill lists, strong verbs on short roles.
- **Gaps and jumps:** short stints (Visnext), overlaps (two internships), stack changes.

| Verb on your resume | What it claims | How to defend it |
|---|---|---|
| Built | You created it | Design decisions, code you wrote, problems you solved |
| Integrated | You connected systems | The contract between them, failure handling, testing |
| Enhanced | You improved an existing system | What it was before, what you changed, how you avoided breaking it |
| Worked on | You contributed to an existing system | Your specific features within the larger system |
| Contributed to | A team effort; you own some pieces | Name your pieces precisely; describe the rest as the team's |
| Proposed | The idea came from you | Why, to whom, how it was accepted |
| Assisted | You supported others' work | What you did in practice (labs, grading, explaining) |

## 2. The summary line

> **Fill in:** paste your resume summary line here, exactly as written.

How to defend it:
- **Every adjective will be tested.** "Scalable", "expert", "proficient", "high-performance" invite hard questions; prefer facts (years, stacks, domains) over adjectives.
- **"Full-stack"** must be backed by both sides: backend (Rails, REST APIs, background jobs, oRPC, DRF, Next.js APIs) and frontend or mobile (React, React on Rails, React Native).
- **"1+ year of experience"** (or however you phrase it) must survive a breakdown: Devntech since November 2025, Visnext for one month, two internships from July to September 2025, plus the part-time TA role during your degree. Be clear about which of these you count.
- **"International clients"** is backed by your Devntech role; be ready to describe how you work with them.
- If the summary claims something the rest of the resume does not show, cut it.

## 3. Experience, line by line

### Devntech: Associate Software Engineer (Nov 2025 to present, onsite)
"Builds full-stack features for international clients in Rails, React, TypeScript and React Native (Expo); works across legacy Rails monoliths, REST APIs, background jobs and a multi-tenant platform migration."

| Phrase | Likely question | Honest stance | Study |
|---|---|---|---|
| "full-stack features" | "Describe one feature you built end to end." | One real feature with its layers (UI, API, job, migration) | [CrystalCommerce](../17-projects-deep-dive/01-crystalcommerce.md), [Colochousing](../17-projects-deep-dive/02-colochousing-web-and-api.md) |
| "international clients" | "Who are they? How do you communicate?" | Clients named on your resume, communication habits, time zones | [Common HR questions](03-common-hr-questions.md) |
| "Rails, React, TypeScript and React Native (Expo)" | "Which do you use most?" | A truthful split of your time | The topic folders for each |
| "legacy Rails monoliths" | "What makes it legacy? How do you change it safely?" | Specific traits you saw, and your safety habits | [Legacy codebases](../05-ruby-on-rails/17-legacy-codebases-and-upgrades.md) |
| "REST APIs" | "How do you design and version an API?" | Resources, status codes, pagination, versioning for mobile clients | [REST API design](../04-web-fundamentals/04-rest-api-design.md) |
| "background jobs" | "How do retries work? What is idempotency?" | At-least-once delivery, idempotent jobs, enqueue after commit | [Background jobs](../05-ruby-on-rails/09-background-jobs-sidekiq.md) |
| "multi-tenant platform migration" | "What was your part?" | Rails sync APIs, marketplace page, org-scoped access control | [Platform migration](../17-projects-deep-dive/04-multi-tenant-platform-migration.md) |

> **Fill in:** the rough split of your time across Rails, React, TypeScript and React Native, and one end-to-end feature per stack.

### Visnext Solutions: Associate Software Engineer (Oct 2025, onsite)
"One month of Ruby on Rails training covering MVC, ActiveRecord and Hotwire (Turbo, Stimulus)."

| Likely question | Honest stance | Study |
|---|---|---|
| "Why only one month?" | One neutral, true sentence, then what you gained | [Common HR questions](03-common-hr-questions.md), section 8 |
| "Your title says engineer, but it was training. Which was it?" | The title was Associate Software Engineer; the month was spent in structured Rails training | |
| "Explain MVC in Rails." | Request flow: router, controller, model, view | [Rails fundamentals](../05-ruby-on-rails/01-rails-fundamentals.md) |
| "What is the N+1 problem in ActiveRecord?" | Lazy loading in a loop; fix with `includes` | [N+1 queries](../05-ruby-on-rails/05-activerecord-querying-n-plus-1.md) |
| "Turbo Frames vs Turbo Streams? What is a Stimulus controller?" | Frames replace one region on navigation; Streams apply targeted DOM updates; Stimulus attaches small JavaScript behavior to HTML | [Hotwire](../05-ruby-on-rails/08-hotwire-turbo-stimulus.md) |

> **Fill in:** what you built during the training, and whether you have used Hotwire since.

### PureLogics: Python (Django) Full-Stack Intern (Jul 2025 to Sep 2025, onsite)
"Proposed and built an LMS (Coursera clone) with subscriptions and role-based dashboards using DRF and React."

| Likely question | Honest stance | Study |
|---|---|---|
| "Why did you propose it?" | The real motivation and how it was approved | [LMS deep-dive](../17-projects-deep-dive/05-lms-coursera-clone.md) |
| "How did subscriptions work?" | Lifecycle, webhooks or simulation (say which), backend gating | Same |
| "How were roles enforced?" | Role storage, DRF permission classes, queryset scoping | [DRF auth and permissions](../10-python-django/05-drf-auth-permissions-advanced.md) |
| "Was it used by real users?" | Only claim real users if there were any | |

### RemoteDev: Full-Stack Intern (Jul 2025 to Sep 2025, remote)
"Built and integrated backend APIs with Next.js (TypeScript) and MongoDB."

| Likely question | Honest stance | Study |
|---|---|---|
| "What did you build?" | Two or three endpoints and what they did | [RemoteDev deep-dive](../17-projects-deep-dive/06-remotedev-nextjs-mongodb.md) |
| "Integrated with what?" | The frontend or services you connected | Same |
| "How did you handle the Mongo connection in Next.js?" | One cached client per process | [Next.js backend APIs and MongoDB](../07-nextjs-and-tanstack/03-nextjs-backend-apis-and-mongodb.md) |
| "Two internships at once?" | Structure, transparency, delivery | [Common HR questions](03-common-hr-questions.md), section 9 |

### PUCIT, University of the Punjab: Teaching Assistant (2023 to 2025)
"Assisted in labs for Programming Fundamentals, OOP, Data Structures, Databases and Web Development."

| Likely question | Honest stance | Study |
|---|---|---|
| "What did you do as a TA?" | Your real duties: helping in labs, checking work, explaining concepts | [STAR stories bank](02-star-stories-bank.md), section 8.8 |
| "Which language were the labs in?" | Answer exactly | |
| "Explain [a concept from those courses] as you would to a student." | Expect pointers, recursion, OOP pillars, linked lists, normalization, joins, HTTP | [OOP](../01-cs-fundamentals/02-oop.md), [Big O](../02-dsa/00-big-o-and-problem-solving.md), [DBMS](../12-databases/01-dbms-fundamentals.md), [How the web works](../04-web-fundamentals/01-how-the-web-works.md) |
| "How were you selected?" | The real process | |

> **Fill in:** your TA duties, the languages used in each lab, and how you were selected.

Because you assisted in these labs, interviewers may hold you to a higher standard on these fundamentals. Revise them before every interview.

## 4. Projects, line by line

### CrystalCommerce (Ruby on Rails, PostgreSQL, background jobs, PriceCharting API)

| Bullet | Likely questions | Study |
|---|---|---|
| Enhanced CrystalCommerce's legacy Rails e-commerce platform serving 700+ game stores and 8M+ customers | "What did you enhance? What made it legacy? Where do those numbers come from?" | [CrystalCommerce deep-dive](../17-projects-deep-dive/01-crystalcommerce.md) |
| Integrated PriceCharting as a sales channel (background jobs sync listings, prices, stock and orders) | "Explain the integration. How was overselling prevented? How were rate limits handled?" | Same, plus [third-party API integration](../13-architecture-and-practices/05-third-party-api-integration.md) |
| Imported PriceCharting market prices into the catalog so stores can price inventory from market data | "How were products matched? How often did prices update? How do prices become store prices?" | Same, plus [e-commerce domain](../13-architecture-and-practices/07-ecommerce-domain.md) |

> **Fill in:** the job library used, and which parts of the integration were yours.

### Colochousing (Ruby on Rails, React on Rails, React Native (Expo), TypeScript, PostgreSQL)

| Bullet | Likely questions | Study |
|---|---|---|
| Worked on the Rails 7 web app and REST API (React on Rails, ERB, Sidekiq) of a Belgian co-living platform with 16K+ users, 65 houses in 8 cities, and 24K+ rental applications | "Why React on Rails plus ERB? Which Sidekiq jobs? How does the mobile app authenticate?" | [Colochousing web and API](../17-projects-deep-dive/02-colochousing-web-and-api.md) |
| Built features across the Coloc resident app (React Native, Expo, TypeScript), live on the App Store and Google Play for 400+ residents: events, tasks, split bills, rent payments, push notifications, and EAS releases | "Which features did you build? How do push notifications work? How do you release with EAS?" | [Coloc resident app](../17-projects-deep-dive/03-coloc-resident-app.md) |
| Contributed to migrating onto a new multi-tenant TypeScript platform (oRPC, TanStack Start, Drizzle): Rails sync APIs, marketplace page migration, and org-scoped access control | "Why migrate? What is org-scoped access control and where was it enforced? oRPC vs tRPC?" | [Platform migration](../17-projects-deep-dive/04-multi-tenant-platform-migration.md) |

### Final Year Project: Recognition and Demonstrations of Human Activities Leveraging LLMs (.NET, Python)
"Built a web app recognizing human activities from smartwatch, phone, and glasses data with fine-tuned LLMs."

| Likely question | Study |
|---|---|
| "Explain your FYP in one minute." | [FYP deep-dive](../16-ai-and-final-year-project/02-human-activity-recognition-project.md) |
| "Why LLMs for activity recognition?" | Same |
| "Which metrics did you use?" | Same, plus [ML and LLM fundamentals](../16-ai-and-final-year-project/01-ml-and-llm-fundamentals.md) |
| "What was your role in the team?" | Same |

## 5. Skills: exposure map
For every skill, know where it comes from. "Exposure" is professional (Devntech, and Visnext training), internship (PureLogics, RemoteDev), academic (coursework, TA, FYP) or self-study. Where the resume does not say, fill it in honestly.

### Languages

| Skill | Evidence on your resume | Exposure | Likely questions | Study |
|---|---|---|---|---|
| Ruby | Rails projects at Devntech; Visnext training | Professional | Blocks and procs, modules and mixins, metaprogramming basics | [Ruby basics](../03-languages/ruby/01-basics.md) |
| TypeScript | Coloc app and the migration at Devntech; RemoteDev | Professional, internship | Generics, narrowing, utility types, `unknown` vs `any` | [TypeScript fundamentals](../03-languages/typescript/01-fundamentals.md) |
| JavaScript | React and React on Rails work | Professional (alongside TypeScript) | Closures, `this`, the event loop, promises | [JavaScript basics](../03-languages/javascript/01-basics-types-coercion.md) |
| Python | LMS at PureLogics; FYP | Internship, academic | Data model, comprehensions, decorators, generators | [Python fundamentals](../03-languages/python/01-fundamentals.md) |
| C# | No project line names it | **Fill in:** FYP .NET side? coursework? | Value vs reference types, LINQ, async/await | [C# fundamentals](../03-languages/csharp/01-fundamentals.md) |
| Java | No project line names it | **Fill in:** which course or project? | OOP, collections, exceptions, JVM basics | [Java essentials](../03-languages/java/01-java-essentials.md) |
| C | No project line names it | **Fill in:** coursework or TA labs? | Pointers, memory, arrays and strings | [C essentials](../03-languages/c-cpp/01-c-essentials.md) |
| C++ | No project line names it | **Fill in:** coursework, TA labs, DSA practice? | OOP in C++, STL, references vs pointers | [C++ fundamentals](../03-languages/c-cpp/02-cpp-fundamentals.md) |
| SQL | PostgreSQL at Devntech; Databases labs as a TA | Professional, academic | Joins, grouping, indexes, transactions | [SQL from zero](../12-databases/02-sql-from-zero.md) |

### Backend frameworks and tools

| Skill | Evidence on your resume | Exposure | Likely questions | Study |
|---|---|---|---|---|
| Ruby on Rails | CrystalCommerce, Colochousing; Visnext training | Professional | Request lifecycle, ActiveRecord, callbacks, jobs | [Rails fundamentals](../05-ruby-on-rails/01-rails-fundamentals.md) |
| Node.js (oRPC) | The multi-tenant TypeScript platform | Professional | Procedures, middleware, typed context, oRPC vs tRPC | [oRPC](../09-nodejs-backend/05-orpc-and-typesafe-apis.md) |
| Node.js (Hono) | No project line names it | **Fill in:** where did you use Hono? | Routing, middleware, validation, runtimes | [Hono](../09-nodejs-backend/04-hono.md) |
| Django REST Framework | LMS at PureLogics | Internship | Serializers, viewsets, permissions, auth | [DRF](../10-python-django/04-django-rest-framework.md) |
| ASP.NET Core MVC | No project line names it | **Fill in:** FYP .NET side? coursework? | Middleware pipeline, controllers, DI, model binding | [ASP.NET Core MVC](../11-dotnet/01-aspnet-core-mvc.md) |
| GraphQL | No project line names it | **Fill in:** course, tutorial or side project? | Schema, resolvers, N+1 and batching, vs REST | [GraphQL](../04-web-fundamentals/05-graphql.md) |
| Sidekiq | Colochousing | Professional | Retries, idempotency, Redis, queues | [Sidekiq](../05-ruby-on-rails/09-background-jobs-sidekiq.md) |

### Frontend and mobile

| Skill | Evidence on your resume | Exposure | Likely questions | Study |
|---|---|---|---|---|
| React | Devntech features, React on Rails; LMS frontend | Professional, internship | Hooks, state, effects, rendering, performance | [React fundamentals](../06-react/01-fundamentals.md) |
| React Native (Expo) | Coloc resident app | Professional | Native vs JS threads, navigation, push, EAS | [RN fundamentals](../08-react-native-expo/01-react-native-fundamentals.md) |
| React on Rails | Colochousing | Professional | `react_component`, props, SSR, hydration | [React on Rails](../05-ruby-on-rails/16-react-on-rails.md) |
| TanStack Start | The migration (marketplace page) | Professional | Server functions, loaders, SSR, vs Next.js | [TanStack Start](../07-nextjs-and-tanstack/05-tanstack-start.md) |
| Next.js | RemoteDev | Internship | App Router, route handlers, caching, server components | [Next.js fundamentals](../07-nextjs-and-tanstack/01-nextjs-fundamentals.md) |
| Hotwire | Visnext training | Training; **Fill in:** any professional use? | Turbo Drive, Frames, Streams, Stimulus | [Hotwire](../05-ruby-on-rails/08-hotwire-turbo-stimulus.md) |
| Tailwind CSS | No project line names it | **Fill in:** which projects used it? | Utility-first trade-offs, configuration, v3 vs v4 | [Tailwind CSS](../04-web-fundamentals/03-tailwind-css.md) |
| Blazor | No project line names it | **Fill in:** FYP? coursework? | Components, render modes, Server vs WebAssembly | [Blazor](../11-dotnet/03-blazor.md) |

### Data

| Skill | Evidence on your resume | Exposure | Likely questions | Study |
|---|---|---|---|---|
| PostgreSQL | CrystalCommerce, Colochousing | Professional | Indexes, transactions, JSONB, upserts | [PostgreSQL](../12-databases/04-postgresql.md) |
| MySQL | No project line names it | **Fill in:** coursework or Databases labs? | Differences from PostgreSQL, InnoDB | [MySQL essentials](../12-databases/07-mysql-essentials.md) |
| MongoDB | RemoteDev | Internship | Embedding vs referencing, indexes, aggregation | [MongoDB fundamentals](../12-databases/08-mongodb-fundamentals.md) |
| Redis | Sidekiq keeps its jobs in Redis | Professional, indirect; **Fill in:** any direct use (caching, rate limits)? | Data types, persistence, caching patterns | [Redis](../12-databases/10-redis.md) |
| Firebase | No project line names it | **Fill in:** where did you use it? | Auth, Firestore, Cloud Messaging | [Firebase](../12-databases/11-firebase.md) |
| ActiveRecord | Rails projects | Professional | Associations, N+1, callbacks, transactions | [ActiveRecord basics](../05-ruby-on-rails/03-activerecord-basics.md) |
| Drizzle | The migration | Professional | Schema, queries, migrations, org scoping | [Drizzle ORM](../12-databases/13-drizzle-orm.md) |
| Entity Framework | No project line names it | **Fill in:** FYP .NET side? coursework? | DbContext, tracking, migrations, `Include` | [EF Core and Dapper](../11-dotnet/02-entity-framework-and-dapper.md) |
| Dapper | No project line names it | **Fill in:** where did you use it? | Micro-ORM vs EF Core, parameterized SQL | [EF Core and Dapper](../11-dotnet/02-entity-framework-and-dapper.md) |

### Tools and concepts

| Skill | Evidence on your resume | Exposure | Likely questions | Study |
|---|---|---|---|---|
| Git, GitHub | Daily work | Professional; **Fill in:** your team's branching and review flow | Rebase vs merge, resolving conflicts, pull requests | [Git](../14-devops-and-tools/01-git.md), [GitHub and workflows](../14-devops-and-tools/02-github-and-workflows.md) |
| Docker | No project line names it | **Fill in:** local development, deployment, coursework? | Images vs containers, Dockerfile, Compose | [Docker](../14-devops-and-tools/03-docker.md) |
| Expo EAS | Coloc releases | Professional | Build, Submit, Update, runtime versions | [EAS](../08-react-native-expo/08-eas-build-submit-update.md) |
| REST APIs | Colochousing API, Rails sync APIs, RemoteDev, LMS | Professional, internship | Methods, status codes, versioning, pagination | [REST API design](../04-web-fundamentals/04-rest-api-design.md) |
| Multi-tenancy | Org-scoped access control in the migration | Professional | Isolation models, enforcement, leak tests | [Multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md) |
| MVC | Rails at work and in training; Django (MVT) | Professional, internship | Responsibilities of each layer, fat models | [MVC and layered architecture](../13-architecture-and-practices/01-mvc-and-layered-architecture.md) |
| Clean Architecture | No project line names it | **Fill in:** where did you apply it (FYP, a .NET course)? | Dependency rule, layers, trade-offs | [Clean Architecture](../13-architecture-and-practices/02-clean-architecture.md) |

## 6. Skills listed without a project
A skill on your resume without a matching project line will be probed. Use this pattern:
1. **State the real exposure** in one sentence (coursework, FYP, self-study, a tutorial project).
2. **Show the fundamentals** in two or three sentences.
3. **Bridge** to a related skill you use in production.

If you cannot do steps 1 and 2 for a skill, remove it from your resume before applying, or move it to a clearly labeled "familiar with" line.

| Skill | Honest framing (fill in the exposure) | Minimum you must know | Bridge |
|---|---|---|---|
| GraphQL | "I haven't used GraphQL in production; my work APIs are REST and oRPC. I've used it in [exposure]." | Schema and types, queries and mutations, resolvers, the N+1 problem and batching, caching trade-offs vs REST | Type-safe APIs with oRPC solve a similar "typed contract" need |
| Firebase | "I've used Firebase in [exposure]." | Authentication, Firestore's document model, Cloud Messaging, security rules | Expo's push service delivers to Android devices through Firebase Cloud Messaging |
| Blazor | "I used Blazor in [exposure]." | Razor components, render modes (static SSR, interactive Server, WebAssembly, Auto since .NET 8), state and events | Component thinking from React |
| Dapper | "I used Dapper in [exposure]." | A micro-ORM: extension methods on `IDbConnection` (`Query<T>`, `Execute`), hand-written parameterized SQL, mapping to objects | Writing and tuning SQL in PostgreSQL |
| Java | "Java comes from [coursework or projects]." | OOP, interfaces vs abstract classes, collections, exceptions, the JVM and garbage collection | OOP concepts you use daily in Ruby and TypeScript |
| MySQL | "I used MySQL in [exposure]; at work I use PostgreSQL." | Differences from PostgreSQL: default isolation (Repeatable Read in InnoDB vs Read Committed), `ON DUPLICATE KEY UPDATE` vs `ON CONFLICT`, JSON vs JSONB | Strong relational and SQL skills from PostgreSQL |
| C# | "C# comes from [exposure, for example the FYP's .NET web app]." | Types, classes and records, LINQ, async/await, ASP.NET Core basics | Typed languages: TypeScript |
| C | "C was part of [coursework or labs]." | Pointers, memory allocation, arrays and strings, structs | Understanding memory explains performance everywhere |
| C++ | "I use C++ for [DSA practice or coursework]." | Classes, references vs pointers, RAII, STL containers and complexity | Your DSA rounds, if you solve them in C++ |

> **Fill in:** the exposure for each of these nine skills, and decide which ones stay on your resume.

Phrases that keep you honest without underselling:
- "I've used it in coursework, not in production."
- "I know the concepts and have built a small project with it; I'd need a short ramp-up for production work."
- "I haven't used it professionally, but here is how I'd approach it, based on [related skill]."

## 7. Attributing the scale numbers

| Number | What it measures | Whose number | Say it like this |
|---|---|---|---|
| 700+ game stores | Stores using CrystalCommerce's platform | The platform's | "a platform serving 700+ game stores" |
| 8M+ customers | Customers across those stores | The platform's | "and 8M+ customers" |
| 16K+ users | Users of the Colochousing platform | The platform's | "a co-living platform with 16K+ users" |
| 65 houses in 8 cities | Colochousing's houses | The business's | "65 houses in 8 cities" |
| 24K+ rental applications | Applications handled by the platform | The platform's | "24K+ rental applications" |
| 400+ residents | People the Coloc app serves | The app's audience | "live on both stores for 400+ residents" |

Rules:
- Never turn them into personal claims ("I handled 24K applications") or invent derived numbers (requests per second, revenue, growth rates) you never measured.
- Know the source and the date of each number in case someone asks "how do you know?" or "is that current?".
- If a number has changed since you wrote the resume, say so: "it was 700+ when I wrote this."

> **Fill in:** the source of each number (client materials, the public website, an admin dashboard) and when you last checked it.

## 8. Client confidentiality and NDAs

| Usually fine to share | Do not share |
|---|---|
| The technologies on your resume | Source code or repository access |
| Your role and the features you built, described in general terms | Credentials, API keys, infrastructure details |
| Architecture patterns ("jobs sync listings with a third-party channel") | Internal metrics beyond what is public or on your resume |
| Public facts about the client and its product | Customer or resident personal data |
| Problems and solutions at the level of patterns | Unreleased features, business plans, contract terms |

Common situations:
- **"Can you show us some code?"** Offer to walk through the architecture verbally or on a whiteboard, or show personal or public projects. Never share client code.
- **"Can you name your clients?"** They appear on your resume, so presumably you may name them; confirm what your contract allows.
- **"How much does the client pay your company?"** Politely decline: "That's confidential, but I can tell you about the work itself."

> **Fill in:** whether you signed an NDA, whether naming clients is allowed, and which personal or public code you can show instead.

If you have no public code to show, consider a small personal project that demonstrates a pattern from your work without any client code or data: for example, a Rails app that syncs listings with a fake third-party API through idempotent background jobs, or a small Expo app with push notifications. It gives interviewers something concrete to read, and you can discuss every line of it freely.

## 9. Education
"BS Information Technology, PUCIT (Old Campus), 2021 to 2025."

| Likely question | How to prepare |
|---|---|
| "Why IT?" | One honest sentence (see the Fill-in below) |
| "Which subjects did you enjoy most?" | Pick ones relevant to the role, and be ready for a question on them |
| "What was your CGPA?" | Exact number; see [Common HR questions](03-common-hr-questions.md), section 10 |
| "IT vs CS: what is the difference?" | Curricula overlap heavily; IT programs often lean more applied. Then point to your fundamentals: you assisted in core CS labs for two years |
| "Tell me about your FYP." | [FYP deep-dive](../16-ai-and-final-year-project/02-human-activity-recognition-project.md) |
| Fundamentals checks (OOP, DSA, DBMS, OS, networking) | [OOP](../01-cs-fundamentals/02-oop.md), [Big O](../02-dsa/00-big-o-and-problem-solving.md), [DBMS](../12-databases/01-dbms-fundamentals.md), [Operating systems](../01-cs-fundamentals/05-operating-systems.md), [Networking](../01-cs-fundamentals/06-networking.md) |

> **Fill in:** why you chose IT, your favorite two courses, your CGPA, and your FYP team size and role.

Classic questions from the five courses you assisted in, so you are never caught out:

| Course | Classic questions |
|---|---|
| Programming Fundamentals | Pass by value vs reference, recursion and the call stack, arrays vs pointers |
| OOP | The four pillars with examples, abstract class vs interface, overloading vs overriding, composition vs inheritance |
| Data Structures | Array vs linked list, stack vs queue uses, hash table collisions, BST operations and complexity |
| Databases | Normalization (1NF to 3NF), joins, primary vs foreign keys, indexes, ACID |
| Web Development | What happens when you type a URL, HTTP methods and status codes, cookies vs sessions vs tokens |

## 10. Your timeline, consistent everywhere

| Period | Role | Mode | What to say about it |
|---|---|---|---|
| 2021 to 2025 | BS Information Technology, PUCIT (Old Campus) | Student | Your degree; the FYP in the final year |
| 2023 to 2025 | Teaching Assistant, PUCIT | Part-time, during the degree | Labs in five core courses |
| Jul to Sep 2025 | PureLogics, Python (Django) Full-Stack Intern | Onsite | Proposed and built the LMS |
| Jul to Sep 2025 | RemoteDev, Full-Stack Intern | Remote, in parallel | Built and integrated Next.js and MongoDB APIs |
| Oct 2025 | Visnext Solutions, Associate Software Engineer | Onsite | One month of Rails training |
| Nov 2025 to present | Devntech, Associate Software Engineer | Onsite | Client work in Rails, React, TypeScript, React Native |

- **There are no gaps** from July 2025 onward; say so confidently if asked.
- **Total experience:** count it the same way every time. A clear version: "about [N] months at Devntech since November 2025, plus a month at Visnext and two internships in 2025." Update N before each interview.
- **The overlap and the short stint** have prepared answers in [common HR questions](03-common-hr-questions.md) (sections 8 and 9).

> **Fill in:** your total experience, stated the way you will say it, as of your next interview date.

## 11. When an interviewer goes deep on a skill you know lightly
It will happen, especially with long skills lists. A calm sequence:
1. **Say what you know** precisely, without padding.
2. **Reason out loud** from principles you do know ("I haven't used Dapper's multi-mapping, but since it maps rows to objects, I'd expect...").
3. **Mark the boundary:** "That's the limit of what I've done with it."
4. **Say how you would find out:** the official docs, a small spike, asking a teammate who knows it.

Interviewers generally respect an honest boundary much more than a confident guess that turns out wrong. One wrong bluff makes them doubt the rest of your resume.

## 12. Rewording options
If a line is hard to defend, reword it rather than hoping nobody asks:
- **Group skills by depth**, for example "Professional: Ruby on Rails, React, TypeScript, React Native (Expo), PostgreSQL, Sidekiq" and "Familiar: GraphQL, Firebase, Blazor, Dapper". Put each skill where your answers in section 5 place it.
- **Keep the verbs exact:** "contributed to" for team efforts is a strength, not a weakness, because you can defend it fully.
- **Attach numbers to platforms** in the wording itself ("a platform serving 700+ stores"), as your resume already does.
- **Drop skills you would not want to be tested on today.** A shorter list you can defend beats a long one you cannot.

> **Fill in:** the changes you will make to your resume after working through this file, if any.

## 13. Rapid drill: questions straight from your resume
Have a friend pick ten at random; answer each in under a minute, then check the linked file.
1. "What did you enhance in CrystalCommerce's platform?" ([CrystalCommerce](../17-projects-deep-dive/01-crystalcommerce.md))
2. "Explain the PriceCharting integration end to end." (same)
3. "How did you prevent overselling across channels?" (same)
4. "How did you handle PriceCharting's rate limits?" (same)
5. "Why React on Rails and ERB together?" ([Colochousing](../17-projects-deep-dive/02-colochousing-web-and-api.md))
6. "Which Sidekiq jobs did you write?" (same)
7. "Which features did you build in the Coloc app?" ([Coloc app](../17-projects-deep-dive/03-coloc-resident-app.md))
8. "How do push notifications work in your app?" (same)
9. "How do you release with EAS, and how would you roll back a bad update?" (same)
10. "Why migrate to a new platform?" ([Platform migration](../17-projects-deep-dive/04-multi-tenant-platform-migration.md))
11. "What is org-scoped access control, and where was it enforced?" (same)
12. "How did subscriptions and role-based dashboards work in your LMS?" ([LMS](../17-projects-deep-dive/05-lms-coursera-clone.md))
13. "What did you build at RemoteDev?" ([RemoteDev](../17-projects-deep-dive/06-remotedev-nextjs-mongodb.md))
14. "Why did you leave Visnext after a month?" ([HR questions](03-common-hr-questions.md))
15. "How did you manage two internships at once?" (same)
16. "What did you do as a TA? Explain normalization as you would to a student." ([DBMS](../12-databases/01-dbms-fundamentals.md))
17. "Explain your FYP in one minute. Why LLMs? Which metrics?" ([FYP](../16-ai-and-final-year-project/02-human-activity-recognition-project.md))
18. "You list GraphQL. Compare it with REST." ([GraphQL](../04-web-fundamentals/05-graphql.md))
19. "You list Docker. What is the difference between an image and a container?" ([Docker](../14-devops-and-tools/03-docker.md))
20. "You list Clean Architecture. Where did you apply it?" ([Clean Architecture](../13-architecture-and-practices/02-clean-architecture.md))

## 14. Final resume hygiene check
- [ ] Every bullet can be defended for five minutes, including follow-ups.
- [ ] Verbs match your real role ("built" vs "contributed to").
- [ ] Every scale number is attributed to its platform, with a known source.
- [ ] Dates match your LinkedIn profile, experience letters and what you say out loud.
- [ ] Every listed skill passes section 6, or is removed or labeled "familiar with".
- [ ] The summary line contains facts, not adjectives you cannot back.
- [ ] Links (GitHub, portfolio, LinkedIn) work and show nothing confidential.
- [ ] You have reread this file and your project deep-dives within a day of the interview.
