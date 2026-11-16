# Resume-to-Topics Map

> Every line of your resume, what an interviewer infers from it, where to study it, and the first questions it invites. Answer scripts live in [resume line-by-line defense](../18-behavioral-and-hr/06-resume-line-by-line-defense.md); this page is the study map.

## How to use this page

1. Before every interview, open the exact resume version the company has and walk it line by line next to these tables.
2. For each line, answer its questions out loud. If you stumble, study the linked files that evening.
3. Complete the **Exposure** column honestly. Interviewers ask "where did you use X?" for every listed skill, and a clear, truthful answer ("coursework and my final-year project") always beats a vague one.

| Exposure level | Meaning |
|---|---|
| **Professional** | Used in your current job at Devntech (Nov 2025 to present) |
| **Training** | The one-month Rails training at Visnext Solutions (Oct 2025) |
| **Internship** | PureLogics (Django, onsite) or RemoteDev (Next.js, remote), Jul to Sep 2025 |
| **Academic** | University coursework, TA labs (PF, OOP, DS, Databases, Web Development) or the final-year project |
| **Listed** | On your skills line with no project on the resume: decide where you really used it and fill it in |

## Summary line

> Full-stack Software Engineer with 1+ year of experience shipping web and mobile apps in Ruby on Rails, React, React Native (Expo), and TypeScript, across legacy codebases, API integrations, and multi-tenant migrations.

| Claim | Study | Questions it invites |
|---|---|---|
| Shipping web and mobile apps | [Colochousing](../17-projects-deep-dive/02-colochousing-web-and-api.md), [Coloc app](../17-projects-deep-dive/03-coloc-resident-app.md) | "Walk me through a feature you shipped end to end." |
| Legacy codebases | [Legacy Rails and upgrades](../05-ruby-on-rails/17-legacy-codebases-and-upgrades.md) | "How do you get productive in a codebase you did not write?" |
| API integrations | [Third-party API integration](../13-architecture-and-practices/05-third-party-api-integration.md), [CrystalCommerce](../17-projects-deep-dive/01-crystalcommerce.md) | "How do you handle rate limits and failures from an external API?" |
| Multi-tenant migrations | [Multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md), [migration strategies](../13-architecture-and-practices/04-platform-migration-strategies.md), [the migration project](../17-projects-deep-dive/04-multi-tenant-platform-migration.md) | "How do you guarantee one tenant never sees another tenant's data?" |

## Experience

### Devntech, Associate Software Engineer (Nov 2025 to present)

| Bullet | What they infer | Study | Questions it invites |
|---|---|---|---|
| Build full-stack features for international clients in Rails, React, TypeScript, and React Native (Expo) | You work across the stack and with foreign clients | [Rails](../05-ruby-on-rails/01-rails-fundamentals.md), [React](../06-react/01-fundamentals.md), [TypeScript](../03-languages/typescript/01-fundamentals.md), [React Native](../08-react-native-expo/01-react-native-fundamentals.md), [code quality and Agile](../13-architecture-and-practices/10-code-quality-and-agile.md) | "How do you communicate with clients in another time zone?" "Which part of the stack do you enjoy most?" |
| Work across legacy Rails monoliths, REST APIs, background jobs, and a multi-tenant platform migration | You can handle messy real systems, not only greenfield code | [Legacy Rails](../05-ruby-on-rails/17-legacy-codebases-and-upgrades.md), [Rails APIs](../05-ruby-on-rails/10-api-development.md), [Sidekiq](../05-ruby-on-rails/09-background-jobs-sidekiq.md), [background patterns](../13-architecture-and-practices/06-background-processing-patterns.md), [multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md) | "What makes a job idempotent?" "How did you find your way around the monolith?" |

### Visnext Solutions, Associate Software Engineer (Oct 2025)

| Bullet | What they infer | Study | Questions it invites |
|---|---|---|---|
| One month of Ruby on Rails training covering MVC, ActiveRecord, and Hotwire (Turbo, Stimulus) | You learned Rails formally and quickly; the short stint will be asked about | [Rails fundamentals](../05-ruby-on-rails/01-rails-fundamentals.md), [ActiveRecord basics](../05-ruby-on-rails/03-activerecord-basics.md), [Hotwire](../05-ruby-on-rails/08-hotwire-turbo-stimulus.md), [HR questions](../18-behavioral-and-hr/03-common-hr-questions.md) | "Why only one month?" "Turbo Frames vs Turbo Streams?" |

### PureLogics, Python (Django) Full-Stack Intern (Jul to Sep 2025)

| Bullet | What they infer | Study | Questions it invites |
|---|---|---|---|
| Proposed and built an LMS (Coursera clone) with subscriptions and role-based dashboards using DRF and React | Initiative ("proposed"), full ownership, payments and RBAC | [LMS project](../17-projects-deep-dive/05-lms-coursera-clone.md), [LMS patterns](../10-python-django/07-lms-project-patterns.md), [DRF](../10-python-django/04-django-rest-framework.md), [DRF permissions](../10-python-django/05-drf-auth-permissions-advanced.md), [payments](../13-architecture-and-practices/08-payments-and-subscriptions.md) | "How did subscription status gate access?" "How were roles modeled?" "How did you protect course videos?" |

### RemoteDev, Full-Stack Intern (Jul to Sep 2025, remote)

| Bullet | What they infer | Study | Questions it invites |
|---|---|---|---|
| Built and integrated backend APIs with Next.js (TypeScript) and MongoDB | Node/TS backend experience and remote collaboration; overlaps PureLogics in time | [RemoteDev](../17-projects-deep-dive/06-remotedev-nextjs-mongodb.md), [Next.js backend and MongoDB](../07-nextjs-and-tanstack/03-nextjs-backend-apis-and-mongodb.md), [MongoDB schema design](../12-databases/09-mongodb-schema-design-aggregation-mongoose.md) | "Route handlers or a separate API, and why?" "How did you reuse the Mongo connection?" "How did you manage two internships at once?" |

### PUCIT, Teaching Assistant (2023 to 2025)

| Bullet | What they infer | Study | Questions it invites |
|---|---|---|---|
| Assisted in labs for Programming Fundamentals, OOP, Data Structures, Databases, and Web Development | Expect a higher bar on fundamentals and clear explanations | [CS fundamentals](../01-cs-fundamentals/02-oop.md), [C++ OOP](../03-languages/c-cpp/03-cpp-oop.md), [DSA](../02-dsa/00-big-o-and-problem-solving.md), [DBMS](../12-databases/01-dbms-fundamentals.md), [how the web works](../04-web-fundamentals/01-how-the-web-works.md) | "Teach me recursion." "Explain normalization to a beginner." "Why does a base class need a virtual destructor?" |

## Projects

### CrystalCommerce (Devntech client)

| Bullet | Study | Questions it invites |
|---|---|---|
| Enhanced the legacy Rails e-commerce platform serving 700+ game stores and 8M+ customers | [CrystalCommerce](../17-projects-deep-dive/01-crystalcommerce.md), [legacy Rails](../05-ruby-on-rails/17-legacy-codebases-and-upgrades.md), [e-commerce domain](../13-architecture-and-practices/07-ecommerce-domain.md) | "Is 8M your impact or the platform's size?" "What made the legacy code hard?" |
| Integrated PriceCharting as a sales channel: background jobs sync listings, prices, stock, and orders | [API integration](../13-architecture-and-practices/05-third-party-api-integration.md), [Sidekiq](../05-ruby-on-rails/09-background-jobs-sidekiq.md), [locking](../05-ruby-on-rails/06-activerecord-advanced.md), [system design from your projects](../15-system-design/03-case-studies-from-your-projects.md) | "Polling or webhooks?" "What is the source of truth for stock?" "What if one item sells on both channels at once?" |
| Imported PriceCharting market prices into the catalog so stores can price inventory from market data | [Bulk operations](../05-ruby-on-rails/06-activerecord-advanced.md), [background patterns](../13-architecture-and-practices/06-background-processing-patterns.md), [indexing](../12-databases/05-indexing-and-query-optimization.md) | "How did you match external products to catalog items?" "How do you import millions of rows safely?" |

### Colochousing (Devntech client)

| Bullet | Study | Questions it invites |
|---|---|---|
| Rails 7 web app and REST API (React on Rails, ERB, Sidekiq) for a Belgian co-living platform: 16K+ users, 65 houses, 8 cities, 24K+ rental applications | [Colochousing](../17-projects-deep-dive/02-colochousing-web-and-api.md), [React on Rails](../05-ruby-on-rails/16-react-on-rails.md), [Rails APIs](../05-ruby-on-rails/10-api-development.md), [Sidekiq](../05-ruby-on-rails/09-background-jobs-sidekiq.md) | "Why React on Rails plus ERB?" "Which jobs ran in Sidekiq?" "How does the mobile app authenticate?" |
| Coloc resident app (React Native, Expo, TypeScript) on the App Store and Google Play for 400+ residents: events, tasks, split bills, rent payments, push notifications, EAS releases | [Coloc app](../17-projects-deep-dive/03-coloc-resident-app.md), [push notifications](../08-react-native-expo/05-push-notifications.md), [payments and split bills](../08-react-native-expo/06-payments-and-split-bills.md), [EAS](../08-react-native-expo/08-eas-build-submit-update.md) | "Why not Apple in-app purchase for rent?" "How do you split a bill without rounding errors?" "How do you roll back a bad OTA update?" |
| Contributed to migrating onto a new multi-tenant TypeScript platform (oRPC, TanStack Start, Drizzle): Rails sync APIs, marketplace page migration, org-scoped access control | [Migration project](../17-projects-deep-dive/04-multi-tenant-platform-migration.md), [oRPC](../09-nodejs-backend/05-orpc-and-typesafe-apis.md), [TanStack Start](../07-nextjs-and-tanstack/05-tanstack-start.md), [Drizzle](../12-databases/13-drizzle-orm.md), [multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md), [migration strategies](../13-architecture-and-practices/04-platform-migration-strategies.md) | "Why rewrite instead of refactor?" "Where is org scoping enforced, and how do you test for leaks?" "How did you keep the marketplace's SEO?" |

### Final-year project: Recognition and Demonstrations of Human Activities Leveraging LLMs (.NET, Python)

| Bullet | Study | Questions it invites |
|---|---|---|
| Web app recognizing human activities from smartwatch, phone, and glasses data with fine-tuned LLMs | [HAR project](../16-ai-and-final-year-project/02-human-activity-recognition-project.md), [ML and LLM fundamentals](../16-ai-and-final-year-project/01-ml-and-llm-fundamentals.md), [ASP.NET Core](../11-dotnet/01-aspnet-core-mvc.md) | "Why an LLM instead of a CNN or LSTM?" "How did you turn sensor data into model input?" "Which metrics, and how did you split the data?" |

## Skills: exposure and the first question to expect

Fill in every `Fill in` cell before your first interview. Where you have little exposure, prepare the honest one-liner from the T3 file's "How to talk about this" section.

### Languages

| Skill | Exposure | Study | First question to expect |
|---|---|---|---|
| Ruby | Professional, Training | [Ruby](../03-languages/ruby/01-basics.md) | "Proc vs lambda?" |
| TypeScript | Professional, Internship | [TypeScript](../03-languages/typescript/01-fundamentals.md) | "Interface vs type alias?" |
| JavaScript | Professional | [JavaScript](../03-languages/javascript/01-basics-types-coercion.md) | "Explain the event loop." |
| Python | Internship, Academic (FYP) | [Python](../03-languages/python/01-fundamentals.md) | "What does a decorator do?" |
| C# | Academic (FYP .NET side): Fill in | [C#](../03-languages/csharp/01-fundamentals.md) | "Value types vs reference types?" |
| Java | Listed: Fill in | [Java](../03-languages/java/01-java-essentials.md) | "`==` vs `equals()`?" |
| C | Academic | [C essentials](../03-languages/c-cpp/01-c-essentials.md) | "What is a dangling pointer?" |
| C++ | Academic (TA labs), your DSA language | [C++](../03-languages/c-cpp/02-cpp-fundamentals.md), [STL](../03-languages/c-cpp/04-stl-for-dsa.md) | "Why does a base class need a virtual destructor?" |
| SQL | Professional | [SQL](../12-databases/02-sql-from-zero.md), [practice](../12-databases/03-sql-practice-problems.md) | "Write a query for the second-highest salary." |

### Backend

| Skill | Exposure | Study | First question to expect |
|---|---|---|---|
| Ruby on Rails | Professional, Training | [Rails](../05-ruby-on-rails/01-rails-fundamentals.md) | "Walk me through the request lifecycle." |
| Node.js (Hono, oRPC) | Professional for oRPC (migration platform); Hono: Fill in | [Node](../09-nodejs-backend/01-nodejs-fundamentals.md), [Hono](../09-nodejs-backend/04-hono.md), [oRPC](../09-nodejs-backend/05-orpc-and-typesafe-apis.md) | "How does Node handle concurrency on one thread?" |
| Django REST Framework | Internship | [DRF](../10-python-django/04-django-rest-framework.md) | "`select_related` vs `prefetch_related`?" |
| ASP.NET Core MVC | Academic (FYP): Fill in | [ASP.NET Core MVC](../11-dotnet/01-aspnet-core-mvc.md) | "Explain the middleware pipeline and DI lifetimes." |
| GraphQL | Listed: Fill in | [GraphQL](../04-web-fundamentals/05-graphql.md) | "How do you solve N+1 in GraphQL?" |
| Sidekiq | Professional (Colochousing) | [Sidekiq](../05-ruby-on-rails/09-background-jobs-sidekiq.md) | "How do retries work, and why must jobs be idempotent?" |

### Frontend and mobile

| Skill | Exposure | Study | First question to expect |
|---|---|---|---|
| React | Professional, Internship | [React](../06-react/01-fundamentals.md) | "What causes a component to re-render?" |
| React Native (Expo) | Professional (Coloc app) | [React Native](../08-react-native-expo/01-react-native-fundamentals.md) | "How does React Native render native UI?" |
| React on Rails | Professional (Colochousing) | [React on Rails](../05-ruby-on-rails/16-react-on-rails.md) | "How do props get from Rails to React?" |
| TanStack Start | Professional (migration platform) | [TanStack Router](../07-nextjs-and-tanstack/04-tanstack-router.md), [TanStack Start](../07-nextjs-and-tanstack/05-tanstack-start.md) | "What is a server function?" |
| Next.js | Internship (RemoteDev) | [Next.js](../07-nextjs-and-tanstack/01-nextjs-fundamentals.md) | "SSR vs SSG vs ISR?" |
| Hotwire | Training; any professional use: Fill in | [Hotwire](../05-ruby-on-rails/08-hotwire-turbo-stimulus.md) | "Turbo Frames vs Turbo Streams?" |
| Tailwind CSS | Listed: Fill in | [Tailwind](../04-web-fundamentals/03-tailwind-css.md) | "Pros and cons of utility-first CSS?" |
| Blazor | Listed: Fill in (FYP?) | [Blazor](../11-dotnet/03-blazor.md) | "Blazor Server vs WebAssembly?" |

### Databases and ORMs

| Skill | Exposure | Study | First question to expect |
|---|---|---|---|
| PostgreSQL | Professional | [Postgres](../12-databases/04-postgresql.md), [indexing](../12-databases/05-indexing-and-query-optimization.md) | "How does an index make a query faster, and when is it skipped?" |
| MySQL | Listed or Academic: Fill in | [MySQL essentials](../12-databases/07-mysql-essentials.md) | "MySQL vs PostgreSQL?" |
| MongoDB | Internship (RemoteDev) | [MongoDB](../12-databases/08-mongodb-fundamentals.md) | "Embed or reference?" |
| Redis | Professional (backs Sidekiq); direct use: Fill in | [Redis](../12-databases/10-redis.md) | "What do you use Redis for besides caching?" |
| Firebase | Listed: Fill in | [Firebase](../12-databases/11-firebase.md) | "When would you pick Firebase over your own backend?" |
| ActiveRecord | Professional | [ActiveRecord basics](../05-ruby-on-rails/03-activerecord-basics.md), [querying](../05-ruby-on-rails/05-activerecord-querying-n-plus-1.md) | "`includes` vs `joins`?" |
| Drizzle | Professional (migration platform) | [Drizzle](../12-databases/13-drizzle-orm.md) | "Why Drizzle over Prisma?" |
| Entity Framework | Academic: Fill in | [EF Core and Dapper](../11-dotnet/02-entity-framework-and-dapper.md) | "What does `AsNoTracking` do?" |
| Dapper | Listed: Fill in | [EF Core and Dapper](../11-dotnet/02-entity-framework-and-dapper.md) | "When would you choose Dapper over EF Core?" |

### Tools and practices

| Skill | Exposure | Study | First question to expect |
|---|---|---|---|
| Git | Professional | [Git](../14-devops-and-tools/01-git.md) | "Merge vs rebase?" |
| GitHub | Professional | [GitHub workflows](../14-devops-and-tools/02-github-and-workflows.md) | "Describe your pull request workflow." |
| Docker | Listed: Fill in | [Docker](../14-devops-and-tools/03-docker.md) | "Image vs container?" |
| Expo EAS | Professional | [EAS](../08-react-native-expo/08-eas-build-submit-update.md) | "What can an OTA update change, and what can it not?" |
| REST APIs | Professional | [REST design](../04-web-fundamentals/04-rest-api-design.md), [Rails APIs](../05-ruby-on-rails/10-api-development.md) | "PUT vs PATCH?" |
| Multi-tenancy | Professional (migration platform) | [Multi-tenancy](../13-architecture-and-practices/03-multi-tenancy.md) | "Shared schema with tenant_id vs schema per tenant?" |
| MVC | Professional, Training | [MVC and layers](../13-architecture-and-practices/01-mvc-and-layered-architecture.md) | "Where should business logic live?" |
| Clean Architecture | Listed: Fill in | [Clean architecture](../13-architecture-and-practices/02-clean-architecture.md) | "What is the dependency rule?" |

## Education

**BS Information Technology, PUCIT (Old Campus), 2021 to 2025.** Expect questions about the final-year project (see above) and, occasionally, your CGPA. The framing for both is in [common HR questions](../18-behavioral-and-hr/03-common-hr-questions.md).

## The numbers on your resume

Interviewers often test whether you understand what a number measures and what you personally contributed.

| Number | What it measures | How to say it |
|---|---|---|
| 700+ game stores, 8M+ customers | The scale of the CrystalCommerce platform | "The platform serves 700+ stores and over 8 million customers. My part was..." (> **Fill in:** your concrete contribution) |
| 16K+ users, 65 houses, 8 cities, 24K+ rental applications | The scale of the Colochousing platform | "It is a co-living platform with 16K+ users across 65 houses. I worked on..." |
| 400+ residents | Users of the Coloc resident app | "The app is live on both stores for 400+ residents, and I built..." |
| 1 month | The Visnext training period | Explain it briefly and truthfully, then move on (see [HR questions](../18-behavioral-and-hr/03-common-hr-questions.md)) |
| 1+ year | Your total professional and internship experience | Count it accurately if asked: internships Jul to Sep 2025, training Oct 2025, Devntech since Nov 2025 |

**Rule:** platform scale is context, not personal impact. Always follow a scale number with what you owned.
