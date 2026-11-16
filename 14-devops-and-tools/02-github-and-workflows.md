# GitHub and Team Workflows

> **Tier:** T2 (working knowledge) · **Related:** [Git](01-git.md), [CI/CD and deployment](04-ci-cd-and-deployment.md), [Code quality and agile](../13-architecture-and-practices/10-code-quality-and-agile.md), [Web security and OWASP](../04-web-fundamentals/07-web-security-owasp.md), [EAS Build, Submit and Update](../08-react-native-expo/08-eas-build-submit-update.md)
> **On your resume:** GitHub in your skills list; pull requests and code review in client repositories at Devntech (CrystalCommerce, Colochousing), plus team work at PureLogics and RemoteDev.

## Quick revision
- **PR workflow:** sync `main`, branch, commit in small steps, push, open a (draft) PR with a clear description, get CI green, review, address feedback, merge, delete the branch, deploy and verify.
- A good PR description covers what and why, how to test, screenshots for UI, risks and rollback, and the linked issue (`Closes #123`).
- GitHub merge buttons: **merge commit** (keeps every commit), **squash and merge** (one commit), **rebase and merge** (linear, keeps commits).
- Authors: small focused PRs, self-review first, context up front, reply to every comment. Reviewers: respond within a day, focus on correctness, security and design, label nits, ask rather than command, approve when good enough.
- **GitHub flow:** an always-deployable `main` plus short-lived branches. **GitFlow:** `develop`, feature, release and hotfix branches for versioned releases. **Trunk-based:** tiny branches merged daily, unfinished work behind feature flags.
- **Branch protection or rulesets:** required reviews and status checks, up-to-date branches, no force pushes, linear history, resolved conversations.
- **CODEOWNERS** (`.github/CODEOWNERS`) auto-requests reviews from path owners; the last matching pattern wins; owner review can be required.
- Issues: templates, labels, milestones, assignees; closing keywords (`Fixes #12`) close the issue when the PR merges into the default branch; Projects for boards.
- Releases are built on tags and can generate notes from merged PRs; versions follow semantic versioning.
- **Dependabot:** vulnerability alerts, security update PRs and scheduled version updates from `.github/dependabot.yml`; group minor and patch bumps, review majors by hand.
- **Client repositories:** follow their conventions, never push to their `main`, keep their code confidential, use their tracker, write PRs that work across time zones.
- **Small PRs:** a few hundred changed lines, refactors separate from features, stacked PRs and feature flags for big work.
- **Conflicts in a PR:** update from `main` (rebase plus `--force-with-lease`, or merge), regenerate lockfiles, re-run tests.
- GitHub Actions runs CI/CD from `.github/workflows/*.yml`; its jobs become the required checks that gate merges.

## 1. The pull request workflow

1. Pick up the ticket and confirm the acceptance criteria.
2. `git switch main && git pull`, then `git switch -c feature/123-split-bills`.
3. Commit in small logical steps and push early (`git push -u origin HEAD`).
4. Open a **draft PR** early, so people can see the direction and comment before you polish.
5. Review your own diff on GitHub: you will spot debug code, leftover TODOs and missing tests.
6. Mark it ready, request reviewers (or let CODEOWNERS do it) and make sure CI is green.
7. Address feedback with new commits so reviewers see exactly what changed, reply to each comment and re-request review.
8. Merge with the agreed strategy once approved and green, and delete the branch.
9. Deploy (or let continuous deployment do it), verify in production through logs and the error tracker, then close the ticket.

```bash
gh pr create --draft --fill           # title and body from your commits
gh pr view --web                      # open the PR in the browser
gh pr checks                          # CI status for the current branch's PR
gh pr checkout 123                    # check out a teammate's PR locally to test it
gh pr review 123 --approve            # or --request-changes --body "..."
gh pr merge 123 --squash --delete-branch
```

## 2. Writing a good PR description

An illustrative example (teams usually store a template in `.github/pull_request_template.md`):

```text
## What
Add percentage-based splitting to the bills API and the resident app.

## Why
Some houses split utilities by room size instead of equally. Closes #123.

## How
- New split_mode column (equal or percentage), defaulting to equal
- Validation that shares add up to 100
- The API returns per-resident amounts; the app shows them on the bill screen

## How to test
1. Create a bill with split_mode=percentage and shares of 50, 30 and 20
2. Check each resident's amount in the app

## Screenshots
Before and after of the bill screen

## Risk and rollback
Additive migration, hidden behind the percentage_splits flag. Disable the flag or revert to roll back.
```

The reviewer should understand the change without asking you anything. Call out the parts where you want careful eyes (a migration, a permission check) and anything you deliberately left out.

## 3. Merge options on GitHub

| Option | What lands on `main` | Pros | Cons |
|---|---|---|---|
| Create a merge commit | every branch commit plus a merge commit | full history; easy to see what landed together | WIP commits in history; non-linear graph |
| Squash and merge | one commit, with the PR title and description as its message | clean history, one commit per PR, easy revert | granular commits are lost; continuing work on the merged branch causes conflicts |
| Rebase and merge | each branch commit replayed linearly | linear and granular | every commit must be clean and buildable; new hashes |

Repository settings decide which buttons exist, and many teams allow only squash. The Git mechanics behind each option are in [Git](01-git.md).

## 4. Code review etiquette

**As the author:**
- Keep PRs small and focused, with refactors and renames in separate PRs from behavior changes.
- Self-review, remove debug code and get CI green before asking for review.
- Give context: what, why, how to test, screenshots, trade-offs considered, and where you want feedback.
- Reply to every comment (fix it, or explain and discuss), and treat feedback as being about the code, not you.
- Push follow-ups as new commits so the reviewer can see what changed since their last pass.

**As the reviewer:**
- Review promptly, ideally within a working day: a blocked PR blocks a person.
- Understand the goal (ticket and description) first, then read the tests, then the code.
- Prioritize correctness, security, data integrity, design and tests; leave formatting to linters and formatters.
- Ask questions ("What happens if the bill has no residents?"), explain why, offer alternatives with GitHub suggestion blocks, label severity, and say what you liked.
- Approve when it is good enough rather than perfect; larger ideas can become follow-up tickets.
- Pull the branch and test locally when the change is risky: migrations, payments, authentication.

```text
blocking: This query is not scoped to the current organization, so any member could read other
          organizations' bills. Could we go through current_org.bills here?
question: Do existing bills need a backfill for split_mode, or does the column default cover them?
nit: amount_cents would match the column name better than amount.
```

Labels such as `blocking:`, `question:`, `suggestion:` and `nit:` (the Conventional Comments style) make intent explicit. When you disagree with feedback, discuss it with reasons or data, propose an alternative, move long threads to a short call, defer to team conventions, and involve the tech lead if you are stuck. Never merge around a reviewer.

## 5. Branching strategies

| | GitHub flow | GitFlow | Trunk-based development |
|---|---|---|---|
| Branches | `main` plus short-lived feature branches | `main`, `develop`, `feature/*`, `release/*`, `hotfix/*` | `main` (the trunk) plus branches that live hours to a day |
| Releases | deploy after every merge to `main` | release branches cut from `develop`, merged to `main` and tagged | continuous; incomplete work hidden behind feature flags |
| Strengths | simple; fits continuous deployment | structured versioned releases, several supported versions | minimal merge pain, fast feedback, real continuous integration |
| Weaknesses | needs good CI and a deployable `main` | heavy process, long-lived branches, merge overhead | needs strong tests, feature flags and discipline |
| Fits | most web apps and APIs | packaged software, products with scheduled releases | high-velocity teams with mature CI/CD |

- Agencies often use **environment branches** (`staging`, `production`) that mirror deployments. They work, but they drift; promoting the same build or tag through environments is more reliable.
- Mobile apps add a twist: store review delays releases, so teams usually tag (or branch) each store version and use EAS Update channels per environment ([EAS Build, Submit and Update](../08-react-native-expo/08-eas-build-submit-update.md)).
- **Feature flags** (Flipper in Rails, LaunchDarkly, Unleash, or simple database flags) decouple deploying code from releasing a feature; remove each flag after rollout ([CI/CD and deployment](04-ci-cd-and-deployment.md)).

## 6. Protected branches, rulesets and merge queues

Branch protection rules, and the newer **rulesets** (which can target many branches and tags and be layered at the organization level), typically enforce on `main`:
- pull requests with a minimum number of approvals, and dismissal of stale approvals after new commits;
- required status checks (the CI jobs), optionally with the branch up to date with `main`;
- review from code owners, and all conversations resolved;
- linear history (squash or rebase merges only) and, if desired, signed commits;
- no force pushes or deletion, with a short list of people who may bypass.

When many PRs must each be up to date with `main`, a **merge queue** tests every PR combined with the PRs ahead of it before merging, which keeps `main` green without endless "update branch" cycles.

## 7. CODEOWNERS

```text
# .github/CODEOWNERS (the last matching pattern wins)
*                       @acme/core-reviewers
/app/javascript/        @acme/frontend
/app/models/billing/    @acme/payments @lead-dev
/db/migrate/            @acme/backend-leads
/.github/workflows/     @acme/devops
```

GitHub automatically requests reviews from the owners of the paths a PR touches, and with "Require review from Code Owners" those approvals become mandatory. The file can live in `.github/`, the repository root or `docs/`. Prefer teams over individuals and keep it current, because stale owners block merges.

## 8. Issues, labels and project boards

- **Issue templates** (`.github/ISSUE_TEMPLATE/`) standardize bug reports (steps to reproduce, expected vs actual, environment, screenshots) and feature requests.
- **Labels** (`bug`, `enhancement`, `priority:high`, `area:billing`), **milestones** for releases or sprints, and **assignees** organize the backlog.
- **Closing keywords** in a PR description or commit message (`Closes #123`, `Fixes #123`, `Resolves #123`) close the issue when the PR merges into the default branch; a plain `#123` only cross-references.
- **GitHub Projects** gives boards and tables with custom fields (status, sprint, estimate) and automations such as moving items to Done on merge.
- Many client teams use Jira or Linear instead; put the ticket key in the branch name and PR title so the integrations link everything.

## 9. Releases and tags

- A GitHub **Release** is a tag plus notes and optional assets. Notes can be generated automatically from the PRs merged since the previous release, grouped by label through `.github/release.yml`.
- **Semantic versioning** (MAJOR.MINOR.PATCH): breaking change, new feature, bug fix.
- Tools such as release-please and changesets open release PRs from Conventional Commits or changeset files and create the tag when the release PR merges.
- Continuously deployed web apps can tag deploys for traceability; mobile apps tag each store build (version and build number) so every binary and OTA update maps back to a commit.

## 10. Dependabot

Dependabot has three parts: **alerts** for known vulnerabilities in your dependency graph, **security updates** (PRs that bump a vulnerable package to a patched version), and **version updates** (scheduled PRs that keep dependencies current), configured like this:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "bundler"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      minor-and-patch:
        update-types: ["minor", "patch"]
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "monthly"
```

- Read the changelog, let CI run, merge grouped minor and patch updates regularly, and review major versions by hand with local testing.
- Framework upgrades (Rails, React Native, the Expo SDK) need coordinated work, not one-line bumps. For Expo, upgrade the SDK as a unit (`npx expo install --fix` aligns package versions with the SDK) instead of merging individual package PRs.
- Auto-merge is reasonable for low-risk development dependencies when CI is strong. Renovate is a popular, more configurable alternative.

## 11. Working in client repositories

- **Access:** you are usually added to the client's GitHub organization, often with mandatory 2FA or SSO. Never fork private client code into a personal account.
- **Conventions:** read the README and contributing notes, and follow their branch naming, commit style, PR template, review rules and CI requirements even where they differ from your company's habits.
- **Process:** never push to protected branches, link every PR to a ticket in their tracker, and keep commit messages professional: they become the client's permanent history.
- **Time zones:** European clients are a few hours behind Pakistan, so write PRs that can be reviewed asynchronously, with enough context that nobody needs a call to understand them.
- **Confidentiality:** client code, data and credentials stay within approved tools and machines: no public gists, no pasting code into forums, production data only as permitted, secrets only through the client's secret management.
- **Offboarding:** at the end of an engagement, leave branches merged or documented and expect access to be removed.

## 12. Keeping PRs small

- **Why:** review quality drops sharply on large diffs, while small PRs merge faster, conflict less and are easy to revert.
- **Target:** a few hundred changed lines and one concern per PR.
- **Techniques:**
  - Land preparatory refactors and renames as their own PRs first.
  - Split by layer in a safe order (migration, then model and API, then UI) using expand/contract so each step deploys safely.
  - Stack PRs, each based on the previous one (`git rebase --update-refs` keeps a stack aligned).
  - Merge unfinished UI behind a feature flag, and use draft PRs to agree on direction before polishing.
- Generated files (lockfiles, schema dumps, snapshots) inflate the line count; mention them in the description so reviewers can skip them.

## 13. Resolving conflicts in PRs

When GitHub reports "This branch has conflicts that must be resolved":
1. **Locally (preferred):** `git fetch origin`, then either `git rebase origin/main` (resolve per commit, then `git push --force-with-lease`) or `git merge origin/main` (resolve once, then a normal push). Follow the team's preference, and never rebase a branch someone else is also pushing to.
2. **In GitHub's web editor** for small text conflicts; it creates a merge commit from `main` into your branch.

Regenerate lockfiles and `db/schema.rb` instead of hand-editing them, and run the full test suite after resolving. If the conflict involves someone else's logic, ask them rather than guessing. Prevent conflicts with small PRs, frequent updates from `main` on longer branches, and a quick message when two people are about to change the same hot files.

## 14. Security features and GitHub Actions in brief

- **Security:** 2FA, secret scanning and **push protection** (blocks pushes that contain recognized secrets), Dependabot, code scanning with CodeQL, least-privilege access through teams and roles, and fine-grained personal access tokens with expiry dates.
- **GitHub Actions:** workflows in `.github/workflows/*.yml` run on events such as `pull_request` and `push`, and their jobs become the status checks that branch protection requires. Pipelines, caching, secrets and deployments are covered in [CI/CD and deployment](04-ci-cd-and-deployment.md).

## Common pitfalls
- Giant PRs that sit unreviewed for days.
- Empty descriptions: no context, no testing steps, no screenshots for UI changes.
- Reviews that only discuss style, or rubber-stamp approvals.
- Merging with failing or skipped checks, or bypassing branch protection "just this once".
- Long-lived feature branches and drifting environment branches.
- Continuing to commit on a branch after it was squash-merged.
- Ignoring Dependabot until dozens of PRs pile up and a forced major upgrade arrives.
- Stale CODEOWNERS entries that block merges.
- Client code or secrets ending up in personal accounts, gists or public tools.

## How to talk about this in an interview
- Describe your real PR routine at Devntech in concrete terms: branch naming, what goes in the description, how you test before requesting review, and how review works with a European client a few hours behind.
- Show reviewer maturity: you look for unscoped queries, N+1s, missing tests, unsafe migrations and permission gaps, and you phrase comments as questions with reasons.
- Show judgment about strategies: GitHub flow for most web apps, trunk-based with feature flags as a goal for mature teams, GitFlow for release-driven products, and release tags for mobile store builds.

> **Fill in:** the review and merge rules in your Devntech client repositories (required approvals, CI checks, squash or merge), and how quickly PRs usually got reviewed.

> **Fill in:** one piece of review feedback you gave or received that changed a design, and how the discussion went.

## Interview questions

**Q1. Walk me through your pull request workflow.**
Sync `main`, create a descriptively named branch, commit in small logical steps, push and open a draft PR early, self-review the diff, fill in the description (what, why, how to test, screenshots, risks), get CI green, request review, address every comment with new commits, merge with the agreed strategy, delete the branch, then verify the deploy in logs and the error tracker.
*Follow-up:* Why open drafts early? Feedback on direction before you invest in polish, and visibility for the team.

**Q2. What makes a good PR description?**
It lets a reviewer understand and verify the change without asking: what changed and why, the approach and alternatives considered, step-by-step testing instructions, screenshots or recordings for UI, risks (migrations, permissions) and the rollback plan, plus the linked ticket.
*Follow-up:* Where do teams standardize it? A `.github/pull_request_template.md` file.

**Q3. Merge commit vs squash vs rebase-and-merge: which do you prefer?**
Squash for most feature PRs: one clean commit per PR that is easy to revert, with the PR description as its message. Rebase-and-merge when the branch's commits are deliberately crafted and each builds. Merge commits when preserving the exact branch history matters, such as long-lived release branches. Above all, follow the repository's convention.
*Follow-up:* Gotcha with squash? Continuing work on the same branch afterwards conflicts, because `main` does not contain the original commits.

**Q4. How do you give a good code review? What do you look for?**
Read the ticket and description first, then the tests, then the code. Prioritize correctness, security (authorization, scoping, injection), data integrity (migrations, transactions), design and test coverage; leave formatting to linters. Write specific, kind comments with reasons, label severity, and approve when the change is good enough.
*Follow-up:* What is a common bug you catch in Rails PRs? Unscoped `find` calls (IDOR) and N+1 queries in new views or serializers.

**Q5. A reviewer requests changes you disagree with. What do you do?**
Assume good intent and make sure I understand their concern, then explain my reasoning with evidence (a benchmark, a doc, a past incident) and suggest alternatives. If the thread drags on, a short call usually settles it; if not, follow the team's convention or ask the tech lead to decide. I never merge around an unresolved review.
*Follow-up:* What if they were right? Say so, make the change, and remember the lesson.

**Q6. GitHub flow vs GitFlow vs trunk-based development?**
GitHub flow keeps `main` deployable with short-lived branches merged via PRs and deployed continuously. GitFlow adds `develop`, release and hotfix branches for scheduled, versioned releases. Trunk-based development merges very small changes into `main` daily, hiding unfinished work behind feature flags. Choose by release cadence, team maturity and CI strength.
*Follow-up:* What do mobile teams add? Release tags or branches per store version, because store review and slow user updates delay releases.

**Q7. What are protected branches and CODEOWNERS for?**
Protection rules (or rulesets) stop direct pushes and force pushes to important branches and require PRs with approvals and passing checks before merging. CODEOWNERS maps paths to people or teams who are auto-requested for review and, when required, must approve changes to those paths.
*Follow-up:* What does a merge queue add? It tests each PR combined with those ahead of it, so `main` stays green under heavy merge traffic.

**Q8. How do you keep PRs small when a feature is big?**
Split it into safe, independently deployable steps: preparatory refactors first, then migrations (expand/contract), then backend logic, then UI behind a feature flag. Stack dependent PRs when needed and agree on direction early with a draft. Each PR should be reviewable in one sitting.
*Follow-up:* Why expand/contract? Every step stays compatible with the code running before and after it, so each PR can deploy on its own.

**Q9. Your PR has merge conflicts with main. What do you do?**
Fetch, then rebase onto `origin/main` and push with `--force-with-lease`, or merge `origin/main` into the branch, depending on team convention. Resolve each conflict by understanding both sides, regenerate lockfiles and schema files, run the tests, and ask the other author when their logic is involved.
*Follow-up:* Why not always use the web editor? It only suits small text conflicts and you cannot run tests there.

**Q10. How do you handle Dependabot PRs?**
Treat them as small, regular maintenance: read the changelog, rely on CI, group minor and patch updates, merge them promptly, and review majors manually with local testing. Coordinate framework upgrades (Rails, Expo SDK) as planned work rather than isolated bumps, and prioritize security updates by severity and exposure.
*Follow-up:* Why not ignore them? Updates pile up into risky big-bang upgrades, and known vulnerabilities stay in production.

**Q11. What is different about working in a client's repository?**
Their conventions override yours (branching, commit style, PR template, CI rules); access goes through their organization with 2FA or SSO; their code, data and credentials stay confidential and inside approved tools; every change links to their tracker; and PRs must be clear enough for asynchronous review across time zones.
*Follow-up:* What is a red flag? Copying client code into personal repositories or public tools, or committing their secrets.

**Q12. How do issues, labels and closing keywords work together?**
Issues capture bugs and features with templates; labels and milestones categorize and schedule them; Projects boards track status. Writing `Closes #123` in a PR description links the PR and closes the issue automatically when the PR merges into the default branch.
*Follow-up:* What if the team uses Jira? Put the ticket key in the branch name and PR title so the integration links them.

## Further reading
- [GitHub Docs: Pull requests](https://docs.github.com/en/pull-requests)
- [GitHub Docs: About code owners](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
- [GitHub Docs: Dependabot](https://docs.github.com/en/code-security/dependabot)
