# Docker

> **Tier:** T2 (working knowledge) · **Related:** [CI/CD and deployment](04-ci-cd-and-deployment.md), [Operating systems](../01-cs-fundamentals/05-operating-systems.md), [Rails 7 and 8: what's new](../05-ruby-on-rails/18-rails-7-8-whats-new.md), [PostgreSQL](../12-databases/04-postgresql.md), [Redis](../12-databases/10-redis.md), [Background jobs with Sidekiq](../05-ruby-on-rails/09-background-jobs-sidekiq.md)
> **On your resume:** Docker in your skills list, alongside the Rails, Node.js, PostgreSQL, Redis and Sidekiq work at Devntech.

## Quick revision
- A **container** is an isolated process that shares the host kernel (namespaces isolate it, cgroups limit its resources); a **VM** virtualizes hardware and runs its own kernel. Containers start in seconds and weigh megabytes.
- An **image** is a read-only stack of layers plus metadata, built from a Dockerfile; a **container** is a running (or stopped) instance of an image with a thin writable layer on top.
- Registries (Docker Hub, GitHub Container Registry, Amazon ECR) store images; tags are movable labels, digests (`@sha256:...`) are immutable.
- Each Dockerfile instruction makes a layer, and Docker reuses cached layers up to the first changed step, so copy dependency manifests and install them before copying the app.
- **CMD** is the default command (replaced by `docker run image other-cmd`); **ENTRYPOINT** is the fixed executable, with CMD as its default arguments. Use exec form (`["bin/rails", "server"]`) so the process is PID 1 and receives signals.
- `ARG` exists only at build time, `ENV` persists into containers; neither is safe for secrets. `EXPOSE` documents a port; `-p 3000:3000` publishes it.
- **Multi-stage builds** compile in a heavy build stage and copy only the results into a slim runtime stage; `.dockerignore` keeps `.git`, `node_modules`, logs and `.env` out of the build context.
- **Named volumes** persist Docker-managed data (databases); **bind mounts** map a host folder in (source code during development).
- On a user-defined network (Compose creates one), containers reach each other by service name: `db:5432`, never `localhost`.
- `docker compose` (no `version:` key) runs multi-container apps; `depends_on` with `condition: service_healthy` waits for healthchecks.
- Development containers mount source and carry dev tools; production images are immutable, slim, non-root and configured by environment variables. Rails 7.1+ generates a production Dockerfile, `.dockerignore` and `bin/docker-entrypoint`; Rails 8 adds Kamal 2 and Thruster to deploy it.
- Windows: keep projects inside the WSL 2 filesystem and scripts in LF line endings. ARM laptops vs x86 servers: watch for `exec format error` and build multi-architecture images.

## 1. Containers vs virtual machines

| | Container | Virtual machine |
|---|---|---|
| What is isolated | processes, using kernel namespaces and cgroups | a whole guest operating system on a hypervisor |
| Kernel | shared with the host | its own |
| Size | megabytes | gigabytes |
| Startup | milliseconds to seconds | tens of seconds or more |
| Isolation strength | good, but the kernel is a shared boundary | stronger |
| Typical use | packaging apps with their dependencies, dev environments, CI, services | other operating systems, strong multi-tenant isolation, legacy systems |

Linux containers need a Linux kernel, so Docker Desktop on Windows and macOS runs them inside a lightweight Linux VM (WSL 2 on Windows). In production the two combine: containers usually run inside cloud VMs.

## 2. Images, layers and containers

- An **image** is an ordered stack of read-only filesystem layers plus configuration (default command, environment, exposed ports, user). Layers are content-addressed and shared, so ten images built on `ruby:3.3-slim` store its layers once.
- A **container** is an image plus a thin writable layer plus runtime settings (environment, ports, mounts, network). Its writes vanish when the container is removed, so anything persistent belongs in a volume. Treat containers as disposable: replace them, do not patch them.
- **Tags** (`ruby:3.3-slim`, `myapp:1.4.0`, `latest`) can be moved to other images; a **digest** pins exact content. Avoid `latest` in production, because you cannot tell what it contains; tag your images with a version or git SHA. Registries (Docker Hub, `ghcr.io`, Amazon ECR) store them; `docker push` and `docker pull` move them.

## 3. Dockerfile instructions

| Instruction | Purpose | Notes |
|---|---|---|
| `FROM ruby:3.3-slim AS base` | base image, optionally naming a stage | pin versions; prefer official slim images |
| `WORKDIR /rails` | working directory for later steps | created if missing |
| `COPY Gemfile Gemfile.lock ./` | copy files from the build context | prefer `COPY`; `ADD` also fetches URLs and unpacks archives |
| `RUN bundle install` | run a command at build time, creating a layer | chain related commands with `&&` and clean up in the same step |
| `ENV RAILS_ENV=production` | environment variable baked into image and containers | visible in `docker inspect`, so no secrets |
| `ARG RUBY_VERSION=3.3.6` | build-time variable, set with `--build-arg` | gone at runtime but visible in image history |
| `EXPOSE 3000` | documents the listening port | publishing needs `-p` |
| `USER rails` | user for later steps and for the running container | avoid running as root |
| `ENTRYPOINT ["bin/docker-entrypoint"]` | the fixed executable | often a script that prepares, then runs `exec "$@"` |
| `CMD ["bin/rails", "server"]` | default command, or default arguments for ENTRYPOINT | replaced by arguments to `docker run` |

`HEALTHCHECK CMD curl -f http://localhost:3000/up || exit 1` lets Docker and Compose report whether the app actually responds (Rails 7.1+ ships the `/up` health endpoint).

**CMD vs ENTRYPOINT in practice:**

```text
ENTRYPOINT ["bin/docker-entrypoint"]
CMD ["bin/rails", "server"]

docker run myapp                      runs: bin/docker-entrypoint bin/rails server
docker run myapp bin/rails console    runs: bin/docker-entrypoint bin/rails console   (CMD replaced)
docker run --entrypoint bash myapp    runs: bash                                      (ENTRYPOINT replaced, CMD cleared)
```

**Exec form vs shell form:** `CMD bin/rails server` (shell form) runs through `/bin/sh -c`, so the shell is PID 1 and does not forward `SIGTERM`; `docker stop` then waits its 10-second timeout and kills the app without a graceful shutdown. Exec form runs the process directly as PID 1. Entrypoint scripts should end with `exec "$@"` so the app replaces the shell, and `docker run --init` adds a tiny init process that reaps zombie processes.

## 4. Layer caching

```text
# Slow: every code change invalidates the COPY layer, so gems reinstall on every build
COPY . .
RUN bundle install

# Fast: gems reinstall only when Gemfile or Gemfile.lock change
COPY Gemfile Gemfile.lock ./
RUN bundle install
COPY . .
```

- Docker reuses a cached layer while the instruction and its inputs (the copied files) are unchanged; the first changed step and everything after it rebuild.
- Order steps from least to most frequently changing: base image, system packages, dependency manifests and install, application code, asset build.
- Run `apt-get update && apt-get install -y --no-install-recommends ... && rm -rf /var/lib/apt/lists/*` as one `RUN`: deleting files in a later layer does not shrink earlier layers.
- In CI, share the cache between runs (`docker buildx build` with `--cache-from` and `--cache-to`, for example the GitHub Actions cache backend). **Going deeper:** BuildKit cache mounts keep package caches between builds without storing them in layers: `RUN --mount=type=cache,target=/root/.npm npm ci`.

## 5. Multi-stage builds for Rails and Node

```text
# syntax=docker/dockerfile:1
# Simplified from the shape of the Rails 7.1+ generated Dockerfile; your generated file is the source of truth.
ARG RUBY_VERSION=3.3.6
FROM ruby:$RUBY_VERSION-slim AS base
WORKDIR /rails
ENV RAILS_ENV=production BUNDLE_DEPLOYMENT=1 BUNDLE_PATH=/usr/local/bundle BUNDLE_WITHOUT=development:test

FROM base AS build
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y build-essential git libpq-dev pkg-config && \
    rm -rf /var/lib/apt/lists/*
COPY Gemfile Gemfile.lock ./
RUN bundle install && rm -rf ~/.bundle "${BUNDLE_PATH}"/ruby/*/cache
COPY . .
RUN SECRET_KEY_BASE_DUMMY=1 ./bin/rails assets:precompile

FROM base
RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y curl libpq5 && \
    rm -rf /var/lib/apt/lists/*
COPY --from=build "${BUNDLE_PATH}" "${BUNDLE_PATH}"
COPY --from=build /rails /rails
RUN groupadd --system --gid 1000 rails && \
    useradd rails --uid 1000 --gid 1000 --create-home --shell /bin/bash && \
    chown -R rails:rails db log storage tmp
USER 1000:1000
ENTRYPOINT ["/rails/bin/docker-entrypoint"]
EXPOSE 3000
CMD ["./bin/rails", "server"]
```

- The build stage has compilers and headers (`build-essential`, `libpq-dev`) for native gems and asset compilation; the final stage gets only runtime libraries (`libpq5`) and the compiled results, so the image is smaller and carries less attack surface.
- `SECRET_KEY_BASE_DUMMY=1` lets assets precompile without real credentials, and the entrypoint runs `db:prepare` before the server starts.
- Rails 8 apps run Puma behind Thruster and expose port 80, so compare with your generated file instead of copying this one.

```text
# syntax=docker/dockerfile:1
FROM node:22-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci                                   # exact versions from the lockfile

FROM deps AS build
COPY . .
RUN npm run build && npm prune --omit=dev    # compile TypeScript, then drop dev dependencies

FROM node:22-slim AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY package.json ./
USER node                                    # non-root user provided by the official image
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

## 6. .dockerignore and image size

```text
.git
.github
node_modules
log/*
tmp/*
storage/*
coverage
.env*
!.env.example
```

- A small build context makes builds faster, stops irrelevant files from busting the cache, and keeps secrets such as `.env` out of images. Keep images small with slim official bases, multi-stage builds, package-cache cleanup in the same layer, and no dev dependencies in production. `docker image ls` and `docker history myapp:1.4.0` show where the size comes from.
- Alpine images (musl libc) are tiny but can break or slow native extensions and prebuilt binaries made for glibc; Debian slim is the safer default for Ruby and Node. Distroless images go further for production by removing even the shell.

## 7. CLI essentials

```bash
docker build -t myapp:dev .                              # build from ./Dockerfile
docker run --rm -p 3000:3000 --env-file .env myapp:dev   # run, publish a port, remove on exit
docker run -d --name web -p 3000:3000 myapp:dev          # detached and named
docker ps                                                # running containers (-a includes stopped ones)
docker logs -f --tail 100 web                            # follow the logs
docker exec -it web bash                                 # shell in a running container (sh on minimal images)
docker exec -it web bin/rails console
docker stop web && docker rm web                         # SIGTERM, then SIGKILL after 10 s; then remove
docker images                                            # local images (docker rmi removes one)
docker inspect web                                       # full config: env, mounts, network, IP address
docker system prune                                      # stopped containers, unused networks, dangling images, build cache
docker system prune -a --volumes                         # also unused images and volumes: deletes database data
```

## 8. Volumes vs bind mounts

| | Named volume | Bind mount | tmpfs mount |
|---|---|---|---|
| Example | `-v pg_data:/var/lib/postgresql/data` | `-v "$(pwd)":/rails` | `--tmpfs /tmp` |
| Managed by | Docker (`docker volume ls`) | you: any host path | memory |
| Lifetime | until removed (`docker volume rm`, `compose down -v`) | as long as the host files exist | until the container stops |
| Use for | database and Redis data, dev uploads | live code reloading during development | scratch data that must not touch disk |

Everything in a container's writable layer disappears on `docker rm`, so data that must survive goes in a volume. On Docker Desktop, bind mounts with many small files are slow unless the project lives inside WSL 2. A bind mount also lets the container write files onto your machine, owned by the container's user (often root).

## 9. Networking

- On a user-defined network (Compose creates one per project), Docker's DNS resolves service names, so the app connects to `db:5432` and `redis:6379`.
- Inside a container, `localhost` is that container: a `DATABASE_URL` pointing at `localhost` fails; use the service name. `-p 8080:3000` publishes container port 3000 on host port 8080 (host first). Unpublished ports are reachable only from containers on the same network. Use `127.0.0.1:5432:5432` to keep a development database off your LAN.
- The server inside must listen on `0.0.0.0`, not `127.0.0.1` (`bin/rails server -b 0.0.0.0`), or the published port reaches nothing.
- To reach the host from a container, use `host.docker.internal` on Docker Desktop; on Linux add `--add-host=host.docker.internal:host-gateway` (`extra_hosts` in Compose).

## 10. Environment variables and secrets

- Configure containers at runtime: `docker run -e KEY=value`, `--env-file .env`, or Compose `environment:` and `env_file:`. `ENV` and `ARG` values are stored in the image (history and `docker inspect`), so anyone who pulls the image can read them. Never put secrets there.
- Build-time secrets, such as a private package token, use BuildKit secret mounts, which exist only during one `RUN` step and are never stored in a layer:

```text
# Dockerfile
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN="$(cat /run/secrets/npm_token)" npm ci
```

```bash
docker build --secret id=npm_token,src=./.npm_token -t myapp .
```

- Runtime secrets come from the platform: Kamal secrets, ECS task definitions that reference Secrets Manager or SSM, Kubernetes secrets, Compose secrets, or a PaaS's config vars. A Rails container gets `RAILS_MASTER_KEY` at runtime.

## 11. docker compose

**Rails, PostgreSQL, Redis and Sidekiq for development:**

```yaml
# compose.yaml: no top-level "version:" key (it is obsolete in the Compose Specification)
x-app: &app
  build:
    context: .
    dockerfile: Dockerfile.dev          # dev image: dev gems, no asset precompile
  environment:
    DATABASE_URL: postgres://app:password@db:5432/app_development
    REDIS_URL: redis://redis:6379/0
  volumes:
    - .:/rails                          # bind mount for live code reloading
    - bundle:/usr/local/bundle          # installed gems survive container rebuilds
  depends_on:
    db:
      condition: service_healthy
    redis:
      condition: service_healthy

services:
  web:
    <<: *app
    command: bin/rails server -b 0.0.0.0 -p 3000
    ports:
      - "3000:3000"
  sidekiq:
    <<: *app
    command: bundle exec sidekiq
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: password
      POSTGRES_DB: app_development
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U app -d app_development"], interval: 5s, timeout: 3s, retries: 10 }
  redis:
    image: redis:7
    healthcheck: { test: ["CMD", "redis-cli", "ping"], interval: 5s, timeout: 3s, retries: 10 }

volumes:
  pg_data:
  bundle:
```

```bash
docker compose up -d                              # start everything in the background
docker compose run --rm web bin/rails db:prepare  # one-off command in a fresh container
docker compose exec web bin/rails console         # console inside the running web container
docker compose logs -f sidekiq
docker compose down                               # remove containers and network (volumes kept; -v deletes them too)
```

- `depends_on` alone only orders startup; `condition: service_healthy` waits until the healthcheck passes (Postgres actually accepting connections). The app should still retry connections, since dependencies can restart later. The `x-app` extension field with a YAML anchor keeps `web` and `sidekiq` identical apart from their commands.
- The official Postgres image changed its data directory layout starting with version 18, so check the image documentation for the right volume path when you upgrade.

**Node and PostgreSQL:**

```yaml
services:
  api:
    build:
      context: .
      target: deps                      # the stage from section 5 with every dependency installed
    command: npm run dev                # a watch-mode dev server
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgres://app:password@db:5432/app
      NODE_ENV: development
    volumes:
      - .:/app                          # live code from the host
      - /app/node_modules               # keep the image's node_modules instead of the host's
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:17
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: password
      POSTGRES_DB: app
    volumes:
      - pg_data:/var/lib/postgresql/data
    healthcheck: { test: ["CMD-SHELL", "pg_isready -U app -d app"], interval: 5s, retries: 10 }

volumes:
  pg_data:
```

Where bind mounts are slow, recent Compose v2 releases also offer Compose Watch (`develop.watch` rules plus `docker compose watch`) to sync changes into the container.

## 12. Development vs production

| | Development | Production |
|---|---|---|
| Code | bind-mounted for live reload | copied into an immutable, versioned image |
| Image and dependencies | dev and test gems and packages, debugging tools | production dependencies only; slim, multi-stage, non-root |
| Configuration | `.env` file and Compose | environment variables and secrets from the platform |
| Data | local volumes and seed data | managed databases (RDS and similar) with backups, rarely containers with volumes |
| Running and logs | `docker compose up`, logs in your terminal | a deploy tool or orchestrator (Kamal, ECS, Kubernetes, a PaaS) with health checks, restarts, rolling deploys and stdout logs collected by the platform |

Many teams run only Postgres and Redis in Docker locally and the app itself natively, which avoids file-sync slowness while keeping services consistent across machines.

## 13. The Rails default Dockerfile and Kamal

- `rails new` (7.1+) generates a production-oriented `Dockerfile`, `.dockerignore` and `bin/docker-entrypoint`: multi-stage, non-root, and in recent versions using jemalloc. It is not a development setup.
- Rails 8 adds **Kamal 2**: `config/deploy.yml` describes servers, the image registry, environment variables and accessories (Postgres, Redis); `kamal setup` prepares servers over SSH, `kamal deploy` builds, pushes and rolls out containers while kamal-proxy switches traffic without downtime, and `kamal rollback` returns to a previous version. **Thruster** runs in front of Puma inside the container, adding HTTP/2, compression, asset caching and X-Sendfile support. Deployment strategies are covered in [CI/CD and deployment](04-ci-cd-and-deployment.md).

## 14. Common issues

- **Slow builds or missed file changes on Windows:** keep the project inside the WSL 2 Linux filesystem (for example `~/projects`, reachable from Windows under `\\wsl$\`) rather than `/mnt/c/...`; bind mounts across the Windows/Linux boundary are slow and may miss change events.
- **Line endings:** a script checked out with CRLF fails in Linux with `/usr/bin/env: 'bash\r': No such file or directory`. Add a `.gitattributes` (`*.sh text eol=lf`, `bin/* text eol=lf`), set `git config --global core.autocrlf input`, and re-checkout the files. A lost executable bit (`permission denied` on `bin/docker-entrypoint`) is fixed with `git update-index --chmod=+x bin/docker-entrypoint` and a commit, or `RUN chmod +x` in the Dockerfile.
- **Bind-mount ownership:** files created by a root container become root-owned on the host, and a non-root container may fail to write to host folders; run the service as your UID (`user: "1000:1000"`) or fix permissions.
- **arm64 vs amd64:** an image built on an Apple Silicon Mac (arm64) fails on most x86 servers with `exec format error`, and vice versa. Build for the target with `docker buildx build --platform linux/amd64`, or publish multi-arch images with `--platform linux/amd64,linux/arm64`; Ruby lockfiles may need the extra platforms (`bundle lock --add-platform x86_64-linux aarch64-linux`).
- **Container exits immediately:** the main process ended or crashed (a container lives only as long as its foreground process); read `docker logs <name>`. In Rails, "A server is already running" means a stale `tmp/pids/server.pid` from a killed container: delete it in the entrypoint or skip the pid file.
- **Everyday errors:** "port is already allocated" (change the host side, `-p 3001:3000`), "cannot connect to the database" (`localhost` instead of the service name, no healthcheck, wrong credentials), and a full disk (check `docker system df`, then prune carefully).

## Common pitfalls
- `COPY . .` before installing dependencies, so every code change reinstalls everything; secrets in `ENV`, `ARG` or copied `.env` files, readable by anyone with the image.
- Shell-form `CMD` or entrypoints without `exec`, so containers ignore `SIGTERM` and get killed after the timeout.
- `localhost` between containers, an app server bound to `127.0.0.1` inside the container, or `depends_on` without healthchecks.
- Running as root, shipping build tools and dev dependencies, or deploying the `latest` tag in production.
- `docker compose down -v` or `prune --volumes` wiping the local database by surprise; using the production Dockerfile for development (or the reverse), or building for the wrong CPU architecture.

## How to talk about this in an interview
- Docker is in your skills list; be specific and honest about how you used it (local Postgres and Redis, a full Compose stack, building images, CI services, deployments), and show the concepts interviewers probe: image vs container, layers and caching, multi-stage builds, CMD vs ENTRYPOINT, Compose networking and secrets.
- Connect to Rails: the 7.1+ generated Dockerfile and Kamal 2 in Rails 8 show you know where the Rails ecosystem is heading, even if you have not deployed with Kamal yourself. If you develop on Windows, WSL 2, line-ending and permission issues make good practical debugging stories.

> **Fill in:** where you used Docker (which project, local services only or the whole app, any Dockerfile you wrote or changed) and one problem you debugged.

> **Fill in:** how Devntech client projects were deployed (Docker images, Heroku, Kamal, other) and whether Docker was part of CI.

## Interview questions

**Q1. What is the difference between a container and a virtual machine?**
A container is an isolated group of processes that shares the host's kernel, using namespaces for isolation and cgroups for resource limits; a VM runs a full guest OS with its own kernel on a hypervisor. Containers are lighter and start in seconds; VMs give stronger isolation and can run other operating systems.
*Follow-up:* How does Docker run Linux containers on Windows? Inside a lightweight Linux VM provided through WSL 2.

**Q2. Image vs container?**
An image is a read-only, layered template (filesystem plus configuration) built from a Dockerfile; a container is a running or stopped instance of that image with a thin writable layer and runtime settings. You can run many containers from one image, and the container's changes disappear when it is removed.
*Follow-up:* Where should persistent data go? In volumes, not the container's writable layer.

**Q3. CMD vs ENTRYPOINT? Exec form vs shell form?**
ENTRYPOINT defines the executable that always runs; CMD supplies default arguments (or the default command if there is no ENTRYPOINT) and is replaced by arguments to `docker run`. Exec form (`["bin/rails", "server"]`) runs the process directly as PID 1 so it receives signals; shell form wraps it in `/bin/sh -c`, which swallows `SIGTERM`.
*Follow-up:* How do you override ENTRYPOINT? `docker run --entrypoint <cmd>`, which also clears the image's CMD.

**Q4. How does layer caching work, and how do you order a Dockerfile?**
Each instruction produces a layer that Docker reuses while the instruction and its inputs are unchanged; once one step changes, it and every later step rebuild. Order from least to most frequently changing: base image, system packages, dependency manifests plus install, then application code and asset builds.
*Follow-up:* Why chain `apt-get update` and `install` in one `RUN`? So the package index is never stale in a cached layer and the cleanup shrinks the same layer.

**Q5. What is a multi-stage build, and why use it?**
A Dockerfile with several `FROM` stages: heavy stages install compilers and build the app, and the final stage copies in only the compiled artifacts on a slim base. Images get smaller, build tools and secrets stay out of production, and the attack surface shrinks.
*Follow-up:* What gets copied from the Rails build stage? The installed gems directory and the app with precompiled assets.

**Q6. How do containers in Compose talk to each other, and why does `localhost` fail?**
Compose puts services on a shared network where Docker's DNS resolves service names, so the app connects to `db:5432`. Inside a container, `localhost` refers to that same container, so a database in another container is not there. Data that must survive lives in named volumes; bind mounts are for live code.
*Follow-up:* And reaching a service on the host machine? `host.docker.internal`, with an extra host mapping on Linux.

**Q7. Does `depends_on` wait for the database to be ready?**
Not by itself: it only orders container startup. With `condition: service_healthy` and a healthcheck such as `pg_isready`, Compose waits until the dependency reports healthy. The application should still retry connections, because dependencies can restart later.
*Follow-up:* What other conditions exist? `service_started` and `service_completed_successfully` (for one-off setup containers).

**Q8. How do you handle secrets with Docker?**
Never bake them into images with `ENV`, `ARG` or copied files, because image metadata and layers are readable. Use BuildKit secret mounts for build-time secrets and inject runtime secrets through the platform (environment variables from a secret manager, Kamal secrets, ECS or Kubernetes secrets), and keep `.env` in `.dockerignore`.
*Follow-up:* Why not `ARG` for a token? Build arguments are recorded in the image history.

**Q9. How would you dockerize a Rails app with Postgres, Redis and Sidekiq?**
For development: a Compose file with `db` (Postgres with a named volume and a `pg_isready` healthcheck), `redis` (with a ping healthcheck), and `web` and `sidekiq` services built from a dev Dockerfile that share configuration, bind-mount the code, keep gems in a volume, use service names in `DATABASE_URL` and `REDIS_URL`, and wait for healthy dependencies. For production: the multi-stage Rails Dockerfile, managed Postgres and Redis, and separate web and worker processes from the same image.
*Follow-up:* How does the database get created? `docker compose run --rm web bin/rails db:prepare`, or the entrypoint running it at boot.

**Q10. Your image works on your Mac but fails on the server with "exec format error". Why?**
It was built for arm64 (Apple Silicon) and the server is amd64, or the reverse. Build for the target platform with `docker buildx build --platform linux/amd64`, or publish a multi-architecture image; the same error can also come from a script with CRLF line endings or a missing shebang.
*Follow-up:* How do you check an image's architecture? `docker image inspect` shows its `Architecture` field.

## Further reading
- [Docker documentation: Get started](https://docs.docker.com/get-started/)
- [Docker documentation: Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- [Docker Compose documentation](https://docs.docker.com/compose/)
- [Kamal documentation](https://kamal-deploy.org/)
