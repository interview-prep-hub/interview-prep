# CI/CD and Deployment

> **Tier:** T2 (working knowledge) · **Related:** [Git](01-git.md), [GitHub and workflows](02-github-and-workflows.md), [Docker](03-docker.md), [Observability and debugging](05-observability-and-debugging.md), [Testing with RSpec](../05-ruby-on-rails/13-testing-rspec.md), [Platform migration strategies](../13-architecture-and-practices/04-platform-migration-strategies.md), [EAS Build, Submit and Update](../08-react-native-expo/08-eas-build-submit-update.md)
> **On your resume:** Rails and TypeScript apps at Devntech (the Colochousing web app and API, CrystalCommerce), EAS releases of the Coloc app, Sidekiq background jobs, and Git, GitHub and Docker in your skills.

## Quick revision
- **CI:** every push is built and tested automatically and changes merge to `main` often. **Continuous delivery:** every green `main` is releasable and deploying is a button. **Continuous deployment:** every green `main` deploys automatically.
- Pipeline: cached install, lint and static analysis, type check, tests, security scans, build an artifact or image, deploy to staging, smoke test, deploy to production, monitor.
- GitHub Actions: workflows (`.github/workflows/*.yml`) run on events; jobs run in parallel unless linked with `needs`; `services` start Postgres and Redis containers; `strategy.matrix` fans out; caches speed up installs.
- Rails CI: Postgres and Redis services, `ruby/setup-ruby` with `bundler-cache: true`, schema load, RSpec, RuboCop, Brakeman, bundler-audit. Node CI: `actions/setup-node` with `cache: npm`, `npm ci`, lint, `tsc --noEmit`, tests, build.
- Secrets live in repository or environment secrets, are masked in logs and are withheld from fork PRs; environments add required reviewers; OIDC replaces long-lived cloud keys.
- Targets: Heroku, Render, Fly.io, AWS (EC2, ECS, RDS, S3, CloudFront), Vercel, Kamal for Rails on your own servers, EAS for mobile.
- Zero downtime: **rolling** (replace instances gradually), **blue-green** (switch traffic between two environments), **canary** (a small share first). All need health checks, draining and graceful shutdown.
- Migrations run while old code is still live: **expand/contract** (add, dual-write, backfill, switch reads, remove later); index concurrently; never rename or drop in one step.
- Rollback = redeploy the previous release; **feature flags** switch features off without a deploy; backward-compatible migrations keep rollbacks safe.
- Linux essentials: `ssh`, `chmod`/`chown`, `ps`/`top`/`kill`, `systemctl`, `journalctl`, `tail`/`grep`, environment variables, `cron`, `curl`, `ss`.
- **Nginx** in front of Puma or Node: TLS termination, static files, buffering, compression, WebSocket upgrades and `X-Forwarded-*` headers. TLS comes from Let's Encrypt (certbot with auto-renewal) or the platform.
- **Twelve-factor:** config in the environment, stateless processes, backing services as attached resources, logs to stdout, disposability, dev/prod parity, separate build, release and run.

## 1. CI vs continuous delivery vs continuous deployment

| | Continuous integration | Continuous delivery | Continuous deployment |
|---|---|---|---|
| Promise | every change is integrated and verified automatically | `main` is always in a releasable state | every verified change reaches production |
| Automation stops at | build and tests | a production-ready artifact; a human presses deploy | nothing: production deploys automatically |
| Requires | fast, reliable tests and small merges | automated deploys to staging, release discipline | strong tests, monitoring, feature flags, fast rollback |

Mobile apps are a special case: store review means binaries ship through delivery, while JavaScript-only fixes can deploy continuously with EAS Update ([EAS Build, Submit and Update](../08-react-native-expo/08-eas-build-submit-update.md)).

## 2. Anatomy of a pipeline

Install dependencies with caching; run the fast checks first (RuboCop or ESLint, formatting, `tsc --noEmit`); run unit and integration tests against real Postgres and Redis services, plus system or end-to-end tests where they earn their cost; scan for vulnerabilities (Brakeman, bundler-audit or `npm audit`, secret scanning); build one artifact (a Docker image tagged with the commit SHA, compiled assets or a mobile binary); deploy it to staging and smoke test; deploy the same artifact to production, automatically or after approval, with migrations run safely; then verify health checks, error rates and latency by release, and roll back if they spike ([Observability and debugging](05-observability-and-debugging.md)).

## 3. GitHub Actions concepts

- **Workflow:** a YAML file in `.github/workflows/`, triggered by events such as `push`, `pull_request`, `workflow_dispatch` (manual) or `schedule` (cron).
- **Jobs** run on fresh runners (`runs-on: ubuntu-latest`) in parallel unless `needs:` orders them; **steps** run shell commands (`run:`) or reusable **actions** (`uses: actions/checkout@v4`).
- **Services** start containers (Postgres, Redis) next to a job; **matrix** strategies run a job once per combination (Node versions, test shards).
- **Caching** reuses dependencies between runs, **artifacts** pass files between jobs and **concurrency** groups cancel superseded runs; **secrets, variables and environments** scope configuration, and `permissions:` limits the built-in `GITHUB_TOKEN`.
- Pin actions to a major version (or a commit SHA for third-party actions) and check each action's README for its current major version.

## 4. A Rails workflow

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true              # a newer push cancels the outdated run

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true           # bundle install, cached on Gemfile.lock; Ruby from .ruby-version
      - run: bundle exec rubocop
      - run: bundle exec brakeman --no-pager
      - run: bundle exec bundler-audit check --update

  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
        options: >-
          --health-cmd "pg_isready -U postgres"
          --health-interval 10s --health-timeout 5s --health-retries 5
      redis:
        image: redis:7
        ports: ["6379:6379"]
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s --health-timeout 5s --health-retries 5
    env:
      RAILS_ENV: test
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/app_test
      REDIS_URL: redis://localhost:6379/0
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - run: bin/rails db:create db:schema:load
      - run: bundle exec rspec
```

Rails 7.2+ apps are generated with a similar workflow (using `bin/rubocop` and `bin/brakeman` binstubs), and bundler-audit must be in the Gemfile for the audit step. Apps that build JavaScript (jsbundling, React on Rails) also need `actions/setup-node` and `npm ci` before the tests, and system tests need a browser, which GitHub's Ubuntu runners provide.

## 5. A Node workflow with a matrix

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  check:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [22, 24]          # the LTS lines you support
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: npm                    # npm cache keyed on package-lock.json
      - run: npm ci
      - run: npm run lint
      - run: npm run typecheck          # for example "tsc --noEmit"
      - run: npm test
      - run: npm run build
```

Add a `postgres` service, as in the Rails workflow, for database-backed integration tests. pnpm projects add `pnpm/action-setup` and `cache: pnpm`. A matrix can also shard a slow suite (`shard: [1, 2, 3, 4]` passed to the runner's shard flag, such as `vitest --shard=1/4`).

## 6. Speeding up CI

- Cache dependencies (`bundler-cache`, the `setup-node` cache, `actions/cache` keyed on `hashFiles('**/package-lock.json')`) and Docker layers (the `gha` cache backend).
- Run independent jobs in parallel and run cheap checks first so failures surface in seconds.
- Split slow test suites across matrix shards (the `parallel_tests` gem or Knapsack for RSpec, built-in sharding in Vitest and Jest).
- Cancel superseded runs with `concurrency`, and use path filters so documentation-only changes skip heavy jobs (mind your required checks). Fix the tests themselves: fewer system tests, stubbed external HTTP, lean factories, and quarantine for flaky tests until they are fixed.

## 7. Environments, secrets and deploy jobs

- **Secrets** (repository, organization or environment level) are referenced as `${{ secrets.NAME }}`, masked in logs, and not passed to workflows triggered by pull requests from forks; use `vars` for non-secret configuration.
- **Environments** such as `staging` and `production` scope secrets and add protection rules: required reviewers, wait timers and branch restrictions. A required approval on `production` is continuous delivery in practice.
- **Least privilege:** declare `permissions:` explicitly, pin third-party actions, and avoid running untrusted pull request code with `pull_request_target`. **OIDC:** instead of storing long-lived cloud keys, a job with `permissions: id-token: write` exchanges a short-lived token for a cloud role (for example with `aws-actions/configure-aws-credentials`).

```yaml
jobs:
  # lint and test jobs as above
  deploy:
    needs: [lint, test]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    environment: production             # protection rules can require an approval here
    steps:
      - uses: actions/checkout@v4
      - run: ./bin/deploy               # your deploy command: kamal deploy, fly deploy or a platform CLI
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

## 8. Deployment targets

| Target | Model | Good for | Notes |
|---|---|---|---|
| Heroku | PaaS: git push or container deploys, dynos, add-ons | fast setup for Rails and Node, small teams | a Procfile defines `web` and `worker`; the `release:` phase runs migrations; cost grows with scale |
| Render | PaaS: web services, workers, cron jobs, managed Postgres and Redis | Heroku-like simplicity | `render.yaml` blueprints describe infrastructure as code |
| Fly.io | containers on Firecracker micro-VMs ("Machines") in many regions | Docker-based apps close to users | `fly.toml`, `fly deploy`, a `release_command` for migrations |
| AWS | building blocks: EC2 (VMs), ECS or Fargate (containers), RDS (managed Postgres), ElastiCache (Redis), S3 (files), CloudFront (CDN), load balancers, IAM | full control and scale; many clients already use it | more operations work, usually with infrastructure as code (Terraform, CDK) |
| Vercel | serverless and edge platform built around Next.js | Next.js and frontends, preview deployments per PR | functions have execution limits, so long jobs and WebSockets live elsewhere |
| Kamal | a deploy tool: Docker containers on your own servers over SSH, with kamal-proxy for zero downtime | the Rails 8 default; affordable VPS or bare metal | you own the servers: patching, backups, monitoring |

## 9. Zero-downtime deployments

| Strategy | How it works | Pros | Cons |
|---|---|---|---|
| Rolling | replace instances a few at a time behind the load balancer | no extra capacity; the default on most platforms | old and new versions serve traffic together; slower rollback |
| Blue-green | start the new version (green) beside the old (blue), then switch all traffic | instant switch and instant rollback | double capacity during deploys; the shared database must suit both versions |
| Canary | route a small share of traffic (say 5%) to the new version, watch metrics, then ramp up | small blast radius, validated on real traffic | needs traffic splitting and good monitoring |

Every strategy depends on the same basics:
- **Health checks:** route traffic only to instances that pass a readiness check (Rails 7.1+ ships `/up`).
- **Draining and graceful shutdown:** on `SIGTERM`, stop accepting new work and finish what is in flight. Puma does this; Sidekiq stops fetching and waits (25 seconds by default) before pushing unfinished jobs back to Redis; a Node server calls `server.close()`.
- **Backward compatibility:** during a rollout, old and new code share the database, queues and caches, so new workers must handle old job payloads and the schema must work for both versions. Mobile clients on old app versions keep calling the API long after a deploy ([REST API design](../04-web-fundamentals/04-rest-api-design.md)).

## 10. Database migrations during deploys

Renaming `users.name` to `full_name` without downtime, using **expand/contract**:
1. **Expand:** add a nullable `full_name` column and deploy.
2. **Dual write:** the code writes both columns; deploy.
3. **Backfill** existing rows in throttled batches (a background job or data migration).
4. **Switch reads** to `full_name`; deploy and verify.
5. **Stop using** `name`: remove it from code and add `self.ignored_columns += ["name"]`; deploy.
6. **Contract:** drop the column in a later deploy.

- Run migrations before new code serves traffic (Heroku's release phase, Fly's `release_command`, a Kamal hook or the entrypoint's `db:prepare`), and make sure the old code still works against the new schema.
- Avoid long locks on big tables: create indexes concurrently, add constraints in steps (`NOT VALID`, then `VALIDATE CONSTRAINT`), batch backfills, and set `lock_timeout` and `statement_timeout`. PostgreSQL 11+ adds a column with a constant default without rewriting the table.
- Never put a destructive change (drop, rename, type change) in the same deploy as the code that stops using the column. The strong_migrations gem flags unsafe migrations in Rails. More in [Platform migration strategies](../13-architecture-and-practices/04-platform-migration-strategies.md) and [PostgreSQL](../12-databases/04-postgresql.md).

```ruby
class AddIndexToBillsOnDueOn < ActiveRecord::Migration[7.1]
  disable_ddl_transaction!                               # CREATE INDEX CONCURRENTLY cannot run inside a transaction

  def change
    add_index :bills, :due_on, algorithm: :concurrently  # avoids blocking writes on a large table
  end
end
```

## 11. Rollbacks and feature flags

- **Code rollback** means redeploying the previous release or image: `heroku releases:rollback`, `kamal rollback <version>`, re-running the previous pipeline, a previous ECS task definition, or Vercel's instant rollback.
- **Database rollback** is the hard part: down migrations can destroy data, so prefer rolling forward with a fix and keep migrations backward compatible, so that rolling back code never requires rolling back the schema.
- **Decide fast:** if errors spike right after a deploy (the error tracker groups errors by release), roll back first and debug afterwards.
- **Feature flags** decouple deploying from releasing: ship code dark, enable it for internal users, one client organization or a percentage, and switch it off instantly without a deploy. Tools: Flipper in Rails, LaunchDarkly, Unleash, PostHog, or a simple flags table. Remove flags after full rollout to avoid flag debt.

```ruby
Flipper.enable_actor(:percentage_splits, current_org)        # one organization first (actors need a flipper_id)
Flipper.enable_percentage_of_actors(:percentage_splits, 25)  # then a quarter of actors
Flipper.enabled?(:percentage_splits, current_org)            # the check in code; Flipper.disable is the kill switch
```

- **Mobile:** a store binary cannot be rolled back. EAS Update can publish a JavaScript fix (or republish an earlier update) to builds with the same runtime version, and server-side flags can hide a broken feature.

## 12. Linux and CLI essentials

```bash
ssh deploy@203.0.113.10                      # log in with your key (ssh-keygen -t ed25519 creates one)
chmod 600 ~/.ssh/id_ed25519                  # private keys readable only by you
chmod +x bin/deploy                          # make a script executable (755 = rwxr-xr-x)
sudo chown -R deploy:deploy /var/www/app     # change owner and group recursively
ps aux | grep puma                           # find processes
top                                          # live CPU and memory (htop if installed)
kill -TERM 12345                             # ask a process to shut down gracefully
kill -9 12345                                # force kill, no cleanup: last resort
sudo systemctl status puma                   # service state and recent log lines
sudo systemctl restart puma                  # also start, stop, enable (start at boot)
journalctl -u puma -f --since "10 min ago"   # follow a service's logs
tail -f log/production.log | grep -i error   # follow an app log, filtered
df -h && free -m                             # disk space and memory
ss -tlnp                                     # listening TCP ports and their processes (or lsof -i :3000)
export RAILS_ENV=production                  # environment variable for this shell session
printenv | grep DATABASE                     # inspect environment variables
crontab -e                                   # schedule jobs: "*/15 * * * * /path/script" runs every 15 minutes
curl -I http://localhost:3000/up             # check the app from the server itself
```

- Permission digits add read (4), write (2) and execute (1) for user, group and others: 755 for directories and scripts, 644 for files, 600 for secrets. `chmod 777` is never the fix.
- A systemd unit describes a service (`ExecStart`, `User`, `WorkingDirectory`, `Restart=always`, `EnvironmentFile`); run `systemctl daemon-reload` after editing one.
- System cron on several servers runs each job once per server; schedule app jobs with sidekiq-cron or Solid Queue recurring tasks instead. More background in [Operating systems](../01-cs-fundamentals/05-operating-systems.md).

## 13. Nginx in front of Puma or Node

```text
upstream app {
  server 127.0.0.1:3000;                 # Puma or Node (a Unix socket also works on the same host)
  keepalive 32;
}

server {
  listen 80;
  server_name app.example.com;
  return 301 https://$host$request_uri;  # force HTTPS
}

server {
  listen 443 ssl;
  http2 on;                              # nginx 1.25.1+; older versions use "listen 443 ssl http2;"
  server_name app.example.com;
  ssl_certificate     /etc/letsencrypt/live/app.example.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/app.example.com/privkey.pem;

  root /var/www/app/public;
  client_max_body_size 20m;

  location /assets/ {                    # fingerprinted files served straight from disk
    add_header Cache-Control "public, max-age=31536000, immutable";
    try_files $uri =404;
  }

  location /cable {                      # WebSocket upgrade for Action Cable
    proxy_pass http://app;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
  }

  location / {                           # everything else goes to the app
    proxy_pass http://app;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 60s;
  }
}
```

Nginx terminates TLS, redirects HTTP to HTTPS, serves precompiled assets with long cache headers, buffers slow clients so app threads are not tied up, caps request bodies, upgrades WebSocket connections, and forwards the client IP and scheme. Without `X-Forwarded-Proto`, Rails cannot tell the request was HTTPS, and `force_ssl` causes redirect loops (Rails 7.1+ also offers `config.assume_ssl` when TLS ends at a proxy). Validate with `nginx -t` and apply with `systemctl reload nginx`, which keeps existing connections. In container deployments, Thruster or kamal-proxy take over some of these duties.

## 14. TLS with Let's Encrypt

```bash
sudo certbot --nginx -d app.example.com    # obtain a certificate and update the Nginx config
sudo certbot renew --dry-run               # test renewal; a systemd timer or cron job renews for real
```

Let's Encrypt issues free, short-lived certificates through the ACME protocol (90 days by default, and it has announced moves toward shorter lifetimes), so automated renewal is mandatory and expiry should still be monitored. Alternatives are Caddy (automatic HTTPS), kamal-proxy (automatic Let's Encrypt certificates) and platform-managed TLS (Heroku, Render, Fly, Vercel, AWS Certificate Manager on load balancers and CloudFront). Handshakes and certificate chains are covered in [Networking essentials](../01-cs-fundamentals/06-networking.md).

## 15. The twelve-factor app

| Factor | Meaning | In a Rails or Node app |
|---|---|---|
| I. Codebase | one repository per app, many deploys | the same repo deploys to staging and production |
| II. Dependencies | declare and isolate them | `Gemfile.lock`, `package-lock.json`, Docker images |
| III. Config | store config in the environment | `DATABASE_URL`, `REDIS_URL` and API keys as env vars, not in code |
| IV. Backing services | treat them as attached resources | swap the Postgres or Redis URL without code changes |
| V. Build, release, run | keep the stages separate | build an image, combine it with config into a release, run it; never edit code on servers |
| VI. Processes | stateless, share-nothing | sessions and uploads in Redis, the database or S3, not on local disk |
| VII. Port binding | the app serves itself on a port | Puma or Node listens on `$PORT` |
| VIII. Concurrency | scale out through process types | `web` and `worker` (Sidekiq) processes, scaled independently |
| IX. Disposability | fast startup, graceful shutdown | handle `SIGTERM`; idempotent jobs survive restarts |
| X. Dev/prod parity | keep environments similar | the same Postgres and Redis versions locally (Docker) as in production |
| XI. Logs | treat logs as event streams | write to stdout and let the platform collect them |
| XII. Admin processes | run one-off tasks in the same environment | `rails db:migrate`, consoles and rake tasks use the release's code and config |

## Common pitfalls
- Slow or flaky CI that people learn to ignore, and merging red builds.
- Secrets echoed in logs or hard-coded in workflow files; long-lived cloud keys in CI.
- Migrations that drop or rename columns while old code is still running.
- No readiness checks, so traffic reaches instances that are still booting; no graceful shutdown, so requests and jobs die mid-flight.
- No tested rollback path, down migrations that destroy data, or environment configuration drifting between staging and production.
- App cron jobs scheduled on every server.
- `kill -9` as a first resort and `chmod 777` as a permissions "fix".
- Missing `X-Forwarded-Proto` (HTTPS redirect loops), or certificate renewals failing silently until the site breaks.

## How to talk about this in an interview
- Explain how code reached production on your Devntech projects: which checks ran on PRs, who reviewed, where the apps deployed and how migrations ran. If you did not own the pipeline, say what you used and understood, then describe the pipeline you would build.
- EAS releases of the Coloc app are your mobile delivery story: store builds through review plus over-the-air JavaScript updates matched by runtime version.
- The multi-tenant platform migration and legacy Rails work are natural places for expand/contract migrations, feature flags and backward-compatible APIs, and Sidekiq makes graceful shutdown concrete: what happens to running jobs during a deploy, and why jobs must be idempotent.

> **Fill in:** the CI checks and deploy target for Colochousing and CrystalCommerce (GitHub Actions or another CI, Heroku, AWS, Kamal or other), and your role in the release process.

> **Fill in:** a deploy that went wrong or a migration you made safe, and what you changed afterwards.

## Interview questions

**Q1. CI vs continuous delivery vs continuous deployment?**
Continuous integration builds and tests every change automatically and keeps merges small and frequent. Continuous delivery keeps `main` always releasable, with production deploys a manual, low-risk step. Continuous deployment removes the manual step: every change that passes the pipeline goes to production.
*Follow-up:* What does continuous deployment demand? Strong automated tests, monitoring, feature flags and fast rollback.

**Q2. Describe the CI/CD pipeline you would set up for a Rails app.**
On every PR: cached `bundle install`, RuboCop, Brakeman and bundler-audit in a lint job, and RSpec against Postgres and Redis services in a test job, all as required checks. On merge to `main`: build an image tagged with the SHA, deploy to staging, smoke test, then deploy to production (automatically or with an approval), running migrations in a release step and watching errors by release.
*Follow-up:* How do migrations fit? They run before the new code serves traffic and follow expand/contract so old code keeps working.

**Q3. How do you manage secrets in CI/CD?**
Store them as repository or environment secrets, never in workflow files; rely on log masking but never echo them; scope production secrets to a protected environment with required reviewers; keep `GITHUB_TOKEN` permissions minimal; and use OIDC federation for cloud access instead of long-lived keys.
*Follow-up:* Why are fork PRs special? Their workflows do not receive secrets, so untrusted code cannot read them.

**Q4. How do you deploy with zero downtime?**
Run several instances behind a load balancer that routes only to instances passing readiness checks, replace them gradually (rolling) or switch between environments (blue-green), and shut old processes down gracefully on `SIGTERM` so in-flight requests and jobs finish. Keep schema and payload changes backward compatible, because old and new code overlap during the rollout.
*Follow-up:* What does Kamal use? kamal-proxy, which starts the new container, waits for its health check and then switches traffic.

**Q5. Rolling vs blue-green vs canary?**
Rolling replaces instances in batches without extra capacity, but versions mix during the rollout. Blue-green runs a full new environment and flips traffic at once, giving instant rollback at the cost of double capacity. Canary sends a small share of traffic to the new version and ramps up while metrics stay healthy, limiting the blast radius.
*Follow-up:* What do they all share? Health checks, graceful shutdown and backward-compatible database changes.

**Q6. How do you run database migrations safely during deploys?**
Use expand/contract: add new structures first, deploy code that writes both and reads the old, backfill in batches, switch reads, stop using the old structure (ignored columns in Rails), and remove it in a later deploy. Create indexes concurrently, add constraints in validated steps, set lock timeouts, and run migrations before the new code serves traffic.
*Follow-up:* How do you rename a column? Never directly: add the new column, dual-write, backfill, switch, then drop the old one later.

**Q7. How do you roll back a bad deploy, including migrations?**
Redeploy the previous release or image (`heroku releases:rollback`, `kamal rollback`, the previous task definition) as soon as errors correlate with the deploy. Because migrations are backward compatible, the old code runs fine on the new schema, so the schema stays and you roll forward with a fix. Feature flags can disable a broken feature without any deploy.
*Follow-up:* Why avoid down migrations in production? They can destroy data written since the deploy.

**Q8. What are feature flags, and why use them?**
Runtime switches that turn code paths on or off per user, organization or percentage without deploying. They decouple deploy from release, enable gradual rollouts and beta access, act as kill switches during incidents, and let unfinished work merge into `main` safely. Clean them up after rollout.
*Follow-up:* Which tool in Rails? Flipper, with actor and percentage gates.

**Q9. What does Nginx do in front of Puma or Node?**
It terminates TLS, redirects HTTP to HTTPS, serves static and precompiled assets directly with cache headers, buffers slow clients so app threads stay free, limits request sizes, compresses responses, proxies WebSocket upgrades and forwards `X-Forwarded-For` and `X-Forwarded-Proto` so the app knows the real client and scheme.
*Follow-up:* Symptom of a missing `X-Forwarded-Proto`? Redirect loops with `force_ssl`, because the app thinks every request is plain HTTP.

**Q10. What is the twelve-factor app? Pick a few factors.**
A set of principles for cloud-friendly apps. Config lives in environment variables, not code; processes are stateless, with sessions and files in backing services; logs go to stdout as event streams; processes start fast and shut down gracefully; and build, release and run are separate stages.
*Follow-up:* How does Sidekiq fit? It is a separate worker process type (concurrency), scaled independently of web processes.

**Q11. A Rails app on a Linux server is down. Which commands do you run first?**
`systemctl status puma` (and the Sidekiq service) to see whether it is running; `journalctl -u puma -n 200` and `tail -n 200 log/production.log` for errors; `curl -I http://localhost:3000/up` to test the app directly, bypassing Nginx; `df -h` and `free -m` for a full disk or memory exhaustion; `ss -tlnp` to confirm what listens on the port; and the Nginx error log for upstream errors.
*Follow-up:* Which signal do you send to restart gracefully? `SIGTERM` (or `systemctl restart`), not `kill -9`.

## Further reading
- [GitHub Actions documentation](https://docs.github.com/en/actions)
- [The Twelve-Factor App](https://12factor.net/)
- [Kamal documentation](https://kamal-deploy.org/docs/)
- [Nginx documentation](https://nginx.org/en/docs/) and [Let's Encrypt documentation](https://letsencrypt.org/docs/)
