# Docker — Zero to Hero

> A complete, beginner-to-advanced guide to Docker and containerization for developers and DevOps engineers.

---

## Table of Contents

1. [Introduction & Theory](#1-introduction--theory)
2. [Installation & Setup](#2-installation--setup)
3. [Core Concepts](#3-core-concepts)
4. [Hands-on Tasks](#4-hands-on-tasks)
5. [Projects](#5-projects)
6. [Best Practices & Common Pitfalls](#6-best-practices--common-pitfalls)
7. [Interview Questions](#7-interview-questions)
8. [Quizzes](#8-quizzes)
9. [Further Resources](#9-further-resources)

---

## 1. Introduction & Theory

### 1.1 What is Docker?

**Docker** is a platform for developing, shipping, and running applications inside **containers** — lightweight, isolated, portable units that package an application together with all its dependencies (libraries, runtime, config). Docker was released in 2013 and popularized containerization to the point that it became the de-facto standard.

The promise: **"Build once, run anywhere."** A container behaves identically on a developer's laptop, a CI server, and production — eliminating the classic "it works on my machine" problem.

### 1.2 Containers vs. Virtual Machines

| Aspect | Virtual Machine | Container |
|--------|-----------------|-----------|
| Isolation | Full OS per VM (hypervisor) | Shares host kernel |
| Size | Gigabytes | Megabytes |
| Startup | Minutes | Seconds / milliseconds |
| Overhead | High (guest OS) | Low |
| Density | Few per host | Many per host |
| Use case | Strong isolation, different OS kernels | Microservices, fast scaling |

```
   VIRTUAL MACHINES                        CONTAINERS
+----------------------+          +-----------------------------+
| App A | App B | App C|          | App A | App B | App C        |
+-------+-------+------+          +-------+-------+--------------+
| Guest | Guest | Guest|          |  Bins/Libs per container     |
|  OS   |  OS   |  OS  |          +------------------------------+
+----------------------+          |     Docker Engine            |
|     Hypervisor       |          +------------------------------+
+----------------------+          |        Host OS (kernel)      |
|      Host OS         |          +------------------------------+
+----------------------+          |        Hardware              |
|      Hardware        |          +------------------------------+
+----------------------+
```

Containers share the host kernel, so they're far lighter than VMs. The trade-off is that all containers on a host run on the **same kernel**.

### 1.3 The technology underneath

Containers are not magic — they're built from Linux kernel features:

- **Namespaces** — isolate what a process can *see*: PID, network (NET), mount (MNT), user (USER), hostname (UTS), inter-process communication (IPC).
- **cgroups (control groups)** — limit and account for resource *usage*: CPU, memory, disk I/O, network.
- **Union filesystems (OverlayFS)** — layered, copy-on-write filesystem that makes images efficient.
- **Capabilities & seccomp** — restrict privileged operations for security.

### 1.4 Docker architecture

```
+-------------+        REST API       +---------------------------+
|   Docker    |  ─────────────────►   |     Docker Daemon         |
|   Client    |   (docker CLI)        |       (dockerd)           |
|  (docker)   |  ◄─────────────────   |  - builds images          |
+-------------+                       |  - runs containers        |
                                      |  - manages networks/vols  |
                                      +-------------+-------------+
                                                    │
                                  +-----------------+-----------------+
                                  │ containerd / runc (runtime)       │
                                  +-----------------+-----------------+
                                                    │
                                          Images  ──► Containers
                                                    │
                                          +---------▼---------+
                                          |   Registry        |
                                          | (Docker Hub, ECR) |
                                          +-------------------+
```

Key components:
- **Docker Client** — the `docker` CLI you type commands into.
- **Docker Daemon (`dockerd`)** — does the actual work: building images, running containers, managing resources.
- **containerd / runc** — the lower-level container runtime that the daemon uses.
- **Registry** — stores and distributes images (Docker Hub, GitHub/GitLab registry, AWS ECR, etc.).

### 1.5 Images vs. Containers (the key distinction)

- An **image** is a read-only template (a blueprint) built from layers — like a class.
- A **container** is a running (or stopped) instance of an image — like an object. You can run many containers from one image.

### 1.6 When to use Docker

- Packaging microservices and web apps for consistent deployment.
- Reproducible development environments and CI/CD pipelines.
- Isolating dependencies (multiple Python/Node versions on one host).
- Building the foundation for orchestration (Kubernetes).

When **not** ideal: GUI desktop apps, workloads needing a different OS kernel (Windows containers on Linux hosts), or scenarios demanding the strong isolation of full VMs.

---

## 2. Installation & Setup

### 2.1 Install Docker Engine (Ubuntu)

```bash
# Remove old versions
sudo apt remove -y docker docker-engine docker.io containerd runc 2>/dev/null

# Set up the official repository
sudo apt update
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 2.2 Post-install: run Docker without sudo

```bash
sudo groupadd docker 2>/dev/null
sudo usermod -aG docker $USER
newgrp docker            # apply group change in current shell (or log out/in)
```

### 2.3 Verify the installation

```bash
docker --version
docker compose version
docker run hello-world
```

**Expected:** A message saying *"Hello from Docker! This message shows that your installation appears to be working correctly."*

### 2.4 Other platforms

- **Windows/macOS:** install **Docker Desktop** (https://docker.com/products/docker-desktop) — bundles the engine, CLI, Compose, and a GUI.
- **Verify daemon:** `docker info` shows server details, storage driver, and resource limits.

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 Running your first containers

```bash
docker run hello-world                 # run and exit
docker run -it ubuntu bash             # interactive shell in a container
docker run -d nginx                    # detached (background)
docker run -d -p 8080:80 nginx         # map host:container port
docker run --name web -d nginx         # named container
docker run --rm alpine echo "hi"       # auto-remove after exit
```

Flag reference:
- `-i` keep STDIN open, `-t` allocate a TTY (combined `-it` for interactive).
- `-d` detached/background.
- `-p host:container` publish a port.
- `--name` assign a name.
- `--rm` remove container when it stops.
- `-e KEY=value` set environment variables.
- `-v host:container` mount a volume.

#### 3.1.2 Managing containers

```bash
docker ps                              # running containers
docker ps -a                           # all (including stopped)
docker stop web                        # graceful stop (SIGTERM)
docker start web                       # start a stopped container
docker restart web
docker kill web                        # force stop (SIGKILL)
docker rm web                          # remove a stopped container
docker rm -f web                       # force remove
docker logs web                        # view logs
docker logs -f web                     # follow logs
docker exec -it web bash               # shell into a running container
docker inspect web                     # full JSON metadata
docker stats                           # live resource usage
docker top web                         # processes in container
```

#### 3.1.3 Managing images

```bash
docker images                          # list local images
docker pull nginx:1.27                 # download an image
docker rmi nginx:1.27                  # remove an image
docker tag myapp myapp:v1              # tag an image
docker history nginx                   # show image layers
docker inspect nginx                   # image metadata
docker image prune                     # remove dangling images
```

#### 3.1.4 Image naming and tags

```
registry/repository:tag
docker.io/library/nginx:1.27-alpine
└──────┬─────┘ └──┬──┘ └────┬─────┘
   registry    repo        tag
```

If omitted: registry defaults to Docker Hub, tag defaults to `latest`.

### 3.2 INTERMEDIATE

#### 3.2.1 Writing a Dockerfile

A **Dockerfile** is a recipe to build an image.

```dockerfile
# Base image
FROM node:20-alpine

# Metadata
LABEL maintainer="you@example.com"

# Set working directory inside the image
WORKDIR /app

# Copy dependency manifests first (better layer caching)
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy the rest of the source
COPY . .

# Document the port the app listens on
EXPOSE 3000

# Environment defaults
ENV NODE_ENV=production

# Default command when the container starts
CMD ["node", "server.js"]
```

#### 3.2.2 Dockerfile instructions reference

| Instruction | Purpose |
|-------------|---------|
| `FROM` | Base image to build on |
| `WORKDIR` | Set working directory |
| `COPY` | Copy files from build context into image |
| `ADD` | Like COPY, but can fetch URLs and auto-extract archives |
| `RUN` | Execute a command at build time (creates a layer) |
| `CMD` | Default command at container start (one per Dockerfile) |
| `ENTRYPOINT` | Fixed executable; CMD becomes its arguments |
| `EXPOSE` | Document a port (does not publish it) |
| `ENV` | Set environment variables |
| `ARG` | Build-time variable |
| `VOLUME` | Declare a mount point |
| `USER` | Set the user to run as |
| `HEALTHCHECK` | Define a container health probe |

**CMD vs. ENTRYPOINT:**
- `ENTRYPOINT` defines the executable that always runs.
- `CMD` provides default arguments that can be overridden at `docker run`.
- Common pattern: `ENTRYPOINT ["python", "app.py"]` + `CMD ["--port", "8000"]`.

#### 3.2.3 Building and running your image

```bash
docker build -t myapp:v1 .             # build from current directory
docker build -t myapp:v1 -f Dockerfile.prod .   # specify a Dockerfile
docker build --no-cache -t myapp:v1 .  # ignore cache
docker run -d -p 3000:3000 myapp:v1
```

#### 3.2.4 Image layers and build cache

Each instruction (`FROM`, `RUN`, `COPY`, ...) creates a **layer**. Layers are cached: if nothing above a layer changed, Docker reuses the cache. This is why you **copy dependency files and install before copying source code** — code changes don't bust the dependency-install cache.

#### 3.2.5 Volumes and persistent data

Containers are ephemeral; data inside a container is lost when it's removed. **Volumes** persist data.

```bash
# Named volume (managed by Docker)
docker volume create mydata
docker run -d -v mydata:/var/lib/mysql mysql

# Bind mount (host path)
docker run -d -v /host/path:/container/path nginx
docker run -d -v "$(pwd)":/app node

# tmpfs (in-memory, non-persistent)
docker run --tmpfs /tmp nginx

docker volume ls
docker volume inspect mydata
docker volume rm mydata
docker volume prune                    # remove unused volumes
```

| Type | Stored where | Use case |
|------|--------------|----------|
| Named volume | Docker-managed area | Databases, persistent app data |
| Bind mount | Specific host path | Dev (live-editing source) |
| tmpfs | Host RAM | Sensitive/temporary data |

#### 3.2.6 Networking

```bash
docker network ls                      # list networks
docker network create mynet            # create a user-defined bridge
docker run -d --network mynet --name db postgres
docker run -d --network mynet --name app myapp   # 'app' can reach 'db' by name
docker network inspect mynet
docker network connect mynet web
docker network rm mynet
```

Network drivers:
- **bridge** (default) — private internal network on the host; containers reach the outside via NAT.
- **host** — container shares the host's network stack (no isolation, fastest).
- **none** — no networking.
- **overlay** — multi-host networking (Swarm/orchestration).

> On a user-defined bridge network, containers can resolve each other by **name** via Docker's embedded DNS — essential for multi-container apps.

#### 3.2.7 Docker Compose

**Compose** defines and runs multi-container applications with a single YAML file.

```yaml
# docker-compose.yml
services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgres://user:pass@db:5432/app
    depends_on:
      - db
    restart: unless-stopped

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      retries: 5

volumes:
  pgdata:
```

```bash
docker compose up                      # start (foreground)
docker compose up -d                   # start (detached)
docker compose up --build              # rebuild then start
docker compose ps                      # list services
docker compose logs -f                 # follow logs
docker compose exec web bash           # shell into a service
docker compose down                    # stop and remove
docker compose down -v                 # also remove volumes
docker compose restart web
```

### 3.3 ADVANCED

#### 3.3.1 Multi-stage builds

Multi-stage builds produce small final images by separating the build environment from the runtime.

```dockerfile
# --- Stage 1: build ---
FROM node:20 AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# --- Stage 2: runtime (tiny) ---
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

The final image contains only nginx + built assets — no Node.js, no source, no dev dependencies.

#### 3.3.2 Optimizing images

- Use **slim/alpine** base images.
- **Combine `RUN` commands** to reduce layers and clean up in the same layer:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
      curl ca-certificates && \
    rm -rf /var/lib/apt/lists/*
```

- Add a **`.dockerignore`** to keep the build context small:

```dockerignore
node_modules
.git
.env
*.md
dist
Dockerfile
.dockerignore
```

- Order instructions from least- to most-frequently-changing for cache efficiency.

#### 3.3.3 Running as a non-root user (security)

```dockerfile
FROM node:20-alpine
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app
COPY --chown=app:app . .
USER app
CMD ["node", "server.js"]
```

#### 3.3.4 Health checks

```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

```bash
docker ps    # STATUS shows (healthy) / (unhealthy)
```

#### 3.3.5 Resource limits

```bash
docker run -d --memory=512m --cpus=1.5 --name limited myapp
docker update --memory=1g limited      # adjust live
docker run --pids-limit=100 myapp      # limit number of processes
```

#### 3.3.6 Registries: push and pull

```bash
# Docker Hub
docker login
docker tag myapp:v1 yourusername/myapp:v1
docker push yourusername/myapp:v1
docker pull yourusername/myapp:v1

# Private/cloud registry (example AWS ECR)
aws ecr get-login-password | docker login --username AWS --password-stdin <acct>.dkr.ecr.<region>.amazonaws.com
docker tag myapp:v1 <acct>.dkr.ecr.<region>.amazonaws.com/myapp:v1
docker push <acct>.dkr.ecr.<region>.amazonaws.com/myapp:v1
```

#### 3.3.7 BuildKit and multi-arch builds

```bash
export DOCKER_BUILDKIT=1               # enable BuildKit (default on modern Docker)

# Build for multiple architectures
docker buildx create --use
docker buildx build --platform linux/amd64,linux/arm64 \
  -t yourusername/myapp:v1 --push .
```

BuildKit features: parallel stages, build secrets (`--secret`), cache mounts (`--mount=type=cache`), and faster builds.

#### 3.3.8 Cleanup and maintenance

```bash
docker system df                       # disk usage
docker system prune                    # remove stopped containers, unused networks, dangling images
docker system prune -a --volumes       # aggressive cleanup (careful!)
docker container prune
docker image prune -a
docker volume prune
docker builder prune                   # clear build cache
```

#### 3.3.9 Security essentials

- Scan images: `docker scout cves myapp:v1` or Trivy (`trivy image myapp:v1`).
- Pin base image versions and digests (`FROM nginx@sha256:...`).
- Don't bake secrets into images; use runtime env/secret managers or BuildKit secrets.
- Run as non-root; drop capabilities (`--cap-drop=ALL`), use read-only root FS (`--read-only`).
- Keep images minimal to reduce attack surface.
- Use `--security-opt no-new-privileges`.

#### 3.3.10 Docker Compose for the advanced (profiles, overrides)

```yaml
services:
  app:
    build: .
    profiles: ["full"]      # only starts with --profile full
```

```bash
docker compose --profile full up -d
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

---

## 4. Hands-on Tasks

### Task 1: Run and inspect a container

```bash
docker run -d --name web -p 8080:80 nginx
curl -s http://localhost:8080 | head -5
docker logs web
docker exec -it web sh -c "ls /usr/share/nginx/html"
```

**Expected:** nginx welcome HTML; logs show an HTTP request; directory listing shows `index.html`.

### Task 2: Build a custom image

Create `app.py` and a `Dockerfile`:

```python
# app.py
from http.server import HTTPServer, BaseHTTPRequestHandler
class H(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200); self.end_headers()
        self.wfile.write(b"Hello from Docker!")
HTTPServer(("0.0.0.0", 8000), H).serve_forever()
```

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
EXPOSE 8000
CMD ["python", "app.py"]
```

```bash
docker build -t pyhello:v1 .
docker run -d -p 8000:8000 --name hello pyhello:v1
curl http://localhost:8000
```

**Expected:** `Hello from Docker!`

### Task 3: Persist data with a volume

```bash
docker volume create appdata
docker run -d -v appdata:/data --name writer alpine \
  sh -c "echo persisted > /data/file.txt && sleep 3600"
docker rm -f writer
docker run --rm -v appdata:/data alpine cat /data/file.txt
```

**Expected:** Prints `persisted` even though the original container was removed.

### Task 4: Connect two containers on a network

```bash
docker network create appnet
docker run -d --network appnet --name db -e POSTGRES_PASSWORD=pass postgres:16
docker run -it --rm --network appnet postgres:16 \
  psql -h db -U postgres -c "SELECT 1;"   # password: pass
```

**Expected:** Query returns `1`, proving name-based DNS works.

### Task 5: Multi-container app with Compose

Use the Compose file from section 3.2.7 (adjust to a simple app), then:

```bash
docker compose up -d
docker compose ps
docker compose logs -f web
docker compose down
```

**Expected:** Both services running; web reachable on the mapped port.

### Task 6: Multi-stage build to shrink an image

Build a Node/React app (or any compiled app) with the multi-stage Dockerfile from 3.3.1, then compare sizes:

```bash
docker images | grep myapp
```

**Expected:** The final image is dramatically smaller than a single-stage build.

### Task 7: Add a health check

```bash
docker run -d --name hc \
  --health-cmd="curl -f http://localhost:80 || exit 1" \
  --health-interval=10s -p 8081:80 nginx
sleep 12
docker ps   # look for (healthy)
```

### Task 8: Limit resources

```bash
docker run -d --name limited --memory=256m --cpus=0.5 nginx
docker stats --no-stream limited
```

**Expected:** `docker stats` shows the memory limit at ~256 MiB.

### Task 9: Push an image to a registry

```bash
docker login
docker tag pyhello:v1 <yourusername>/pyhello:v1
docker push <yourusername>/pyhello:v1
```

**Expected:** Image appears in your Docker Hub repositories.

### Task 10: Clean up

```bash
docker compose down -v 2>/dev/null
docker rm -f $(docker ps -aq) 2>/dev/null
docker system prune -af --volumes
docker system df
```

**Expected:** Reclaimed space; minimal usage reported.

---

## 5. Projects

### Project 1 (Beginner): Containerize a Web Application

**Goal:** Take a simple Flask/Express/static app and containerize it properly.

Steps:
1. Write the app with a `/health` endpoint.
2. Create a Dockerfile (slim base, non-root user, `EXPOSE`, `CMD`).
3. Add a `.dockerignore`.
4. Build, tag, and run; verify with `curl`.
5. Add a `HEALTHCHECK`.
6. Push to Docker Hub.

**Skills:** Dockerfile basics, layering, health checks, registries.

### Project 2 (Intermediate): Full-Stack App with Docker Compose

**Goal:** Run a frontend + backend API + database + cache together.

Architecture:
```
[ frontend (nginx) ] → [ backend (API) ] → [ postgres ]
                                   └──────► [ redis ]
```

Steps:
1. Write a `docker-compose.yml` with four services on a shared network.
2. Use named volumes for Postgres data.
3. Use environment variables and a `.env` file for config/secrets.
4. Add `depends_on` + health checks so services start in order.
5. Configure the frontend to proxy API calls to the backend by service name.
6. Bring it all up with `docker compose up -d` and test end-to-end.

**Skills:** multi-service orchestration, networking, volumes, env config, healthchecks.

### Project 3 (Advanced): Production-Grade Image Pipeline

**Goal:** Build secure, optimized, multi-arch images in CI and publish them.

Steps:
1. Write a **multi-stage** Dockerfile producing a minimal runtime image running as non-root.
2. Add a `.dockerignore`, pin base image by digest, drop capabilities.
3. Integrate **image scanning** (Trivy/Docker Scout) into CI; fail on HIGH/CRITICAL CVEs.
4. Use **BuildKit** with build cache and `--secret` for private package access.
5. Build **multi-arch** (`amd64` + `arm64`) with `buildx` and push to a registry.
6. Tag images with semantic version + git SHA; sign images (cosign) optionally.
7. Document an update/rollback strategy.

**Skills:** multi-stage builds, security hardening, scanning, BuildKit, multi-arch, CI integration.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **One process/concern per container** — keep containers focused.
- **Use small base images** (alpine/slim/distroless) to reduce size and attack surface.
- **Leverage layer caching:** copy dependency manifests and install before copying source.
- **Use multi-stage builds** to exclude build tools from runtime images.
- **Never store secrets in images** — use runtime env vars, secret managers, or BuildKit secrets.
- **Run as a non-root user.**
- **Add a `.dockerignore`** to keep the build context small and avoid leaking files.
- **Pin versions** (base images and dependencies) for reproducibility.
- **Define health checks** so orchestrators know container status.
- **Tag meaningfully** (semantic version + git SHA), avoid relying on `latest` in production.
- **Scan images** for vulnerabilities in CI.

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Using `latest` tag in prod | Unpredictable deploys | Pin specific versions/digests |
| Storing data inside the container | Data lost on removal | Use volumes |
| Running as root | Security risk | `USER` non-root, drop caps |
| Huge images (full OS + build tools) | Slow pulls, bigger attack surface | Multi-stage + slim base |
| Baking secrets into layers | Secrets leak in image history | Runtime env / BuildKit secrets |
| No `.dockerignore` | Slow builds, leaked files | Add `.dockerignore` |
| `COPY . .` before installing deps | Cache busted on every code change | Copy manifests first |
| Ignoring zombie/cleanup | Disk fills up | `docker system prune` regularly |
| Assuming containers keep state | Surprises on restart | Treat containers as ephemeral |
| Publishing all ports / `--network host` blindly | Exposure risk | Publish only needed ports |

---

## 7. Interview Questions

### Beginner

1. **What is the difference between an image and a container?**
   *An image is a read-only template; a container is a running instance of an image. Many containers can run from one image.*

2. **What does `docker run -d -p 8080:80 nginx` do?**
   *Runs nginx detached, mapping host port 8080 to container port 80.*

3. **How do you list running vs. all containers?**
   *`docker ps` (running) and `docker ps -a` (all, including stopped).*

4. **What is a Dockerfile?**
   *A text file with instructions Docker uses to build an image.*

5. **How do you get a shell inside a running container?**
   *`docker exec -it <container> bash` (or `sh`).*

### Intermediate

6. **Explain the difference between CMD and ENTRYPOINT.**
   *ENTRYPOINT defines the fixed executable; CMD provides default arguments (overridable at run). Used together, ENTRYPOINT is the command and CMD its default args.*

7. **What's the difference between a volume and a bind mount?**
   *A named volume is managed by Docker in its own area (good for persistent data); a bind mount maps a specific host path into the container (good for development).*

8. **How does the Docker build cache work?**
   *Each instruction creates a layer; if an instruction and everything above it are unchanged, Docker reuses the cached layer. Ordering instructions well maximizes cache hits.*

9. **How do containers on a user-defined bridge network communicate?**
   *Via Docker's embedded DNS — they resolve each other by container/service name.*

10. **What does `EXPOSE` actually do?**
    *It documents the port the app uses; it does not publish it. You still need `-p` to publish to the host.*

### Advanced

11. **What kernel features make containers possible?**
    *Namespaces (isolation of PID, NET, MNT, UTS, IPC, USER), cgroups (resource limits), and union filesystems (OverlayFS).*

12. **How do multi-stage builds help?**
    *They separate build-time tooling from the runtime image, producing a small final image containing only what's needed to run, improving size and security.*

13. **How would you reduce a Docker image's size?**
    *Use slim/alpine/distroless bases, multi-stage builds, combine RUN layers with cleanup, add `.dockerignore`, remove caches, and avoid unnecessary packages.*

14. **How do you secure a container in production?**
    *Run as non-root, drop capabilities, read-only root FS, `no-new-privileges`, scan images, pin versions/digests, limit resources, avoid host networking, and never embed secrets.*

15. **Explain copy-on-write in Docker's storage.**
    *Image layers are read-only and shared. A container gets a thin writable layer on top; modifying a file copies it up into the writable layer (copy-on-write), so changes don't affect the shared image and aren't persisted unless using volumes.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** Which command builds an image tagged `myapp:v1` from the current directory?
- A) `docker run -t myapp:v1 .`  B) `docker build -t myapp:v1 .`  C) `docker image myapp:v1`  D) `docker create myapp:v1 .`

**Q2.** What does the `-d` flag do in `docker run`?
- A) Delete after exit  B) Debug mode  C) Detached/background  D) Disable network

**Q3.** Which persists data beyond a container's lifetime?
- A) Container writable layer  B) tmpfs  C) Volume  D) EXPOSE

**Q4.** What does `EXPOSE 80` do?
- A) Publishes port 80 to the host  B) Documents the port  C) Opens the firewall  D) Maps 80 to 8080

**Q5.** Which kernel feature limits container resource usage?
- A) namespaces  B) cgroups  C) OverlayFS  D) seccomp

**Q6.** In a Dockerfile, which provides default arguments overridable at runtime?
- A) ENTRYPOINT  B) RUN  C) CMD  D) ENV

**Q7.** What's the main benefit of a multi-stage build?
- A) Faster networking  B) Smaller final image  C) More layers  D) Root access

**Q8.** Which file excludes paths from the build context?
- A) `.gitignore`  B) `.dockerignore`  C) `.ignore`  D) `exclude.txt`

**Q9.** How do containers on a user-defined bridge reach each other?
- A) By IP only  B) By container name via DNS  C) Through the host's `/etc/hosts`  D) They can't

**Q10.** Which command removes unused images, networks, and stopped containers?
- A) `docker rm -a`  B) `docker clean`  C) `docker system prune`  D) `docker delete all`

### Short Answer

**S1.** Write a command to run `redis` in the background named `cache`, mapping port 6379.

**S2.** What instruction in a Dockerfile sets the user the container runs as?

**S3.** Write the command to follow the logs of a container named `api`.

**S4.** How do you start a multi-container app defined in `docker-compose.yml` in detached mode?

**S5.** Why should you copy `package.json` and install dependencies before copying the rest of the source?

### Answer Key

**Multiple Choice:** Q1-B, Q2-C, Q3-C, Q4-B, Q5-B, Q6-C, Q7-B, Q8-B, Q9-B, Q10-C

**Short Answer:**
- **S1.** `docker run -d --name cache -p 6379:6379 redis`
- **S2.** `USER`
- **S3.** `docker logs -f api`
- **S4.** `docker compose up -d`
- **S5.** *To leverage layer caching — dependencies only reinstall when the manifest changes, not on every source-code edit.*

---

## 9. Further Resources

### Official Documentation
- Docker Docs — https://docs.docker.com
- Dockerfile reference — https://docs.docker.com/reference/dockerfile/
- Docker Compose spec — https://docs.docker.com/compose/
- Best practices for writing Dockerfiles — https://docs.docker.com/build/building/best-practices/

### Interactive Learning
- Play with Docker (browser labs) — https://labs.play-with-docker.com
- Docker's official "Get Started" — https://docs.docker.com/get-started/
- KillerCoda Docker scenarios — https://killercoda.com

### Tools
- Docker Scout / Trivy / Grype — image vulnerability scanning
- Dive — https://github.com/wagoodman/dive (inspect image layers)
- hadolint — https://github.com/hadolint/hadolint (Dockerfile linter)
- ctop — https://github.com/bcicen/ctop (container metrics TUI)
- Buildx — multi-arch builds

### Books & Guides
- *Docker Deep Dive* — Nigel Poulton
- *Docker in Action* — Jeff Nickoloff & Stephen Kuenzli
- The Twelve-Factor App — https://12factor.net

### Registries
- Docker Hub — https://hub.docker.com
- GitHub Container Registry, GitLab Registry, AWS ECR, Google Artifact Registry

---

*End of Docker — Zero to Hero.*
