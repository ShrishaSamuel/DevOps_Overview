# JFrog — Zero to Advanced: Complete Training Guide
### Your JFrog Trainer & AWS DevOps Mentor

> **Who is this for?** Absolute beginners to advanced practitioners.
> **What you'll learn?** JFrog Artifactory, Xray, Pipelines, CLI, AWS integration, real projects, troubleshooting, best practices, assignments, and interview prep.

---

## Table of Contents

1. [What is JFrog?](#1-what-is-jfrog)
2. [JFrog Platform Components](#2-jfrog-platform-components)
3. [Core Concepts — Repositories & Artifacts](#3-core-concepts--repositories--artifacts)
4. [JFrog Artifactory — Deep Dive](#4-jfrog-artifactory--deep-dive)
5. [Installing JFrog Locally (Docker)](#5-installing-jfrog-locally-docker)
6. [JFrog CLI — Setup & Commands](#6-jfrog-cli--setup--commands)
7. [Repository Types — Local, Remote, Virtual](#7-repository-types--local-remote-virtual)
8. [Working with Package Managers](#8-working-with-package-managers)
9. [JFrog Xray — Security & Compliance](#9-jfrog-xray--security--compliance)
10. [JFrog Pipelines — CI/CD](#10-jfrog-pipelines--cicd)
11. [Real-Time Project Workflow](#11-real-time-project-workflow)
12. [AWS Integration with JFrog](#12-aws-integration-with-jfrog)
13. [GitLab CI/CD + JFrog Integration](#13-gitlab-cicd--jfrog-integration)
14. [REST API — Automate Everything](#14-rest-api--automate-everything)
15. [Permissions, Users & Groups](#15-permissions-users--groups)
16. [Build Info & Traceability](#16-build-info--traceability)
17. [High Availability & Disaster Recovery](#17-high-availability--disaster-recovery)
18. [Troubleshooting & Bug Fixing](#18-troubleshooting--bug-fixing)
19. [Best Practices](#19-best-practices)
20. [Assignments](#20-assignments)
21. [Interview Questions & Answers](#21-interview-questions--answers)

---

## 1. What is JFrog?

### Plain English Definition

JFrog is a **Universal Artifact Management Platform**. Think of it as a super-powered storage and delivery system for all your software build outputs (binaries, Docker images, npm packages, Maven JARs, Python wheels, Helm charts, etc.).

### Why do you need it?

In a DevOps pipeline:

```
Developer writes code
     │
     ▼
CI/CD builds the code  →  produces artifacts (JAR, Docker image, .whl, etc.)
     │
     ▼
Where do you STORE these artifacts securely?
     │
     ▼
Where do you SCAN them for vulnerabilities?
     │
     ▼
How do you DISTRIBUTE them to servers/environments?
     │
     ▼
  ← JFrog answers ALL these questions →
```

### JFrog vs Other Tools

| Feature                | JFrog Artifactory | Nexus Repository | AWS CodeArtifact |
|------------------------|-------------------|------------------|------------------|
| Universal package types| ✅ 30+             | ✅ ~20            | ✅ ~10            |
| Built-in Security Scan | ✅ Xray            | ❌ (plugin)       | ❌                |
| CI/CD pipelines        | ✅ JFrog Pipelines | ❌                | ❌                |
| Cloud/Self-hosted       | Both              | Both             | Cloud only        |
| Free tier              | ✅ (OSS/Cloud free)| ✅                | ✅                |

### Real-World Analogy

> JFrog Artifactory = A highly organized, secure **warehouse** for all your software packages.
> - Local repos = Your own shelves
> - Remote repos = External supplier shelves you proxy through your warehouse
> - Virtual repos = A single door that gives access to multiple shelves

---

## 2. JFrog Platform Components

```
┌─────────────────────────────────────────────────────┐
│                  JFrog Platform                      │
│                                                     │
│  ┌─────────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ Artifactory │  │  Xray    │  │   Pipelines   │  │
│  │ (Storage)   │  │(Security)│  │   (CI/CD)     │  │
│  └─────────────┘  └──────────┘  └───────────────┘  │
│                                                     │
│  ┌─────────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ Distribution│  │  Curation│  │   Catalog     │  │
│  │ (Release)   │  │(OSS Mgmt)│  │  (Metadata)   │  │
│  └─────────────┘  └──────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────┘
```

| Component         | Purpose                                                       |
|-------------------|---------------------------------------------------------------|
| **Artifactory**   | Universal binary repository manager — store, manage, retrieve |
| **Xray**          | Deep recursive security scanning of artifacts and dependencies |
| **Pipelines**     | Native CI/CD pipeline orchestration                          |
| **Distribution**  | Distribute release bundles to remote edges securely           |
| **Curation**      | Automated OSS package curation and approval workflows         |
| **Catalog**       | Centralized metadata catalog for all software components      |

---

## 3. Core Concepts — Repositories & Artifacts

### What is an Artifact?

An artifact is **any file produced by a build process**:
- Java → `.jar`, `.war`, `.ear`
- Python → `.whl`, `.tar.gz`
- Docker → image layers
- Node.js → `.tgz` (npm package)
- Helm → `.tgz` chart
- Generic → `.zip`, `.rpm`, `.deb`, binary

### What is a Repository?

A repository is a **named storage location** in Artifactory that holds artifacts of a specific package type.

```
Artifactory
├── libs-release-local      (Maven local, stable releases)
├── libs-snapshot-local     (Maven local, dev snapshots)
├── npm-local               (npm local packages)
├── docker-local            (Docker images)
├── npm-remote              (proxies registry.npmjs.org)
├── pypi-remote             (proxies pypi.org)
└── libs-virtual            (virtual — merges local + remote)
```

### Repository Naming Convention (Best Practice)

```
<team>-<type>-<packagemanager>-<local|remote|virtual>

Examples:
  myapp-release-maven-local
  myapp-snapshot-maven-local
  myapp-docker-local
  central-maven-remote
  npm-virtual
```

---

## 4. JFrog Artifactory — Deep Dive

### Artifactory Web UI Tour

After login at `http://localhost:8082/ui`:

```
Left Sidebar:
  Packages     → Browse all artifacts by package type
  Builds       → View build info, dependencies, and history
  Xray         → Security violations and license compliance
  Admin        → Users, repos, permissions, config

Top Bar:
  Search bar   → Global artifact search
  Set Me Up    → Auto-generates config snippets for your package manager
```

### How Artifact Resolution Works

```
Developer runs: mvn install
        │
        ▼
Maven looks for dependencies
        │
        ▼
Checks Virtual Repo (libs-virtual)
        │
        ├── Checks Local Repo (libs-release-local) ── Found? Return it.
        │
        └── Checks Remote Repo (central-remote)
                │
                Fetches from Maven Central, CACHES in Artifactory
                │
                Returns artifact to developer
```

> Next time anyone needs the same artifact → served from Artifactory cache. No internet needed!

---

## 5. Installing JFrog Locally (Docker)

### Prerequisites

- Docker installed
- At least 4 GB RAM free

### Step 1 — Pull and Run Artifactory OSS

```bash
# Pull the JFrog Artifactory OSS (Open Source) image
docker pull releases-docker.jfrog.io/jfrog/artifactory-oss:latest

# Create a data directory so your data persists across container restarts
mkdir -p $HOME/artifactory/var/etc

# Set correct permissions (JFrog runs as UID 1030)
chown -R 1030:1030 $HOME/artifactory

# Run the container
docker run -d \
  --name artifactory \
  -p 8081:8081 \
  -p 8082:8082 \
  -v $HOME/artifactory/var:/var/opt/jfrog/artifactory \
  releases-docker.jfrog.io/jfrog/artifactory-oss:latest
```

**Line-by-line explanation:**

| Line | Explanation |
|------|-------------|
| `docker run -d` | Run in detached (background) mode |
| `--name artifactory` | Name the container "artifactory" |
| `-p 8081:8081` | Map host port 8081 → container 8081 (REST API) |
| `-p 8082:8082` | Map host port 8082 → container 8082 (Web UI) |
| `-v $HOME/artifactory/var:/var/opt/jfrog/artifactory` | Mount local dir for persistent data |

### Step 2 — First-Time Login

```
URL:      http://localhost:8082/ui
Username: admin
Password: password
```

> You will be prompted to change the password on first login.

### Step 3 — Verify Container is Running

```bash
# Check container status
docker ps | grep artifactory

# View logs (useful for troubleshooting startup)
docker logs -f artifactory

# Look for this line in logs — means it's ready:
# ###########################################################
# ###   All Services started successfully in XX.XXX seconds ###
# ###########################################################
```

### Step 4 — Using Docker Compose (Recommended for Dev)

```yaml
# docker-compose.yml
version: '3.8'

services:
  artifactory:
    image: releases-docker.jfrog.io/jfrog/artifactory-oss:latest
    container_name: artifactory
    ports:
      - "8081:8081"   # REST API port
      - "8082:8082"   # Web UI / Reverse proxy port
    volumes:
      - artifactory_data:/var/opt/jfrog/artifactory
    environment:
      - JF_SHARED_DATABASE_TYPE=derby   # Default embedded DB (fine for dev)
    restart: unless-stopped

volumes:
  artifactory_data:
    driver: local
```

```bash
# Start with Docker Compose
docker-compose up -d

# Stop without deleting data
docker-compose stop

# Stop and remove containers (data volume persists)
docker-compose down
```

---

## 6. JFrog CLI — Setup & Commands

### Why use JFrog CLI?

The JFrog CLI (`jf`) is the **command-line interface** to interact with all JFrog services. Use it to:
- Upload/download artifacts
- Configure package managers (Maven, npm, Docker, pip)
- Trigger builds
- Scan with Xray
- Manage repositories

### Installation

```bash
# macOS (Homebrew)
brew install jfrog-cli

# Linux (one-liner install script)
curl -fL https://install-cli.jfrog.io | sh

# Verify installation
jf --version
# Expected: jf version 2.x.x ...

# Windows (Chocolatey)
choco install jfrog-cli
```

### Configure CLI to Talk to Your Artifactory

```bash
# Interactive setup wizard
jf config add

# You will be prompted:
# Server ID:    myserver          ← a local alias/nickname
# URL:          http://localhost:8082
# Username:     admin
# Password:     <your-password>
# Access Token: (leave blank if using user/pass)

# Verify configuration
jf config show

# Test connection
jf rt ping
# Expected: OK
```

### Essential JFrog CLI Commands

```bash
# ──────────────────────────────────────────────────────
# UPLOAD a file to Artifactory
# ──────────────────────────────────────────────────────
jf rt upload \
  "build/myapp-1.0.0.jar" \        # local file (supports wildcards)
  "libs-release-local/com/myapp/1.0.0/"  # target repo path

# Upload with properties (metadata tags)
jf rt upload \
  "dist/*.whl" \
  "pypi-local/" \
  --target-props "env=prod;approved=true"

# ──────────────────────────────────────────────────────
# DOWNLOAD an artifact
# ──────────────────────────────────────────────────────
jf rt download \
  "libs-release-local/com/myapp/1.0.0/myapp-1.0.0.jar" \
  "./downloads/"

# Download all JARs matching a pattern
jf rt download \
  "libs-release-local/com/myapp/*/myapp-*.jar" \
  "./downloads/"

# ──────────────────────────────────────────────────────
# SEARCH for artifacts
# ──────────────────────────────────────────────────────
jf rt search "libs-release-local/com/myapp/*"

# Search with property filter
jf rt search \
  --props "env=prod;approved=true" \
  "libs-release-local/*"

# ──────────────────────────────────────────────────────
# DELETE an artifact
# ──────────────────────────────────────────────────────
jf rt delete "libs-release-local/com/myapp/1.0.0/myapp-1.0.0.jar"

# ──────────────────────────────────────────────────────
# SET PROPERTIES on an artifact
# ──────────────────────────────────────────────────────
jf rt set-props \
  "libs-release-local/com/myapp/1.0.0/myapp-1.0.0.jar" \
  "status=approved;deployed-by=jenkins"

# ──────────────────────────────────────────────────────
# COPY artifact to another repo
# ──────────────────────────────────────────────────────
jf rt copy \
  "libs-snapshot-local/com/myapp/1.0.0-SNAPSHOT/" \
  "libs-release-local/com/myapp/1.0.0/"

# ──────────────────────────────────────────────────────
# MOVE artifact to another repo
# ──────────────────────────────────────────────────────
jf rt move \
  "staging-local/myapp-1.0.0.jar" \
  "release-local/myapp-1.0.0.jar"

# ──────────────────────────────────────────────────────
# BUILD INFO — publish build metadata
# ──────────────────────────────────────────────────────
jf rt build-publish myapp-build 42    # buildName=myapp-build, buildNumber=42

# ──────────────────────────────────────────────────────
# SCAN with Xray
# ──────────────────────────────────────────────────────
jf xr scan --watches my-security-watch --fail-no-op
```

---

## 7. Repository Types — Local, Remote, Virtual

### Local Repository

A repository that **Artifactory owns and stores directly**.

```bash
# Create a local Maven repository via REST API
curl -X PUT \
  -H "Content-Type: application/json" \
  -u admin:password \
  http://localhost:8081/artifactory/api/repositories/libs-release-local \
  -d '{
    "rclass": "local",
    "packageType": "maven",
    "description": "Local Maven release artifacts",
    "repoLayoutRef": "maven-2-default",
    "handleSnapshots": false,
    "handleReleases": true
  }'
```

**Field explanation:**

| Field | Meaning |
|-------|---------|
| `rclass` | Repository class: `local`, `remote`, or `virtual` |
| `packageType` | `maven`, `npm`, `docker`, `pypi`, `helm`, `generic`, etc. |
| `handleSnapshots` | Allow `-SNAPSHOT` versions? `false` for release repos |
| `handleReleases`  | Allow release versions? `true` for release repos |

### Remote Repository

A **proxy** to an external repository. Artifactory fetches from the external source and **caches** the artifact locally.

```bash
# Create a remote repo that proxies Maven Central
curl -X PUT \
  -H "Content-Type: application/json" \
  -u admin:password \
  http://localhost:8081/artifactory/api/repositories/central-remote \
  -d '{
    "rclass": "remote",
    "packageType": "maven",
    "url": "https://repo1.maven.org/maven2",
    "description": "Proxy to Maven Central",
    "repoLayoutRef": "maven-2-default",
    "retrievalCachePeriodSecs": 600
  }'
```

> `retrievalCachePeriodSecs`: How long (seconds) to cache the remote metadata locally before re-fetching.

### Virtual Repository

An **aggregation** of local and remote repos behind a single URL. Your package manager points here — it never needs to know where artifacts actually live.

```bash
# Create a virtual repo combining local + remote
curl -X PUT \
  -H "Content-Type: application/json" \
  -u admin:password \
  http://localhost:8081/artifactory/api/repositories/libs-virtual \
  -d '{
    "rclass": "virtual",
    "packageType": "maven",
    "description": "Virtual Maven repo — all sources",
    "repositories": [
      "libs-release-local",
      "libs-snapshot-local",
      "central-remote"
    ],
    "defaultDeploymentRepo": "libs-release-local"
  }'
```

> `defaultDeploymentRepo`: When you deploy/upload to the virtual repo, artifacts land here.

### Visual Summary

```
Developer/CI → Virtual Repo (single URL)
                    │
          ┌─────────┴──────────┐
          ▼                    ▼
     Local Repo           Remote Repo
  (your artifacts)    (proxied from internet)
                            │
                            ▼
                    External Registry
                  (Maven Central, PyPI, etc.)
```

---

## 8. Working with Package Managers

### 8.1 Maven Integration

```bash
# Step 1: Configure JFrog CLI for Maven
jf mvn-config \
  --server-id-resolve myserver \
  --repo-resolve-releases libs-release-local \
  --repo-resolve-snapshots libs-snapshot-local \
  --server-id-deploy myserver \
  --repo-deploy-releases libs-release-local \
  --repo-deploy-snapshots libs-snapshot-local
```

This creates a `.jfrog/projects/maven.yaml` in your project:

```yaml
# .jfrog/projects/maven.yaml — auto-generated by jf mvn-config
version: 1
type: maven
resolver:
  serverID: myserver
  releaseRepo: libs-release-local
  snapshotRepo: libs-snapshot-local
deployer:
  serverID: myserver
  releaseRepo: libs-release-local
  snapshotRepo: libs-snapshot-local
```

```bash
# Step 2: Build and deploy with JFrog CLI (tracks build info)
jf mvn clean install \
  --build-name=myapp-build \
  --build-number=42

# Step 3: Publish build info to Artifactory
jf rt build-publish myapp-build 42
```

Your `pom.xml` — no JFrog-specific changes needed:

```xml
<!-- pom.xml -->
<project>
  <groupId>com.mycompany</groupId>
  <artifactId>myapp</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>

  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
      <version>3.2.0</version>
    </dependency>
  </dependencies>
</project>
```

> JFrog CLI wraps Maven — your `pom.xml` stays clean. All Artifactory config lives in `.jfrog/`.

### 8.2 npm Integration

```bash
# Configure JFrog CLI for npm
jf npm-config \
  --server-id-resolve myserver \
  --repo-resolve npm-virtual \
  --server-id-deploy myserver \
  --repo-deploy npm-local

# Install packages (resolves through Artifactory)
jf npm install --build-name=myapp --build-number=1

# Publish package to Artifactory
jf npm publish --build-name=myapp --build-number=1

# Publish build info
jf rt build-publish myapp 1
```

```json
// package.json
{
  "name": "my-frontend-app",
  "version": "1.0.0",
  "dependencies": {
    "react": "^18.0.0",
    "axios": "^1.6.0"
  }
}
```

### 8.3 Docker Integration

```bash
# Step 1: Tag your image with the Artifactory Docker registry URL
# Format: <artifactory-host>/<repo-name>/<image-name>:<tag>

docker build -t myapp:latest .

docker tag myapp:latest \
  localhost:8082/docker-local/myapp:1.0.0

# Step 2: Login to Artifactory Docker registry
docker login localhost:8082 \
  --username admin \
  --password password

# Step 3: Push image via JFrog CLI (captures build info)
jf docker push \
  localhost:8082/docker-local/myapp:1.0.0 \
  --build-name=docker-build \
  --build-number=5

# Step 4: Pull image through Artifactory (uses virtual repo)
docker pull localhost:8082/docker-virtual/myapp:1.0.0

# Step 5: Publish build info
jf rt build-publish docker-build 5
```

### 8.4 Python (pip) Integration

```bash
# Configure JFrog CLI for pip
jf pip-config \
  --server-id-resolve myserver \
  --repo-resolve pypi-virtual

# Install Python packages through Artifactory
jf pip install -r requirements.txt \
  --build-name=python-build \
  --build-number=3

# Publish your Python package (wheel/sdist)
jf rt upload "dist/*.whl" pypi-local/

# Publish build info
jf rt build-publish python-build 3
```

### 8.5 Helm Charts

```bash
# Configure Helm to use Artifactory as chart museum
helm repo add myrepo \
  http://localhost:8081/artifactory/helm-virtual \
  --username admin \
  --password password

helm repo update

# Search charts
helm search repo myrepo/

# Push a chart to Artifactory
helm package ./mychart
jf rt upload "mychart-1.0.0.tgz" helm-local/

# Install chart from Artifactory
helm install myrelease myrepo/mychart --version 1.0.0
```

---

## 9. JFrog Xray — Security & Compliance

### What is Xray?

JFrog Xray is the **security and compliance engine** built into the JFrog platform. It:
- Scans all artifacts stored in Artifactory
- Performs **deep recursive scanning** (scans inside JARs, inside Docker layers, etc.)
- Identifies **CVEs** (Common Vulnerabilities and Exposures)
- Checks **OSS license compliance**
- Can **block** builds or downloads based on policy violations

### Xray Key Concepts

```
Watches  →  What to scan?  (repos, builds, release bundles)
Policies →  What are the rules?  (block on Critical CVE, deny GPL, etc.)
Rules    →  Conditions that trigger policy violations
```

### Create a Security Watch via UI

```
Admin → Xray → Watches → New Watch
  Name: my-security-watch
  Resources: Select repos → libs-release-local, docker-local
  Assigned Policies: my-security-policy
```

### Create a Security Policy (REST API)

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -u admin:password \
  http://localhost:8081/xray/api/v2/policies \
  -d '{
    "name": "block-critical-cves",
    "type": "security",
    "description": "Block artifacts with critical vulnerabilities",
    "rules": [
      {
        "name": "critical-cve-rule",
        "priority": 1,
        "criteria": {
          "min_severity": "Critical",
          "fix_version_dependant": false
        },
        "actions": {
          "webhooks": [],
          "mails": ["security-team@mycompany.com"],
          "fail_build": true,
          "block_download": {
            "active": true,
            "unscanned": false
          }
        }
      }
    ]
  }'
```

**Field explanation:**

| Field | Meaning |
|-------|---------|
| `min_severity` | Minimum CVE severity to trigger: `Low`, `Medium`, `High`, `Critical` |
| `fail_build` | Fail the CI build if this rule is violated |
| `block_download` | Prevent downloading violating artifacts |
| `unscanned` | Also block artifacts not yet scanned |

### Scan from CLI

```bash
# Scan the current directory (your source code)
jf audit

# Scan a specific build
jf xr build-scan myapp-build 42

# Scan a Docker image
jf docker scan localhost:8082/docker-local/myapp:1.0.0

# Output scan results in table format
jf audit --format table

# Output as JSON (for CI parsing)
jf audit --format json > scan-results.json
```

### Sample Xray Scan Output

```
┌──────────────┬─────────────────┬──────────┬────────────────────────────────┐
│ CVE          │ Component       │ Severity │ Fixed Version                  │
├──────────────┼─────────────────┼──────────┼────────────────────────────────┤
│ CVE-2021-44228│ log4j:2.14.0  │ Critical │ 2.15.0                         │
│ CVE-2022-0778 │ openssl:1.1.1k │ High     │ 1.1.1n                         │
│ CVE-2023-1234 │ axios:0.21.1   │ Medium   │ 0.21.4                         │
└──────────────┴─────────────────┴──────────┴────────────────────────────────┘
```

---

## 10. JFrog Pipelines — CI/CD

> **Note:** JFrog Pipelines is available in the JFrog Cloud (SaaS) and self-hosted Enterprise tiers. For free/OSS, use GitLab CI or GitHub Actions with JFrog CLI (covered in Section 13).

### Pipeline YAML Structure

```yaml
# pipelines.yml

resources:
  # A Git repository resource — pipeline triggers on code push
  - name: app_git_repo
    type: GitRepo
    configuration:
      gitProvider: my_github           # Integration name configured in admin
      path: myorg/myapp                # GitHub repo path
      branches:
        include: main                  # Only trigger on main branch

  # A build info resource — tracks which build was produced
  - name: app_build_info
    type: BuildInfo
    configuration:
      sourceArtifactory: my_artifactory

pipelines:
  - name: build_and_publish
    steps:

      # Step 1: Build the Maven application
      - name: build_maven
        type: MvnBuild
        configuration:
          mvnCommand: "clean install -DskipTests"
          sourceLocation: .
          inputResources:
            - name: app_git_repo
          outputResources:
            - name: app_build_info
          integrations:
            - name: my_artifactory

      # Step 2: Scan with Xray
      - name: scan_build
        type: XrayScan
        configuration:
          inputResources:
            - name: app_build_info
          integrations:
            - name: my_artifactory
          fail: true                   # Fail pipeline if Critical CVE found

      # Step 3: Promote build from staging to production repo
      - name: promote_build
        type: PromoteBuild
        configuration:
          targetRepository: libs-release-local
          status: "Promoted"
          comment: "Promoted after Xray scan passed"
          inputResources:
            - name: app_build_info
          integrations:
            - name: my_artifactory
```

---

## 11. Real-Time Project Workflow

### Project: Build, Scan, and Deploy a Java Microservice

```
Developer pushes code
       │
       ▼
GitLab CI triggers
       │
       ▼
Step 1: jf mvn clean install   (build JAR, resolve deps from Artifactory)
       │
       ▼
Step 2: jf rt build-publish    (publish build metadata to Artifactory)
       │
       ▼
Step 3: jf xr build-scan       (Xray scans the build)
       │
       ├── Critical CVE? → FAIL pipeline → notify team
       │
       ▼
Step 4: docker build + jf docker push  (build & push Docker image)
       │
       ▼
Step 5: jf rt build-promote    (promote build from staging to prod repo)
       │
       ▼
Step 6: helm upgrade --install  (deploy to Kubernetes from Artifactory Helm repo)
       │
       ▼
      PRODUCTION
```

### Full GitLab CI Pipeline File

```yaml
# .gitlab-ci.yml

variables:
  ARTIFACTORY_URL: "http://artifactory.mycompany.com/artifactory"
  ARTIFACTORY_USER: "$JFROG_USER"          # Store in GitLab CI/CD variables
  ARTIFACTORY_PASSWORD: "$JFROG_PASSWORD"  # Store in GitLab CI/CD variables
  BUILD_NAME: "myapp-build"
  BUILD_NUMBER: "$CI_PIPELINE_ID"          # Use GitLab pipeline ID as build number
  DOCKER_IMAGE: "artifactory.mycompany.com/docker-local/myapp"

stages:
  - setup
  - build
  - scan
  - package
  - promote
  - deploy

# ─────────────────────────────────────────────────────────
# STAGE 1: Setup JFrog CLI
# ─────────────────────────────────────────────────────────
setup_jfrog_cli:
  stage: setup
  image: releases-docker.jfrog.io/jfrog/jfrog-cli
  script:
    # Configure JFrog CLI with Artifactory credentials
    - jf config add myserver \
        --url=$ARTIFACTORY_URL \
        --user=$ARTIFACTORY_USER \
        --password=$ARTIFACTORY_PASSWORD \
        --interactive=false
    - jf rt ping

# ─────────────────────────────────────────────────────────
# STAGE 2: Build with Maven
# ─────────────────────────────────────────────────────────
build_maven:
  stage: build
  image: releases-docker.jfrog.io/jfrog/jfrog-cli
  script:
    # Configure Maven to resolve/deploy via Artifactory
    - jf mvn-config \
        --server-id-resolve=myserver \
        --repo-resolve-releases=libs-release-local \
        --repo-resolve-snapshots=libs-snapshot-local \
        --server-id-deploy=myserver \
        --repo-deploy-releases=libs-release-local \
        --repo-deploy-snapshots=libs-snapshot-local

    # Build and deploy (JFrog CLI wraps Maven to collect build info)
    - jf mvn clean install \
        --build-name=$BUILD_NAME \
        --build-number=$BUILD_NUMBER

    # Add environment variables to build info
    - jf rt build-add-environment \
        $BUILD_NAME $BUILD_NUMBER

    # Publish build info to Artifactory
    - jf rt build-publish \
        $BUILD_NAME $BUILD_NUMBER

  artifacts:
    paths:
      - target/*.jar
    expire_in: 1 hour

# ─────────────────────────────────────────────────────────
# STAGE 3: Security Scan with Xray
# ─────────────────────────────────────────────────────────
xray_scan:
  stage: scan
  image: releases-docker.jfrog.io/jfrog/jfrog-cli
  script:
    # Scan the published build — fail if critical vulnerability found
    - jf xr build-scan \
        --fail=true \
        $BUILD_NAME $BUILD_NUMBER
  allow_failure: false   # This stage MUST pass

# ─────────────────────────────────────────────────────────
# STAGE 4: Build and Push Docker Image
# ─────────────────────────────────────────────────────────
build_docker:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: ""
  script:
    # Build the Docker image
    - docker build -t $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA .
    - docker tag $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA $DOCKER_IMAGE:latest

    # Login to Artifactory Docker registry
    - docker login \
        -u $ARTIFACTORY_USER \
        -p $ARTIFACTORY_PASSWORD \
        artifactory.mycompany.com

    # Push using JFrog CLI to capture build info
    - jf docker push \
        $DOCKER_IMAGE:$CI_COMMIT_SHORT_SHA \
        --build-name=$BUILD_NAME-docker \
        --build-number=$BUILD_NUMBER

    - jf rt build-publish \
        $BUILD_NAME-docker $BUILD_NUMBER

# ─────────────────────────────────────────────────────────
# STAGE 5: Promote Build (Staging → Production)
# ─────────────────────────────────────────────────────────
promote_build:
  stage: promote
  image: releases-docker.jfrog.io/jfrog/jfrog-cli
  script:
    # Promote the build — copy artifacts from staging to prod repo
    - jf rt build-promote \
        $BUILD_NAME $BUILD_NUMBER \
        libs-release-local \
        --status="Released" \
        --comment="Promoted after all checks passed" \
        --copy=true   # copy, not move — keep artifacts in staging too
  only:
    - main   # Only promote from main branch

# ─────────────────────────────────────────────────────────
# STAGE 6: Deploy to Kubernetes via Helm
# ─────────────────────────────────────────────────────────
deploy_k8s:
  stage: deploy
  image: alpine/helm:latest
  script:
    # Add Artifactory Helm repo
    - helm repo add myrepo \
        http://artifactory.mycompany.com/artifactory/helm-virtual \
        --username $ARTIFACTORY_USER \
        --password $ARTIFACTORY_PASSWORD

    - helm repo update

    # Deploy/upgrade the Helm release
    - helm upgrade --install myapp myrepo/myapp \
        --namespace production \
        --set image.repository=$DOCKER_IMAGE \
        --set image.tag=$CI_COMMIT_SHORT_SHA \
        --wait \
        --timeout 5m
  environment:
    name: production
  only:
    - main
```

---

## 12. AWS Integration with JFrog

### Architecture

```
AWS ECR   ←──── JFrog Artifactory (remote repo proxying ECR)
AWS S3    ←──── JFrog file storage backend (Enterprise)
AWS EKS   ←──── Pulls Docker images from Artifactory
CodeBuild ←──── Uses JFrog CLI to build and publish artifacts
```

### Option A — JFrog Cloud on AWS (SaaS)

Sign up at [jfrog.com/start-free](https://jfrog.com/start-free).
Your instance URL: `https://<yourname>.jfrog.io`

```bash
# Configure CLI for JFrog Cloud
jf config add cloud-server \
  --url=https://yourname.jfrog.io \
  --user=admin \
  --password=<your-api-key> \
  --interactive=false
```

### Option B — Self-hosted on AWS EC2

```bash
# Launch EC2 (t3.large recommended, Amazon Linux 2)
# Then install Docker and run:

docker run -d \
  --name artifactory \
  -p 8081:8081 \
  -p 8082:8082 \
  -v /opt/artifactory/var:/var/opt/jfrog/artifactory \
  releases-docker.jfrog.io/jfrog/artifactory-oss:latest
```

### AWS CodeBuild + JFrog Integration

```yaml
# buildspec.yml (AWS CodeBuild)
version: 0.2

env:
  secrets-manager:
    JFROG_USER: "prod/jfrog:username"       # Stored in AWS Secrets Manager
    JFROG_PASSWORD: "prod/jfrog:password"

phases:
  install:
    runtime-versions:
      java: corretto17
    commands:
      # Install JFrog CLI
      - curl -fL https://install-cli.jfrog.io | sh
      - mv jf /usr/local/bin/jf

      # Configure JFrog CLI
      - jf config add myserver \
          --url=https://yourname.jfrog.io/artifactory \
          --user=$JFROG_USER \
          --password=$JFROG_PASSWORD \
          --interactive=false

  build:
    commands:
      # Configure Maven resolver
      - jf mvn-config \
          --server-id-resolve=myserver \
          --repo-resolve-releases=libs-release-local \
          --repo-deploy-releases=libs-release-local

      # Build
      - jf mvn clean package \
          --build-name=codebuild-myapp \
          --build-number=$CODEBUILD_BUILD_NUMBER

  post_build:
    commands:
      # Publish build info
      - jf rt build-publish codebuild-myapp $CODEBUILD_BUILD_NUMBER

      # Scan with Xray
      - jf xr build-scan \
          --fail=true \
          codebuild-myapp $CODEBUILD_BUILD_NUMBER

artifacts:
  files:
    - target/*.jar
```

### S3-Backed Artifactory (Enterprise)

```yaml
# system.yaml (Enterprise configuration)
shared:
  database:
    type: postgresql
    driver: org.postgresql.Driver
    url: "jdbc:postgresql://rds-endpoint:5432/artifactory"
    username: "artifactory"
    password: "<password>"
  node:
    storage:
      binariesDir: "/var/opt/jfrog/artifactory/data/filestore"
  filestore:
    type: s3-storage-v3
    awsS3V3:
      region: us-east-1
      bucketName: my-artifactory-bucket
      endpoint: ""
      useInstanceCredentials: true   # Use EC2 IAM Role
```

---

## 13. GitLab CI/CD + JFrog Integration

### Store Credentials Securely in GitLab

```
GitLab Project → Settings → CI/CD → Variables

Variable Name          | Value                          | Protected | Masked
JFROG_URL              | http://localhost:8082           | No        | No
JFROG_USER             | admin                          | Yes       | No
JFROG_PASSWORD         | <your-password>                | Yes       | Yes  ← MASK THIS
```

> Always mask passwords/tokens. Never hardcode credentials in `.gitlab-ci.yml`.

### Minimal Working Example

```yaml
# .gitlab-ci.yml — minimal JFrog upload example
image: releases-docker.jfrog.io/jfrog/jfrog-cli:latest

before_script:
  - jf config add myserver \
      --url=$JFROG_URL \
      --user=$JFROG_USER \
      --password=$JFROG_PASSWORD \
      --interactive=false

build_and_upload:
  stage: build
  script:
    - mvn clean package -DskipTests
    - jf rt upload \
        "target/*.jar" \
        "libs-release-local/com/myapp/$CI_COMMIT_TAG/" \
        --build-name=myapp \
        --build-number=$CI_PIPELINE_ID
    - jf rt build-publish myapp $CI_PIPELINE_ID
  only:
    - tags  # Run only when a Git tag is pushed
```

---

## 14. REST API — Automate Everything

JFrog Artifactory exposes a comprehensive REST API. Every action done in the UI can be automated via API.

### Base URL

```
http://<host>:8081/artifactory/api/
```

### Authentication Options

```bash
# Option 1: Basic Auth (user:password)
curl -u admin:password http://localhost:8081/artifactory/api/system/ping

# Option 2: API Token (recommended for CI/CD)
curl -H "Authorization: Bearer <your-api-token>" \
  http://localhost:8081/artifactory/api/system/ping

# Generate an API token
curl -X POST \
  -u admin:password \
  http://localhost:8081/artifactory/api/security/token \
  -d "username=ci-user&scope=member-of-groups:readers"
```

### Useful API Examples

```bash
# ─── Get system info ───────────────────────────────────
curl -u admin:password \
  http://localhost:8081/artifactory/api/system/info | python3 -m json.tool

# ─── List all repositories ─────────────────────────────
curl -u admin:password \
  http://localhost:8081/artifactory/api/repositories | python3 -m json.tool

# ─── Get artifact info (metadata, checksums) ───────────
curl -u admin:password \
  "http://localhost:8081/artifactory/api/storage/libs-release-local/com/myapp/1.0.0/myapp-1.0.0.jar"

# ─── AQL (Artifactory Query Language) search ───────────
# Find all artifacts deployed in the last 24 hours
curl -X POST \
  -u admin:password \
  http://localhost:8081/artifactory/api/search/aql \
  -H "Content-Type: text/plain" \
  -d '
  items.find({
    "repo": "libs-release-local",
    "created": { "$gt": "now-1d" }
  }).include("name", "repo", "path", "created", "size")
  '

# ─── AQL: Find artifacts by property ──────────────────
curl -X POST \
  -u admin:password \
  http://localhost:8081/artifactory/api/search/aql \
  -H "Content-Type: text/plain" \
  -d '
  items.find({
    "@env": "prod",
    "@approved": "true"
  }).include("name", "repo", "path")
  '

# ─── Get build info ────────────────────────────────────
curl -u admin:password \
  "http://localhost:8081/artifactory/api/build/myapp-build/42"

# ─── Delete old builds (cleanup) ───────────────────────
curl -X DELETE \
  -u admin:password \
  "http://localhost:8081/artifactory/api/build/myapp-build?buildNumbers=1,2,3&artifacts=1"
```

### AQL — Artifactory Query Language

AQL is like SQL for Artifactory. It lets you search artifacts with complex criteria.

```bash
# AQL Syntax
items.find(
  { <criteria> }
)
.include(<fields>)
.sort({ <field>: "asc|desc" })
.limit(<number>)
.offset(<number>)
```

```bash
# Real example: Find top 10 largest JARs in release repo
curl -X POST \
  -u admin:password \
  http://localhost:8081/artifactory/api/search/aql \
  -H "Content-Type: text/plain" \
  -d '
  items.find({
    "repo": "libs-release-local",
    "name": { "$match": "*.jar" }
  })
  .include("name", "size", "repo", "path")
  .sort({"$desc": ["size"]})
  .limit(10)
  '
```

---

## 15. Permissions, Users & Groups

### Permission Model

```
User → belongs to → Group → has → Permission Target → on → Repository
```

### Create User (REST API)

```bash
curl -X PUT \
  -H "Content-Type: application/json" \
  -u admin:password \
  http://localhost:8081/artifactory/api/security/users/ci-user \
  -d '{
    "name": "ci-user",
    "email": "ci@mycompany.com",
    "password": "Str0ngP@ss!",
    "admin": false,
    "groups": ["ci-group"],
    "disableUIAccess": true
  }'
```

> `disableUIAccess: true` — Best practice for service accounts. They should only access via API/CLI.

### Create Group

```bash
curl -X PUT \
  -H "Content-Type: application/json" \
  -u admin:password \
  http://localhost:8081/artifactory/api/security/groups/ci-group \
  -d '{
    "name": "ci-group",
    "description": "CI/CD service accounts",
    "autoJoin": false
  }'
```

### Create Permission Target

```bash
curl -X PUT \
  -H "Content-Type: application/json" \
  -u admin:password \
  http://localhost:8081/artifactory/api/security/permissions/ci-permission \
  -d '{
    "name": "ci-permission",
    "repositories": ["libs-release-local", "libs-snapshot-local", "docker-local"],
    "principals": {
      "groups": {
        "ci-group": ["r", "d", "w", "n", "m"]
      },
      "users": {
        "developer1": ["r"]
      }
    }
  }'
```

**Permission codes:**

| Code | Permission |
|------|-----------|
| `r`  | Read      |
| `d`  | Deploy/write |
| `w`  | Write (edit properties) |
| `n`  | Annotate  |
| `m`  | Manage (admin for this repo) |
| `managedXrayMeta` | Manage Xray metadata |

---

## 16. Build Info & Traceability

### What is Build Info?

Build Info is a **JSON document** that Artifactory stores with every published build. It contains:
- Which source code commit triggered the build
- Which artifacts were produced
- Which dependencies were used (and their checksums)
- Environment variables at build time
- Xray scan results

```json
// Example Build Info structure (simplified)
{
  "version": "1.0",
  "name": "myapp-build",
  "number": "42",
  "started": "2025-01-15T10:30:00.000Z",
  "buildAgent": { "name": "Maven", "version": "3.9.5" },
  "vcs": [
    {
      "revision": "a1b2c3d4e5f6",
      "branch": "main",
      "url": "https://gitlab.com/myorg/myapp.git",
      "message": "feat: add user authentication"
    }
  ],
  "modules": [
    {
      "id": "com.mycompany:myapp:1.0.0",
      "artifacts": [
        {
          "name": "myapp-1.0.0.jar",
          "sha256": "abc123...",
          "md5": "def456..."
        }
      ],
      "dependencies": [
        {
          "id": "org.springframework:spring-core:6.1.0",
          "sha256": "ghi789..."
        }
      ]
    }
  ]
}
```

### Add Git Info to Build

```bash
# Automatically collect git info and add to build
jf rt build-add-git myapp-build 42

# Add custom properties
jf rt build-add-environment myapp-build 42
```

### Promote a Build

```bash
# Promote — move/copy build artifacts between repos with status change
jf rt build-promote myapp-build 42 \
  libs-release-local \
  --status="Released" \
  --comment="Approved by QA on 2025-01-15" \
  --source-repo=libs-staging-local \
  --copy=false \   # true = copy, false = move
  --props="release.version=1.0.0;release.date=2025-01-15"
```

---

## 17. High Availability & Disaster Recovery

### HA Architecture

```
                    Load Balancer (nginx / AWS ALB)
                           │
          ┌────────────────┴────────────────┐
          ▼                                 ▼
  Artifactory Node 1              Artifactory Node 2
  (Active)                        (Active — both serve traffic)
          │                                 │
          └──────────────┬──────────────────┘
                         │
                 Shared Components:
                 ┌───────────────┐
                 │  PostgreSQL   │  (shared DB — all nodes use same DB)
                 │  (RDS / ext)  │
                 └───────────────┘
                 ┌───────────────┐
                 │  Shared File  │  (NFS / S3 / Azure Blob)
                 │  Storage      │  (all nodes read/write same binaries)
                 └───────────────┘
```

### Key HA Configuration Points

```yaml
# system.yaml — HA node configuration
shared:
  database:
    type: postgresql
    url: "jdbc:postgresql://postgres-host:5432/artifactory"
  node:
    id: "node1"               # Unique ID per node
    haEnabled: true           # Enable HA mode
  filestore:
    type: s3-storage-v3       # Shared binary storage on S3
```

### Backup Strategy

```bash
# Trigger a manual backup via API
curl -X POST \
  -u admin:password \
  "http://localhost:8081/artifactory/api/backup/execute" \
  -d "backupKey=backup-daily"

# Configure scheduled backup in UI:
# Admin → Artifactory → Backups → New Backup
#   Backup Key:  backup-daily
#   Cron:        0 0 2 * * ?   (2am daily)
#   Retention:   7 builds
#   Repos:       All (or select specific)
```

---

## 18. Troubleshooting & Bug Fixing

### Problem 1 — `jf rt ping` fails (Connection refused)

```bash
# Symptom:
$ jf rt ping
[Error] Artifactory response: 404 Not Found for url: http://localhost:8082/artifactory/api/system/ping

# Diagnosis steps:
docker ps | grep artifactory          # Is container running?
docker logs artifactory --tail 50     # Check for startup errors
curl http://localhost:8082/ui         # Can browser reach UI?

# Fix 1: Container not running
docker start artifactory

# Fix 2: Wrong URL in config (missing /artifactory path)
jf config edit myserver
# Correct URL format: http://localhost:8082/artifactory   ← NOTE: /artifactory at end
# OR for JFrog Cloud: https://yourname.jfrog.io           ← no /artifactory
```

### Problem 2 — 401 Unauthorized on upload

```bash
# Symptom:
$ jf rt upload "target/myapp.jar" libs-release-local/
[Error] Artifactory response: 401 Unauthorized

# Diagnosis:
# 1. Check credentials
jf config show        # Verify URL, user are correct
jf rt ping            # Should return OK

# 2. Test credentials directly
curl -u myuser:mypassword http://localhost:8082/artifactory/api/system/ping

# Fix 1: Re-configure CLI with correct credentials
jf config edit myserver

# Fix 2: User may not have Deploy permission on that repo
# Admin → Security → Permissions → check user's group has 'd' permission
```

### Problem 3 — 403 Forbidden on repo (snapshot into release repo)

```bash
# Symptom:
Forbidden (403): Cannot deploy 'com/myapp/myapp-1.0.0-SNAPSHOT.jar' into
'libs-release-local' since Snapshot deployments are not allowed in this repository

# Root cause: Trying to deploy a SNAPSHOT artifact into a release-only repo

# Fix: Either
# (a) Change artifact version in pom.xml from 1.0.0-SNAPSHOT to 1.0.0
# (b) Deploy to snapshot repo instead:
jf rt upload "target/myapp-1.0.0-SNAPSHOT.jar" libs-snapshot-local/
```

### Problem 4 — Docker push fails (name unknown)

```bash
# Symptom:
$ docker push localhost:8082/docker-local/myapp:1.0.0
Error response from daemon: Get "https://localhost:8082/v2/": http: server gave HTTP response to HTTPS client

# Fix 1: Tell Docker to trust this insecure registry
# Edit /etc/docker/daemon.json:
{
  "insecure-registries": ["localhost:8082"]
}
# Then restart Docker: sudo systemctl restart docker

# Fix 2: For production — use HTTPS with a proper certificate
# Set up nginx/traefik with a TLS cert in front of Artifactory
```

### Problem 5 — Build info not showing in Artifactory UI

```bash
# Symptom: Build exists but shows no artifacts in UI

# Diagnosis: Did you run build-publish after the build?
# Fix: Always publish build info after build completes
jf rt build-publish $BUILD_NAME $BUILD_NUMBER

# Also check: Did you use jf mvn (not plain mvn)?
# Plain mvn does NOT collect build info
# Wrong:  mvn clean install
# Right:  jf mvn clean install --build-name=myapp --build-number=42
```

### Problem 6 — Xray scan times out

```bash
# Symptom:
[Error] Xray scan exceeded timeout

# Fix 1: Increase timeout in CLI command
jf xr build-scan --fail=true myapp-build 42 --timeout=300

# Fix 2: Check Xray service health
curl -u admin:password http://localhost:8082/xray/api/v1/system/ping

# Fix 3: Check Xray has indexed the repos (Admin → Xray → Indexed Resources)
```

### Problem 7 — Disk space full

```bash
# Check Artifactory storage usage
curl -u admin:password \
  http://localhost:8081/artifactory/api/storageinfo | python3 -m json.tool

# Run garbage collection to clean orphaned binaries
curl -X POST \
  -u admin:password \
  "http://localhost:8081/artifactory/api/system/storage/gc"

# Set up artifact cleanup policies in UI:
# Admin → Artifactory → Cleanup Policies → New Policy
# e.g., Delete all artifacts in libs-snapshot-local not downloaded in 30 days
```

### Useful Log Files

```bash
# Application log (main Artifactory log)
docker exec artifactory cat /var/opt/jfrog/artifactory/log/artifactory.log

# Access log (all HTTP requests)
docker exec artifactory tail -f /var/opt/jfrog/artifactory/log/access.log

# Request log (slow query analysis)
docker exec artifactory tail -f /var/opt/jfrog/artifactory/log/request.log

# Xray log
docker exec artifactory tail -f /var/opt/jfrog/artifactory/log/xray.log
```

---

## 19. Best Practices

### Repository Strategy

```
✅  DO:
- Use virtual repos as the single entry point for all resolvers
- Separate local repos by environment (dev, staging, release)
- Use consistent naming conventions: <team>-<env>-<pkgtype>-<local|remote|virtual>
- Create a snapshot repo and a release repo per package type (never mix)

❌  DON'T:
- Don't point build tools directly at local repos — use virtual repos
- Don't store secrets or passwords in artifact metadata/properties
- Don't delete artifacts manually — use cleanup policies
```

### Security Best Practices

```
✅  DO:
- Use API tokens (not passwords) for CI/CD authentication
- Enable Xray scanning on ALL repos used in production pipelines
- Set minimum severity threshold to "High" (fail build on High/Critical)
- Store JFrog credentials in secret managers (Vault, AWS Secrets Manager, GitLab vars)
- Use "disableUIAccess: true" for all CI service accounts
- Enable anonymous read only if truly needed (disabled by default)

❌  DON'T:
- Don't use admin credentials in CI pipelines
- Don't store credentials in .gitlab-ci.yml or buildspec.yml
- Don't skip Xray scans to save time — one critical CVE can destroy your reputation
```

### Build Info Best Practices

```
✅  DO:
- Always use jf mvn / jf npm / jf docker instead of bare package manager commands
- Always publish build info after every build: jf rt build-publish
- Add git info to builds: jf rt build-add-git
- Include environment info: jf rt build-add-environment
- Promote builds through stages (snapshot → staging → release) — never deploy straight to prod

❌  DON'T:
- Don't use arbitrary build numbers — tie them to CI system IDs (pipeline ID, build number)
- Don't skip build promotion — it creates traceability gaps
```

### Retention / Cleanup Policies

```yaml
# Recommended retention policy (set in UI or via API):

# Snapshots: Delete after 30 days or keep only last 5 builds
Repo: libs-snapshot-local
  Policy: Delete artifacts not downloaded in 30 days
  OR:     Keep only last 5 versions per module

# Releases: Keep indefinitely (unless archiving strategy exists)
Repo: libs-release-local
  Policy: No auto-delete (manual release lifecycle management)

# Docker: Keep last 10 tags per image name
Repo: docker-local
  Policy: Delete tags beyond the last 10 per image
```

---

## 20. Assignments

### Assignment 1 — Basic (Beginner)

**Task:** Set up JFrog Artifactory locally and upload your first artifact.

1. Install Artifactory using Docker (Section 5)
2. Create a local generic repository named `my-files-local`
3. Create a file `hello.txt` with content "Hello JFrog!"
4. Upload it using JFrog CLI: `jf rt upload hello.txt my-files-local/`
5. Download it back: `jf rt download my-files-local/hello.txt ./downloaded/`
6. Verify the file content matches

**Expected result:** File downloaded successfully, content matches.

---

### Assignment 2 — Intermediate

**Task:** Create a full Maven artifact lifecycle.

1. Create a Maven project with `mvn archetype:generate`
2. Create repositories:
   - `myapp-snapshot-maven-local` (local, maven, snapshots only)
   - `myapp-release-maven-local` (local, maven, releases only)
   - `central-maven-remote` (remote, proxying Maven Central)
   - `maven-virtual` (virtual, aggregating all 3)
3. Configure JFrog CLI for Maven (`jf mvn-config`)
4. Build and deploy SNAPSHOT: `jf mvn clean deploy`
5. Change version from `1.0-SNAPSHOT` to `1.0.0`
6. Deploy release: `jf mvn clean deploy`
7. Promote the release build from snapshot to release repo

**Expected result:** Both SNAPSHOT and release artifacts visible in Artifactory. Build info published.

---

### Assignment 3 — Advanced

**Task:** Build a full CI/CD pipeline with security scanning.

1. Create a GitLab repository with a Java Spring Boot application
2. Set up GitLab CI/CD with JFrog integration (Section 13)
3. Implement these stages:
   - `build` — Build JAR with JFrog Maven
   - `docker` — Build and push Docker image to Artifactory
   - `scan` — Scan with Xray (fail on High+ CVEs)
   - `promote` — Promote build to release on `main` branch
4. Introduce a known vulnerable dependency (e.g., `log4j:2.14.0`) and verify that Xray **blocks** the build
5. Fix the vulnerability and verify the pipeline passes
6. Add artifact cleanup policy: delete snapshots older than 7 days

**Expected result:** Pipeline enforces security. Vulnerable builds blocked. Fixed builds promoted successfully.

---

### Assignment 4 — Expert

**Task:** JFrog REST API automation.

Write a Python script that:
1. Connects to Artifactory API
2. Lists all repositories
3. Finds all artifacts in `libs-release-local` older than 90 days (using AQL)
4. Generates a report (CSV) with: artifact name, size, created date, last downloaded date
5. (Optional) Deletes artifacts not downloaded in the last 90 days (with a `--dry-run` flag)

```python
# Starter code for Assignment 4
import requests
import json
import csv
from datetime import datetime

ARTIFACTORY_URL = "http://localhost:8081/artifactory"
USER = "admin"
PASSWORD = "password"

def search_old_artifacts(days: int) -> list:
    aql_query = f"""
    items.find({{
        "repo": "libs-release-local",
        "created": {{ "$lt": "now-{days}d" }}
    }}).include("name", "repo", "path", "created", "size", "stat.downloaded")
    """
    response = requests.post(
        f"{ARTIFACTORY_URL}/api/search/aql",
        data=aql_query,
        auth=(USER, PASSWORD),
        headers={"Content-Type": "text/plain"}
    )
    response.raise_for_status()
    return response.json()["results"]

# TODO: Complete the script — generate CSV report
# TODO: Add --dry-run flag for deletion
```

---

## 21. Interview Questions & Answers

### Beginner Level

**Q1: What is JFrog Artifactory and why is it used?**
> JFrog Artifactory is a universal binary repository manager. It stores, manages, and distributes software artifacts (JARs, Docker images, npm packages, etc.) across the development lifecycle. It's used because it provides a single source of truth for all binaries, enables artifact caching (proxy remote repos), provides security scanning via Xray, and ensures traceability from source code to deployment.

**Q2: What are the three types of repositories in Artifactory?**
> - **Local**: Artifactory owns and stores the artifacts directly. Used for your own artifacts.
> - **Remote**: A proxy to an external repository (Maven Central, PyPI, Docker Hub). Caches artifacts locally to reduce internet dependency and improve build speed.
> - **Virtual**: An aggregation of local and remote repositories behind a single URL. Your build tools point to this — they never need to know about the underlying repos.

**Q3: What is the difference between snapshot and release repositories?**
> - **Snapshot repos** store development/in-progress versions (e.g., `1.0-SNAPSHOT`). These can be overwritten — the same version number may be updated multiple times.
> - **Release repos** store stable, final versions (e.g., `1.0.0`). These are immutable — once published, a release artifact should never be overwritten.

**Q4: How do you upload a file to Artifactory using JFrog CLI?**
> ```bash
> jf rt upload "target/myapp.jar" "libs-release-local/com/myapp/1.0.0/"
> ```

---

### Intermediate Level

**Q5: What is Build Info and why is it important?**
> Build Info is a JSON document that Artifactory stores with every build. It captures: the source code commit, all artifacts produced, all dependencies used (with checksums), environment variables, and Xray scan results. It provides **full traceability** — you can answer "which exact code, with which exact dependencies, produced this artifact that's running in production?"

**Q6: What is JFrog Xray and how does it work?**
> Xray is the security and compliance component of the JFrog platform. It:
> 1. Performs **deep recursive scanning** of artifacts (scans inside JAR/ZIP files, Docker layers, etc.)
> 2. Matches components against vulnerability databases (CVE, NVD, JFrog Research)
> 3. Checks license compliance
> 4. Applies **Watches** (what to scan) and **Policies** (what to do when violations are found)
> 5. Can block downloads or fail CI builds on policy violations

**Q7: How would you configure Maven to use Artifactory as its repository?**
> Two approaches:
> 1. **JFrog CLI** (recommended): Run `jf mvn-config` to configure resolver/deployer, then use `jf mvn` instead of `mvn` to build with build info collection.
> 2. **settings.xml**: Add the Artifactory virtual repo URL in Maven's `~/.m2/settings.xml` under `<mirrors>` and `<servers>`.

**Q8: What is artifact promotion and when do you use it?**
> Promotion is the act of moving or copying build artifacts from one repository to another as they progress through your release pipeline. Example: `staging-local → qa-local → release-local`. Each promotion represents a quality gate. It ensures that only tested and approved artifacts reach production repositories, and it maintains full traceability of which builds were approved at each stage.

---

### Advanced Level

**Q9: How would you implement a JFrog HA setup?**
> A JFrog HA setup requires:
> 1. Multiple Artifactory nodes (all active-active)
> 2. A shared external database (PostgreSQL — all nodes use same DB)
> 3. Shared binary storage (NFS or cloud storage like S3/Azure Blob — all nodes read/write same files)
> 4. A load balancer (nginx/HAProxy/AWS ALB) in front of all nodes
> 5. Each node configured with `haEnabled: true` and a unique node ID in `system.yaml`

**Q10: Explain AQL and give a use case.**
> AQL (Artifactory Query Language) is a query language for searching artifacts in Artifactory. It supports filtering by repository, name, path, properties, dates, file size, and more. Use case: Find all production-approved Docker images pushed in the last 7 days that haven't been downloaded yet (potential orphans):
> ```
> items.find({
>   "repo": "docker-local",
>   "@env": "prod",
>   "created": { "$gt": "now-7d" },
>   "stat.downloads": { "$eq": null }
> }).include("name", "created", "size")
> ```

**Q11: How do you handle credential rotation for Artifactory in CI/CD?**
> 1. Use API tokens (scoped access tokens) instead of user passwords — tokens can be revoked independently
> 2. Store tokens in a secret manager (HashiCorp Vault, AWS Secrets Manager, GitLab CI masked variables)
> 3. Use short-lived tokens where possible (set expiry)
> 4. For Kubernetes: Use external secrets operator to sync secrets from Vault/AWS SM to K8s secrets
> 5. Rotate by: create new token → update secret manager → verify CI passes → revoke old token

**Q12: How would you clean up old artifacts automatically?**
> Several approaches:
> 1. **Artifactory Cleanup Policies** (UI): Admin → Cleanup Policies — configure rules like "delete artifacts not downloaded in X days"
> 2. **AQL + CLI script**: Write a cron job that uses AQL to find old artifacts and `jf rt delete` to remove them
> 3. **Xray Retention Policies**: Configure Xray to automatically quarantine and clean up violating artifacts
> 4. **Build Retention**: Set `maxBuilds` or date-based retention on builds and their artifacts
> Best practice: Start with dry-run, log what would be deleted, then enable actual deletion after validation.

**Q13: What is the difference between `jf rt copy` and `jf rt move`?**
> - `jf rt copy`: Duplicates the artifact — it exists in both source and destination repos. Used for promotion when you want to keep the original (e.g., for audit purposes).
> - `jf rt move`: Transfers the artifact — it only exists in the destination repo. Source no longer has it. Use move when the source is a staging/temporary area.

**Q14: How does JFrog Distribution work?**
> JFrog Distribution allows you to create **Release Bundles** — signed, immutable collections of artifacts — and distribute them to remote **Edge nodes** (lightweight Artifactory instances). This enables:
> - Secure distribution to air-gapped environments
> - Guaranteed delivery with cryptographic verification
> - Rollback capability (release bundles are versioned)
> - Bandwidth-efficient delta syncs

**Q15: How do you integrate JFrog with Kubernetes?**
> 1. Store Docker images in Artifactory `docker-local` repo
> 2. Configure Kubernetes `imagePullSecrets` to authenticate with Artifactory registry:
> ```bash
> kubectl create secret docker-registry artifactory-secret \
>   --docker-server=artifactory.mycompany.com \
>   --docker-username=k8s-svc-user \
>   --docker-password=<api-token>
> ```
> 3. Reference in Pod/Deployment spec:
> ```yaml
> spec:
>   imagePullSecrets:
>     - name: artifactory-secret
>   containers:
>     - image: artifactory.mycompany.com/docker-local/myapp:1.0.0
> ```
> 4. Use Artifactory as Helm chart repository for `helm install/upgrade`
> 5. (Optional) Use JFrog Xray admission controller to block vulnerable images at Kubernetes admission time

---

## Quick Reference Card

```bash
# ═══════════════════════════════════════════════════════
#               JFrog CLI Quick Reference
# ═══════════════════════════════════════════════════════

# CONFIG
jf config add <id>                          # Add server config
jf config show                              # Show all configs
jf rt ping                                  # Test connection

# UPLOAD / DOWNLOAD
jf rt upload <local> <remote>              # Upload file
jf rt download <remote> <local>            # Download file

# BUILD INFO
jf mvn clean install --build-name=X --build-number=Y
jf rt build-publish <name> <number>        # Publish build info
jf rt build-add-git <name> <number>        # Add Git info
jf rt build-promote <name> <number> <repo> # Promote build

# SEARCH
jf rt search "repo/path/*"                 # Search by pattern
jf rt search --props "key=val" "repo/*"   # Search by property

# PROPERTIES
jf rt set-props "repo/path/file" "k=v"    # Set properties

# SECURITY
jf audit                                   # Scan source code
jf xr build-scan <name> <number>          # Scan build
jf docker scan <image>                    # Scan Docker image

# CLEANUP
jf rt delete "repo/path/*"                # Delete artifacts
jf rt build-discard <name> --max-builds=5 # Keep last 5 builds
```

---

*Guide version: 1.0 | Last updated: June 2026*
*Your JFrog Trainer & AWS DevOps Mentor*
