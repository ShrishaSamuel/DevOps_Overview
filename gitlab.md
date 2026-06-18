# GitLab — Zero to Advanced: Complete Training Guide
### Your GitLab Trainer & AWS DevOps Mentor

> **Who is this for?** Absolute beginners to advanced practitioners.  
> **What you'll learn?** GitLab concepts, CI/CD pipelines, AWS integration, real projects, troubleshooting, best practices, and interview prep.

---

## Table of Contents

1. [Chapter 1 — What is GitLab? Core Concepts](#chapter-1--what-is-gitlab-core-concepts)
2. [Chapter 2 — GitLab Setup & Account](#chapter-2--gitlab-setup--account)
3. [Chapter 3 — Git Basics (The Foundation)](#chapter-3--git-basics-the-foundation)
4. [Chapter 4 — GitLab Repository Deep Dive](#chapter-4--gitlab-repository-deep-dive)
5. [Chapter 5 — Branching, Merging & Merge Requests](#chapter-5--branching-merging--merge-requests)
6. [Chapter 6 — GitLab CI/CD Fundamentals](#chapter-6--gitlab-cicd-fundamentals)
7. [Chapter 7 — `.gitlab-ci.yml` — Line by Line](#chapter-7--gitlab-ciyml--line-by-line)
8. [Chapter 8 — GitLab Runners](#chapter-8--gitlab-runners)
9. [Chapter 9 — Advanced CI/CD Pipelines](#chapter-9--advanced-cicd-pipelines)
10. [Chapter 10 — GitLab + AWS DevOps Integration](#chapter-10--gitlab--aws-devops-integration)
11. [Chapter 11 — Real-Time Project Workflow](#chapter-11--real-time-project-workflow)
12. [Chapter 12 — GitLab Security & Compliance](#chapter-12--gitlab-security--compliance)
13. [Chapter 13 — Troubleshooting & Bug Fixing](#chapter-13--troubleshooting--bug-fixing)
14. [Chapter 14 — Best Practices](#chapter-14--best-practices)
15. [Chapter 15 — Assignments](#chapter-15--assignments)
16. [Chapter 16 — Interview Questions & Answers](#chapter-16--interview-questions--answers)

---

## Chapter 1 — What is GitLab? Core Concepts

### 1.1 GitLab vs GitHub vs Bitbucket

| Feature | GitLab | GitHub | Bitbucket |
|---|---|---|---|
| CI/CD | Built-in (free) | GitHub Actions | Pipelines |
| Self-hosted | Yes (Community Edition) | Limited | Yes |
| Container Registry | Built-in | Yes | No |
| Issue Tracking | Built-in | Built-in | Jira integration |
| DevSecOps | Built-in SAST/DAST | Third-party | Limited |

### 1.2 GitLab Architecture (How it Works)

```
Developer's Machine
      │
      ▼
   Git Push ──────────────────────────────►  GitLab Server
                                                │
                                     ┌──────────┴──────────┐
                                     │                     │
                               Repository            CI/CD Engine
                               (Source Code)              │
                                                   ┌───────┴───────┐
                                                GitLab Runner   GitLab Runner
                                                (Shell/Docker)  (Kubernetes)
                                                     │
                                              ┌──────┴──────┐
                                           Build         Deploy
                                           & Test        to AWS/K8s
```

### 1.3 Key GitLab Terminology

| Term | Definition | Real Example |
|---|---|---|
| **Repository** | Where your code lives | `my-web-app` project |
| **Pipeline** | Automated workflow triggered by a push | Build → Test → Deploy |
| **Job** | A single unit of work in a pipeline | `run_tests`, `docker_build` |
| **Stage** | Group of jobs that run in parallel | `build`, `test`, `deploy` |
| **Runner** | Machine that executes jobs | EC2 instance on AWS |
| **Artifact** | Files produced by a job | `app.jar`, coverage report |
| **Environment** | Deployment target | `staging`, `production` |
| **Merge Request (MR)** | Request to merge a branch | Feature → main |
| **Group** | Collection of projects | `my-company/backend` |
| **Namespace** | User or group path | `gitlab.com/john/project` |

---

## Chapter 2 — GitLab Setup & Account

### 2.1 Create a Free GitLab Account

1. Go to [https://gitlab.com](https://gitlab.com)
2. Click **Register** → Use work email
3. Verify email
4. Choose **Free** plan (includes 400 CI/CD minutes/month)

### 2.2 Create Your First Project

```
GitLab UI → New Project → Create blank project
  Project name: my-first-app
  Visibility: Private
  Initialize with README: ✓
```

### 2.3 SSH Key Setup (Recommended)

```bash
# Step 1: Generate SSH key pair on your machine
ssh-keygen -t ed25519 -C "your_email@example.com"
# Press Enter for all prompts (default location: ~/.ssh/id_ed25519)

# Step 2: Copy the PUBLIC key
cat ~/.ssh/id_ed25519.pub
# Output: ssh-ed25519 AAAAC3Nza... your_email@example.com

# Step 3: Add to GitLab
# GitLab → Preferences (top-right) → SSH Keys → Paste → Add key

# Step 4: Test connection
ssh -T git@gitlab.com
# Expected: Welcome to GitLab, @yourname!
```

### 2.4 Configure Git Identity

```bash
# These details appear in every commit — set them once globally
git config --global user.name "Your Name"
git config --global user.email "your_email@example.com"
git config --global core.editor "vim"         # default editor
git config --global init.defaultBranch "main" # default branch name

# Verify configuration
git config --list
```

---

## Chapter 3 — Git Basics (The Foundation)

> GitLab uses Git under the hood. Master Git first.

### 3.1 Core Git Commands

```bash
# Clone a GitLab repository to your local machine
git clone git@gitlab.com:yourname/my-first-app.git
cd my-first-app

# Check status of working directory
git status
# Shows: untracked files, staged changes, modified files

# Stage a file (prepare for commit)
git add filename.txt        # specific file
git add .                   # all changes in current directory
git add -p                  # interactive — choose which hunks to stage

# Commit staged changes with a message
git commit -m "feat: add login page"

# Push commits to GitLab remote
git push origin main

# Pull latest changes from GitLab
git pull origin main

# View commit history
git log --oneline --graph --all
```

### 3.2 The Git Workflow Explained

```
Working Directory    Staging Area (Index)    Local Repository    Remote (GitLab)
      │                     │                      │                   │
      │── edit files ──────►│                      │                   │
      │                     │── git add ──────────►│                   │
      │                     │                      │── git commit ────►│
      │                     │                      │                   │── git push ──►
      │◄─────────────────────────────────────────────────── git pull ──│
```

### 3.3 Understanding `.gitignore`

```gitignore
# .gitignore — tells Git which files NOT to track

# Python
__pycache__/
*.pyc
.env
venv/

# Node.js
node_modules/
dist/
.env.local

# IDE files
.vscode/
.idea/
*.swp

# OS files
.DS_Store
Thumbs.db

# Build outputs
build/
*.log
```

---

## Chapter 4 — GitLab Repository Deep Dive

### 4.1 Repository Structure Best Practices

```
my-web-app/
├── .gitlab/                    # GitLab-specific configs
│   ├── issue_templates/        # Issue templates
│   └── merge_request_templates/# MR templates
├── .gitlab-ci.yml              # CI/CD Pipeline definition ← MOST IMPORTANT
├── src/                        # Source code
├── tests/                      # Test files
├── docs/                       # Documentation
├── Dockerfile                  # Container definition
├── docker-compose.yml          # Local dev environment
├── requirements.txt            # Python dependencies
├── README.md                   # Project documentation
└── .gitignore                  # Files to ignore
```

### 4.2 Protected Branches

```
GitLab UI → Repository → Settings → Repository → Protected Branches

Branch: main
  Allowed to merge: Maintainers
  Allowed to push: No one (force pipelines + MRs)
  
Branch: develop
  Allowed to merge: Developers + Maintainers
  Allowed to push: Developers + Maintainers
```

**Why protect branches?**  
- Prevents accidental direct pushes to `main`  
- Forces code review via Merge Requests  
- Ensures CI/CD passes before merge  

### 4.3 Tags & Releases

```bash
# Create a lightweight tag
git tag v1.0.0

# Create an annotated tag (recommended for releases)
git tag -a v1.0.0 -m "Release version 1.0.0"

# Push tag to GitLab
git push origin v1.0.0

# Push all tags
git push origin --tags

# List all tags
git tag -l

# Delete a tag locally
git tag -d v1.0.0

# Delete a tag on GitLab remote
git push origin --delete v1.0.0
```

---

## Chapter 5 — Branching, Merging & Merge Requests

### 5.1 Branching Strategy (GitFlow)

```
main ──────────────────────────────────────────────────────►
  │                                                         (production)
  └── develop ──────────────────────────────────────────────►
          │                                                 (integration)
          ├── feature/user-auth ──────────┐
          │                              merge back to develop
          ├── feature/payment-api ────────┘
          │
          └── release/v1.2 ──────────────────────────────────►
                                                          merge to main + tag
          hotfix/critical-bug ──────────────────────────►
          (branches from main, merges to main + develop)
```

### 5.2 Branch Commands

```bash
# Create and switch to a new branch
git checkout -b feature/user-authentication
# Modern equivalent:
git switch -c feature/user-authentication

# List all branches (local + remote)
git branch -a

# Push new branch to GitLab
git push -u origin feature/user-authentication
# -u sets upstream tracking (future: just git push)

# Switch between branches
git switch main
git switch feature/user-authentication

# Delete a local branch (safe — checks if merged)
git branch -d feature/user-authentication

# Force delete (unmerged branch)
git branch -D feature/user-authentication

# Delete remote branch on GitLab
git push origin --delete feature/user-authentication
```

### 5.3 Merge Strategies

```bash
# Regular merge (creates merge commit)
git merge feature/user-auth
# History: A─B─C─M  (M = merge commit)
#               │
#          D─E─F (feature branch)

# Squash merge (squashes all feature commits into one)
git merge --squash feature/user-auth
git commit -m "feat: add user authentication"
# History: A─B─C─G  (G = squashed commit)

# Rebase (replays feature commits on top of main)
git rebase main
# History: A─B─C─D'─E'─F'  (clean linear history)
```

### 5.4 Creating a Merge Request (MR)

**Via UI:**
```
GitLab → Your Project → Merge Requests → New Merge Request
  Source branch: feature/user-authentication
  Target branch: main
  Title: feat: Add user authentication
  Description: (use MR template)
  Assignees: yourself
  Reviewers: senior developer
  Labels: feature, in-review
  Milestone: v1.2
```

**MR Description Template** (`.gitlab/merge_request_templates/feature.md`):

```markdown
## What does this MR do?
<!-- Brief description of changes -->

## Why was this needed?
<!-- Link to issue: Closes #123 -->

## How to test?
1. Step one
2. Step two

## Checklist
- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] No secrets committed
- [ ] Pipeline passes
```

### 5.5 Resolving Merge Conflicts

```bash
# Scenario: main has new commits, your branch conflicts

# Step 1: Update your local main
git switch main
git pull origin main

# Step 2: Switch to your feature branch
git switch feature/user-auth

# Step 3: Merge main into your branch
git merge main
# OR rebase (cleaner):
git rebase main

# Step 4: Git shows conflicts
# CONFLICT (content): Merge conflict in src/auth.py
# Open the file:
# <<<<<<< HEAD (your branch)
# def login():
#     return "feature version"
# =======
# def login():
#     return "main version"
# >>>>>>> main

# Step 5: Edit the file to the correct version, remove markers
# Save the file

# Step 6: Mark as resolved
git add src/auth.py
git rebase --continue   # if rebasing
# OR
git commit              # if merging

# Step 7: Push the resolved branch
git push origin feature/user-auth --force-with-lease
# --force-with-lease is safer than --force (fails if remote was updated)
```

---

## Chapter 6 — GitLab CI/CD Fundamentals

### 6.1 What is CI/CD?

```
CI = Continuous Integration
  Every code push triggers automated: Build → Test → Static Analysis

CD = Continuous Delivery / Deployment
  Continuous Delivery: Auto-deploys to staging, manual approval for production
  Continuous Deployment: Fully automated all the way to production
```

### 6.2 Pipeline Flow

```
git push → GitLab detects .gitlab-ci.yml → Pipeline Created
                                                  │
                              ┌───────────────────┼───────────────────┐
                              ▼                   ▼                   ▼
                         [Stage: build]      [Stage: test]      [Stage: deploy]
                              │                   │                   │
                         ┌────┴────┐        ┌────┴────┐        ┌────┴────┐
                         │compile  │        │unit_test│        │deploy   │
                         │docker   │        │lint     │        │staging  │
                         │build    │        │sast     │        │deploy   │
                         └─────────┘        └─────────┘        │prod     │
                                                                └─────────┘
```

### 6.3 Your First `.gitlab-ci.yml`

```yaml
# .gitlab-ci.yml — place in root of repository

# Define the stages (order matters)
stages:
  - build
  - test
  - deploy

# A simple job
hello_world:
  stage: build
  script:
    - echo "Hello from GitLab CI!"
    - echo "Branch: $CI_COMMIT_BRANCH"
    - echo "Pipeline ID: $CI_PIPELINE_ID"
```

---

## Chapter 7 — `.gitlab-ci.yml` — Line by Line

### 7.1 Full Annotated Example

```yaml
# ============================================================
# .gitlab-ci.yml — Complete Annotated Example
# Project: Python Flask Web App
# ============================================================

# --- GLOBAL IMAGE ---
# Default Docker image used by all jobs unless overridden
image: python:3.11-slim

# --- STAGES ---
# Defines the ORDER of execution.
# Jobs in the same stage run in PARALLEL.
# Next stage only starts when ALL jobs in current stage pass.
stages:
  - build      # Stage 1: Install dependencies, compile
  - test       # Stage 2: Run tests
  - security   # Stage 3: Security scans
  - deploy     # Stage 4: Deploy application

# --- GLOBAL VARIABLES ---
# Available in ALL jobs as environment variables
variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"   # Cache pip packages
  APP_NAME: "my-flask-app"
  DOCKER_IMAGE: "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"

# --- CACHE ---
# Cache persists between pipeline runs (saves time)
# Key: cache is unique per branch
cache:
  key: "$CI_COMMIT_REF_SLUG"    # e.g., "feature-user-auth"
  paths:
    - .cache/pip                 # Cache pip downloads
    - venv/                      # Cache virtual environment

# ============================================================
# STAGE: build
# ============================================================

install_dependencies:
  stage: build

  # Override global image for this job
  image: python:3.11-slim

  # Commands to run
  script:
    - python -m venv venv                      # Create virtual env
    - source venv/bin/activate                 # Activate it
    - pip install --upgrade pip                # Upgrade pip
    - pip install -r requirements.txt          # Install app deps
    - pip install -r requirements-dev.txt      # Install dev/test deps

  # Artifacts: files produced by this job, passed to next stages
  artifacts:
    paths:
      - venv/                    # Pass venv to test jobs
    expire_in: 1 hour            # Auto-delete after 1 hour

  # Only run on these branches
  only:
    - main
    - develop
    - merge_requests             # Run on every MR

# ============================================================
# STAGE: test
# ============================================================

unit_tests:
  stage: test

  # This job depends on install_dependencies artifact (venv/)
  script:
    - source venv/bin/activate
    - pytest tests/unit/                       # Run unit tests
      --junitxml=report.xml                    # JUnit format report
      --cov=src/                               # Coverage for src/
      --cov-report=xml:coverage.xml            # Coverage XML report
      --cov-report=html:coverage_html/         # HTML coverage report
      -v                                       # Verbose output

  # Test reports: GitLab parses these and shows in MR UI
  artifacts:
    when: always                               # Save even on failure
    reports:
      junit: report.xml                        # Shows test results in MR
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml                     # Shows coverage in MR
    paths:
      - coverage_html/                         # Download HTML report
    expire_in: 1 week

  # Extract coverage % from pytest output (shows in pipeline)
  coverage: '/TOTAL.*\s+(\d+%)$/'

lint_code:
  stage: test

  script:
    - source venv/bin/activate
    - flake8 src/ --max-line-length=120        # PEP8 linter
    - black src/ --check                       # Code formatter check
    - isort src/ --check-only                  # Import sorting check

  # Allow lint to fail without failing the pipeline (optional)
  allow_failure: false                         # true = warning, not error

integration_tests:
  stage: test

  # Run a service alongside the job (database for integration tests)
  services:
    - name: postgres:15                        # PostgreSQL container
      alias: db                               # Accessible as "db" hostname
    - name: redis:7                            # Redis container
      alias: redis

  variables:
    DATABASE_URL: "postgresql://user:pass@db:5432/testdb"
    REDIS_URL: "redis://redis:6379"
    POSTGRES_DB: testdb
    POSTGRES_USER: user
    POSTGRES_PASSWORD: pass

  script:
    - source venv/bin/activate
    - pytest tests/integration/ -v

# ============================================================
# STAGE: security
# ============================================================

# GitLab built-in SAST (Static Application Security Testing)
# Scans your code for vulnerabilities
sast:
  stage: security
  # This is a GitLab template — no script needed
  include:
    - template: Security/SAST.gitlab-ci.yml

dependency_scanning:
  stage: security
  include:
    - template: Security/Dependency-Scanning.gitlab-ci.yml

secret_detection:
  stage: security
  include:
    - template: Security/Secret-Detection.gitlab-ci.yml

# ============================================================
# STAGE: deploy
# ============================================================

build_docker_image:
  stage: deploy
  image: docker:24                             # Use Docker image
  services:
    - docker:24-dind                           # Docker-in-Docker service

  variables:
    DOCKER_TLS_CERTDIR: "/certs"

  before_script:
    # Login to GitLab Container Registry
    - docker login -u "$CI_REGISTRY_USER"
                   -p "$CI_REGISTRY_PASSWORD"
                   "$CI_REGISTRY"

  script:
    # Build Docker image tagged with commit SHA
    - docker build -t "$DOCKER_IMAGE" .
    # Also tag as 'latest' for the branch
    - docker tag "$DOCKER_IMAGE" "$CI_REGISTRY_IMAGE:latest"
    # Push both tags to GitLab Container Registry
    - docker push "$DOCKER_IMAGE"
    - docker push "$CI_REGISTRY_IMAGE:latest"

  # Only run on main branch
  only:
    - main

deploy_staging:
  stage: deploy
  image: alpine:latest

  before_script:
    - apk add --no-cache openssh-client       # Install SSH
    - eval $(ssh-agent -s)                    # Start SSH agent
    - echo "$STAGING_SSH_KEY" | ssh-add -     # Add private key from CI var
    - mkdir -p ~/.ssh
    - ssh-keyscan -H "$STAGING_HOST" >> ~/.ssh/known_hosts

  script:
    # SSH into staging server and deploy
    - ssh deploy@$STAGING_HOST "
        docker pull $DOCKER_IMAGE &&
        docker stop $APP_NAME || true &&
        docker rm $APP_NAME || true &&
        docker run -d
          --name $APP_NAME
          -p 5000:5000
          -e DATABASE_URL=$PROD_DATABASE_URL
          $DOCKER_IMAGE
      "

  environment:
    name: staging
    url: https://staging.myapp.com

  only:
    - main

deploy_production:
  stage: deploy

  script:
    - echo "Deploying to production..."
    # Add production deployment commands here

  environment:
    name: production
    url: https://myapp.com

  # MANUAL: requires a human to click the play button
  when: manual

  # Only run from main branch
  only:
    - main
```

### 7.2 Predefined CI/CD Variables (Most Used)

```yaml
# GitLab provides these automatically — no configuration needed

$CI_COMMIT_SHA          # Full commit hash: a3f4b2c...
$CI_COMMIT_SHORT_SHA    # Short hash: a3f4b2c
$CI_COMMIT_BRANCH       # Branch name: main, develop
$CI_COMMIT_REF_SLUG     # URL-safe branch: feature-user-auth
$CI_COMMIT_MESSAGE      # Commit message text
$CI_COMMIT_TAG          # Tag name (if triggered by a tag)
$CI_COMMIT_AUTHOR       # Author name <email>

$CI_PIPELINE_ID         # Unique pipeline ID: 123456
$CI_PIPELINE_URL        # Full URL to the pipeline
$CI_JOB_ID              # Unique job ID
$CI_JOB_NAME            # Job name: unit_tests
$CI_JOB_STATUS          # Job status: success, failed

$CI_PROJECT_ID          # Project numeric ID
$CI_PROJECT_NAME        # Project name: my-flask-app
$CI_PROJECT_PATH        # Full path: mygroup/my-flask-app
$CI_PROJECT_DIR         # Path where repo is cloned: /builds/...
$CI_PROJECT_URL         # https://gitlab.com/mygroup/my-flask-app

$CI_REGISTRY            # Registry URL: registry.gitlab.com
$CI_REGISTRY_IMAGE      # Full image path: registry.gitlab.com/group/project
$CI_REGISTRY_USER       # Registry login user
$CI_REGISTRY_PASSWORD   # Registry login password

$CI_MERGE_REQUEST_ID    # MR ID (only in merge request pipelines)
$CI_MERGE_REQUEST_IID   # MR internal ID
$CI_MERGE_REQUEST_SOURCE_BRANCH_NAME  # Source branch

$GITLAB_USER_NAME       # Who triggered the pipeline
$GITLAB_USER_EMAIL      # Their email
```

### 7.3 Job Control Keywords

```yaml
# --- RULES (modern approach, replaces only/except) ---
deploy_prod:
  rules:
    # Only run on main branch, and only on push events
    - if: '$CI_COMMIT_BRANCH == "main" && $CI_PIPELINE_SOURCE == "push"'
      when: on_success
    # Run on tags matching v*.*.* pattern
    - if: '$CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/'
      when: on_success
    # Skip for all other cases
    - when: never

# --- NEEDS (DAG — override stage ordering) ---
# deploy_staging can start as soon as build_docker_image finishes
# without waiting for all other test jobs
deploy_staging:
  stage: deploy
  needs:
    - job: build_docker_image
      artifacts: true     # Download artifacts from that job

# --- EXTENDS (reuse job definitions) ---
.base_test:               # Jobs starting with . are hidden (templates)
  image: python:3.11-slim
  before_script:
    - pip install -r requirements.txt

unit_tests:
  extends: .base_test     # Inherits image and before_script
  script:
    - pytest tests/unit/

lint:
  extends: .base_test
  script:
    - flake8 src/

# --- PARALLEL (run job N times with different variables) ---
test_matrix:
  parallel:
    matrix:
      - PYTHON_VERSION: ["3.9", "3.10", "3.11"]
        DATABASE: ["postgres", "mysql"]
  image: python:$PYTHON_VERSION
  script:
    - echo "Testing Python $PYTHON_VERSION with $DATABASE"

# --- TRIGGER (start another project's pipeline) ---
trigger_downstream:
  trigger:
    project: mygroup/another-project
    branch: main
    strategy: depend      # Wait for downstream pipeline
```

---

## Chapter 8 — GitLab Runners

### 8.1 What is a Runner?

A **Runner** is an agent (a machine or container) that picks up CI/CD jobs and executes them.

```
GitLab Server                           Runner Machine
     │                                       │
     │── assigns job to runner ─────────────►│
     │                                       │── executes scripts
     │                                       │── sends logs back
     │◄────────── reports success/failure ───│
```

### 8.2 Runner Types

| Type | Description | Best For |
|---|---|---|
| **Shared Runner** | GitLab.com managed, free 400 min/month | Quick start, OSS projects |
| **Group Runner** | Shared across all projects in a group | Team projects |
| **Project Runner** | Dedicated to one project | Special requirements |
| **Self-hosted** | Your own machine/EC2 | Unlimited minutes, private data |

### 8.3 Register a Self-Hosted Runner on AWS EC2

```bash
# Step 1: Launch EC2 instance (Amazon Linux 2 or Ubuntu)
# Instance type: t3.medium (minimum for builds)

# Step 2: SSH into EC2
ssh -i your-key.pem ec2-user@<EC2-PUBLIC-IP>

# Step 3: Install GitLab Runner
# For Ubuntu:
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install gitlab-runner

# For Amazon Linux / RHEL:
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.rpm.sh" | sudo bash
sudo yum install gitlab-runner

# Step 4: Get registration token
# GitLab → Your Project → Settings → CI/CD → Runners → Registration token

# Step 5: Register the runner
sudo gitlab-runner register
# Interactive prompts:
#   URL: https://gitlab.com
#   Token: <paste-registration-token>
#   Description: my-aws-ec2-runner
#   Tags: aws, ec2, production
#   Executor: docker
#   Default image: alpine:latest

# Step 6: Start the runner
sudo systemctl start gitlab-runner
sudo systemctl enable gitlab-runner   # Start on boot
sudo systemctl status gitlab-runner   # Check status

# Step 7: Verify in GitLab
# GitLab → Settings → CI/CD → Runners
# You should see a green dot next to your runner
```

### 8.4 Runner Executors Explained

```yaml
# Shell executor — runs directly on host machine
# Fast but no isolation
executor: shell

# Docker executor — runs in Docker containers
# Isolated, clean environment per job (RECOMMENDED)
executor: docker
# config.toml:
[[runners]]
  executor = "docker"
  [runners.docker]
    image = "alpine:latest"
    volumes = ["/cache"]

# Docker Machine executor — auto-scales runners on AWS
executor: docker+machine

# Kubernetes executor — runs as pods in K8s cluster
executor: kubernetes
```

### 8.5 Runner Configuration (`/etc/gitlab-runner/config.toml`)

```toml
concurrent = 4            # Max jobs running simultaneously
check_interval = 0        # How often to poll for jobs (0 = default)

[[runners]]
  name = "my-aws-runner"
  url = "https://gitlab.com"
  token = "your-runner-token"
  executor = "docker"

  [runners.docker]
    tls_verify = false
    image = "python:3.11-slim"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    shm_size = 0

  [runners.cache]
    Type = "s3"
    Path = "gitlab-runner-cache"
    Shared = true
    [runners.cache.s3]
      ServerAddress = "s3.amazonaws.com"
      BucketName = "my-runner-cache-bucket"
      BucketLocation = "us-east-1"
      Insecure = false
```

---

## Chapter 9 — Advanced CI/CD Pipelines

### 9.1 Multi-Project Pipelines

```yaml
# Parent pipeline (.gitlab-ci.yml in parent project)
stages:
  - triggers

trigger_backend:
  stage: triggers
  trigger:
    project: mygroup/backend-service
    branch: main
    strategy: depend          # Parent waits for child

trigger_frontend:
  stage: triggers
  trigger:
    project: mygroup/frontend-app
    branch: main
    strategy: depend

# Pass variables to child pipelines
trigger_with_vars:
  trigger:
    project: mygroup/child-project
  variables:
    DEPLOY_ENV: production
    APP_VERSION: $CI_COMMIT_TAG
```

### 9.2 Dynamic Pipelines (Generate CI YAML)

```yaml
# Generate a pipeline dynamically using a script
generate_pipeline:
  stage: build
  script:
    - python scripts/generate_ci.py > generated-pipeline.yml
  artifacts:
    paths:
      - generated-pipeline.yml

# Use the generated pipeline
run_generated:
  trigger:
    include:
      - artifact: generated-pipeline.yml
        job: generate_pipeline
    strategy: depend
```

### 9.3 Environments and Deployments

```yaml
# Define deployment environments with rollback capability

deploy_staging:
  script:
    - ./deploy.sh staging
  environment:
    name: staging
    url: https://staging.myapp.com
    on_stop: stop_staging     # Job to run when stopping env

stop_staging:
  script:
    - ./undeploy.sh staging
  environment:
    name: staging
    action: stop
  when: manual

deploy_production:
  script:
    - ./deploy.sh production
  environment:
    name: production
    url: https://myapp.com
  when: manual
  only:
    - main
```

### 9.4 Pipeline Schedules (Cron Jobs)

```
GitLab → CI/CD → Schedules → New Schedule
  Description: Nightly regression tests
  Interval Pattern: 0 2 * * *   (2 AM every day)
  Target branch: main
  Variables: SCHEDULE_TYPE=nightly
```

```yaml
# In .gitlab-ci.yml: run extra jobs only on schedule
nightly_tests:
  script:
    - pytest tests/ --full-regression
  rules:
    - if: '$CI_PIPELINE_SOURCE == "schedule" && $SCHEDULE_TYPE == "nightly"'
```

### 9.5 Caching vs Artifacts

```yaml
# CACHE: Persist data BETWEEN pipeline runs (speed up)
# Uploaded to shared storage, downloaded at job start
cache:
  key:
    files:
      - package.json          # Cache changes when package.json changes
  paths:
    - node_modules/

# ARTIFACTS: Pass data WITHIN a pipeline (between jobs)
# Stage 1 → artifact → Stage 2
build_job:
  artifacts:
    paths:
      - dist/                 # Pass built files to deploy job
    reports:
      junit: test-results.xml # GitLab-parsed reports
    expire_in: 30 days        # Auto-cleanup

# Download artifacts from another job
deploy_job:
  needs:
    - job: build_job
      artifacts: true         # Automatically downloads build_job artifacts
```

---

## Chapter 10 — GitLab + AWS DevOps Integration

### 10.1 Architecture Overview

```
Developer Push
      │
      ▼
   GitLab CI/CD
      │
      ├──► Build Docker Image ──► GitLab Container Registry
      │
      ├──► Run Tests
      │
      ├──► Deploy to AWS ECS ──────────────────────────────────►
      │         │                                          AWS ECS
      │         ▼                                     (Fargate Cluster)
      │    ECR (Elastic Container Registry)               │
      │         │                                    Running Containers
      │         └──► Push Image ──────────────────────────────►
      │
      └──► Terraform Apply ──────────────────────────────────►
                                                        AWS Infrastructure
                                                     (VPC, RDS, S3, etc.)
```

### 10.2 GitLab CI Variables for AWS (Secure Setup)

```
GitLab → Settings → CI/CD → Variables

Add the following (mark as Protected + Masked):
  AWS_ACCESS_KEY_ID       = AKIA...
  AWS_SECRET_ACCESS_KEY   = wJalr...
  AWS_DEFAULT_REGION      = us-east-1
  ECR_REGISTRY            = 123456789.dkr.ecr.us-east-1.amazonaws.com
  ECS_CLUSTER             = my-app-cluster
  ECS_SERVICE             = my-app-service
```

### 10.3 Deploy to AWS ECR + ECS

```yaml
# .gitlab-ci.yml — Full AWS ECS Deployment Pipeline

image: python:3.11-slim

stages:
  - build
  - test
  - push
  - deploy

variables:
  ECR_IMAGE: "$ECR_REGISTRY/my-app:$CI_COMMIT_SHORT_SHA"

# Build Docker image
build_image:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    - docker build -t my-app:local .
    - docker save my-app:local > image.tar
  artifacts:
    paths:
      - image.tar
    expire_in: 1 hour

# Push to AWS ECR
push_to_ecr:
  stage: push
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    # Install AWS CLI
    - apk add --no-cache aws-cli
    # Authenticate Docker to ECR
    - aws ecr get-login-password --region $AWS_DEFAULT_REGION |
      docker login --username AWS --password-stdin $ECR_REGISTRY
  script:
    - docker load < image.tar
    - docker tag my-app:local $ECR_IMAGE
    - docker push $ECR_IMAGE
  only:
    - main

# Deploy to AWS ECS
deploy_ecs:
  stage: deploy
  image: python:3.11-slim
  before_script:
    - pip install awscli --quiet
  script:
    # Update ECS service to use new image
    - |
      aws ecs update-service \
        --cluster $ECS_CLUSTER \
        --service $ECS_SERVICE \
        --force-new-deployment \
        --region $AWS_DEFAULT_REGION
    # Wait for service to stabilize
    - |
      aws ecs wait services-stable \
        --cluster $ECS_CLUSTER \
        --services $ECS_SERVICE \
        --region $AWS_DEFAULT_REGION
    - echo "Deployment complete!"
  environment:
    name: production
    url: https://myapp.com
  when: manual
  only:
    - main
```

### 10.4 Terraform Infrastructure with GitLab CI

```yaml
# Infrastructure as Code pipeline

stages:
  - validate
  - plan
  - apply

variables:
  TF_ROOT: "${CI_PROJECT_DIR}/terraform"
  TF_STATE_NAME: "production"

# Use GitLab-managed Terraform state (no S3 needed)
before_script:
  - cd $TF_ROOT
  - export TF_ADDRESS="${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/${TF_STATE_NAME}"
  - terraform init
      -backend-config="address=${TF_ADDRESS}"
      -backend-config="lock_address=${TF_ADDRESS}/lock"
      -backend-config="unlock_address=${TF_ADDRESS}/lock"
      -backend-config="username=${CI_JOB_JWT_V2}"
      -backend-config="password=${CI_JOB_TOKEN}"
      -backend-config="lock_method=POST"
      -backend-config="unlock_method=DELETE"
      -backend-config="retry_wait_min=5"

validate:
  stage: validate
  script:
    - terraform validate
    - terraform fmt -check

plan:
  stage: plan
  script:
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - terraform/tfplan
    expire_in: 1 week

apply:
  stage: apply
  script:
    - terraform apply tfplan
  when: manual
  only:
    - main
```

### 10.5 AWS S3 Static Site Deployment

```yaml
deploy_s3:
  stage: deploy
  image: python:3.11-slim
  before_script:
    - pip install awscli --quiet
  script:
    # Sync built files to S3
    - aws s3 sync dist/ s3://$S3_BUCKET_NAME/
      --delete                              # Remove old files
      --cache-control "max-age=31536000"    # 1 year cache for assets
    # Invalidate CloudFront cache
    - aws cloudfront create-invalidation
      --distribution-id $CLOUDFRONT_DIST_ID
      --paths "/*"
  environment:
    name: production
    url: https://myapp.com
  only:
    - main
```

---

## Chapter 11 — Real-Time Project Workflow

### 11.1 Complete Project: Python Flask App CI/CD

**Project Structure:**
```
flask-app/
├── .gitlab-ci.yml
├── Dockerfile
├── requirements.txt
├── requirements-dev.txt
├── src/
│   ├── __init__.py
│   ├── app.py
│   └── config.py
└── tests/
    ├── unit/
    │   └── test_app.py
    └── integration/
        └── test_api.py
```

**`src/app.py`:**
```python
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/health')
def health():
    return jsonify({"status": "healthy", "version": "1.0.0"})

@app.route('/api/users')
def get_users():
    return jsonify({"users": []})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

**`Dockerfile`:**
```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY src/ ./src/

EXPOSE 5000

CMD ["python", "src/app.py"]
```

**`.gitlab-ci.yml` (Complete):**
```yaml
image: python:3.11-slim

stages:
  - build
  - test
  - security
  - package
  - deploy

variables:
  DOCKER_IMAGE: "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA"

cache:
  key: "$CI_COMMIT_REF_SLUG-pip"
  paths:
    - .cache/pip/

install:
  stage: build
  script:
    - pip install -r requirements.txt -r requirements-dev.txt
      --cache-dir .cache/pip
  artifacts:
    paths:
      - .cache/pip/

unit_test:
  stage: test
  script:
    - pip install -r requirements-dev.txt --cache-dir .cache/pip
    - pytest tests/unit/ -v --junitxml=junit.xml --cov=src --cov-report=xml
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    when: always
    reports:
      junit: junit.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

lint:
  stage: test
  script:
    - pip install flake8 --cache-dir .cache/pip
    - flake8 src/ tests/ --max-line-length=120

sast:
  stage: security
  include:
    - template: Security/SAST.gitlab-ci.yml

build_docker:
  stage: package
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
  script:
    - docker build -t "$DOCKER_IMAGE" .
    - docker push "$DOCKER_IMAGE"
  only:
    - main
    - tags

deploy_staging:
  stage: deploy
  environment:
    name: staging
    url: https://staging.myapp.com
  script:
    - echo "Deploying $DOCKER_IMAGE to staging"
  only:
    - main

deploy_production:
  stage: deploy
  environment:
    name: production
    url: https://myapp.com
  script:
    - echo "Deploying $DOCKER_IMAGE to production"
  when: manual
  only:
    - tags
```

### 11.2 Day-to-Day Developer Workflow

```bash
# ─── Monday Morning: Start a new feature ───────────────────

# 1. Sync with latest code
git switch main
git pull origin main

# 2. Create feature branch (linked to issue #42)
git switch -c feature/42-add-user-profile

# 3. Work, commit frequently
# (after writing some code)
git add src/user_profile.py tests/unit/test_user_profile.py
git commit -m "feat(user): add user profile model (#42)"

# 4. Push and create MR
git push -u origin feature/42-add-user-profile
# GitLab auto-suggests creating an MR — click the URL in output

# ─── During Development: Keep branch updated ───────────────

# If main has new commits while you work:
git fetch origin
git rebase origin/main
# Fix any conflicts, then:
git push --force-with-lease

# ─── Code Review: Address feedback ─────────────────────────

# After reviewer leaves comments:
git add src/user_profile.py
git commit -m "fix: address review comments — add input validation"
git push origin feature/42-add-user-profile

# ─── Merging: After approval ───────────────────────────────

# GitLab UI → MR → Merge (squash and merge recommended)
# OR via CLI:
git switch main
git merge --squash feature/42-add-user-profile
git commit -m "feat(user): add user profile (#42)"
git push origin main

# ─── Cleanup ───────────────────────────────────────────────

git branch -d feature/42-add-user-profile
git push origin --delete feature/42-add-user-profile
```

---

## Chapter 12 — GitLab Security & Compliance

### 12.1 Built-in Security Scanning

```yaml
# Add all security scans to your pipeline
include:
  - template: Security/SAST.gitlab-ci.yml                # Source code scan
  - template: Security/DAST.gitlab-ci.yml                # Dynamic app scan
  - template: Security/Dependency-Scanning.gitlab-ci.yml # Dependency vulns
  - template: Security/Container-Scanning.gitlab-ci.yml  # Docker image scan
  - template: Security/Secret-Detection.gitlab-ci.yml    # Leaked secrets scan
  - template: Security/License-Scanning.gitlab-ci.yml    # License compliance
```

### 12.2 Prevent Secrets in Code

```bash
# Install pre-commit hooks locally (run before every commit)
pip install pre-commit

# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks

# Install hooks
pre-commit install

# Now every git commit automatically scans for secrets
```

### 12.3 Audit Events

```
GitLab → Group/Project → Security & Compliance → Audit Events
```
Shows: who pushed, who changed settings, failed login attempts, etc.

---

## Chapter 13 — Troubleshooting & Bug Fixing

### 13.1 Common CI/CD Errors and Fixes

---

**Error 1: Job stuck in "Pending" state**

```
Symptom: Job shows "pending" for more than 5 minutes

Root Cause: No runner available to pick up the job

Fix:
1. Check if runners are online:
   GitLab → Settings → CI/CD → Runners
   
2. Check runner tags — job requires a tag the runner doesn't have:
   # In .gitlab-ci.yml:
   my_job:
     tags:
       - docker     ← Runner must have this tag
   
3. Check runner is not "paused"
4. Restart runner: sudo systemctl restart gitlab-runner
```

---

**Error 2: `docker: command not found`**

```yaml
# Wrong: Shell executor can't run Docker commands without setup
my_job:
  script:
    - docker build .      # FAILS on shell executor

# Fix Option 1: Use Docker executor
my_job:
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  script:
    - docker build .

# Fix Option 2: Mount Docker socket (shell executor)
# In config.toml:
# volumes = ["/var/run/docker.sock:/var/run/docker.sock"]
```

---

**Error 3: `fatal: not a git repository`**

```yaml
# Cause: Working in wrong directory

# Fix: Use CI_PROJECT_DIR
my_job:
  script:
    - cd $CI_PROJECT_DIR    # Always start from project root
    - python setup.py
```

---

**Error 4: Permission denied on SSH deploy**

```bash
# Symptom: Permission denied (publickey)

# Fix:
# 1. Add private key to CI variable (Protected, File type)
#    Variable name: SSH_PRIVATE_KEY

# 2. In job:
before_script:
  - eval $(ssh-agent -s)
  - chmod 400 $SSH_PRIVATE_KEY          # Fix permissions
  - ssh-add $SSH_PRIVATE_KEY
  - mkdir -p ~/.ssh && chmod 700 ~/.ssh
  - ssh-keyscan -H $DEPLOY_HOST >> ~/.ssh/known_hosts

# 3. Verify the PUBLIC key is in ~/.ssh/authorized_keys on target server
```

---

**Error 5: Cache not working (different runner executes job)**

```yaml
# Fix: Use S3-backed cache shared across runners
# In config.toml on runner:
[runners.cache]
  Type = "s3"
  [runners.cache.s3]
    ServerAddress = "s3.amazonaws.com"
    BucketName = "my-runner-cache"
    BucketLocation = "us-east-1"
    Insecure = false

# In .gitlab-ci.yml: clear and rebuild cache
clear_cache:
  script:
    - echo "Clearing cache"
  cache:
    key: "$CI_COMMIT_REF_SLUG-v2"   # Bump version to invalidate old cache
    paths:
      - .cache/
```

---

**Error 6: Pipeline passes but deployment is wrong version**

```bash
# Problem: Using :latest tag instead of commit-specific tag

# Wrong:
DOCKER_IMAGE: "$CI_REGISTRY_IMAGE:latest"  # Race condition!

# Correct: Use unique immutable tag
DOCKER_IMAGE: "$CI_REGISTRY_IMAGE:$CI_COMMIT_SHA"

# Also push :latest as a convenience alias, but deploy with SHA
script:
  - docker build -t $DOCKER_IMAGE .
  - docker tag $DOCKER_IMAGE $CI_REGISTRY_IMAGE:latest
  - docker push $DOCKER_IMAGE          # Deploy THIS specific image
  - docker push $CI_REGISTRY_IMAGE:latest
```

---

**Error 7: `exit code 1` with no clear error message**

```yaml
# Debug: Add -x flag to bash for verbose output
my_job:
  script:
    - set -x              # Print every command before executing
    - your_command
    - set +x              # Turn off verbose

# Or: Run with error handling
my_job:
  script:
    - |
      set -euo pipefail   # Exit on any error, unset var, pipe failure
      your_command
```

---

### 13.2 Git Troubleshooting

```bash
# Undo last commit (keep changes in working directory)
git reset --soft HEAD~1

# Undo last commit (discard changes) — DESTRUCTIVE
git reset --hard HEAD~1

# Undo a pushed commit (safe — creates new commit)
git revert HEAD
git push origin main

# Find when a bug was introduced
git bisect start
git bisect bad                    # Current commit is broken
git bisect good v1.0.0            # This version was working
# Git checks out middle commit — test it, then:
git bisect good   # OR
git bisect bad
# Repeat until git identifies the exact bad commit
git bisect reset  # Done

# Recover a deleted branch
git reflog                        # Shows all recent HEAD movements
git checkout -b recovered-branch <SHA>   # Restore from reflog

# See what changed in last commit
git show HEAD

# See changes between two branches
git diff main..feature/my-branch

# Find which commit introduced a specific line
git log -S "def login" src/auth.py      # Search for function
git blame src/auth.py                   # Line-by-line commit info
```

---

## Chapter 14 — Best Practices

### 14.1 Pipeline Best Practices

```yaml
# ✅ DO: Use specific image tags (not :latest)
image: python:3.11.4-slim    # Pinned version

# ❌ DON'T:
image: python:latest         # Non-deterministic

# ✅ DO: Fail fast with syntax check first
validate:
  stage: .pre               # Runs before all stages
  script:
    - python -m py_compile src/*.py

# ✅ DO: Set job timeouts
long_job:
  timeout: 30 minutes       # Prevent hanging jobs

# ✅ DO: Use rules over only/except (more flexible)
deploy:
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_TAG'

# ✅ DO: Keep pipeline fast — parallelize
# Jobs in same stage run in PARALLEL — use this!
stages:
  - test

unit_test:     # These run in parallel
  stage: test
  script: pytest tests/unit/

lint:          # ← Same stage = runs at same time as unit_test
  stage: test
  script: flake8 src/

# ✅ DO: Use .pre and .post stages
setup:
  stage: .pre              # Always runs first
  script: echo "setup"

cleanup:
  stage: .post             # Always runs last (even if pipeline fails)
  script: echo "cleanup"
```

### 14.2 Security Best Practices

```yaml
# ✅ NEVER hardcode secrets — use CI/CD variables
# ❌ DON'T:
script:
  - export DB_PASSWORD="mypassword123"

# ✅ DO:
script:
  - export DB_PASSWORD="$DB_PASSWORD_VAR"   # From CI/CD variables

# ✅ Use Protected variables for production secrets
# GitLab → Settings → CI/CD → Variables → Protected ✓

# ✅ Use Masked variables to hide from logs
# GitLab → Settings → CI/CD → Variables → Masked ✓

# ✅ Rotate tokens regularly
# GitLab → User Settings → Access Tokens → Set expiry date

# ✅ Limit runner scope
runner:
  tags:
    - production-only      # Only specific runners can deploy to prod

# ✅ Add CODEOWNERS for sensitive files
# .gitlab/CODEOWNERS:
/terraform/           @devops-team
/.gitlab-ci.yml       @devops-team
/src/auth/            @security-team
```

### 14.3 Git Commit Message Convention

```
# Format: <type>(<scope>): <subject>
# Types: feat, fix, docs, style, refactor, test, chore, perf, ci

feat(auth): add OAuth2 login with Google
^    ^       ^
│    │       └─ Subject: short, imperative, lowercase
│    └─ Scope: affected module (optional)
└─ Type: feat = new feature

# More examples:
fix(api): handle null response from payment service
docs(readme): update deployment instructions
test(user): add unit tests for password reset
ci: add DAST security scanning stage
refactor(db): extract database connection to config module
chore(deps): upgrade Flask from 2.3 to 3.0

# Breaking changes — add ! or footer
feat(api)!: remove deprecated /v1/ endpoints

BREAKING CHANGE: The /v1/users endpoint has been removed.
Use /v2/users instead.
```

---

## Chapter 15 — Assignments

### Assignment 1 — Beginner
**Goal:** Set up a basic GitLab project with CI

**Tasks:**
1. Create a new GitLab project called `my-first-pipeline`
2. Add a Python script `hello.py` that prints "Hello, CI/CD!"
3. Write a `.gitlab-ci.yml` that:
   - Has one stage: `test`
   - Runs `python hello.py`
   - Uses `python:3.11-slim` image
4. Push and verify the pipeline passes

**Expected `.gitlab-ci.yml`:**
```yaml
image: python:3.11-slim

stages:
  - test

run_script:
  stage: test
  script:
    - python hello.py
```

---

### Assignment 2 — Intermediate
**Goal:** Multi-stage pipeline with testing and Docker

**Tasks:**
1. Create a Flask app (use the example from Chapter 11)
2. Write a `.gitlab-ci.yml` with stages: `test`, `build`
3. In `test` stage: run `pytest` and collect coverage
4. In `build` stage: build a Docker image (use GitLab Container Registry)
5. Only run the `build` stage on `main` branch
6. Add a scheduled pipeline to run tests every night at midnight

---

### Assignment 3 — Advanced
**Goal:** Full DevOps pipeline with AWS deployment

**Tasks:**
1. Create a Python/Node.js application
2. Build a complete CI/CD pipeline with:
   - Linting (flake8/eslint)
   - Unit tests with coverage
   - SAST security scan
   - Docker build and push to ECR
   - Automatic deploy to staging on merge to `main`
   - Manual deploy to production (requires approval)
3. Set up a self-hosted runner on an EC2 instance
4. Configure Terraform to provision an ECS cluster via GitLab CI
5. Implement rollback mechanism (redeploy previous image on failure)

---

### Assignment 4 — GitLab Flow
**Goal:** Practice real team workflow

**Tasks:**
1. Work in pairs (or solo simulating a team)
2. Create an issue: "Add user authentication"
3. Create a branch from the issue
4. Write the feature code
5. Open a Merge Request with the MR template
6. Review your own MR, leave comments
7. Resolve comments, update the MR
8. Merge and verify deployment pipeline runs

---

## Chapter 16 — Interview Questions & Answers

### Beginner Level

**Q1: What is GitLab and how does it differ from Git?**
> **A:** Git is the version control system (a tool). GitLab is a platform built on top of Git that provides: hosted repositories, CI/CD pipelines, issue tracking, container registry, and DevOps tools. Think of Git as the engine and GitLab as the car.

**Q2: What is a GitLab Pipeline?**
> **A:** A pipeline is an automated workflow defined in `.gitlab-ci.yml` that runs when code is pushed. It consists of stages (build, test, deploy) containing jobs. Jobs in the same stage run in parallel; stages run sequentially.

**Q3: What is the difference between `git merge` and `git rebase`?**
> **A:** `git merge` creates a new merge commit preserving both branch histories. `git rebase` replays commits on top of another branch creating a linear history. Rebase creates cleaner history but should NOT be used on shared public branches (rewrites history).

**Q4: What is a Merge Request?**
> **A:** A Merge Request (MR) is a request to merge one branch into another. It provides a place for code review, discussion, CI/CD status checks, and approval gates before code enters the main branch.

---

### Intermediate Level

**Q5: What are GitLab Runners and what executor types exist?**
> **A:** Runners are agents that execute CI/CD jobs. Executor types: Shell (runs on host), Docker (isolated containers, recommended), Docker Machine (auto-scaling), Kubernetes (runs as pods), VirtualBox, Parallels. Docker executor is the most common for clean, isolated, reproducible builds.

**Q6: Explain the difference between `cache` and `artifacts` in GitLab CI.**
> **A:** Cache persists data BETWEEN pipeline runs on the same runner (speeds up builds by reusing dependencies like `node_modules`). Artifacts pass data WITHIN a pipeline between stages/jobs (e.g., compiled binary from build stage passed to deploy stage). Artifacts are stored on GitLab server; cache is stored locally on runner.

**Q7: What is the `needs` keyword in GitLab CI?**
> **A:** `needs` creates a DAG (Directed Acyclic Graph) pipeline where a job can start as soon as its dependencies finish, without waiting for the entire previous stage. This reduces total pipeline duration by enabling out-of-order execution.

**Q8: How do you pass secrets to CI/CD jobs securely?**
> **A:** Use GitLab CI/CD Variables (Settings → CI/CD → Variables). Mark as Protected (only available on protected branches) and Masked (hidden from job logs). Never hardcode secrets in `.gitlab-ci.yml`. For files (SSH keys, certificates), use File-type variables.

---

### Advanced Level

**Q9: How would you design a GitLab CI/CD pipeline for a microservices architecture?**
> **A:** Use multi-project pipelines. Each microservice has its own repository and pipeline. A parent orchestrator pipeline triggers child pipelines with `trigger:`. Use `strategy: depend` to wait for child pipelines. Pass versions via variables between pipelines. Use GitLab Environments for deployment tracking across services.

**Q10: How do you implement zero-downtime deployment with GitLab CI and AWS?**
> **A:** Use ECS rolling updates or blue-green deployments. Pipeline: Build image → push to ECR → update ECS task definition → trigger ECS service update with rolling deployment config (minimum 50% healthy, maximum 200%). Use `aws ecs wait services-stable` to confirm before closing pipeline. Implement health checks on the ALB target group.

**Q11: Explain GitLab's security scanning capabilities.**
> **A:** GitLab Ultimate/Gold includes: SAST (static code analysis for vulnerabilities), DAST (dynamic testing of running app), Dependency Scanning (CVEs in libraries), Container Scanning (image vulnerabilities), Secret Detection (leaked credentials in code), License Compliance (open-source license management), Fuzzing, and API Security testing. All integrate into MR with vulnerability reports.

**Q12: What is GitLab's "Review Apps" feature?**
> **A:** Review Apps automatically deploy a live version of your application for each Merge Request, accessible at a dynamic URL. Reviewers can test the actual running app, not just the code. Configured in `.gitlab-ci.yml` using `environment: name: review/$CI_COMMIT_REF_SLUG` and `on_stop:` for cleanup when the MR is merged/closed.

**Q13: How do you handle database migrations in a CI/CD pipeline?**
> **A:** Run migrations as a separate pre-deployment job. For zero downtime: use backward-compatible migrations (additive only — add columns, don't remove). Order: 1) deploy new code (compatible with old schema), 2) run migration, 3) deploy code that uses new schema. Use tools like Flyway or Alembic. On failure: trigger rollback job that runs `migrate down`.

**Q14: What is the difference between Protected branches, Protected tags, and Protected environments?**
> **A:** Protected Branches control who can push/merge (e.g., only maintainers can push to `main`). Protected Tags control who can create tags (prevent anyone from creating release tags). Protected Environments control who can trigger deployments to environments like production (require specific roles or manual approval). All three work together to enforce change control.

**Q15: How would you reduce a slow 45-minute CI pipeline?**
> **A:** 
> 1. Parallelize jobs within stages (put independent tests in same stage)
> 2. Use `needs:` for DAG to remove unnecessary sequential waiting  
> 3. Optimize caching (cache dependencies, invalidate only when lock files change)
> 4. Use faster Docker images (Alpine instead of Ubuntu)
> 5. Only run expensive jobs on main branch (rules)
> 6. Run tests in parallel with `parallel: matrix:`
> 7. Skip CI for documentation changes: `[skip ci]` in commit message or path rules
> 8. Profile what's slow: check job durations in pipeline view

---

## Quick Reference Card

```bash
# ── Git Daily Commands ──────────────────────────────────────
git status                          # What changed?
git add -p                          # Stage interactively
git commit -m "type: message"       # Commit with convention
git push origin branch-name         # Push to GitLab
git pull --rebase origin main       # Update with rebase

# ── Branch Operations ───────────────────────────────────────
git switch -c feature/name          # Create + switch branch
git switch main                     # Switch branch
git branch -d feature/name          # Delete local branch
git push origin --delete feature/name  # Delete remote branch

# ── Debugging ───────────────────────────────────────────────
git log --oneline --graph --all     # Visual history
git diff main..feature/branch       # Compare branches
git blame filename                  # Who wrote each line?
git bisect start/good/bad           # Find bad commit
git reflog                          # Recovery log

# ── Undoing Things ──────────────────────────────────────────
git reset --soft HEAD~1             # Undo commit, keep changes
git restore filename                # Discard file changes
git revert HEAD                     # Safe undo (creates commit)
git stash / git stash pop           # Temporarily save changes

# ── GitLab Runner ───────────────────────────────────────────
sudo gitlab-runner register         # Register new runner
sudo systemctl restart gitlab-runner # Restart runner
sudo gitlab-runner status           # Check status
sudo gitlab-runner verify           # Test connectivity
gitlab-runner exec docker job_name  # Test job locally
```

---

*Last updated: 2026-06-18 | GitLab Trainer & AWS DevOps Mentor*
