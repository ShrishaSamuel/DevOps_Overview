# Docker Interview Preparation — Hands-On Tasks

> A curated set of real-world Docker challenges organized by difficulty.  
> Each task mirrors scenarios you will encounter in DevOps interviews and on the job.

---

## Table of Contents

1. [Beginner Tasks](#beginner-tasks)
2. [Intermediate Tasks](#intermediate-tasks)
3. [Advanced Tasks](#advanced-tasks)
4. [Troubleshooting & Debugging Tasks](#troubleshooting--debugging-tasks)
5. [Container Architecture Design Tasks](#container-architecture-design-tasks)

---

## Beginner Tasks

---

### Task B1 — Write Your First Production-Grade Dockerfile

**Problem Statement**  
A Python Flask application exists with a `requirements.txt` and an `app.py`. The current Dockerfile uses `python:3.11` (full image), runs as `root`, copies everything into the container, and installs dependencies on every build. Rewrite it to follow production best practices.

**Expected Outcome**
- Image uses `python:3.11-slim` as the base.
- Dependencies are installed before copying application code (layer cache optimization).
- The container runs as a non-root user (`appuser`).
- `docker build` with unchanged dependencies uses the cache for the `pip install` layer.
- `docker inspect <image>` shows a non-root `User` in the config.

**Hints**
- Order Dockerfile instructions from least-changing to most-changing for maximum cache reuse.
- Use `COPY requirements.txt .` → `RUN pip install` → `COPY . .` (not the other way around).
- Create a user with `RUN useradd -m appuser` and switch with `USER appuser`.
- Use `--no-cache-dir` with pip to reduce image size.
- Set `WORKDIR` before any `COPY` or `RUN` instructions.

**Skills Tested**
- Dockerfile instruction order and layer caching
- Base image selection (`slim`, `alpine`, `distroless`)
- Non-root container execution
- Build cache optimization fundamentals

---

### Task B2 — Understand and Use Docker Volumes for Data Persistence

**Problem Statement**  
You run a PostgreSQL container but every time you stop and remove it, all data is lost. Set up a named volume so that data persists across container restarts and removals. Then simulate a disaster by deleting the container and prove data survived by recreating it.

**Expected Outcome**
- A named volume `pg-data` exists (`docker volume ls`).
- PostgreSQL container mounts `pg-data` at `/var/lib/postgresql/data`.
- After `docker rm -f postgres`, recreating the container with the same volume shows the previously created database.
- `docker volume inspect pg-data` shows mount path and creation metadata.

**Hints**
- Use `-v pg-data:/var/lib/postgresql/data` in `docker run` — Docker auto-creates named volumes.
- Named volumes (`pg-data`) survive `docker rm`; anonymous volumes (random hash) do not.
- Use `docker exec -it postgres psql -U postgres -c "CREATE DATABASE testdb;"` to write data.
- Never use bind mounts for database data in production — understand why.
- `docker volume prune` removes all unused volumes — dangerous in production.

**Skills Tested**
- Named vs anonymous vs bind mount volumes
- Data persistence lifecycle
- Volume management commands
- Understanding Docker volume storage drivers

---

### Task B3 — Explore and Use Docker Networking Modes

**Problem Statement**  
You have two containers: a Node.js API and a Redis cache. The API must reach Redis by hostname (`redis`), not by IP (which changes on restart). Connect them using a user-defined bridge network. Then, observe the difference between `bridge`, `host`, and `none` network modes.

**Expected Outcome**
- A custom network `app-net` created with `docker network create`.
- Both containers attached to `app-net`.
- From inside the API container, `ping redis` resolves and succeeds.
- The same `ping redis` fails from a container on the default `bridge` network.
- You can explain when you would use `host` mode and its security implications.

**Hints**
- The default `bridge` network does NOT provide automatic DNS resolution between containers.
- User-defined bridge networks embed a DNS server that resolves container names.
- Attach containers to a network with `--network app-net` at `docker run` time, or after with `docker network connect`.
- Use `docker network inspect app-net` to see connected containers and their IPs.
- `host` mode removes network isolation — the container shares the host's network stack.

**Skills Tested**
- Docker network types and their use cases
- Container DNS resolution via user-defined networks
- Network isolation concepts
- `docker network` management commands

---

### Task B4 — Manage Environment Variables and `.env` Files

**Problem Statement**  
A containerized application requires database credentials and API keys at runtime. These must never be baked into the image. Set up the container to accept environment variables via three methods: `-e` flag, `--env-file`, and Docker Compose `env_file`. Verify each method works and understand their security trade-offs.

**Expected Outcome**
- `docker run -e DB_HOST=localhost` passes a single variable correctly.
- `docker run --env-file .env` loads all variables from a file.
- `docker inspect <container>` shows the env vars (understand why this is a security concern).
- A `.dockerignore` file prevents `.env` from being accidentally copied into the image.

**Hints**
- Never use `ENV` in a Dockerfile for secrets — it's baked into every layer and visible in `docker inspect`.
- Add `.env` to `.dockerignore` to prevent `COPY . .` from including it.
- Use `ARG` for build-time values and `ENV` for runtime defaults of non-sensitive config only.
- In production, use Docker Secrets (Swarm) or a secrets manager (Vault, AWS Secrets Manager) instead of env files.
- Check `docker history <image>` to see all layers including those set via `ENV`.

**Skills Tested**
- `ENV` vs `ARG` distinction
- Runtime secret injection patterns
- `.dockerignore` usage
- Security awareness around environment variables

---

## Intermediate Tasks

---

### Task I1 — Reduce Image Size with Multi-Stage Builds

**Problem Statement**  
A Go application's Docker image is 1.2 GB because it uses `golang:1.21` and includes the full SDK, build tools, and source code in the final image. Refactor the Dockerfile to use a multi-stage build so the final image contains only the compiled binary.

**Expected Outcome**
- Final image is under 20 MB (using `scratch` or `alpine` as the runtime stage).
- `docker images` shows the old single-stage image vs the new multi-stage image side by side.
- The binary runs correctly in the final container despite the minimal base image.
- `docker history <image>` shows only the runtime layers, not the build tools.

**Hints**
- Stage 1 (`FROM golang:1.21 AS builder`): compile the binary with `CGO_ENABLED=0 GOOS=linux go build`.
- Stage 2 (`FROM scratch` or `FROM alpine:3.19`): `COPY --from=builder /app/binary /app/binary`.
- `CGO_ENABLED=0` produces a statically linked binary that runs on `scratch` (no libc needed).
- If using `scratch`, you'll also need to copy SSL certificates: `COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/`.
- Compare `docker image inspect` sizes before and after.

**Skills Tested**
- Multi-stage build pattern
- Static binary compilation
- `scratch` and `distroless` base images
- Image layer inspection with `docker history`

---

### Task I2 — Orchestrate a Multi-Container App with Docker Compose

**Problem Statement**  
Build a local development environment for a three-tier web application: a React frontend (`nginx`), a Python Django API, and a PostgreSQL database. The API must not start until the database is healthy. The frontend proxies API requests to avoid CORS issues. Use Docker Compose with proper health checks and dependency ordering.

**Expected Outcome**
- `docker compose up` starts all three services in the correct order.
- The database is healthy before the API starts (`depends_on` with `condition: service_healthy`).
- A `healthcheck` on the database container uses `pg_isready`.
- `docker compose ps` shows all services as `healthy` or `running`.
- `docker compose down -v` tears down everything including volumes cleanly.

**Hints**
- `depends_on` with just a service name only waits for the container to start, not for it to be ready — always pair with `condition: service_healthy`.
- PostgreSQL healthcheck: `test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER"]`.
- Use a named volume for postgres data in the `volumes:` section at the top level.
- The nginx frontend needs a `nginx.conf` with a `proxy_pass http://api:8000/` block to forward `/api/` requests.
- Use `networks:` to separate the frontend-facing network from the backend-only network.

**Skills Tested**
- Docker Compose service dependencies and health checks
- Multi-network topology in Compose
- Nginx reverse proxy configuration
- Volume and network lifecycle in Compose

---

### Task I3 — Implement Docker Layer Caching in a CI/CD Pipeline

**Problem Statement**  
Your GitLab CI pipeline rebuilds the full Docker image on every push, including re-running `npm install` even when `package.json` hasn't changed. This makes the pipeline take 8+ minutes. Optimize the Dockerfile and pipeline configuration to maximize cache hits and reduce build time to under 2 minutes for code-only changes.

**Expected Outcome**
- A refactored `Dockerfile` that separates dependency installation from source code copying.
- CI pipeline uses `--cache-from` to pull the previous image and use it as a cache source.
- A pipeline run where only `src/` files changed completes the build stage in under 30 seconds.
- A pipeline run where `package.json` changes correctly invalidates and rebuilds the dependency layer.

**Hints**
- Copy `package.json` and `package-lock.json` first, run `npm ci`, then copy the rest of the source.
- In GitLab CI, use `docker pull myimage:latest || true` before `docker build --cache-from myimage:latest`.
- Use `DOCKER_BUILDKIT=1` and `--build-arg BUILDKIT_INLINE_CACHE=1` to enable BuildKit inline caching.
- With BuildKit, you can also use `--cache-to type=inline` and `--cache-from type=registry`.
- Alternatively, use `docker buildx` with `--cache-to type=gha` for GitHub Actions native caching.

**Skills Tested**
- Dockerfile layer ordering for cache efficiency
- Registry-based build cache (`--cache-from`)
- BuildKit and `docker buildx` caching strategies
- CI/CD pipeline optimization

---

### Task I4 — Harden a Container for Production Security

**Problem Statement**  
A security audit flagged your container deployment for multiple issues: running as root, container can gain new privileges, no read-only filesystem, all Linux capabilities are enabled, and the image contains a package manager and shell. Fix all findings without breaking the application.

**Expected Outcome**
- Container runs as a non-root user (UID > 1000).
- `docker run --security-opt no-new-privileges` is enforced (or set in Compose).
- Filesystem is read-only (`--read-only`) with tmpfs mounts only for directories needing writes.
- Capabilities are dropped to the minimum: `--cap-drop ALL` with only required ones added back.
- Final image uses `distroless` or `scratch` — no shell, no package manager.
- `docker scout cve <image>` (or `trivy image <image>`) shows zero critical CVEs.

**Hints**
- Use `RUN addgroup -S appgroup && adduser -S appuser -G appgroup` (Alpine syntax) in the Dockerfile.
- In Docker Compose: `security_opt: [no-new-privileges:true]`, `read_only: true`, `cap_drop: [ALL]`.
- Identify writable paths the app needs (`/tmp`, `/var/cache`) and mount them as `tmpfs`.
- `distroless/python3` or `distroless/nodejs` images have no shell — use `exec` form `CMD ["node", "server.js"]`.
- Run `docker run --rm -it <image> sh` — it should fail on a distroless image (no shell = attack surface reduced).

**Skills Tested**
- Container security hardening checklist
- Linux capabilities model
- Read-only filesystem with tmpfs exceptions
- Distroless and minimal images
- CVE scanning with Trivy or Docker Scout

---

### Task I5 — Build and Push a Multi-Platform Image

**Problem Statement**  
Your team has a mix of developers on Apple Silicon (ARM64) and a production cluster running on AMD64. A single-platform image causes `exec format error` when run on the wrong architecture. Build a multi-platform image that runs natively on both `linux/amd64` and `linux/arm64` using a single image tag.

**Expected Outcome**
- `docker buildx build --platform linux/amd64,linux/arm64 -t myrepo/myapp:latest --push .` succeeds.
- `docker manifest inspect myrepo/myapp:latest` shows manifests for both platforms.
- Running the image on both an AMD64 machine and an ARM64 machine works without emulation warnings.
- Image size for each platform is optimal (not the emulated size).

**Hints**
- Create a new buildx builder: `docker buildx create --name multiarch --use`.
- The builder needs QEMU for cross-compilation: `docker run --rm --privileged multiarch/qemu-user-static --reset -p yes`.
- Multi-platform builds require `--push` (cannot load a multi-arch image into the local daemon).
- Use `--platform $BUILDPLATFORM` in the build stage and `--platform $TARGETPLATFORM` in the runtime stage to avoid emulating the build tools.
- Use `TARGETARCH` build arg inside the Dockerfile to download architecture-specific binaries.

**Skills Tested**
- `docker buildx` and BuildKit
- Multi-platform image manifests
- QEMU cross-compilation emulation
- OCI image manifest specification

---

## Advanced Tasks

---

### Task A1 — Implement a Rootless Docker Setup

**Problem Statement**  
Your company's security policy prohibits running the Docker daemon as root. Set up rootless Docker for a CI runner user (`cirunner`) so the entire Docker workflow (build, run, push) works without root privileges. Identify the limitations vs standard Docker.

**Expected Outcome**
- Docker daemon runs as `cirunner` without sudo.
- `docker run hello-world` succeeds as `cirunner`.
- Volume mounts work correctly with user namespace remapping.
- You document at least 3 features not available in rootless mode (e.g., overlay networking, certain storage drivers).

**Hints**
- Install rootless Docker with `dockerd-rootless-setuptool.sh install` as the target user.
- Requires `newuidmap` and `newgidmap` tools (part of `uidmap` package).
- Set `DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock` in the user's environment.
- Storage driver may fall back to `fuse-overlayfs` — understand its performance implications.
- Use `systemctl --user enable --now docker` to manage the user-scoped daemon.

**Skills Tested**
- Linux user namespaces and UID/GID mapping
- Rootless container security model
- Daemon configuration and socket paths
- Security vs functionality trade-offs

---

### Task A2 — Design a Zero-Downtime Deployment with Docker Swarm

**Problem Statement**  
You manage a Docker Swarm cluster with 3 manager nodes and 5 worker nodes. Deploy a web service with 6 replicas and implement a zero-downtime rolling update. When deploying `v2`, no more than 1 replica should be unavailable at a time and the new container must pass a health check before the next replica is updated.

**Expected Outcome**
- A Swarm service `webapp` running 6 replicas across worker nodes.
- `docker service update --image webapp:v2` performs a rolling update with 1 replica at a time.
- `docker service ps webapp` shows old and new tasks interleaved during the update.
- A failing health check in `v2` causes the update to pause automatically.
- `docker service rollback webapp` restores `v1` across all 6 replicas.

**Hints**
- Configure rolling updates with `--update-parallelism 1 --update-delay 10s --update-failure-action pause`.
- The `HEALTHCHECK` instruction in the Dockerfile drives Swarm's `update-monitor` period.
- Use `--update-order start-first` to start the new container before stopping the old one (requires spare capacity).
- `docker service inspect webapp --pretty` shows all update configuration.
- Constrain replicas to worker nodes only with `--constraint 'node.role == worker'`.

**Skills Tested**
- Docker Swarm service management
- Rolling update configuration and failure handling
- Health check integration with orchestration
- Service rollback mechanics

---

### Task A3 — Optimize a Docker Build with BuildKit Cache Mounts

**Problem Statement**  
A Rust application takes 12 minutes to build because cargo downloads and compiles all dependencies on every build. The source code changes frequently but `Cargo.toml` rarely changes. Use BuildKit cache mounts to persist the cargo registry and build cache between builds, reducing incremental build time to under 60 seconds.

**Expected Outcome**
- First build takes full time (cold cache).
- Second build with only `src/` changes completes in under 60 seconds.
- `docker system df` shows the build cache growing but not the image layers.
- Clearing the build cache (`docker builder prune`) forces a full rebuild on the next run.

**Hints**
- Enable BuildKit: `DOCKER_BUILDKIT=1` or `"features": {"buildkit": true}` in `/etc/docker/daemon.json`.
- Use `RUN --mount=type=cache,target=/usr/local/cargo/registry cargo build --release`.
- Also cache the build output: `--mount=type=cache,target=/app/target`.
- Cache mounts are NOT stored in image layers — they're stored in BuildKit's cache backend.
- Use `--mount=type=cache,id=cargo-registry,sharing=locked` to prevent cache corruption in parallel builds.

**Skills Tested**
- BuildKit cache mount (`RUN --mount=type=cache`)
- Build performance optimization for compiled languages
- BuildKit cache backend and `docker builder` commands
- Separation of build cache from image layers

---

### Task A4 — Implement Image Vulnerability Scanning in CI

**Problem Statement**  
Your CI pipeline builds and pushes Docker images but never checks for vulnerabilities. Integrate Trivy into the pipeline so that: builds fail if any `CRITICAL` severity CVE is found in the final image, a full vulnerability report is uploaded as a CI artifact, and base image updates are automatically suggested.

**Expected Outcome**
- CI pipeline has a `scan` stage after `build`.
- `trivy image --exit-code 1 --severity CRITICAL myimage:$CI_COMMIT_SHA` fails the pipeline on critical CVEs.
- Full JSON report is saved as a CI artifact and viewable in the pipeline UI.
- Updating the base image from `python:3.11.0` to `python:3.11-slim` (latest patch) reduces the CVE count.

**Hints**
- Run Trivy as a container in CI: `docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image`.
- Use `--format template --template "@contrib/gitlab.tpl"` for GitLab security dashboard integration.
- `--ignore-unfixed` skips CVEs with no available fix — useful for reducing noise.
- For base image suggestions, use `trivy image --dependency-tree` to see where vulnerabilities originate.
- Cache Trivy's vulnerability DB in CI to avoid re-downloading on every run.

**Skills Tested**
- Container security scanning workflow
- CVE severity levels and triage
- CI pipeline security gate integration
- Base image lifecycle management

---

## Troubleshooting & Debugging Tasks

---

### Task T1 — Debug a Container That Exits Immediately

**Problem Statement**  
A colleague runs `docker run myapp` and the container exits with code 1 immediately. There are no logs visible. The image was built successfully. Diagnose and fix the issue using only Docker CLI tools — no access to the source code initially.

**Expected Outcome**
- Root cause is identified (e.g., wrong entrypoint, missing binary, wrong working directory, permission error).
- Container runs successfully after the fix.
- Investigation steps are documented as a debugging runbook.

**Hints**
- Override the entrypoint to get a shell: `docker run -it --entrypoint /bin/sh myapp` (fails if no shell in distroless — use `sh` or `bash`).
- Check `docker inspect myapp --format='{{.Config.Cmd}} {{.Config.Entrypoint}}'` to see what command is being run.
- Use `docker run --entrypoint "" myapp cat /proc/1/cmdline` to inspect without running the app.
- Check `docker logs <container-id>` even after exit — logs persist until `docker rm`.
- Use `docker diff <container>` to see filesystem changes made before the crash.

**Skills Tested**
- Container startup debugging methodology
- Entrypoint and CMD override for debugging
- `docker inspect`, `docker logs`, `docker diff`
- Container filesystem investigation

---

### Task T2 — Fix an Out-of-Control Container Consuming All Resources

**Problem Statement**  
A runaway container is consuming 100% CPU and growing memory unboundedly, starving other containers on the same host. You cannot stop the service immediately. Apply resource limits to cap the container's resource usage and investigate the root cause.

**Expected Outcome**
- Running container is updated (or recreated) with CPU limit of 0.5 cores and memory limit of 512Mi.
- `docker stats` shows the container's CPU and memory capped at the new limits.
- After OOM-killing (if memory limit is hit), `docker inspect` shows `OOMKilled: true` in the `State`.
- Root cause is identified and a `docker-compose.yml` with proper `deploy.resources.limits` is produced.

**Hints**
- You cannot update resource limits on a running container — you must `docker update` or recreate it.
- `docker update --cpus 0.5 --memory 512m --memory-swap 512m <container>` applies limits to a running container.
- `docker stats --no-stream` gives a snapshot of all container resource usage.
- In Compose, use `deploy.resources.limits` (Compose v3) or `mem_limit`/`cpus` (Compose v2).
- Set `--memory-swap` equal to `--memory` to disable swap for the container.

**Skills Tested**
- Container resource constraints (CPU, memory)
- `docker stats` and `docker update`
- OOM behavior understanding
- Resource limits in Docker Compose

---

### Task T3 — Diagnose Networking Issues Between Containers

**Problem Statement**  
Two containers (`frontend` and `backend`) are on the same Compose network but `frontend` cannot reach `backend` at `http://backend:3000`. Both containers are `running`. DNS resolves `backend` to an IP, but the connection times out. Find all the issues and fix them.

**Expected Outcome**
- `curl http://backend:3000` from `frontend` returns a valid response.
- All networking misconfigurations are identified and documented.
- You can explain the difference between `EXPOSE` in a Dockerfile and publishing ports with `-p`.

**Hints**
- `EXPOSE` is documentation only — it does NOT open ports between containers on the same network. Containers on the same network can reach each other on ANY port the app is listening on.
- Check that the backend app is actually binding to `0.0.0.0`, not `127.0.0.1` — a common mistake.
- Use `docker exec frontend curl -v http://backend:3000` to test from inside the network.
- Use `docker exec backend ss -tlnp` to confirm the app is listening on port 3000.
- `docker network inspect <network>` shows connected containers and their aliases.

**Skills Tested**
- `EXPOSE` vs port publishing vs inter-container networking
- Application bind address debugging (`127.0.0.1` vs `0.0.0.0`)
- Network diagnostic commands inside containers
- Docker Compose network defaults

---

### Task T4 — Recover Disk Space on a Docker Host

**Problem Statement**  
A CI host's disk is at 97% capacity. Pipelines are failing with `no space left on device`. You must recover disk space safely without deleting data that is actively in use. Identify what's consuming space and clean it up systematically.

**Expected Outcome**
- `docker system df` is used to quantify space used by images, containers, volumes, and build cache.
- At least 10 GB is recovered without removing any running containers or named volumes with data.
- A cron-job-friendly cleanup script is written that can run safely in CI environments.
- Disk usage is reduced below 70% and `docker system df` reflects the change.

**Hints**
- `docker system df -v` shows detailed breakdown per image, container, and volume.
- `docker image prune -a --filter "until=72h"` removes images not used in the last 72 hours.
- `docker container prune` removes stopped containers — these accumulate rapidly in CI.
- `docker volume prune` is DANGEROUS — it removes all unnamed volumes. Always check first.
- `docker builder prune --keep-storage 5gb` keeps only 5 GB of build cache.
- Never run `docker system prune -a -f --volumes` on a production host without verifying first.

**Skills Tested**
- Docker disk usage analysis (`docker system df`)
- Safe cleanup of images, containers, volumes, build cache
- `prune` command filter options
- CI host maintenance discipline

---

### Task T5 — Debug a Failing Multi-Stage Build

**Problem Statement**  
A multi-stage Dockerfile builds successfully locally but fails in CI with `COPY --from=builder: file not found`. The binary that should be copied from the `builder` stage does not exist at the expected path. Debug the build failure without changing the CI environment.

**Expected Outcome**
- Root cause is identified (e.g., build failed silently in a layer, wrong output path, architecture mismatch, conditional build logic not met).
- Fix applied to the Dockerfile.
- `docker build --target builder -t debug-builder .` is used as a debugging technique.
- CI build succeeds and the final image runs correctly.

**Hints**
- Build only the intermediate stage: `docker build --target builder -t debug-builder .` then `docker run -it debug-builder sh` to inspect what's actually there.
- A `RUN` command that exits with code 0 but produces no output (e.g., a failed `go build` that was suppressed) is a common trap.
- Check `WORKDIR` consistency between stages — relative paths depend on the current `WORKDIR`.
- Cross-compilation: if CI runs on AMD64 but you tested on ARM64, the binary might be built for the wrong target.
- Add `RUN ls -la /app/` before the `COPY --from` to print the builder filesystem in CI logs.

**Skills Tested**
- Multi-stage build debugging methodology
- `--target` flag for intermediate stage inspection
- Cross-platform build issue identification
- CI vs local environment parity

---

## Container Architecture Design Tasks

---

### Task D1 — Design a Local Development Environment with Hot Reload

**Problem Statement**  
Design a Docker Compose setup for a full-stack application (React + Node.js API + PostgreSQL) that supports hot reload for both frontend and backend during development, without rebuilding images on code changes. Production images must be separate and optimized.

**Expected Outcome**
- `docker compose -f docker-compose.dev.yml up` mounts source code as bind mounts and enables hot reload.
- Changing a `.js` file in the API automatically restarts the process inside the container (nodemon or similar).
- Changing a React component triggers HMR without a full page reload.
- `docker compose -f docker-compose.prod.yml up` uses multi-stage built, optimized images with no bind mounts.

**Hints**
- Dev Compose file: use `volumes: - ./api:/app` bind mount + `command: nodemon src/index.js`.
- For Vite/CRA hot reload inside Docker, set `CHOKIDAR_USEPOLLING=true` (filesystem events don't propagate in some Docker environments).
- Use `extends` in Compose or a `docker-compose.override.yml` to share common config between dev and prod.
- Prod Dockerfile: multi-stage build producing a minimal image with `node --max-old-space-size` tuning.
- Never bind-mount `node_modules` from the host — use an anonymous volume to override it: `- /app/node_modules`.

**Skills Tested**
- Dev vs prod Compose file patterns
- Bind mounts for hot reload
- `node_modules` volume override trick
- Compose override and extension patterns

---

### Task D2 — Implement a Private Docker Registry with Access Control

**Problem Statement**  
Your team needs a private Docker registry on-premises (no Docker Hub). Deploy a private registry with TLS, basic authentication, and a retention policy that automatically deletes images older than 30 days. CI/CD pipelines must authenticate and push/pull images securely.

**Expected Outcome**
- Registry running at `https://registry.internal:5000` with a valid TLS certificate.
- `docker login registry.internal:5000` requires a username and password.
- `docker push registry.internal:5000/myapp:v1.0.0` succeeds from an authenticated session.
- Unauthenticated pushes are rejected with `401 Unauthorized`.
- A cron job or registry GC command prunes old manifests and blobs.

**Hints**
- Use the official `registry:2` image with a `config.yml` for auth and storage configuration.
- Generate a `htpasswd` file: `docker run --rm --entrypoint htpasswd httpd:2 -Bbn user password > htpasswd`.
- For TLS, use a self-signed cert or Let's Encrypt; configure `REGISTRY_HTTP_TLS_CERTIFICATE` and `REGISTRY_HTTP_TLS_KEY`.
- Add the self-signed CA to Docker's trusted certs: `/etc/docker/certs.d/registry.internal:5000/ca.crt`.
- Run garbage collection: `docker exec registry bin/registry garbage-collect /etc/docker/registry/config.yml`.

**Skills Tested**
- Private registry deployment and configuration
- TLS certificate management for Docker
- Registry authentication (htpasswd)
- Registry garbage collection and storage management

---

### Task D3 — Build a Reproducible Build System with Docker

**Problem Statement**  
Your team's builds are not reproducible — different developers get different binary outputs depending on their local environment. Design a Docker-based build system where `docker run --rm -v $(pwd):/src builder make` always produces identical output regardless of the host OS, installed tools, or timezone.

**Expected Outcome**
- A `builder` image that pins every tool version (compiler, linter, test runner) explicitly.
- Build output is identical on macOS, Linux, and Windows hosts.
- `SOURCE_DATE_EPOCH` is set to ensure reproducible timestamps in build artifacts.
- The `builder` image is versioned and stored in the registry, so `git blame` on `Dockerfile` explains every tool version choice.

**Hints**
- Pin every tool: `apt-get install gcc=12.2.0-14` not `apt-get install gcc`.
- Use `--no-install-recommends` to avoid implicit dependency version variance.
- Set `ENV TZ=UTC` and `ENV LANG=C.UTF-8` for locale/timezone reproducibility.
- Use `ARG BUILDKIT_SBOM_SCAN_CONTEXT=true` to generate a Software Bill of Materials (SBOM) with BuildKit.
- Run `docker build --no-cache` in CI to validate there are no implicit host dependencies leaking in.

**Skills Tested**
- Reproducible builds with Docker
- Dependency pinning and SBOM
- Build environment isolation
- `SOURCE_DATE_EPOCH` and timestamp determinism

---

## Quick Reference — Key Docker Commands

| Command | Purpose |
|---|---|
| `docker build -t name:tag --no-cache .` | Build image, ignoring cache |
| `docker build --target <stage> .` | Build only up to a specific multi-stage stage |
| `docker history <image>` | Show image layers and sizes |
| `docker inspect <container/image>` | Full JSON metadata |
| `docker stats --no-stream` | Snapshot of all container resource usage |
| `docker system df -v` | Detailed disk usage breakdown |
| `docker system prune -a --filter until=24h` | Remove resources older than 24h |
| `docker exec -it <container> sh` | Shell into a running container |
| `docker run --entrypoint sh -it <image>` | Override entrypoint for debugging |
| `docker logs <container> --tail 100 -f` | Stream last 100 lines of logs |
| `docker cp <container>:/path ./local` | Copy file from container to host |
| `docker diff <container>` | Show filesystem changes in container |
| `docker network inspect <network>` | Show network details and connected containers |
| `docker volume inspect <volume>` | Show volume mount path and driver |
| `docker buildx build --platform linux/amd64,linux/arm64` | Multi-platform build |
| `docker manifest inspect <image>` | Inspect multi-platform manifest |
| `DOCKER_BUILDKIT=1 docker build .` | Enable BuildKit for advanced features |

---

## Interview Cheat Sheet

- **"What is the difference between CMD and ENTRYPOINT?"**  
  `ENTRYPOINT` defines the executable that always runs. `CMD` provides default arguments to `ENTRYPOINT`. If only `CMD` is set, it defines the full command. Using both together: `ENTRYPOINT ["node"]` + `CMD ["server.js"]` — the CMD can be overridden at `docker run` time without replacing the entrypoint.

- **"Explain Docker layers and why they matter."**  
  Each `RUN`, `COPY`, `ADD` instruction creates a new read-only layer. Layers are content-addressed and cached. If a layer's content changes, all subsequent layers are invalidated. Order instructions from least-to-most-changing to maximize cache efficiency.

- **"What is the difference between COPY and ADD?"**  
  `COPY` copies files/dirs literally. `ADD` additionally supports URL downloads and auto-extraction of tar archives. Best practice: always use `COPY` unless you specifically need `ADD`'s extra features.

- **"Named volume vs bind mount vs tmpfs?"**  
  Named volume: Docker-managed, persists data, portable. Bind mount: maps a host path directly, good for dev but creates host dependency. tmpfs: in-memory only, lost on container stop, for sensitive temporary data.

- **"How do you reduce Docker image size?"**  
  Multi-stage builds, minimal base images (alpine/distroless/scratch), combine `RUN` commands to reduce layers, use `.dockerignore`, `--no-install-recommends`, and clean up package manager caches in the same `RUN` layer that installs packages.

- **"What is Docker networking — bridge vs host vs overlay?"**  
  Bridge: default, isolated NAT network per host. Host: container shares host network stack (no isolation). Overlay: multi-host network for Swarm/Kubernetes. User-defined bridge adds automatic DNS resolution between containers.

- **"How do you handle secrets in Docker?"**  
  Never use `ENV` for secrets (visible in `docker inspect` and layers). Use runtime injection (`-e`, `--env-file`), Docker Secrets (Swarm), Kubernetes Secrets, or a secrets manager (Vault, AWS Secrets Manager). Always add `.env` to `.dockerignore`.

- **"What is BuildKit?"**  
  BuildKit is the modern build engine replacing the legacy builder. It enables: parallel stage building, cache mounts (`RUN --mount=type=cache`), secret mounts (`RUN --mount=type=secret`), SSH agent forwarding, multi-platform builds, and inline build cache.

---

## Study Path

```
Week 1  → B1, B2, B3, B4         (Dockerfile basics, volumes, networks, env vars)
Week 2  → I1, I2, I3             (Multi-stage builds, Compose, CI caching)
Week 3  → I4, I5, T1, T2         (Security hardening, multi-platform, debugging)
Week 4  → T3, T4, T5             (Network issues, disk cleanup, build debugging)
Week 5  → A1, A2, A3, A4         (Rootless, Swarm, BuildKit cache mounts, scanning)
Week 6  → D1, D2, D3             (Dev environments, private registry, reproducible builds)
```

---

*Run every task in a local Docker environment. Use `docker events` in a separate terminal while working — seeing the real-time event stream builds deep intuition for how Docker orchestrates container lifecycle.*
