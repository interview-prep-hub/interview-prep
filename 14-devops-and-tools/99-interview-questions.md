# DevOps and Tools: Rapid-Fire Interview Questions

> How to use: cover the answer, say yours out loud, then check. Deeper explanations live in the topic files linked under each section.

## Git (see [Git](01-git.md))

**Q1. What object types does git store?**
Blobs (file contents), trees (directories: names pointing to blobs and subtrees), commits (a tree snapshot plus parents, author, committer and message) and annotated tags. Every object is stored under the hash of its content (SHA-1 by default; SHA-256 repositories also exist), so identical content is stored once and history is tamper-evident.

**Q2. What are branches and HEAD, really?**
A branch is a ref, a tiny file such as `refs/heads/main` holding a commit hash, that moves forward when you commit. HEAD points at the current branch, or directly at a commit in "detached HEAD" state. Creating a branch only writes a small file, which is why branches are cheap.

**Q3. Working tree vs staging area vs repository?**
The working tree is your files on disk; the index (staging area) is the snapshot you are building for the next commit with `git add`; the repository holds committed history. `git diff` compares the working tree with the index, and `git diff --staged` compares the index with the last commit.

**Q4. Does git store diffs or snapshots?**
Snapshots: every commit points to a complete tree. Unchanged files reuse the same blob, and packfiles delta-compress similar objects on disk, but conceptually each commit is a full snapshot.

**Q5. `git fetch` vs `git pull`?**
`fetch` downloads new commits and updates remote-tracking branches such as `origin/main` without touching your work. `pull` is `fetch` followed by a merge (or a rebase with `--rebase` or the `pull.rebase` setting) into the current branch. Fetching first lets you look before you integrate.

**Q6. Merge vs rebase?**
Merge joins branches with a merge commit that has two parents: history is preserved exactly as it happened, but busy repositories get a tangled graph. Rebase replays your commits on top of the target branch as new commits with new hashes: the history is linear and readable, but it is rewritten, conflicts are resolved commit by commit, and you must never rebase commits others have already pulled. A common workflow: rebase your private feature branch onto `main` to stay current, then merge it through a pull request (often squashed).

**Q7. What is a fast-forward merge?**
If the target branch has no new commits since you branched, git simply moves its pointer to your latest commit, with no merge commit. `--no-ff` forces a merge commit to keep the feature grouped; `--ff-only` refuses anything but a fast-forward.

**Q8. What is the golden rule of rebasing?**
Never rebase commits that exist outside your repository and that others may have built on: rewriting shared history leaves everyone reconciling duplicate commits. Rebasing your own local branch, or a pushed branch only you use, is fine; push it afterwards with `--force-with-lease`.

**Q9. `--force` vs `--force-with-lease`?**
`--force` overwrites the remote branch no matter what it contains, possibly destroying a teammate's commits. `--force-with-lease` overwrites only if the remote branch still points where your last fetch saw it, so it fails instead of clobbering new work.

**Q10. What can interactive rebase do?**
`git rebase -i HEAD~5` opens a todo list where you reorder, `squash` or `fixup` (combine), `reword` (edit a message), `edit` (stop to amend) or `drop` commits; it is how you tidy a feature branch before review. `git commit --fixup <sha>` plus `git rebase -i --autosquash` automates the fixups.

**Q11. Reset vs revert?**
`git revert <sha>` creates a new commit that undoes an earlier commit's changes: safe on shared branches because history is kept. `git reset <target>` moves the current branch pointer back, removing later commits from the branch: fine for local work, destructive on shared branches. Rule of thumb: revert what is pushed, reset what is local.

**Q12. `reset --soft` vs `--mixed` vs `--hard`?**
All three move the branch pointer. `--soft` keeps the undone changes staged; `--mixed` (the default) keeps them in the working tree, unstaged; `--hard` discards them, including uncommitted work. `git reset --soft HEAD~1` is the classic "undo my last commit but keep the changes".

**Q13. What is the reflog, and how does it save you?**
A local log of every position HEAD and each branch has pointed to, including before resets, rebases and amends. Find the lost commit with `git reflog`, then restore it with `git reset --hard <sha>` or `git branch rescue <sha>`. Entries expire eventually (by default after 90 days, or 30 for unreachable commits) and garbage collection then removes the objects.

**Q14. How does `git bisect` work?**
It binary-searches history for the commit that introduced a bug: `git bisect start`, mark a bad commit (`git bisect bad`) and a known good one (`git bisect good v1.4`), then test each commit git checks out and mark it good or bad until it names the culprit, in about log2(n) steps. `git bisect run <test command>` automates the testing and `git bisect reset` returns you to where you started.

**Q15. What is cherry-pick for?**
`git cherry-pick <sha>` applies one commit's changes as a new commit on the current branch: backporting a hotfix to a release branch, or rescuing a commit from an abandoned branch. Overusing it duplicates commits across branches.

**Q16. How do you undo the most common mistakes?**
Unstage a file: `git restore --staged file`. Discard local edits: `git restore file`. Fix the last commit's message or add a forgotten file: `git commit --amend` (only if not pushed). Undo the last local commit: `git reset --soft HEAD~1`. Undo a pushed commit: `git revert <sha>`. Find lost commits: `git reflog`.

**Q17. What does `git stash` do?**
It shelves uncommitted changes on a stack and cleans the working tree so you can switch tasks: `git stash push -m "wip"`, `git stash list`, then `git stash pop` (apply and drop) or `git stash apply` (apply and keep). Add `-u` to include untracked files.

**Q18. What is detached HEAD?**
HEAD points directly at a commit instead of a branch, for example after checking out a tag or a sha, or during a rebase or bisect. Commits made there belong to no branch and become unreachable when you switch away, so create a branch (`git switch -c fix`) to keep them.

**Q19. How do you resolve a merge conflict?**
Open each conflicted file, understand both sides between the `<<<<<<<`, `=======` and `>>>>>>>` markers, edit to the correct result (often a combination), run the tests, `git add` the files and continue (`git commit`, `git merge --continue` or `git rebase --continue`). `git merge --abort` or `git rebase --abort` backs out; talk to the other author when the intent is unclear.

**Q20. How do you stop tracking a file that is already committed, such as `.env`?**
Add it to `.gitignore`, then run `git rm --cached .env` and commit: the file stays on disk but leaves the index. `.gitignore` alone does not affect files that are already tracked. If the file held secrets, rotate them, because they remain in history.

**Q21. You pushed a secret to GitHub. What now?**
Revoke and rotate the secret immediately: that is the real fix. Then, if needed, remove it from history (`git filter-repo` or BFG, followed by a force push and asking collaborators to re-clone), check for abuse, and add secret scanning and pre-commit checks so it does not happen again.

**Q22. Lightweight vs annotated tags?**
A lightweight tag is just a name for a commit. An annotated tag is a full object with tagger, date, message and an optional signature, so use annotated tags for releases: `git tag -a v1.2.0 -m "Release 1.2.0"`. Tags are not pushed by default: `git push origin v1.2.0`, or `--tags` for all.

**Q23. `git switch` and `git restore` vs `git checkout`?**
`checkout` historically did two unrelated jobs: switching branches and restoring files. Git 2.23 split them into `switch` (branches) and `restore` (files), which are clearer and harder to misuse; `checkout` still works.

**Q24. What are git hooks?**
Scripts git runs at certain events: `pre-commit` (lint and format staged files), `commit-msg` (enforce a message format), `pre-push` (run tests). They live in `.git/hooks`, which is not versioned, so teams share them through tools such as Husky, lefthook or overcommit. CI must still enforce the same checks, because hooks can be skipped locally.

**Q25. How do you find out who changed a line and why?**
`git blame file` shows the last commit for each line; `git log -S "price_cents"` (the pickaxe) finds commits that added or removed a string; `git log -L 10,20:app/models/order.rb` traces the history of a line range. Then read the pull request linked to the commit for the reasoning.

**Q26. What is `git worktree` for?**
It checks out another branch into a second directory backed by the same repository, so you can review a pull request or build a hotfix without stashing your current work: `git worktree add ../hotfix hotfix-branch`.

## GitHub workflows and code review (see [GitHub and workflows](02-github-and-workflows.md))

**Q27. Describe a typical pull request workflow.**
Branch from an up-to-date `main`, commit in small logical steps, push and open a pull request (a draft early if you want feedback) with context, screenshots and testing notes. CI runs and reviewers comment; you address feedback with new commits. Once it is approved and green, merge it (often squashed) and delete the branch; the merge triggers a deploy to staging or production.

**Q28. Merge commit vs squash merge vs rebase merge on GitHub?**
A merge commit keeps every commit plus a merge commit: full history, noisier graph. Squash merge turns the pull request into one commit on `main`: clean history and easy reverts, but individual commits are lost. Rebase merge replays the commits linearly with no merge commit. Many teams squash by default.

**Q29. What are branch protection rules?**
Repository settings that guard important branches: require pull requests and a number of approvals, require passing status checks, require resolved conversations, block force pushes and deletion, and optionally require linear history or signed commits. They turn team agreements into enforced rules; GitHub's rulesets offer the same controls in a more flexible form.

**Q30. What is a CODEOWNERS file?**
`.github/CODEOWNERS` maps paths to owners, for example `/app/billing/ @org/payments-team`; GitHub automatically requests their review on pull requests touching those paths, and branch protection can require their approval.

**Q31. Gitflow vs GitHub flow vs trunk-based development?**
Gitflow has long-lived `develop` and `main` branches plus feature, release and hotfix branches; it suits scheduled, versioned releases but is heavy. GitHub flow uses short-lived branches off `main`, pull requests, and a deploy after each merge. Trunk-based development has everyone integrating into `main` at least daily, with feature flags hiding unfinished work; it needs strong CI and enables continuous delivery.

**Q32. What makes a good commit message?**
A short imperative subject ("Add retry to price sync job", around 50 characters), a blank line, then a body explaining why the change was made and any trade-offs. Conventional Commits prefixes (`feat:`, `fix:`, `chore:`) add machine-readable types for changelogs and version bumps.

**Q33. What is semantic versioning?**
`MAJOR.MINOR.PATCH`: a major bump for breaking changes, minor for backward-compatible features, patch for backward-compatible fixes. Lockfiles pin exact versions; ranges decide what updates are allowed: `^1.4.0` in npm allows minor and patch updates below 2.0, and `~> 7.1.0` in a Gemfile allows only patch updates (at least 7.1.0, below 7.2).

**Q34. What do you look for when reviewing a pull request?**
Whether it does what the ticket asks, correctly, including edge cases and error paths; security (authorization, injection, secrets); data safety (migrations, backfills); performance (N+1 queries, missing indexes); tests that would fail without the change; readability and naming; and consistency with the codebase. Run risky changes locally.

**Q35. How do you respond to review feedback you disagree with?**
Assume good intent, ask what problem the reviewer sees, explain your reasoning with evidence (a benchmark, the docs, a failing test) and offer alternatives. If it is a matter of taste, follow the team convention or the reviewer's preference and move on; take long debates to a call instead of a 30-comment thread.

**Q36. What is a GitHub Actions workflow made of?**
YAML files in `.github/workflows/`: triggers (`on: push`, `pull_request`, `schedule`, `workflow_dispatch`), jobs that run on runners (`runs-on: ubuntu-latest`) in parallel unless they declare `needs`, and steps that either run shell commands (`run:`) or use reusable actions (`uses: actions/checkout@v4`). Secrets come from repository or environment settings as `${{ secrets.NAME }}`, and `strategy.matrix` runs a job across several versions. Pin third-party actions to a version or commit sha.

**Q37. Show a minimal CI workflow for a Rails app.**
```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
    env:
      RAILS_ENV: test
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/app_test
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true # runs bundle install and caches the gems
      - run: bin/rails db:prepare
      - run: bundle exec rspec
```
The service container gives each run a throwaway database, and the health check makes the job wait until Postgres accepts connections.

**Q38. How do you keep CI fast?**
Cache dependencies (`bundler-cache`, or `actions/setup-node` with `cache: npm`), run lint, type checks and tests as parallel jobs, split the test suite across machines, skip expensive jobs when unrelated paths change, keep Docker layers cache-friendly, and fix slow tests instead of buying bigger runners.

**Q39. How do you manage dependency updates?**
Let Dependabot or Renovate open pull requests for new versions and security advisories, let CI test them, and merge small updates often instead of doing big painful upgrades. `bundle audit`, `npm audit` and GitHub's dependency alerts flag known vulnerabilities.

## Docker and Compose (see [Docker](03-docker.md))

**Q40. Docker image vs container?**
An image is a read-only, layered template (a filesystem plus metadata such as the default command), built from a Dockerfile and identified by a digest. A container is a running (or stopped) instance of an image with its own writable layer, process, network and resource limits. One image can start many containers, like a class and its objects.

**Q41. Container vs virtual machine?**
A VM virtualizes hardware and runs a full guest operating system with its own kernel: strong isolation, but heavy (gigabytes, slow boots). Containers share the host's kernel and isolate processes with namespaces and cgroups: lightweight (megabytes, start in seconds) but with weaker isolation. On macOS and Windows, Docker Desktop runs containers inside a Linux VM.

**Q42. What do the common Dockerfile instructions do?**
`FROM` picks the base image; `WORKDIR` sets the working directory; `COPY` adds files from the build context (`ADD` can also fetch URLs and unpack archives, so prefer `COPY`); `RUN` executes a command at build time and creates a layer; `ENV` sets environment variables for build and runtime; `ARG` defines build-time variables; `EXPOSE` documents a port without publishing it; `USER` switches to a non-root user; `ENTRYPOINT` and `CMD` define what runs when a container starts.

**Q43. CMD vs ENTRYPOINT?**
`ENTRYPOINT` is the executable that always runs; `CMD` supplies default arguments to it (or the whole default command when there is no entrypoint) and is replaced by anything you pass to `docker run image ...`. A common pattern is `ENTRYPOINT ["./bin/docker-entrypoint"]`, a script that prepares the database and then runs `exec "$@"`, plus `CMD ["./bin/rails", "server"]`. Use the exec (JSON array) form so your process runs as PID 1 and receives SIGTERM; the shell form wraps it in `/bin/sh -c`, which does not forward signals.

**Q44. How does layer caching work, and how should you order a Dockerfile?**
Each instruction produces a layer cached by its inputs; once a layer changes, every later layer is rebuilt. Copy dependency manifests and install them before copying the source: `COPY Gemfile Gemfile.lock ./`, then `RUN bundle install`, then `COPY . .`, so editing code does not reinstall gems. The same applies to `package.json` and `npm ci`.

**Q45. What are multi-stage builds?**
Several `FROM` stages in one Dockerfile: a build stage with compilers and development dependencies compiles assets or binaries, and the final stage copies only the results (`COPY --from=build ...`) onto a slim base. Images become smaller and safer because build tools never ship. The Dockerfile that Rails 7.1+ generates uses this approach.

**Q46. How do you make images smaller and safer?**
Slim or distroless base images, multi-stage builds, a `.dockerignore` (exclude `.git`, `node_modules`, logs and `.env`), install and clean up in the same `RUN` so caches never persist in a layer, production dependencies only, a non-root `USER`, pinned base image versions, and vulnerability scanning in CI.

**Q47. Why should secrets never be baked into an image?**
Anyone who can pull the image can read every layer, and `ARG` and `ENV` values show up in the image metadata and history. Provide secrets at runtime (environment variables from the orchestrator, mounted secret files), and use BuildKit secret mounts (`RUN --mount=type=secret,id=...`) when a build step needs one.

**Q48. Volumes vs bind mounts?**
A named volume is storage managed by Docker (`pgdata:/var/lib/postgresql/data`) that survives container removal: use it for databases. A bind mount maps a host directory into the container (`.:/rails`): use it in development so code changes appear immediately. Anything written only to the container's writable layer disappears when the container is removed.

**Q49. How do containers talk to each other in Compose?**
Compose creates a network per project on which every service is reachable by its service name, so the app connects to `postgres://db:5432`, not `localhost`. Inside a container, `localhost` means that container itself, which is also why servers must listen on `0.0.0.0` rather than `127.0.0.1` to be reachable from outside.

**Q50. `ports` vs `expose` in Compose?**
`ports: ["3000:3000"]` publishes a container port on the host (`HOST:CONTAINER`) so your browser can reach it. `expose` only documents ports for other services on the same network, which can reach each other anyway. Do not publish database ports in production.

**Q51. Does `depends_on` wait until the database is ready?**
Not by default: it only orders container startup. Add a `healthcheck` to the database service and use the long form with `condition: service_healthy`; the app should still retry its connection at boot.

**Q52. Show a small Compose file for Rails with Postgres and Redis.**
```yaml
services:
  web:
    build: .
    command: bin/rails server -b 0.0.0.0
    ports: ["3000:3000"]
    environment:
      DATABASE_URL: postgres://postgres:postgres@db:5432/app_development
      REDIS_URL: redis://redis:6379/0
    volumes: [".:/rails"]
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: postgres
    volumes: ["pgdata:/var/lib/postgresql/data"]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 10
  redis:
    image: redis:7
volumes:
  pgdata:
```
There is no top-level `version:` key: current Compose ignores it. A `worker` service running Sidekiq would reuse the same build with a different `command`.

**Q53. `docker run` vs `docker exec`, and which commands help debugging?**
`docker run` creates and starts a new container from an image; `docker exec -it <container> sh` runs a command inside a container that is already running (`bash` if the image has it). For debugging: `docker ps -a`, `docker logs -f <container>`, `docker inspect`, `docker stats`, and `docker compose logs -f web`.

**Q54. Why does PID 1 matter inside a container?**
The process with PID 1 receives the stop signal and must reap zombie child processes. A shell-form `CMD` makes `/bin/sh` PID 1, which does not forward SIGTERM, so `docker stop` waits out its timeout and then kills the app with no graceful shutdown. Use the exec form, `exec` in entrypoint scripts, or `--init` (tini).

**Q55. Why avoid the `latest` tag?**
It is only a default name, not "the newest stable version", and it moves, so the same Dockerfile builds different images over time and rollbacks become guesswork. Pin versions (`ruby:3.3-slim`), deploy immutable tags such as the git sha, and pin digests when you need full reproducibility.

## CI/CD and deployment (see [CI/CD and deployment](04-ci-cd-and-deployment.md))

**Q56. Continuous integration vs continuous delivery vs continuous deployment?**
Continuous integration: everyone merges small changes often, and every push is built and tested automatically. Continuous delivery: every green build on `main` is releasable, and a person triggers the production deploy. Continuous deployment: every green build goes to production automatically.

**Q57. What stages does a typical pipeline have?**
Install dependencies (cached), lint and format checks, type checks, unit and integration tests in parallel, security scans (Brakeman, dependency audits), build an artifact (a Docker image tagged with the git sha), deploy to staging and smoke test, then deploy to production after an approval or automatically, followed by monitoring.

**Q58. Rolling vs blue-green vs canary deployment?**
Rolling: replace instances a few at a time; cheap, but old and new versions serve traffic side by side during the rollout. Blue-green: bring up a complete new environment (green), switch all traffic at once, and keep blue for an instant rollback; it needs double resources during the switch. Canary: send a small percentage of traffic to the new version, watch the metrics, then increase it; problems hit only a few users.

**Q59. How do you achieve zero-downtime deploys?**
Start new instances and route traffic to them only after their health checks pass, drain old instances (stop sending new requests and let in-flight ones finish on SIGTERM), and make every change compatible with the version running beside it: backward-compatible database migrations (expand and contract), assets from both versions available, job arguments understood by old and new workers, and additive API changes for old mobile clients. Puma phased restarts, Kamal's proxy and Kubernetes rolling updates with readiness probes all implement the traffic part.

**Q60. How do you run database migrations without downtime?**
Split risky changes into safe steps: add a nullable column, deploy code that writes it, backfill in batches, then add constraints. Remove a column in two deploys: first stop using it (and add it to `ignored_columns`), then drop it. Create Postgres indexes concurrently (`algorithm: :concurrently` with `disable_ddl_transaction!`), avoid long locks on large tables, and let a gem like strong_migrations flag dangerous operations.

**Q61. Should migrations run before or after the new code is deployed?**
Usually before (as a release phase or pre-deploy step), which is safe only when the old code still works with the new schema, so the change must be additive. Destructive changes such as drops and renames ship in a later deploy, once no running code uses the old shape.

**Q62. How do you roll back a bad deploy?**
Redeploy the previous image or release (immutable artifacts make this fast) or switch off the feature flag. Database rollbacks are risky because data written by the new version can be lost, which is why schema changes stay backward compatible: a rollback moves code, not data.

**Q63. What are feature flags good for?**
Decoupling deploy from release: ship code dark, enable it per user, tenant or percentage of traffic, run betas, keep kill switches, and turn a feature off instantly without a deploy. The costs are extra branches in code and stale flags, so remove each flag after rollout. Flipper is a common choice in Rails.

**Q64. How do you handle configuration and secrets across environments?**
Twelve-factor style: configuration in environment variables, not in code, with separate values for development, staging and production; secrets come from the platform's secret store, Rails encrypted credentials (with `RAILS_MASTER_KEY` provided at runtime) or CI secrets. Validate required variables at boot so a missing one fails fast.

**Q65. What does Kamal do?**
Kamal, the default deployment tool in Rails 8, deploys Docker containers to your own servers over SSH: it builds and pushes the image, pulls it on each host and swaps containers behind kamal-proxy for zero-downtime deploys, with "accessories" for services such as databases and Redis. It gives PaaS-like deploys on plain virtual machines.

**Q66. What is a health check endpoint, and what is the difference between liveness and readiness?**
An endpoint that load balancers and orchestrators call to decide whether to send traffic to an instance or restart it. Liveness asks "is the process alive?" (restart it if not); readiness asks "can it serve traffic right now?" (booted, warmed up, dependencies reachable). Rails 7.1+ generates `/up`, which returns 200 when the app boots without errors.

## Linux basics (see [CI/CD and deployment](04-ci-cd-and-deployment.md) and [Operating systems](../01-cs-fundamentals/05-operating-systems.md))

**Q67. How do Linux file permissions work?**
Each file has an owner, a group, and read, write and execute bits for the owner, the group and everyone else. `chmod 755 script.sh` gives `rwxr-xr-x` (owner everything, others read and execute), `chmod 600 key.pem` makes a file private to its owner, and `chown deploy:deploy file` changes ownership. Execute permission on a directory means you can enter it.

**Q68. How do you find and stop a process?**
`ps aux | grep puma`, `pgrep -f sidekiq`, or `top` and `htop` for live CPU and memory. `kill <pid>` sends SIGTERM (graceful) and `kill -9 <pid>` sends SIGKILL (last resort). `lsof -i :3000` or `ss -ltnp` shows which process listens on a port.

**Q69. How do you check disk, memory and CPU on a server?**
`df -h` for disk usage per filesystem, `du -sh * | sort -h` to find what is big in a directory, `free -m` for memory (including cache and swap), `top`, `htop` and `uptime` for CPU and load average, and `vmstat` or `iostat` for I/O. A disk filled by unrotated logs is a classic outage.

**Q70. How do you read logs on a Linux server?**
`tail -f log/production.log`, `less +F`, or `grep -i error file | tail -50`; for systemd services, `journalctl -u puma -f` and `systemctl status puma`. Combine `awk`, `sort` and `uniq -c` to count things, such as status codes in an Nginx access log.

**Q71. What do `>`, `>>`, `2>&1` and `|` do?**
`>` redirects standard output to a file (overwriting it), `>>` appends, `2>&1` sends standard error wherever standard output goes, and `|` pipes one command's output into the next command. `cmd > out.log 2>&1` captures everything; `cmd 2>/dev/null` discards errors.

**Q72. How do you manage a service with systemd?**
A unit file such as `/etc/systemd/system/puma.service` defines the command, user, working directory, environment and restart policy. Then `systemctl start`, `stop`, `restart` and `status`, `systemctl enable puma` to start it at boot, `journalctl -u puma` for its logs, and `systemctl daemon-reload` after editing a unit.

**Q73. How does SSH key authentication work?**
You generate a key pair (`ssh-keygen -t ed25519`) and put the public key in the server's `~/.ssh/authorized_keys`; at login the client proves it holds the private key, which never leaves your machine. Disable password logins, protect the private key with a passphrase, and use `ssh-agent` so you type it once.

**Q74. How do you read a cron expression?**
Five fields: minute, hour, day of month, month and day of week. `0 2 * * *` runs daily at 02:00, `*/15 * * * *` every 15 minutes, and `0 9 * * 1` on Mondays at 09:00. Cron runs with a minimal environment in the server's time zone, so use absolute paths, log the output, and prefer your app's scheduler for application jobs.

**Q75. What do exit codes mean?**
0 means success and anything else means failure: 1 is a general error, 127 means command not found, 137 means killed by SIGKILL (often the out-of-memory killer) and 143 means terminated by SIGTERM. CI fails a step on a non-zero exit, and `set -e` makes a bash script stop at the first failing command.

## Nginx and Puma (see [CI/CD and deployment](04-ci-cd-and-deployment.md) and [Networking](../01-cs-fundamentals/06-networking.md))

**Q76. Why put Nginx in front of Puma?**
Nginx terminates TLS, serves static files efficiently, buffers slow clients so Puma threads are not tied up with uploads and downloads, compresses responses, enforces limits (body size, rate limiting) and balances load across app servers, leaving Puma to run Ruby. Rails 8 also ships Thruster, a small proxy that runs in front of Puma, for setups without Nginx.

**Q77. Show a minimal Nginx reverse proxy configuration for a Rails app.**
```text
upstream app {
  server 127.0.0.1:3000;
}

server {
  listen 443 ssl;
  server_name shop.example.com;
  ssl_certificate     /etc/ssl/shop.crt;
  ssl_certificate_key /etc/ssl/shop.key;
  client_max_body_size 20m;

  location /assets/ {
    root /var/www/shop/public;
    expires max;               # fingerprinted files never change
  }

  location / {
    proxy_pass http://app;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 60s;
  }
}
```
Assets are served straight from disk, and everything else goes to Puma with the original host, client IP and scheme in headers.

**Q78. What do 502, 504 and 413 errors from Nginx usually mean?**
502 Bad Gateway: Nginx got no valid response from the upstream (Puma is down, crashed, or listening on a different socket or port). 504 Gateway Timeout: the upstream did not answer within `proxy_read_timeout` (a slow query or external call). 413 Request Entity Too Large: an upload exceeded `client_max_body_size`.

**Q79. Why do `X-Forwarded-For` and `X-Forwarded-Proto` matter?**
Behind a proxy, the app sees the proxy's IP address and plain HTTP. These headers carry the client's IP and the original scheme, so Rails builds correct `https://` URLs, `force_ssl` does not redirect forever, secure cookies work, and logs and rate limits use the real client IP. Trust them only from known proxies (Rails `trusted_proxies`), because clients can forge them.

**Q80. Puma workers vs threads?**
Workers are forked processes (clustered mode, set with `WEB_CONCURRENCY`) that give real parallelism across CPU cores despite the GVL, at the cost of memory per process. Threads (`RAILS_MAX_THREADS`, commonly 3 to 5) serve concurrent requests inside each worker, which helps while other requests wait on I/O. A typical start is one worker per core with a few threads each, then tune by measuring memory and latency.

**Q81. How should the database pool relate to Puma threads?**
Each thread can hold a database connection, so `pool` in `database.yml` must be at least the number of threads per process (the generated config reads `RAILS_MAX_THREADS`), and the total (workers times threads, plus Sidekiq concurrency, plus consoles and cron) must stay below the database's `max_connections`, or you add a pooler such as PgBouncer.

**Q82. Puma hot restart vs phased restart?**
A hot restart (`SIGUSR2`) restarts the whole server in place: new code loads, and requests queue briefly. A phased restart (`SIGUSR1`, clustered mode only) replaces workers one at a time, so the app keeps serving throughout; it cannot be combined with `preload_app!`, and because the master process is not replaced, upgrading Puma itself needs a hot restart.

**Q83. What does `preload_app!` do, and what is the trade-off?**
It loads the application in the master process before forking workers, so the workers share memory pages through copy-on-write and boot faster. The trade-offs: phased restarts are no longer possible, and connections opened in the master must be re-established in each worker through Puma's worker boot hook (Rails handles Active Record connections itself).

**Q84. How do you load balance across several app servers?**
Put an L7 load balancer or an Nginx `upstream` with several servers in front: round robin by default, `least_conn` when request durations vary a lot, `ip_hash` or cookies for stickiness (avoid it by keeping sessions in cookies or Redis). Health checks remove dead nodes, and every node must be stateless: uploads go to object storage, and sessions and caches go to shared stores.

## Observability and production debugging (see [Observability and debugging](05-observability-and-debugging.md))

**Q85. Logs vs metrics vs traces?**
Logs are timestamped records of discrete events with rich context (an error with its request id and parameters): best for "what exactly happened". Metrics are numeric time series aggregated cheaply (request rate, p95 latency, queue size): best for dashboards, trends and alerts. Traces follow one request across services and components as timed spans (controller, SQL, HTTP calls): best for "where did the time go". You alert on metrics, investigate with traces, and confirm details in logs.

**Q86. What is structured logging, and why use it?**
Writing logs as key-value data, usually one JSON object per line (`{"level":"error","request_id":"...","user_id":42,"msg":"payment failed"}`), instead of free text, so the log platform can filter, group and graph them. Put a request or correlation id on every line (Rails `config.log_tags = [:request_id]`) and pass it into jobs and outgoing calls.

**Q87. What should you never log?**
Passwords, tokens, API keys, full card numbers, session cookies and unnecessary personal data. Rails `filter_parameters` masks parameters such as `password` and `token`; also scrub headers, error-tracker payloads and job arguments. Logs are kept for a long time and read by many people.

**Q88. Counter vs gauge vs histogram?**
A counter only goes up (requests served, errors), and you graph its rate. A gauge goes up and down (memory used, queue depth, connections in use). A histogram records a distribution in buckets (request durations), from which you compute percentiles.

**Q89. Why use percentiles instead of averages for latency?**
Averages hide the slow tail: a 120 ms average can coexist with 5% of requests taking 3 seconds. p50 describes the typical request; p95 and p99 describe the unlucky users, who are often your heaviest customers. Alert on p95 or p99 for critical endpoints.

**Q90. What are the four golden signals, RED and USE?**
The golden signals: latency, traffic, errors and saturation. RED, for request-driven services: rate, errors and duration. USE, for resources such as CPUs, disks and connection pools: utilization, saturation and errors. They are checklists for building dashboards and alerts.

**Q91. SLI vs SLO vs SLA, and what is an error budget?**
An SLI is a measured indicator (the percentage of requests that succeed in under 500 ms). An SLO is the internal target for it (99.9% over 30 days). An SLA is a contractual promise with penalties, set looser than the SLO. The error budget is the allowed failure (0.1%, about 43 minutes a month); when it is spent, reliability work takes priority over features.

**Q92. What makes a good alert?**
It fires on user-facing symptoms (error rate, latency, queue latency, failed payments) rather than on every possible cause (CPU at 80%), it is actionable, it links to a runbook and a dashboard, and it rarely fires falsely. Noisy alerts get ignored, and then the real one is missed.

**Q93. How does distributed tracing work?**
Each request gets a trace id, and every unit of work records a span (name, start time, duration, attributes, parent span). The ids travel to downstream services in headers (the W3C `traceparent` header) and into background jobs, so the tracing backend can rebuild the whole tree. OpenTelemetry is the vendor-neutral standard for instrumenting and exporting traces, metrics and logs; tracing usually samples a fraction of requests.

**Q94. Why is metric cardinality a problem?**
Every unique combination of label values creates a separate time series. Labels such as `user_id`, `order_id` or raw URLs multiply the number of series, the cost and the query time. Keep labels bounded (route template, status class, region) and put high-cardinality details in logs and traces.

**Q95. An endpoint became slow after a deploy. How do you investigate?**
Confirm it with metrics (which endpoint, since when, p95 or every request, and whether it lines up with the deploy), open slow traces in the APM to see where the time goes (SQL, external calls, rendering), then check the diff for new queries (N+1 queries or a missing index, confirmed with `EXPLAIN ANALYZE`), new external calls or heavier payloads. Mitigate first by rolling back or turning off the flag, then fix it and add a regression check.

**Q96. Users report 500 errors. What do you do?**
Check the error tracker (Sentry, Honeybadger, Bugsnag or similar) for the exception, stack trace, release and affected users; find the request in the logs by its request id; check whether it started with a deploy, a configuration change or a third-party outage; mitigate (roll back, turn off the flag, hotfix); reproduce it with a failing test; keep stakeholders informed; and write a blameless postmortem for significant incidents.

**Q97. How do you debug a growing Sidekiq queue?**
Check queue latency and which job classes dominate (the Sidekiq Web UI), whether workers are running and busy, error and retry rates, and slow dependencies (a third-party API timing out, database locks). Fixes include scaling workers or concurrency, separating queues, adding timeouts, throttling producers and optimizing the slow job. Also check that a recent deploy did not break the workers or rename a queue.

**Q98. A Ruby process's memory keeps growing. What could it be?**
A true leak (objects retained by globals, class-level caches or unbounded memoization) or, more often in Ruby, memory bloat and fragmentation: large allocations from loading big result sets at once (use `find_each`) or building huge strings. Common mitigations are jemalloc or `MALLOC_ARENA_MAX=2`, batching and streaming, and profiling with memory_profiler or derailed_benchmarks; periodically restarting workers is only a stopgap.

**Q99. What is a blameless postmortem?**
A written review after an incident that covers the impact, a timeline, the root cause and contributing factors, what went well, and action items with owners. "Blameless" means focusing on how the system allowed the failure (missing tests, alerts or guardrails) rather than on who made the mistake, so people report problems honestly.

**Q100. Monitoring vs observability?**
Monitoring watches known failure modes with predefined dashboards and alerts ("is the error rate above 1%?"). Observability is being able to ask new questions of a running system from its telemetry (structured logs, traces, high-cardinality context) to debug problems nobody predicted. You need both: monitoring tells you something is wrong, observability helps you find out why.
