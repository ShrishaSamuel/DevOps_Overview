# Docker Training: Zero to Advanced with AWS Real-Time Examples

> **Your Docker Trainer & AWS DevOps Mentor**
> Covers: Basics → Intermediate → Advanced | AWS Integration | Real-Time Projects | Troubleshooting | Interview Prep

---

## Table of Contents

1. [What is Docker? Why Docker?](#1-what-is-docker-why-docker)
2. [Core Concepts & Architecture](#2-core-concepts--architecture)
3. [Installation & Setup](#3-installation--setup)
4. [Docker CLI Fundamentals](#4-docker-cli-fundamentals)
5. [Working with Images](#5-working-with-images)
6. [Writing Dockerfiles](#6-writing-dockerfiles)
7. [Docker Volumes & Persistent Storage](#7-docker-volumes--persistent-storage)
8. [Docker Networking](#8-docker-networking)
9. [Docker Compose](#9-docker-compose)
10. [Docker Registry & Amazon ECR](#10-docker-registry--amazon-ecr)
11. [Real-Time Project: Deploy a Python Flask App to AWS ECS](#11-real-time-project-deploy-a-python-flask-app-to-aws-ecs)
12. [Docker Multi-Stage Builds](#12-docker-multi-stage-builds)
13. [Docker Secrets & Security Best Practices](#13-docker-secrets--security-best-practices)
14. [Docker Health Checks & Logging](#14-docker-health-checks--logging)
15. [Docker Swarm (Orchestration Basics)](#15-docker-swarm-orchestration-basics)
16. [Docker + AWS ECS (Elastic Container Service)](#16-docker--aws-ecs-elastic-container-service)
17. [Docker + AWS EKS (Kubernetes on AWS)](#17-docker--aws-eks-kubernetes-on-aws)
18. [CI/CD Pipeline: GitHub Actions + ECR + ECS](#18-cicd-pipeline-github-actions--ecr--ecs)
19. [Troubleshooting & Bug Fixing](#19-troubleshooting--bug-fixing)
20. [Best Practices](#20-best-practices)
21. [Assignments](#21-assignments)
22. [Interview Questions & Answers](#22-interview-questions--answers)

---

## 1. What is Docker? Why Docker?

### The Problem Before Docker

Imagine you write a Python app on your laptop:

```
Your Laptop:  Python 3.11, Ubuntu 22.04  → App works ✅
Dev Server:   Python 3.8,  CentOS 7      → App crashes ❌
Production:   Python 3.9,  Amazon Linux  → App crashes ❌
```

The classic developer complaint: **"It works on my machine!"**

### What Docker Solves

Docker packages your application together with **everything it needs** — the code, runtime, libraries, environment variables, and config files — into a single portable unit called a **container**.

```
Your Laptop  →  Container  →  Dev Server   ✅
                            →  Production  ✅
                            →  AWS ECS     ✅
                            →  AWS EKS     ✅
```

### Virtual Machine vs Docker Container

```
┌─────────────────────────────────┐    ┌─────────────────────────────────┐
│         Virtual Machine          │    │         Docker Container         │
├─────────────────────────────────┤    ├─────────────────────────────────┤
│  App A  │  App B  │  App C      │    │  App A  │  App B  │  App C      │
├─────────┼─────────┼─────────────┤    ├─────────┼─────────┼─────────────┤
│ Guest   │ Guest   │ Guest       │    │  Libs   │  Libs   │  Libs       │
│  OS     │  OS     │  OS         │    ├─────────┴─────────┴─────────────┤
├─────────┴─────────┴─────────────┤    │         Docker Engine            │
│          Hypervisor              │    ├─────────────────────────────────┤
├─────────────────────────────────┤    │           Host OS               │
│           Host OS               │    ├─────────────────────────────────┤
├─────────────────────────────────┤    │           Hardware              │
│           Hardware              │    └─────────────────────────────────┘
└─────────────────────────────────┘
  Heavy, slow, GBs per VM              Light, fast, MBs per container
  Full OS per VM                        Shares Host OS kernel
  Minutes to start                      Milliseconds to start
```

| Feature        | Virtual Machine | Docker Container |
|----------------|-----------------|-----------------|
| Boot Time      | Minutes         | Milliseconds    |
| Size           | GBs             | MBs             |
| OS             | Full Guest OS   | Shares Host OS  |
| Isolation      | Strong          | Process-level   |
| Portability    | Limited         | Excellent       |

---

## 2. Core Concepts & Architecture

### Key Terms

| Term           | Analogy                        | Description                                          |
|----------------|--------------------------------|------------------------------------------------------|
| **Image**      | Blueprint / Recipe             | Read-only template to create containers             |
| **Container**  | Running instance of image      | Isolated process running from an image              |
| **Dockerfile** | Cooking instructions           | Script to build a Docker image                      |
| **Registry**   | App Store for images           | Storage for Docker images (Docker Hub, Amazon ECR)  |
| **Volume**     | External hard drive            | Persistent storage outside the container filesystem |
| **Network**    | Internal network               | How containers communicate with each other          |
| **Layer**      | Layers of a cake               | Each instruction in Dockerfile adds a layer         |

### Docker Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                         Docker Client                         │
│     docker build  │  docker pull  │  docker run              │
└────────────────────────┬─────────────────────────────────────┘
                         │  REST API
┌────────────────────────▼─────────────────────────────────────┐
│                      Docker Daemon (dockerd)                  │
│                                                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Containers  │  │   Images    │  │  Networks/Volumes   │  │
│  └─────────────┘  └─────────────┘  └─────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────────┐
│                  Container Registry                           │
│           Docker Hub  │  Amazon ECR  │  GitHub GHCR          │
└──────────────────────────────────────────────────────────────┘
```

### Image Layers (How Images Work)

```dockerfile
FROM ubuntu:22.04          # Layer 1: Base OS (~77MB)
RUN apt-get update         # Layer 2: Updated package index
RUN apt-get install python # Layer 3: Python installed
COPY app.py /app/          # Layer 4: Your application code
CMD ["python", "/app/app.py"] # Layer 5: Default start command
```

Each layer is **cached**. If you change only `app.py`, Docker only rebuilds Layer 4 and 5. Layers 1-3 are reused from cache → **faster builds**.

---

## 3. Installation & Setup

### Install Docker on Ubuntu/Debian (Linux)

```bash
# Step 1: Update package list
sudo apt-get update

# Step 2: Install required packages for HTTPS transport
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# Step 3: Add Docker's official GPG key (security verification)
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
    sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Step 4: Set up the stable repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Step 5: Install Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Step 6: Start Docker service
sudo systemctl start docker
sudo systemctl enable docker

# Step 7: Add your user to docker group (avoid sudo every time)
sudo usermod -aG docker $USER
newgrp docker

# Step 8: Verify installation
docker --version
docker run hello-world
```

**Line-by-line explanation of `docker run hello-world`:**
1. Docker client contacts the Docker daemon
2. Daemon pulls `hello-world` image from Docker Hub (no local image found)
3. Daemon creates a container from that image
4. Container runs and prints the "Hello from Docker!" message
5. Container exits (it's a one-time task container)

---

## 4. Docker CLI Fundamentals

### The Most Important Commands

```bash
# ─── IMAGE COMMANDS ───────────────────────────────────────────

docker images                    # List all local images
docker pull nginx                # Download nginx image from Docker Hub
docker pull nginx:1.25           # Pull specific version (tag)
docker rmi nginx                 # Remove an image
docker rmi $(docker images -q)   # Remove ALL images

# ─── CONTAINER COMMANDS ───────────────────────────────────────

docker run nginx                          # Run container (foreground, blocking)
docker run -d nginx                       # Run in detached (background) mode
docker run -d -p 8080:80 nginx            # Map host port 8080 → container port 80
docker run -d -p 8080:80 --name web nginx # Give container a name

docker ps                        # List running containers
docker ps -a                     # List ALL containers (including stopped)

docker stop web                  # Gracefully stop container (sends SIGTERM)
docker kill web                  # Force stop container (sends SIGKILL)
docker start web                 # Start a stopped container
docker restart web               # Restart a container

docker rm web                    # Remove a stopped container
docker rm -f web                 # Force remove a running container
docker rm $(docker ps -aq)       # Remove ALL stopped containers

# ─── INTERACT WITH CONTAINER ──────────────────────────────────

docker exec -it web bash         # Open interactive bash shell inside container
# -i = interactive (keep STDIN open)
# -t = allocate a pseudo-TTY (terminal)
# bash = command to run inside container

docker logs web                  # View container logs
docker logs -f web               # Follow (tail) container logs in real time
docker logs --tail 100 web       # Show last 100 lines

docker inspect web               # Detailed JSON info about container
docker stats                     # Live CPU/memory/network stats for all containers
docker top web                   # Show running processes inside container

# ─── SYSTEM COMMANDS ──────────────────────────────────────────

docker system df                 # Show Docker disk usage
docker system prune              # Remove unused containers, images, networks
docker system prune -a           # Remove ALL unused resources (be careful!)
```

### Hands-On Example: Run an nginx Web Server

```bash
# Pull and run nginx
docker run -d -p 8080:80 --name my-nginx nginx

# Verify it's running
docker ps

# Test it
curl http://localhost:8080

# Check logs
docker logs my-nginx

# Open a shell inside the container
docker exec -it my-nginx bash

# Inside the container:
# ls /usr/share/nginx/html    ← default web root
# cat /etc/nginx/nginx.conf   ← nginx config
# exit                        ← leave the container

# Stop and clean up
docker stop my-nginx
docker rm my-nginx
```

---

## 5. Working with Images

### Image Naming Convention

```
registry/namespace/repository:tag
    │         │         │      │
    │         │         │      └── version (default: latest)
    │         │         └───────── image name
    │         └─────────────────── username or org
    └───────────────────────────── registry host (default: docker.io)

Examples:
  nginx                           → docker.io/library/nginx:latest
  python:3.11-slim                → docker.io/library/python:3.11-slim
  123456789.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0  → Amazon ECR image
```

### Searching and Inspecting Images

```bash
# Search Docker Hub
docker search python

# Pull specific versions
docker pull python:3.11          # Full Python image (~900MB)
docker pull python:3.11-slim     # Minimal Python (~130MB)  ← prefer this
docker pull python:3.11-alpine   # Alpine-based (~50MB)     ← smallest

# Inspect image details
docker image inspect python:3.11-slim

# Show image history (all layers)
docker history python:3.11-slim

# Show image size
docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
```

### Tagging Images

```bash
# Tag an image for Amazon ECR
docker tag my-app:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0

# Format:
# docker tag <source-image> <target-image>
```

---

## 6. Writing Dockerfiles

### Dockerfile Instruction Reference

| Instruction  | Purpose                                               |
|--------------|-------------------------------------------------------|
| `FROM`       | Base image (always first)                             |
| `WORKDIR`    | Set working directory inside container                |
| `COPY`       | Copy files from host to container                     |
| `ADD`        | Like COPY but supports URLs and tar extraction        |
| `RUN`        | Execute commands during image build                   |
| `ENV`        | Set environment variables                             |
| `ARG`        | Build-time variables (not available in final image)   |
| `EXPOSE`     | Document which port the app uses (metadata only)      |
| `CMD`        | Default command when container starts (overridable)   |
| `ENTRYPOINT` | Fixed command that always runs                        |
| `VOLUME`     | Declare mount points for persistent data              |
| `USER`       | Switch to a non-root user (security best practice)    |
| `LABEL`      | Add metadata (author, version, description)           |
| `HEALTHCHECK`| Define how Docker checks container health             |

### Example 1: Simple Python Flask App

**Project structure:**
```
my-flask-app/
├── app.py
├── requirements.txt
└── Dockerfile
```

**app.py:**
```python
from flask import Flask
app = Flask(__name__)

@app.route('/')
def home():
    return "Hello from Docker + Flask!"

@app.route('/health')
def health():
    return {"status": "healthy"}, 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

**requirements.txt:**
```
flask==3.0.0
gunicorn==21.2.0
```

**Dockerfile:**
```dockerfile
# ── Layer 1: Base image ──────────────────────────────────────────
# Use official Python 3.11 slim image (smaller than full image)
FROM python:3.11-slim

# ── Layer 2: Metadata ─────────────────────────────────────────────
# Add image metadata (optional but good practice)
LABEL maintainer="yourname@example.com"
LABEL version="1.0"
LABEL description="Flask REST API"

# ── Layer 3: Working directory ────────────────────────────────────
# All subsequent commands run from /app directory
# Docker creates this directory if it doesn't exist
WORKDIR /app

# ── Layer 4: Install dependencies ─────────────────────────────────
# Copy requirements FIRST (before code) for better caching
# If requirements.txt hasn't changed, Docker reuses cached layer
COPY requirements.txt .

# Run pip install inside the container
# --no-cache-dir: don't cache pip downloads (saves space)
RUN pip install --no-cache-dir -r requirements.txt

# ── Layer 5: Copy application code ───────────────────────────────
# Copy everything else after dependencies
# (changes to app code won't invalidate pip install cache)
COPY . .

# ── Layer 6: Security - run as non-root user ─────────────────────
# Create a system user 'appuser' with no home directory
RUN adduser --system --no-create-home appuser

# Switch to that user (avoids running as root)
USER appuser

# ── Metadata: document the port ──────────────────────────────────
# This is documentation only - does NOT open the port
# You still need -p flag in docker run
EXPOSE 5000

# ── Default command ───────────────────────────────────────────────
# Use gunicorn as production WSGI server
# -w 2: 2 worker processes
# -b 0.0.0.0:5000: listen on all interfaces, port 5000
CMD ["gunicorn", "-w", "2", "-b", "0.0.0.0:5000", "app:app"]
```

**Build and run:**
```bash
# Build the image
# -t: tag the image with name:version
# . : build context is current directory (where Dockerfile is)
docker build -t my-flask-app:v1.0 .

# Verify the image was created
docker images my-flask-app

# Run the container
docker run -d -p 5000:5000 --name flask-app my-flask-app:v1.0

# Test it
curl http://localhost:5000
curl http://localhost:5000/health
```

### Example 2: Node.js Application

```dockerfile
FROM node:20-alpine

# Alpine Linux uses 'node' user (not root) built-in
WORKDIR /app

# Copy package files first for dependency caching
COPY package*.json ./

# npm ci = clean install (faster, reproducible for CI/CD)
RUN npm ci --only=production

# Copy source code
COPY src/ ./src/

# Switch to non-root user (built into node:alpine image)
USER node

EXPOSE 3000

CMD ["node", "src/index.js"]
```

### CMD vs ENTRYPOINT — Important Difference

```dockerfile
# CMD: default command, can be overridden at runtime
CMD ["python", "app.py"]
# docker run my-image                  → runs: python app.py
# docker run my-image python debug.py  → runs: python debug.py (override)

# ENTRYPOINT: fixed command, arguments are appended
ENTRYPOINT ["python"]
CMD ["app.py"]
# docker run my-image           → runs: python app.py
# docker run my-image debug.py  → runs: python debug.py (CMD overridden, ENTRYPOINT fixed)

# Real-world pattern: Use ENTRYPOINT for the executable, CMD for default args
ENTRYPOINT ["gunicorn"]
CMD ["-w", "2", "-b", "0.0.0.0:5000", "app:app"]
```

### .dockerignore File

Create `.dockerignore` to exclude files from build context (like `.gitignore`):

```
# .dockerignore
__pycache__/
*.pyc
*.pyo
*.pyd
.Python
.env
.env.*
.venv/
venv/
*.log
.git/
.gitignore
README.md
Dockerfile
.dockerignore
node_modules/
dist/
build/
.pytest_cache/
.coverage
htmlcov/
```

> **Why it matters:** Smaller build context = faster builds. Prevents secrets in `.env` from being copied into the image.

---

## 7. Docker Volumes & Persistent Storage

### The Problem: Containers are Ephemeral

```bash
# Start a container and create a file
docker run -it ubuntu bash
# Inside container:
echo "important data" > /data/myfile.txt
exit

# Container is stopped. Start a new container from same image:
docker run -it ubuntu bash
cat /data/myfile.txt   # ← FILE IS GONE! Data doesn't persist
```

### Types of Storage

```
┌──────────────────────────────────────────────────────┐
│                    Container                          │
│  ┌─────────────────────────────────────────────────┐ │
│  │  Container Layer (writable, ephemeral)          │ │
│  └─────────────────────────────────────────────────┘ │
│  ┌────────────────┐  ┌───────────────────────────┐   │
│  │  Named Volume  │  │  Bind Mount               │   │
│  │  /var/lib/     │  │  /host/path → /container/ │   │
│  │  docker/volumes│  │  (host directory mounted) │   │
│  └────────────────┘  └───────────────────────────┘   │
└──────────────────────────────────────────────────────┘
```

### Named Volumes (Recommended for Production)

```bash
# Create a named volume
docker volume create my-data

# List volumes
docker volume ls

# Mount volume to container
# -v my-data:/app/data  means: mount 'my-data' volume at /app/data in container
docker run -d \
  -v my-data:/app/data \
  --name app \
  my-flask-app:v1.0

# Inspect volume (find actual location on host)
docker volume inspect my-data

# Remove volume
docker volume rm my-data

# Remove all unused volumes
docker volume prune
```

### Bind Mounts (Good for Development)

```bash
# Mount current directory into container
# Great for development: edit on host, see changes immediately in container
docker run -d \
  -p 5000:5000 \
  -v $(pwd)/app:/app \    # host path : container path
  --name dev-flask \
  my-flask-app:v1.0

# Any changes to files in $(pwd)/app are immediately visible in container
```

### Real-Time Example: PostgreSQL with Persistent Data on AWS ECS

```bash
# Local development with persistent database
docker run -d \
  --name postgres-db \
  -e POSTGRES_USER=admin \
  -e POSTGRES_PASSWORD=secret123 \
  -e POSTGRES_DB=myapp \
  -v postgres-data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:15-alpine

# Connect to it
docker exec -it postgres-db psql -U admin -d myapp
```

---

## 8. Docker Networking

### Default Network Types

```bash
# List networks
docker network ls

# Output:
# NETWORK ID     NAME      DRIVER    SCOPE
# abc123         bridge    bridge    local   ← default for containers
# def456         host      host      local   ← shares host network
# ghi789         none      null      local   ← no network
```

### Bridge Network (Default)

```bash
# Containers on the same bridge network can communicate by name
docker network create my-app-network

# Run containers on the same network
docker run -d \
  --name database \
  --network my-app-network \
  -e POSTGRES_PASSWORD=secret \
  postgres:15-alpine

docker run -d \
  --name webapp \
  --network my-app-network \
  -p 5000:5000 \
  -e DATABASE_URL=postgresql://postgres:secret@database:5432/app \
  my-flask-app:v1.0

# 'webapp' can reach 'database' using the container name 'database' as hostname
# Docker's built-in DNS resolves container names to their IP addresses
```

### Port Mapping

```bash
# -p HOST_PORT:CONTAINER_PORT
docker run -p 8080:80 nginx     # host:8080 → container:80
docker run -p 443:443 nginx     # same port number
docker run -p 127.0.0.1:8080:80 nginx  # bind to localhost only (more secure)

# Expose all ports declared with EXPOSE
docker run -P nginx             # Docker assigns random host ports
docker port nginx-container     # See what ports were mapped
```

---

## 9. Docker Compose

### What is Docker Compose?

Docker Compose lets you define and run **multi-container applications** using a single YAML file. Instead of running multiple `docker run` commands, you write one `docker-compose.yml`.

### docker-compose.yml Syntax

```yaml
# docker-compose.yml for a Flask + PostgreSQL + Redis stack

version: "3.9"   # Compose file format version

services:        # Define each container as a service

  # ── Service 1: Web Application ────────────────────────────────
  web:
    build:
      context: .          # Build from Dockerfile in current directory
      dockerfile: Dockerfile
    image: my-flask-app:v1.0
    container_name: flask-web
    ports:
      - "5000:5000"       # host:container port mapping
    environment:
      - FLASK_ENV=production
      - DATABASE_URL=postgresql://admin:secret@db:5432/myapp
      - REDIS_URL=redis://cache:6379/0
    depends_on:
      db:
        condition: service_healthy   # Wait until db is healthy
      cache:
        condition: service_started
    volumes:
      - app-logs:/app/logs           # Mount named volume for logs
    networks:
      - app-network
    restart: unless-stopped          # Auto-restart unless manually stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # ── Service 2: PostgreSQL Database ────────────────────────────
  db:
    image: postgres:15-alpine
    container_name: postgres-db
    environment:
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret      # In production, use Docker secrets!
      POSTGRES_DB: myapp
    volumes:
      - postgres-data:/var/lib/postgresql/data   # Persist database files
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # Run on first start
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d myapp"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── Service 3: Redis Cache ─────────────────────────────────────
  cache:
    image: redis:7-alpine
    container_name: redis-cache
    command: redis-server --appendonly yes --requirepass cachepassword
    volumes:
      - redis-data:/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "cachepassword", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

  # ── Service 4: Nginx Reverse Proxy ────────────────────────────
  nginx:
    image: nginx:1.25-alpine
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro   # :ro = read-only
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - web
    networks:
      - app-network
    restart: unless-stopped

# ── Named Volumes ─────────────────────────────────────────────
volumes:
  postgres-data:
    driver: local
  redis-data:
    driver: local
  app-logs:
    driver: local

# ── Networks ──────────────────────────────────────────────────
networks:
  app-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

### Docker Compose Commands

```bash
# Start all services in background
docker compose up -d

# Start with rebuild (when Dockerfile changed)
docker compose up -d --build

# View running services
docker compose ps

# View logs for all services
docker compose logs

# View logs for specific service, follow real-time
docker compose logs -f web

# Execute command in running service
docker compose exec web bash
docker compose exec db psql -U admin -d myapp

# Stop all services (containers still exist)
docker compose stop

# Stop and remove containers, networks (volumes preserved)
docker compose down

# Stop, remove containers AND volumes (DATA LOSS - be careful!)
docker compose down -v

# Scale a service (run multiple instances)
docker compose up -d --scale web=3

# Pull latest images
docker compose pull
```

---

## 10. Docker Registry & Amazon ECR

### Amazon ECR (Elastic Container Registry)

ECR is AWS's managed Docker container registry — like Docker Hub, but private and integrated with AWS services (ECS, EKS, Lambda, IAM).

```
Your Machine  →  Build Image  →  Push to ECR  →  ECS/EKS pulls image  →  Runs container
```

### Setting Up ECR

```bash
# Step 1: Install AWS CLI
pip install awscli
aws configure
# Enter: AWS Access Key ID, Secret, Region (e.g., us-east-1), output format (json)

# Step 2: Create an ECR repository
aws ecr create-repository \
    --repository-name my-flask-app \
    --region us-east-1 \
    --image-scanning-configuration scanOnPush=true \
    --encryption-configuration encryptionType=AES256

# Step 3: Authenticate Docker to ECR
# This command generates a token and logs Docker in
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin \
    123456789012.dkr.ecr.us-east-1.amazonaws.com
# 123456789012 = your AWS account ID

# Step 4: Build your image
docker build -t my-flask-app:v1.0 .

# Step 5: Tag for ECR
# Format: ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/REPO_NAME:TAG
docker tag my-flask-app:v1.0 \
    123456789012.dkr.ecr.us-east-1.amazonaws.com/my-flask-app:v1.0

# Also tag as latest
docker tag my-flask-app:v1.0 \
    123456789012.dkr.ecr.us-east-1.amazonaws.com/my-flask-app:latest

# Step 6: Push to ECR
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-flask-app:v1.0
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-flask-app:latest

# Step 7: Verify push
aws ecr list-images --repository-name my-flask-app --region us-east-1

# Step 8: Pull from ECR (on any machine with AWS credentials)
docker pull 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-flask-app:v1.0
```

### ECR Lifecycle Policy (Auto-clean old images)

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 10 production images",
      "selection": {
        "tagStatus": "tagged",
        "tagPrefixList": ["v"],
        "countType": "imageCountMoreThan",
        "countNumber": 10
      },
      "action": { "type": "expire" }
    },
    {
      "rulePriority": 2,
      "description": "Delete untagged images older than 7 days",
      "selection": {
        "tagStatus": "untagged",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 7
      },
      "action": { "type": "expire" }
    }
  ]
}
```

```bash
# Apply lifecycle policy
aws ecr put-lifecycle-policy \
    --repository-name my-flask-app \
    --lifecycle-policy-text file://ecr-lifecycle.json \
    --region us-east-1
```

---

## 11. Real-Time Project: Deploy a Python Flask App to AWS ECS

### Project Overview

```
Developer  →  GitHub  →  GitHub Actions CI/CD  →  Build & Push to ECR  →  Deploy to ECS Fargate
                                                                              ↓
                                                                      Users access via ALB
                                                                      (Application Load Balancer)
```

### Step 1: Project Structure

```
flask-ecs-app/
├── app/
│   ├── __init__.py
│   ├── main.py
│   └── config.py
├── tests/
│   └── test_main.py
├── Dockerfile
├── docker-compose.yml
├── docker-compose.override.yml   ← local dev overrides
├── requirements.txt
├── .dockerignore
└── .env.example
```

### Step 2: Application Code

**app/main.py:**
```python
import os
from flask import Flask, jsonify
import psycopg2

app = Flask(__name__)

DATABASE_URL = os.environ.get("DATABASE_URL", "sqlite:///dev.db")
APP_VERSION  = os.environ.get("APP_VERSION", "1.0.0")

@app.route("/")
def index():
    return jsonify({
        "app": "Flask ECS Demo",
        "version": APP_VERSION,
        "environment": os.environ.get("ENVIRONMENT", "development")
    })

@app.route("/health")
def health():
    return jsonify({"status": "healthy"}), 200

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000, debug=False)
```

### Step 3: Production Dockerfile

```dockerfile
# ── Stage 1: Build dependencies ────────────────────────────────
FROM python:3.11-slim AS builder

WORKDIR /build

# Install build tools (only needed at build time)
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# ── Stage 2: Final production image ───────────────────────────
FROM python:3.11-slim

# Install runtime libraries only
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

WORKDIR /app

# Copy installed packages from builder stage
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY app/ ./app/

# Set Python path to include user's local packages
ENV PATH=/home/appuser/.local/bin:$PATH
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1

# Switch to non-root user
USER appuser

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

CMD ["gunicorn", \
     "--bind", "0.0.0.0:5000", \
     "--workers", "2", \
     "--threads", "4", \
     "--timeout", "60", \
     "--access-logfile", "-", \
     "--error-logfile", "-", \
     "app.main:app"]
```

### Step 4: ECS Task Definition (JSON)

```json
{
  "family": "flask-app-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::123456789012:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "flask-app",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-flask-app:latest",
      "portMappings": [
        {
          "containerPort": 5000,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {"name": "ENVIRONMENT", "value": "production"},
        {"name": "APP_VERSION", "value": "1.0.0"}
      ],
      "secrets": [
        {
          "name": "DATABASE_URL",
          "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/db-url"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/flask-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:5000/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
```

```bash
# Register task definition
aws ecs register-task-definition \
    --cli-input-json file://task-definition.json \
    --region us-east-1

# Create ECS cluster
aws ecs create-cluster --cluster-name my-app-cluster --region us-east-1

# Create ECS service with load balancer
aws ecs create-service \
    --cluster my-app-cluster \
    --service-name flask-app-service \
    --task-definition flask-app-task:1 \
    --desired-count 2 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[subnet-abc123,subnet-def456],securityGroups=[sg-xyz789],assignPublicIp=ENABLED}" \
    --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=flask-app,containerPort=5000" \
    --region us-east-1
```

---

## 12. Docker Multi-Stage Builds

Multi-stage builds create smaller, more secure final images by separating the build environment from the runtime environment.

### Before Multi-Stage (Problem)

```dockerfile
# Single-stage: final image contains ALL build tools (~800MB)
FROM golang:1.21
RUN apt-get install -y gcc make
COPY . .
RUN make build
CMD ["./myapp"]
```

### After Multi-Stage (Solution)

```dockerfile
# ── Stage 1: Build ────────────────────────────────────────────
FROM golang:1.21-alpine AS builder
# 'AS builder' names this stage so we can reference it later

WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download          # Download dependencies (cached layer)

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o /app/server .
# CGO_ENABLED=0: static binary (no C dependencies)
# GOOS=linux: compile for Linux

# ── Stage 2: Final image ──────────────────────────────────────
FROM scratch
# 'scratch' = completely empty base image (ultra-minimal)
# Only contains what we explicitly copy

# Copy ONLY the compiled binary from builder stage
COPY --from=builder /app/server /server

EXPOSE 8080
CMD ["/server"]
# Final image size: ~10MB instead of ~800MB
```

### Multi-Stage for Python (Dev vs Prod)

```dockerfile
# ── Base Stage ────────────────────────────────────────────────
FROM python:3.11-slim AS base
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# ── Development Stage ─────────────────────────────────────────
FROM base AS development
# Add dev tools not needed in production
COPY requirements-dev.txt .
RUN pip install --no-cache-dir -r requirements-dev.txt
ENV FLASK_ENV=development
ENV FLASK_DEBUG=1
CMD ["flask", "run", "--host=0.0.0.0", "--reload"]

# ── Production Stage ──────────────────────────────────────────
FROM base AS production
COPY . .
RUN adduser --system appuser && chown -R appuser /app
USER appuser
ENV FLASK_ENV=production
CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:5000", "app:app"]
```

```bash
# Build specific stage
docker build --target development -t my-app:dev .
docker build --target production -t my-app:prod .

# Use compose to select stage
# docker-compose.yml:
#   build:
#     context: .
#     target: development
```

---

## 13. Docker Secrets & Security Best Practices

### Never Bake Secrets into Images

```dockerfile
# ❌ BAD: Secret visible in image history
ENV DATABASE_PASSWORD=mysecretpassword

# ❌ BAD: ARG values visible in image history
ARG DB_PASS
RUN some-command --password=$DB_PASS
```

```dockerfile
# ✅ GOOD: Pass secrets at runtime via environment
# docker run -e DATABASE_PASSWORD=$DB_PASS ...
# Or use AWS Secrets Manager / Parameter Store
```

### Docker Secrets (Docker Swarm)

```bash
# Create a secret from a file
echo "supersecretpassword" | docker secret create db_password -

# Use secret in service
docker service create \
    --name db \
    --secret db_password \
    -e POSTGRES_PASSWORD_FILE=/run/secrets/db_password \
    postgres:15
# Secrets are mounted as files in /run/secrets/ (not env vars)
```

### AWS Secrets Manager with ECS

```python
# In your app code - fetch secret at startup
import boto3
import json

def get_secret(secret_name, region="us-east-1"):
    client = boto3.client("secretsmanager", region_name=region)
    response = client.get_secret_value(SecretId=secret_name)
    return json.loads(response["SecretString"])

secrets = get_secret("prod/myapp/database")
DATABASE_URL = secrets["url"]
```

### Security Scanning

```bash
# Scan image for vulnerabilities using Docker Scout
docker scout cves my-flask-app:v1.0

# Or use Trivy (open source)
# Install Trivy
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh

# Scan image
trivy image my-flask-app:v1.0

# Scan for CRITICAL vulnerabilities only
trivy image --severity CRITICAL my-flask-app:v1.0

# ECR built-in scanning
aws ecr describe-image-scan-findings \
    --repository-name my-flask-app \
    --image-id imageTag=v1.0 \
    --region us-east-1
```

### Security Checklist

```dockerfile
# 1. Use specific version tags (not 'latest')
FROM python:3.11.7-slim    # ✅ Pinned version
FROM python:latest          # ❌ Unpredictable

# 2. Run as non-root user
RUN useradd -r -u 1001 appuser
USER appuser

# 3. Read-only filesystem where possible
# docker run --read-only --tmpfs /tmp my-image

# 4. Drop Linux capabilities
# docker run --cap-drop ALL --cap-add NET_BIND_SERVICE my-image

# 5. No privileged mode
# NEVER: docker run --privileged (gives root access to host)

# 6. Minimize image size (smaller attack surface)
FROM python:3.11-slim      # ✅ Minimal base
FROM python:3.11           # ❌ Includes unnecessary tools

# 7. Copy only what's needed (use .dockerignore)
# 8. Don't install unnecessary packages
RUN apt-get install -y --no-install-recommends python3   # ✅
RUN apt-get install -y python3                            # ❌
```

---

## 14. Docker Health Checks & Logging

### Health Checks

```dockerfile
# In Dockerfile
HEALTHCHECK --interval=30s \    # Check every 30 seconds
            --timeout=10s \     # Wait 10 seconds for response
            --start-period=40s \ # Wait 40s before first check (startup grace)
            --retries=3 \       # Mark unhealthy after 3 consecutive failures
    CMD curl -f http://localhost:5000/health || exit 1
    # exit 0 = healthy, exit 1 = unhealthy
```

```bash
# Check container health status
docker inspect --format='{{.State.Health.Status}}' flask-app
# Output: healthy / unhealthy / starting

# View health check history
docker inspect --format='{{json .State.Health}}' flask-app | python3 -m json.tool
```

### Logging

```bash
# View logs
docker logs flask-app
docker logs -f flask-app           # Follow (real-time)
docker logs --tail 50 flask-app    # Last 50 lines
docker logs --since 1h flask-app   # Logs from last 1 hour
docker logs --since 2024-01-01T00:00:00 flask-app  # Since timestamp

# Log drivers
# Configure in daemon.json or per-container
docker run \
    --log-driver=awslogs \
    --log-opt awslogs-group=/ecs/my-app \
    --log-opt awslogs-region=us-east-1 \
    --log-opt awslogs-stream=my-container \
    my-flask-app:v1.0
```

**Configure global log driver `/etc/docker/daemon.json`:**
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

```bash
# Restart Docker to apply
sudo systemctl restart docker
```

---

## 15. Docker Swarm (Orchestration Basics)

Docker Swarm is Docker's built-in container orchestration. It manages clusters of Docker nodes.

```
                ┌─────────────────┐
                │   Swarm Manager  │  ← Orchestrates everything
                │   (Leader node)  │
                └────────┬────────┘
           ┌─────────────┼─────────────┐
           ▼             ▼             ▼
    ┌──────────┐  ┌──────────┐  ┌──────────┐
    │ Worker 1 │  │ Worker 2 │  │ Worker 3 │
    │ (2 tasks)│  │ (2 tasks)│  │ (2 tasks)│
    └──────────┘  └──────────┘  └──────────┘
```

```bash
# Initialize swarm on manager node
docker swarm init --advertise-addr 192.168.1.100

# Join worker nodes (run on worker machines with token from above command)
docker swarm join --token SWMTKN-1-xxxxx 192.168.1.100:2377

# Deploy a stack (multi-service app) to Swarm
docker stack deploy -c docker-compose.yml my-app-stack

# List services in stack
docker stack services my-app-stack

# Scale a service
docker service scale my-app-stack_web=5

# Rolling update (zero-downtime deployment)
docker service update \
    --image my-flask-app:v2.0 \
    --update-parallelism 1 \      # Update 1 container at a time
    --update-delay 10s \          # Wait 10s between updates
    my-app-stack_web

# List nodes in swarm
docker node ls
```

---

## 16. Docker + AWS ECS (Elastic Container Service)

### ECS Architecture

```
Internet → ALB (Application Load Balancer)
               ↓
         ECS Cluster (Fargate)
               ↓
    ┌──────────────────────┐
    │  ECS Service         │
    │  ┌────────────────┐  │
    │  │ Task (container)│  │
    │  │  Flask App      │  │
    │  │  Port 5000      │  │
    │  └────────────────┘  │
    │  ┌────────────────┐  │
    │  │ Task (container)│  │  ← desired count = 2
    │  │  Flask App      │  │
    │  │  Port 5000      │  │
    │  └────────────────┘  │
    └──────────────────────┘
               ↓
         Amazon ECR (Image Registry)
         Amazon RDS (Database)
         AWS Secrets Manager (Secrets)
         CloudWatch Logs (Logging)
```

### ECS Launch Types

| Feature       | EC2 Launch Type                    | Fargate Launch Type               |
|---------------|-------------------------------------|-----------------------------------|
| Management    | You manage EC2 instances            | AWS manages infrastructure        |
| Cost          | Pay for EC2 instances               | Pay per task (vCPU + memory)      |
| Scaling       | Manual EC2 scaling + task scaling   | Automatic                         |
| Use case      | Large workloads, GPU, custom AMIs   | Most workloads, serverless        |
| Networking    | EC2 networking                      | ENI per task (awsvpc)             |

### ECS Auto Scaling

```bash
# Register scalable target
aws application-autoscaling register-scalable-target \
    --service-namespace ecs \
    --scalable-dimension ecs:service:DesiredCount \
    --resource-id service/my-app-cluster/flask-app-service \
    --min-capacity 2 \
    --max-capacity 10

# Scale based on CPU utilization
aws application-autoscaling put-scaling-policy \
    --service-namespace ecs \
    --scalable-dimension ecs:service:DesiredCount \
    --resource-id service/my-app-cluster/flask-app-service \
    --policy-name cpu-scaling \
    --policy-type TargetTrackingScaling \
    --target-tracking-scaling-policy-configuration '{
        "PredefinedMetricSpecification": {
            "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
        },
        "TargetValue": 70.0,
        "ScaleInCooldown": 60,
        "ScaleOutCooldown": 60
    }'
```

---

## 17. Docker + AWS EKS (Kubernetes on AWS)

### EKS Overview

EKS is AWS's managed Kubernetes service. Docker containers run as Kubernetes Pods.

```bash
# Install eksctl
curl --silent --location \
    "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
    | tar xz -C /usr/local/bin

# Create EKS cluster
eksctl create cluster \
    --name my-cluster \
    --region us-east-1 \
    --nodegroup-name standard-workers \
    --node-type t3.medium \
    --nodes 3 \
    --nodes-min 2 \
    --nodes-max 5 \
    --managed

# Configure kubectl
aws eks update-kubeconfig --name my-cluster --region us-east-1

# Deploy your Docker image to EKS
```

**Kubernetes deployment manifest:**
```yaml
# k8s-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
  labels:
    app: flask-app
spec:
  replicas: 3                          # Run 3 pods
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
    spec:
      containers:
      - name: flask-app
        # Your Docker image from ECR
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-flask-app:v1.0
        ports:
        - containerPort: 5000
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        env:
        - name: ENVIRONMENT
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-url
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: flask-app-service
spec:
  selector:
    app: flask-app
  ports:
  - port: 80
    targetPort: 5000
  type: LoadBalancer    # Creates AWS ALB automatically
```

```bash
# Apply to cluster
kubectl apply -f k8s-deployment.yml

# Check pods
kubectl get pods
kubectl describe pod flask-app-xxxxx

# Check service and get ALB URL
kubectl get service flask-app-service

# Rolling update
kubectl set image deployment/flask-app \
    flask-app=123456789012.dkr.ecr.us-east-1.amazonaws.com/my-flask-app:v2.0

# Watch rollout
kubectl rollout status deployment/flask-app

# Rollback if needed
kubectl rollout undo deployment/flask-app
```

---

## 18. CI/CD Pipeline: GitHub Actions + ECR + ECS

### Full Automated Pipeline

Every git push triggers: Test → Build → Push to ECR → Deploy to ECS

**.github/workflows/deploy.yml:**
```yaml
name: Build and Deploy to AWS ECS

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: my-flask-app
  ECS_SERVICE: flask-app-service
  ECS_CLUSTER: my-app-cluster
  CONTAINER_NAME: flask-app

jobs:
  # ── Job 1: Test ──────────────────────────────────────────────
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Run tests
        run: pytest tests/ -v --tb=short

      - name: Run linter
        run: flake8 app/

  # ── Job 2: Build and Push to ECR ────────────────────────────
  build-and-push:
    needs: test          # Only runs if test job passes
    runs-on: ubuntu-latest
    outputs:
      image: ${{ steps.build-image.outputs.image }}

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image to ECR
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          # Build Docker image
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .

          # Tag as latest
          docker tag $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG \
                     $ECR_REGISTRY/$ECR_REPOSITORY:latest

          # Push both tags
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest

          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

  # ── Job 3: Deploy to ECS ─────────────────────────────────────
  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: production    # Requires manual approval in GitHub

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Download task definition
        run: |
          aws ecs describe-task-definition \
              --task-definition flask-app-task \
              --query taskDefinition > task-definition.json

      - name: Update ECS task definition with new image
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-definition.json
          container-name: ${{ env.CONTAINER_NAME }}
          image: ${{ needs.build-and-push.outputs.image }}

      - name: Deploy updated task definition to ECS
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true   # Wait until deployment is healthy
```

---

## 19. Troubleshooting & Bug Fixing

### Common Errors and Solutions

#### Error 1: `docker: Cannot connect to the Docker daemon`

```bash
# Error
docker ps
# Error response from daemon: Cannot connect to the Docker daemon at unix:///var/run/docker.sock

# Diagnosis
sudo systemctl status docker

# Fix: Start Docker service
sudo systemctl start docker

# Fix: If Docker socket permissions issue
sudo chmod 666 /var/run/docker.sock
# Better fix: add user to docker group
sudo usermod -aG docker $USER && newgrp docker
```

#### Error 2: Port Already in Use

```bash
# Error
docker run -p 8080:80 nginx
# Error: Bind for 0.0.0.0:8080 failed: port is already allocated

# Find what's using the port
sudo lsof -i :8080
sudo ss -tlnp | grep 8080
# or
docker ps  # Check if another container is already using it

# Fix: Stop the conflicting container
docker stop $(docker ps -q --filter publish=8080)

# Or use a different host port
docker run -p 9090:80 nginx
```

#### Error 3: Container Exits Immediately

```bash
# Container stops as soon as you start it
docker run my-app
docker ps -a  # Shows container in 'Exited' state

# Diagnosis: Check the logs
docker logs <container-id>

# Common causes and fixes:
# 1. CMD is wrong or app crashes at startup
#    Fix: Check logs, fix the application error

# 2. CMD runs as background process (daemon)
#    Problem Dockerfile: CMD ["nginx", "-g", "daemon on;"]
#    Fix Dockerfile:     CMD ["nginx", "-g", "daemon off;"]  # Run in foreground

# 3. Missing environment variable
docker run -e REQUIRED_VAR=value my-app

# Debug: Override CMD to get a shell
docker run -it --entrypoint bash my-app
docker run -it --entrypoint sh my-app   # for Alpine
```

#### Error 4: `No space left on device`

```bash
# Check Docker disk usage
docker system df

# Clean up
docker system prune                  # Remove unused containers, networks, images
docker system prune -a               # Also remove unused images
docker volume prune                  # Remove unused volumes
docker builder prune                 # Clear build cache

# Check disk space
df -h
du -sh /var/lib/docker               # Docker data directory size
```

#### Error 5: Image Not Found / Pull Failures

```bash
# Error: Unable to find image 'myapp:latest' locally
# Error: pull access denied

# Fix 1: Check image name and tag
docker images                        # List available images
docker search <image-name>           # Search Docker Hub

# Fix 2: Login if it's a private registry
docker login                         # Docker Hub
aws ecr get-login-password | docker login --username AWS --password-stdin ACCOUNT.dkr.ecr.REGION.amazonaws.com

# Fix 3: Check network connectivity
curl -I https://registry-1.docker.io/v2/
```

#### Error 6: Container Can't Connect to Database

```bash
# Error inside container: Connection refused to database:5432

# Diagnosis
docker network ls
docker network inspect my-network    # Check if both containers are on same network

# Check if database container is healthy
docker inspect db-container | grep -A 10 '"Health"'

# Fix 1: Make sure containers are on the same network
docker network connect my-network app-container
docker network connect my-network db-container

# Fix 2: Use container NAME as hostname (not IP or localhost)
# ❌ Wrong:  DATABASE_URL=postgresql://localhost:5432/db
# ✅ Right:  DATABASE_URL=postgresql://db-container:5432/db

# Fix 3: Database isn't ready yet — add startup wait
# Add health check to db service in docker-compose.yml
# Use depends_on with condition: service_healthy
```

#### Error 7: Out of Memory (OOM Kill)

```bash
# Container keeps restarting with code 137 (OOM killed)

# Check
docker inspect <container> | grep -i oom
docker stats --no-stream

# Fix: Set appropriate memory limits
docker run --memory=512m --memory-swap=512m my-app

# In docker-compose.yml:
services:
  web:
    deploy:
      resources:
        limits:
          memory: 512M
```

#### Error 8: Permission Denied on Volume Mount

```bash
# Error: PermissionError: [Errno 13] Permission denied: '/app/data/file'

# Cause: Container running as non-root but host directory is owned by root

# Fix 1: Match user IDs
docker run -u $(id -u):$(id -g) -v $(pwd)/data:/app/data my-app

# Fix 2: Change ownership of host directory
sudo chown -R 1001:1001 ./data     # Match container user ID
chmod 755 ./data

# Fix 3: In Dockerfile, create directory with correct permissions
RUN mkdir -p /app/data && chown -R appuser:appuser /app/data
```

### Debugging Commands Quick Reference

```bash
# Get a shell in a running container
docker exec -it <container> bash

# Get a shell in a stopped/crashed container
docker run -it --entrypoint bash <image>

# Check resource usage
docker stats

# Inspect full container details
docker inspect <container> | python3 -m json.tool

# Check network connectivity from inside container
docker exec -it <container> wget -qO- http://other-service:port/health

# View all events
docker events --since 1h

# Check container processes
docker top <container>
```

---

## 20. Best Practices

### Dockerfile Best Practices

```dockerfile
# ✅ 1. Use specific version tags
FROM python:3.11.7-slim-bookworm    # Pinned and minimal

# ✅ 2. Combine RUN commands to reduce layers
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
       curl \
       libpq5 \
    && rm -rf /var/lib/apt/lists/*    # Clean apt cache in same layer

# ✅ 3. Order layers by change frequency (least → most)
COPY requirements.txt .              # Changes rarely
RUN pip install -r requirements.txt  # Cached if requirements.txt unchanged
COPY . .                             # Changes frequently (so it's last)

# ✅ 4. Use .dockerignore
# ✅ 5. Run as non-root
# ✅ 6. Use multi-stage builds for production
# ✅ 7. Set WORKDIR explicitly
# ✅ 8. Add HEALTHCHECK
# ✅ 9. Use COPY not ADD (unless you need URL or tar features)
# ✅ 10. Pin base image digest for maximum reproducibility
FROM python:3.11.7-slim-bookworm@sha256:abc123...
```

### General Docker Best Practices

```
1. One process per container (separation of concerns)
2. Keep containers stateless (store state in databases/volumes)
3. Use environment variables for configuration
4. Never store secrets in images
5. Log to stdout/stderr (let orchestrator handle logs)
6. Make containers ephemeral (can be stopped/replaced at any time)
7. Use health checks in production
8. Scan images for vulnerabilities before deploying
9. Tag images with commit SHA for traceability
10. Use resource limits (CPU, memory) in production
```

### AWS-Specific Best Practices

```
1. Use ECR private repositories (not Docker Hub in prod)
2. Enable ECR image scanning on push
3. Use Fargate to avoid managing EC2 for containers
4. Store secrets in AWS Secrets Manager, not env vars
5. Use CloudWatch Container Insights for monitoring
6. Enable ECS task-level IAM roles (least privilege)
7. Use VPC with private subnets for ECS tasks
8. Put ALB in public subnet, ECS tasks in private subnet
9. Use ECR lifecycle policies to clean old images
10. Use ECS Service Connect or AWS Cloud Map for service discovery
```

---

## 21. Assignments

### Assignment 1 — Basic (Beginner)

**Task:** Create and run your first custom Docker container

1. Write a simple Python script `hello.py` that prints your name and the current date
2. Write a `Dockerfile` to run this script using `python:3.11-slim`
3. Build the image tagged as `hello-docker:v1`
4. Run the container and verify output
5. Check the image size with `docker images`

**Expected commands:**
```bash
docker build -t hello-docker:v1 .
docker run hello-docker:v1
docker images hello-docker
```

---

### Assignment 2 — Intermediate

**Task:** Multi-container application with Docker Compose

1. Create a Flask TODO app with endpoints: `GET /todos`, `POST /todos`, `DELETE /todos/<id>`
2. Use PostgreSQL as the database
3. Write a `docker-compose.yml` with services: `web`, `db`
4. Add a health check to the database service
5. Use named volumes for database persistence
6. Test that data persists after `docker compose restart`

**Bonus:** Add Redis for caching GET requests

---

### Assignment 3 — Advanced

**Task:** Production-grade deployment to AWS

1. Create a multi-stage Dockerfile for a Node.js or Python app
2. Set up Amazon ECR repository
3. Push your image to ECR
4. Create an ECS Fargate task definition using:
   - AWS Secrets Manager for database credentials
   - CloudWatch Logs for logging
   - Health check endpoint
5. Create an ECS service with:
   - 2 desired tasks
   - ALB for load balancing
   - Auto-scaling based on CPU > 70%
6. Set up a GitHub Actions workflow that:
   - Runs tests on every push
   - Builds and pushes to ECR on merge to main
   - Deploys to ECS automatically

---

### Assignment 4 — Troubleshooting Challenge

Debug and fix these intentionally broken Dockerfiles:

**Broken Dockerfile 1** — Find 5 issues:
```dockerfile
FROM ubuntu:latest
RUN apt-get install python3
WORKDIR app
COPY . .
RUN pip3 install flask
EXPOSE 5000
CMD python3 app.py
```

**Broken Dockerfile 2** — Find 3 issues:
```dockerfile
FROM node:latest
COPY . .
RUN npm install
USER root
CMD ["npm", "start"]
```

<details>
<summary>Answers (click to reveal)</summary>

**Dockerfile 1 fixes:**
1. `ubuntu:latest` → `ubuntu:22.04` (pin version)
2. `apt-get install` → `apt-get update && apt-get install -y --no-install-recommends` (update first, no cache)
3. `WORKDIR app` → `WORKDIR /app` (use absolute path)
4. Copy `requirements.txt` BEFORE source code for caching
5. `CMD python3 app.py` → `CMD ["python3", "app.py"]` (use JSON array form)

**Dockerfile 2 fixes:**
1. `node:latest` → `node:20-alpine` (pin version, smaller image)
2. Copy `package*.json` first, `npm install`, THEN `COPY . .` (cache optimization)
3. `USER root` → `USER node` (never run as root; use node's built-in user)
</details>

---

## 22. Interview Questions & Answers

### Beginner Level

**Q1: What is Docker and why is it used?**
> Docker is an open-source containerization platform that packages applications and their dependencies into isolated containers. It solves the "works on my machine" problem by ensuring consistent environments across development, testing, and production. Containers are lightweight (share host OS kernel), portable, and start in milliseconds.

**Q2: What is the difference between a Docker image and a container?**
> An **image** is a read-only template (like a blueprint or class). A **container** is a running instance of an image (like an object). You can have one image and run many containers from it. Images are layered and immutable; containers add a thin writable layer on top.

**Q3: What is Docker Hub?**
> Docker Hub is the default public container registry at hub.docker.com. It hosts official images (nginx, python, postgres) and community images. In AWS production environments, you'd use Amazon ECR instead for private, secure image storage.

**Q4: Explain Docker networking modes.**
> - **bridge** (default): Containers get their own network namespace, can communicate via container names on the same bridge network
> - **host**: Container shares the host's network namespace (no isolation, better performance)
> - **none**: No network access — maximum isolation
> - **overlay**: Multi-host networking for Docker Swarm clusters

**Q5: What is the difference between CMD and ENTRYPOINT?**
> `CMD` provides a default command that can be **overridden** at `docker run`. `ENTRYPOINT` sets the main executable that always runs. When both are used, CMD provides default **arguments** to ENTRYPOINT. Best practice: `ENTRYPOINT ["executable"]`, `CMD ["default-arg"]`.

---

### Intermediate Level

**Q6: How do Docker layers and caching work?**
> Each Dockerfile instruction creates a new read-only layer. Docker caches layers — if a layer hasn't changed, Docker reuses the cached version instead of rebuilding. Cache is invalidated when a layer changes, and all subsequent layers must be rebuilt. This is why you should order Dockerfile instructions from least-changed to most-changed (dependencies before source code).

**Q7: What is a multi-stage build and when would you use it?**
> Multi-stage builds use multiple `FROM` statements in a single Dockerfile. Each stage can be named with `AS`. You copy artifacts from build stages into smaller final stages using `COPY --from=stage-name`. Use case: compile a Go binary in a golang image (800MB), copy only the binary into a scratch image (10MB). This dramatically reduces final image size and attack surface.

**Q8: How do you manage secrets in Docker?**
> Never bake secrets into images. Options:
> 1. **Runtime env vars**: `docker run -e SECRET=value` (visible in process list)
> 2. **Docker Secrets** (Swarm): Mounted as files in `/run/secrets/` — secure
> 3. **AWS Secrets Manager**: App fetches secrets at startup using boto3
> 4. **ECS secrets**: Reference Secrets Manager ARN in task definition — injected as env vars by ECS, never in image
> 5. **.env files**: Pass to `docker run --env-file .env` (but never commit .env to git)

**Q9: What's the difference between COPY and ADD?**
> `COPY` simply copies files from host to container. `ADD` also supports: downloading from URLs and automatically extracting tar archives. Best practice: use `COPY` by default; use `ADD` only when you specifically need URL downloads or tar auto-extraction. `COPY` is more explicit and predictable.

**Q10: How does Docker Compose differ from Docker CLI?**
> Docker CLI manages individual containers. Docker Compose manages multi-container applications defined in a YAML file. Compose handles service dependencies, networking, volumes, and scaling in a declarative way. One `docker compose up` replaces multiple `docker run` commands with complex flags.

---

### Advanced Level

**Q11: Explain Docker's overlay file system.**
> Docker uses a union file system (OverlayFS on Linux). Images consist of stacked read-only layers. When a container runs, a thin writable layer is added on top. When a file is modified, it's copied to the writable layer (copy-on-write). Deleting a layer file adds a "whiteout" file. This allows multiple containers to share the same image layers while having independent writable state.

**Q12: How do you achieve zero-downtime deployments with Docker?**
> 1. **Docker Swarm**: `docker service update --update-parallelism 1 --update-delay 10s` — updates one container at a time
> 2. **ECS**: Rolling update strategy — new tasks start, pass health checks, then old tasks stop
> 3. **Kubernetes**: Rolling update — configures `maxUnavailable` and `maxSurge`
> 4. **Blue-Green**: Deploy new version in parallel, switch load balancer when healthy, keep old version for instant rollback

**Q13: What is the difference between ECS EC2 and ECS Fargate?**
> - **EC2**: You provision and manage EC2 instances; more control, supports GPU, cheaper for large scale
> - **Fargate**: Serverless — AWS manages infrastructure; you define CPU/memory per task; pay per task; no EC2 to manage; better security (each task has its own kernel with Firecracker microVM)

**Q14: How would you debug a container that keeps restarting?**
> 1. `docker ps -a` — check exit code (137=OOM, 1=app crash, 0=process exited normally)
> 2. `docker logs <container>` — check last logs before crash
> 3. `docker inspect <container>` — check health check, restart policy, OOM kill status
> 4. Override entrypoint: `docker run -it --entrypoint bash <image>` — manually investigate
> 5. Check resource limits: `docker stats` — is it hitting memory limits?
> 6. Check exit code meaning: 127=command not found, 126=permission denied

**Q15: How does Docker integrate with AWS IAM?**
> - **ECR Authentication**: `aws ecr get-login-password` generates a 12-hour token; the ECS task execution role automatically pulls from ECR without any credentials
> - **ECS Task Role**: IAM role attached to ECS task definition; containers can call AWS APIs (S3, DynamoDB, Secrets Manager) without storing credentials — uses EC2 instance metadata / ECS credentials endpoint
> - **Fargate**: Task credentials are isolated per task using dedicated credential endpoints
> - **Principle of least privilege**: Create specific IAM policies for each application

**Q16: What is Docker BuildKit and how does it improve builds?**
> BuildKit is Docker's next-generation build engine (default since Docker 23.0). Improvements:
> - **Parallel builds**: Independent stages build in parallel
> - **Secret mounts**: `RUN --mount=type=secret,id=mykey` — secret available during build but NOT stored in layer
> - **SSH agent forwarding**: `RUN --mount=type=ssh` — clone private repos without embedding keys
> - **Build cache export**: Share cache between CI runs using `--cache-to/--cache-from`
> - **Better caching**: Detects file changes more accurately

```dockerfile
# BuildKit secret mount example
# docker build --secret id=pip_conf,src=pip.conf .
RUN --mount=type=secret,id=pip_conf,dst=/etc/pip.conf \
    pip install --no-cache-dir -r requirements.txt
# pip.conf is available during this RUN but NOT in the final image
```

---

### Scenario-Based Questions

**Q17: Your container works locally but fails in ECS. How do you troubleshoot?**
> 1. Check ECS task stopped reason: AWS Console → ECS → Task → Events/Details
> 2. Check CloudWatch Logs for application errors
> 3. Verify environment variables/secrets are correctly configured in task definition
> 4. Check IAM roles: task execution role (ECR pull, CloudWatch), task role (app permissions)
> 5. Verify security group allows inbound traffic on container port
> 6. Check if image was successfully pushed to ECR and task definition references correct tag
> 7. Run task locally with same environment: `docker run -e VAR=value image:tag`

**Q18: How would you reduce a Docker image from 1GB to under 100MB?**
> 1. Switch base image: `ubuntu` → `python:3.11-slim` or `python:3.11-alpine`
> 2. Multi-stage build: separate build tools from runtime
> 3. Combine RUN commands: fewer layers, clean apt cache in same layer
> 4. Use `--no-install-recommends` with apt
> 5. Use `pip install --no-cache-dir`
> 6. Remove build tools: gcc, make, headers after compilation
> 7. Add proper `.dockerignore` to exclude tests, docs, .git
> 8. Analyze layers: `docker history image` or use `dive` tool

**Q19: How do you handle database migrations in containers?**
> Options:
> 1. **Init container** (Kubernetes): Run migration as separate pod that completes before app starts
> 2. **ECS**: Run a separate one-off ECS task for migrations before deploying new app version
> 3. **Application startup**: App runs migrations on startup (acceptable for small apps)
> 4. **Separate migration container** in compose with `depends_on`
> Best practice: Migrations should be idempotent (safe to run multiple times). Never run migrations and new app deployment simultaneously.

---

## Quick Reference Card

```
┌──────────────────────────────────────────────────────────────┐
│                    DOCKER CHEAT SHEET                         │
├──────────────────────────────────────────────────────────────┤
│ docker build -t name:tag .          Build image               │
│ docker run -d -p h:c --name n img   Run container             │
│ docker ps / ps -a                   List containers           │
│ docker logs -f <container>          Follow logs               │
│ docker exec -it <container> bash    Shell access              │
│ docker stop / rm <container>        Stop / delete             │
│ docker images                       List images               │
│ docker rmi <image>                  Delete image              │
│ docker system prune -a              Clean everything          │
├──────────────────────────────────────────────────────────────┤
│ docker compose up -d                Start all services        │
│ docker compose down                 Stop and remove           │
│ docker compose logs -f web          Follow service logs       │
│ docker compose exec web bash        Shell in service          │
├──────────────────────────────────────────────────────────────┤
│ AWS ECR:                                                       │
│ aws ecr get-login-password | docker login --username AWS \    │
│   --password-stdin ACCT.dkr.ecr.REGION.amazonaws.com         │
│ docker tag app:v1 ACCT.dkr.ecr.REGION.amazonaws.com/app:v1  │
│ docker push ACCT.dkr.ecr.REGION.amazonaws.com/app:v1         │
└──────────────────────────────────────────────────────────────┘
```

---

*Docker Trainer & AWS DevOps Mentor — Continuously updated*
*Last updated: June 2026*
