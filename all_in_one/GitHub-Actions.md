# GitHub Actions — Zero to Hero

> A complete, beginner-to-advanced guide to GitHub Actions for CI/CD and workflow automation.

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

### 1.1 What is GitHub Actions?

**GitHub Actions** is a **CI/CD and automation platform** built directly into GitHub. It lets you automate workflows — building, testing, and deploying code — triggered by events in your repository (pushes, pull requests, releases, issues, schedules, and more). Launched in 2019, it has become one of the most popular CI/CD tools because it requires no external server and lives alongside your code.

### 1.2 Why GitHub Actions?

| Feature | Explanation |
|---------|-------------|
| **Native to GitHub** | No separate CI server; configured in `.github/workflows/` |
| **Event-driven** | React to 30+ event types (push, PR, release, schedule, webhook) |
| **Marketplace** | Thousands of reusable, community-built actions |
| **Matrix builds** | Test across many OS/language-version combinations in parallel |
| **Hosted & self-hosted runners** | Use GitHub's VMs or your own infrastructure |
| **Free tier** | Generous free minutes for public repos and personal use |
| **Secrets & environments** | Built-in secret management and deployment gates |

### 1.3 How it works

GitHub Actions follows an **event → workflow → jobs → steps** model:

```
   Event (push, PR, schedule…)
        │ triggers
        ▼
   Workflow  (.github/workflows/ci.yml)
        │ contains one or more
        ▼
   Jobs  (run in parallel by default, on a runner each)
        │ each job has ordered
        ▼
   Steps  (run commands or use actions, sequentially)
```

- **Workflow:** An automated process defined in a YAML file in `.github/workflows/`.
- **Event:** What triggers the workflow.
- **Job:** A set of steps executed on the same **runner**; jobs run in parallel unless they declare dependencies.
- **Step:** An individual task — either a shell command (`run`) or a reusable **action** (`uses`).
- **Runner:** A server (GitHub-hosted or self-hosted) that executes a job.
- **Action:** A reusable unit of code (from the Marketplace or your own).

### 1.4 Architecture

```
   GitHub Repo (events) ──► GitHub Actions service ──► schedules workflows
                                       │
              ┌────────────────────────┼────────────────────────┐
              ▼                        ▼                         ▼
        +-----------+            +-----------+             +-------------+
        | Runner 1  |            | Runner 2  |             | Self-hosted |
        | (ubuntu)  |            | (windows) |             | runner      |
        | Job A     |            | Job B     |             | Job C       |
        +-----------+            +-----------+             +-------------+
            │ uses actions from Marketplace + your repo
            ▼
        Artifacts, deployments, status checks → back to GitHub
```

### 1.5 When to use GitHub Actions

- CI/CD for any project hosted on GitHub.
- Automating routine repo tasks (labeling, releasing, dependency updates).
- Scheduled jobs (cron) and event-driven automation.
- Building/publishing packages, containers, and docs.

When **not** ideal: code hosted elsewhere (GitLab/Bitbucket have their own CI), very heavy self-hosted enterprise needs that favor Jenkins, or complex multi-repo orchestration better served by dedicated CD tools (Argo CD for GitOps).

---

## 2. Installation & Setup

### 2.1 No installation required

GitHub Actions is built into GitHub. You only need a repository and a workflow file. Everything runs on GitHub-hosted runners by default.

### 2.2 Create your first workflow

Add a file at `.github/workflows/ci.yml`:

```yaml
name: CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Say hello
        run: echo "Hello, GitHub Actions!"
```

Commit and push — the workflow runs automatically. View results under the **Actions** tab.

### 2.3 Anatomy of the file

```yaml
name: CI                    # workflow name (shown in UI)
on: [push]                  # event(s) that trigger it
jobs:                       # one or more jobs
  build:                    # job ID
    runs-on: ubuntu-latest  # runner OS
    steps:                  # ordered steps
      - uses: actions/checkout@v4   # an action
      - run: echo "hi"              # a shell command
```

### 2.4 Test locally (optional)

Use [`act`](https://github.com/nektos/act) to run workflows locally in Docker:

```bash
# install act, then:
act push                    # simulate a push event
act -j build                # run a specific job
```

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 Events (triggers)

```yaml
on:
  push:
    branches: [main, develop]
    paths: ['src/**']           # only when these files change
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'          # daily at 02:00 UTC
  workflow_dispatch:             # manual trigger (UI button)
  release:
    types: [published]
```

Common events: `push`, `pull_request`, `schedule`, `workflow_dispatch`, `release`, `issues`, `workflow_call` (reusable).

#### 3.1.2 Jobs and runners

```yaml
jobs:
  test:
    runs-on: ubuntu-latest      # also: windows-latest, macos-latest
    steps:
      - run: echo "testing"
```

GitHub-hosted runners come preinstalled with common tools (git, Docker, language runtimes). Each job gets a fresh VM.

#### 3.1.3 Steps: run vs. uses

```yaml
steps:
  - uses: actions/checkout@v4        # reusable action (from Marketplace/repo)

  - name: Install deps               # named shell step
    run: npm ci

  - name: Multi-line script
    run: |
      echo "line 1"
      echo "line 2"
    shell: bash
```

#### 3.1.4 Using Marketplace actions

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: actions/setup-node@v4
    with:
      node-version: '20'
  - run: npm ci && npm test
```

Pin actions to a version (`@v4`) or, for security, a full commit SHA.

#### 3.1.5 Viewing results

Each push/PR shows status checks. The **Actions** tab displays logs per job/step, with green ✓ / red ✗ indicators. PRs can require workflows to pass before merging.

### 3.2 INTERMEDIATE

#### 3.2.1 Environment variables and contexts

```yaml
env:
  GLOBAL_VAR: production          # workflow-level

jobs:
  build:
    env:
      JOB_VAR: value              # job-level
    steps:
      - run: echo "$GLOBAL_VAR - $JOB_VAR"
      - run: echo "Branch is ${{ github.ref_name }}"
      - run: echo "Actor is ${{ github.actor }}"
```

**Contexts** expose runtime data via `${{ }}`:
- `github` — event, repo, ref, actor, sha.
- `env` — environment variables.
- `secrets` — secret values.
- `job`, `steps`, `runner`, `matrix`, `needs`.

#### 3.2.2 Secrets and variables

Store secrets under **Settings → Secrets and variables → Actions**.

```yaml
steps:
  - name: Deploy
    env:
      API_KEY: ${{ secrets.API_KEY }}
    run: ./deploy.sh
```

> Secrets are masked in logs. Never `echo` a secret. Repository, environment, and organization-level secrets are supported.

#### 3.2.3 Job dependencies and outputs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.set.outputs.name }}
    steps:
      - id: set
        run: echo "name=app-${{ github.sha }}" >> "$GITHUB_OUTPUT"

  deploy:
    needs: build                  # waits for build to succeed
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying ${{ needs.build.outputs.artifact-name }}"
```

`needs` creates an execution order; otherwise jobs run in parallel.

#### 3.2.4 Matrix builds

Run a job across combinations in parallel:

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [18, 20, 22]
        exclude:
          - os: macos-latest
            node: 18
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci && npm test
```

This produces a grid of jobs (3 OS × 3 Node = 9, minus excludes).

#### 3.2.5 Conditionals

```yaml
steps:
  - name: Deploy to prod
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    run: ./deploy.sh

  - name: Run on failure
    if: failure()
    run: ./notify-failure.sh

  - name: Always run cleanup
    if: always()
    run: ./cleanup.sh
```

Status functions: `success()`, `failure()`, `cancelled()`, `always()`.

#### 3.2.6 Artifacts and caching

```yaml
steps:
  # Cache dependencies for faster runs
  - uses: actions/cache@v4
    with:
      path: ~/.npm
      key: npm-${{ hashFiles('package-lock.json') }}
      restore-keys: npm-

  # Upload build output
  - uses: actions/upload-artifact@v4
    with:
      name: dist
      path: dist/

  # In another job, download it
  - uses: actions/download-artifact@v4
    with:
      name: dist
```

- **Artifacts** pass files between jobs and persist after the run.
- **Caching** speeds up runs by reusing dependencies.

#### 3.2.7 Services (containers)

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: pass
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready --health-interval 10s --health-retries 5
    steps:
      - run: psql -h localhost -U postgres -c "SELECT 1;"
        env: { PGPASSWORD: pass }
```

### 3.3 ADVANCED

#### 3.3.1 Reusable workflows

Define once, call from many repos/workflows:

```yaml
# .github/workflows/reusable-build.yml
on:
  workflow_call:
    inputs:
      node-version:
        type: string
        default: '20'
    secrets:
      token:
        required: true
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: ${{ inputs.node-version }} }
      - run: npm ci && npm run build
```

```yaml
# caller
jobs:
  call-build:
    uses: ./.github/workflows/reusable-build.yml
    with: { node-version: '22' }
    secrets: { token: ${{ secrets.TOKEN }} }
```

#### 3.3.2 Composite actions

Bundle multiple steps into a custom action:

```yaml
# .github/actions/setup/action.yml
name: 'Setup'
runs:
  using: 'composite'
  steps:
    - uses: actions/setup-node@v4
      with: { node-version: '20' }
    - run: npm ci
      shell: bash
```

```yaml
steps:
  - uses: ./.github/actions/setup
```

Action types: **composite** (steps), **JavaScript** (Node), **Docker container**.

#### 3.3.3 Environments and deployment protection

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://myapp.com
    steps:
      - run: ./deploy.sh
```

**Environments** (Settings → Environments) add: required reviewers (manual approval), wait timers, branch restrictions, and environment-scoped secrets — perfect for gated production deploys.

#### 3.3.4 OIDC for keyless cloud auth

Instead of long-lived cloud keys, use **OpenID Connect** to obtain short-lived credentials:

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/gha-deploy
          aws-region: us-east-1
      - run: aws s3 sync ./dist s3://my-bucket
```

This is the **recommended** way to authenticate to AWS/Azure/GCP — no secrets stored.

#### 3.3.5 Concurrency control

```yaml
concurrency:
  group: deploy-${{ github.ref }}
  cancel-in-progress: true        # cancel older runs of the same group
```

Prevents overlapping deploys and saves minutes.

#### 3.3.6 Permissions and security hardening

```yaml
permissions:
  contents: read                  # least privilege for GITHUB_TOKEN
  pull-requests: write
```

Security best practices:
- Set least-privilege `permissions` for the `GITHUB_TOKEN`.
- **Pin actions to a full commit SHA** (not just a tag) to prevent supply-chain attacks.
- Use **OIDC** instead of static cloud secrets.
- Restrict which actions can run (org policy).
- Be careful with `pull_request_target` and untrusted forks (secret exposure risk).

#### 3.3.7 Self-hosted runners

```bash
# On your own machine (from repo Settings → Actions → Runners)
./config.sh --url https://github.com/org/repo --token <token>
./run.sh
```

```yaml
jobs:
  build:
    runs-on: self-hosted          # or labels: [self-hosted, gpu]
```

Use for custom hardware, private networks, or special tooling. Secure them (ephemeral runners, isolation) — never use self-hosted runners on public repos with untrusted PRs.

#### 3.3.8 Building and pushing Docker images

```yaml
jobs:
  docker:
    runs-on: ubuntu-latest
    permissions: { contents: read, packages: write }
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## 4. Hands-on Tasks

### Task 1: Create a hello-world workflow

Add `.github/workflows/hello.yml` with a single `echo` step, push, and check the Actions tab.

**Expected:** Green run with your message in the logs.

### Task 2: Build and test a Node/Python app

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: actions/setup-node@v4
    with: { node-version: '20' }
  - run: npm ci
  - run: npm test
```

**Expected:** Dependencies install, tests run, status reported on the commit/PR.

### Task 3: Add a matrix build

Use the matrix from section 3.2.4 (multiple OS/versions).

**Expected:** A grid of parallel jobs in the Actions UI.

### Task 4: Use a secret

Add a repo secret `MY_SECRET`, reference it via `${{ secrets.MY_SECRET }}` in an env var.

**Expected:** The value is masked (`***`) in logs.

### Task 5: Cache dependencies

Add `actions/cache@v4` keyed on the lockfile hash; run twice.

**Expected:** Second run restores the cache and runs faster.

### Task 6: Pass data between jobs

Set a `GITHUB_OUTPUT` in `build`, consume it in `deploy` via `needs`.

**Expected:** `deploy` prints the value produced by `build`.

### Task 7: Add a conditional deploy step

Only run a deploy step when `github.ref == 'refs/heads/main'`.

**Expected:** Deploy runs on main but is skipped on feature branches.

### Task 8: Upload and download an artifact

Build output → `upload-artifact`; another job → `download-artifact`.

**Expected:** Files transfer between jobs and appear under the run's artifacts.

### Task 9: Use a service container

Add a `postgres` service and run a query against it.

**Expected:** The step connects to the DB and returns `1`.

### Task 10: Manual deploy with an environment gate

Add `workflow_dispatch` + an `environment: production` job requiring a reviewer.

**Expected:** The job pauses for approval before deploying.

---

## 5. Projects

### Project 1 (Beginner): CI for an Application

**Goal:** Automatically lint, build, and test on every push and PR.

Steps:
1. Add a workflow triggered on `push`/`pull_request`.
2. Steps: checkout → setup language → install → lint → test.
3. Cache dependencies.
4. Add a build status badge to the README.
5. Require the workflow to pass before merging (branch protection).

**Skills:** events, steps, actions, caching, status checks.

### Project 2 (Intermediate): Build, Containerize, and Publish

**Goal:** On release, build a Docker image and publish to GHCR.

Steps:
1. Trigger on `release: published` and `workflow_dispatch`.
2. Run a test matrix across versions.
3. Build a multi-stage Docker image with `build-push-action` + GHA cache.
4. Push to GitHub Container Registry using `GITHUB_TOKEN`.
5. Tag images with semantic version + SHA.
6. Upload build artifacts and generate release notes.

**Skills:** matrix, Docker, registries, permissions, artifacts, releases.

### Project 3 (Advanced): Full CI/CD with OIDC, Environments, and Reusable Workflows

**Goal:** A production pipeline deploying to a cloud with no stored secrets.

Steps:
1. Create a **reusable workflow** for build/test shared across repos.
2. Use a **composite action** for repeated setup.
3. Deploy to staging automatically; production behind an **environment** with required reviewers.
4. Authenticate to AWS/Azure/GCP via **OIDC** (no long-lived keys).
5. Add **concurrency** control and least-privilege `permissions`.
6. **Pin actions to commit SHAs**; add dependency scanning (Dependabot/CodeQL).
7. Implement rollback and Slack notifications.

**Skills:** reusable workflows, composite actions, OIDC, environments, security hardening, CodeQL.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Pin actions to a commit SHA** (or at least a major version) to avoid supply-chain risk.
- **Set least-privilege `permissions`** for `GITHUB_TOKEN`.
- **Use OIDC** for cloud auth instead of storing long-lived secrets.
- **Cache dependencies** to speed up and reduce cost.
- **Use matrix builds** to cover multiple versions/OS efficiently.
- **Use reusable workflows and composite actions** to avoid duplication.
- **Gate production with environments** (required reviewers, branch limits).
- **Add concurrency control** to cancel superseded runs.
- **Keep secrets out of logs**; never echo them.
- **Fail fast** on lint/test; keep workflows readable and modular.
- **Use `paths`/`branches` filters** to avoid unnecessary runs.

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Using `@main`/floating tags for actions | Supply-chain compromise | Pin to commit SHA |
| Over-broad `GITHUB_TOKEN` permissions | Privilege escalation | Set least-privilege `permissions` |
| Storing cloud keys as secrets | Leakage risk | Use OIDC short-lived creds |
| `pull_request_target` with fork code | Secret exfiltration | Avoid; treat fork code as untrusted |
| No caching | Slow, costly runs | Use `actions/cache` |
| Echoing secrets | Exposure | Never print secrets |
| Self-hosted runners on public repos | RCE from untrusted PRs | Use hosted/ephemeral, isolate |
| Monolithic copy-pasted workflows | Hard to maintain | Reusable workflows/composite actions |
| No concurrency control | Overlapping deploys, wasted minutes | `concurrency` with cancel-in-progress |
| Running on every push to everything | Wasted minutes | `paths`/`branches` filters |

---

## 7. Interview Questions

### Beginner

1. **What is GitHub Actions?**
   *A CI/CD and automation platform built into GitHub that runs event-triggered workflows defined in YAML.*

2. **Where do workflow files live?**
   *In `.github/workflows/` in the repository.*

3. **What is the difference between a job and a step?**
   *A job is a set of steps run on one runner; a step is a single command or action. Jobs run in parallel by default; steps run sequentially.*

4. **What is a runner?**
   *A server (GitHub-hosted or self-hosted) that executes a job.*

5. **What's the difference between `run` and `uses`?**
   *`run` executes shell commands; `uses` invokes a reusable action.*

### Intermediate

6. **How do you pass data between jobs?**
   *Via job `outputs` + `needs`, or by uploading/downloading artifacts.*

7. **What is a matrix build?**
   *A strategy that runs a job across combinations of variables (e.g., OS × language version) in parallel.*

8. **How are secrets handled?**
   *Stored in repo/environment/org settings, injected via `${{ secrets.NAME }}`, and masked in logs.*

9. **What does `needs` do?**
   *Declares job dependencies, forcing execution order and enabling output passing.*

10. **What's the difference between artifacts and caching?**
    *Artifacts persist outputs to share between jobs and after a run; caches speed up runs by reusing dependencies but aren't meant as deliverables.*

### Advanced

11. **What are reusable workflows and composite actions?**
    *Reusable workflows (`workflow_call`) are entire workflows invoked by others; composite actions bundle multiple steps into a single custom action. Both reduce duplication.*

12. **Why use OIDC for cloud deployments?**
    *It exchanges a short-lived token for cloud credentials at runtime, eliminating stored long-lived secrets and reducing breach risk.*

13. **How do you secure GitHub Actions against supply-chain attacks?**
    *Pin actions to commit SHAs, set least-privilege `permissions`, restrict allowed actions, use OIDC, and avoid running untrusted fork code with secrets (`pull_request_target`).*

14. **What are environments and their protections?**
    *Named deployment targets with rules: required reviewers (manual approval), wait timers, branch restrictions, and environment-scoped secrets.*

15. **When and how would you use self-hosted runners safely?**
    *For custom hardware/private networks; secure with ephemeral runners, network isolation, least privilege, and never on public repos accepting untrusted PRs.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** Where are GitHub Actions workflows stored?
- A) `.github/`  B) `.github/workflows/`  C) `.ci/`  D) root of repo

**Q2.** What triggers a workflow?
- A) A job  B) A step  C) An event  D) A runner

**Q3.** By default, jobs run:
- A) Sequentially  B) In parallel  C) Once per repo  D) Only on main

**Q4.** Which keyword invokes a reusable action?
- A) `run`  B) `uses`  C) `with`  D) `needs`

**Q5.** How do you create job dependencies?
- A) `depends`  B) `after`  C) `needs`  D) `requires`

**Q6.** What runs a job across OS/version combinations?
- A) Concurrency  B) Matrix  C) Cache  D) Environment

**Q7.** Which is recommended for cloud auth without stored keys?
- A) Personal access token  B) OIDC  C) SSH key  D) Basic auth

**Q8.** How should you reference actions most securely?
- A) `@main`  B) `@latest`  C) Pinned commit SHA  D) No version

**Q9.** What persists files to share between jobs?
- A) Cache  B) Artifacts  C) Secrets  D) Outputs

**Q10.** What adds required reviewers before deploy?
- A) Matrix  B) Environment  C) Concurrency  D) Service

### Short Answer

**S1.** Write the `on:` block to trigger on push to `main` and manual runs.

**S2.** Which context exposes the branch name?

**S3.** How do you cancel in-progress runs of the same group?

**S4.** What permission must `GITHUB_TOKEN` have to use OIDC?

**S5.** Name the three types of custom actions.

### Answer Key

**Multiple Choice:** Q1-B, Q2-C, Q3-B, Q4-B, Q5-C, Q6-B, Q7-B, Q8-C, Q9-B, Q10-B

**Short Answer:**
- **S1.** `on:\n  push:\n    branches: [main]\n  workflow_dispatch:`
- **S2.** The `github` context (`github.ref_name`).
- **S3.** Use a `concurrency` group with `cancel-in-progress: true`.
- **S4.** `id-token: write`.
- **S5.** Composite, JavaScript, and Docker container actions.

---

## 9. Further Resources

### Official Documentation
- GitHub Actions docs — https://docs.github.com/actions
- Workflow syntax — https://docs.github.com/actions/using-workflows/workflow-syntax-for-github-actions
- Security hardening — https://docs.github.com/actions/security-guides/security-hardening-for-github-actions
- OIDC — https://docs.github.com/actions/deployment/security-hardening-your-deployments

### Learning
- GitHub Skills — https://skills.github.com
- Awesome Actions — https://github.com/sdras/awesome-actions
- GitHub Actions Marketplace — https://github.com/marketplace?type=actions

### Tools
- `act` — run workflows locally — https://github.com/nektos/act
- Dependabot, CodeQL (security)
- actionlint — workflow linter — https://github.com/rhysd/actionlint

### Books & Guides
- *Learning GitHub Actions* — Brent Laster
- GitHub Actions cheat sheets (DevHints, official quickstart)

---

*End of GitHub Actions — Zero to Hero.*
