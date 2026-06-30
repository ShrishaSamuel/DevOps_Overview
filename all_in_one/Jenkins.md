# Jenkins — Zero to Hero

> A complete, beginner-to-advanced guide to Jenkins for Continuous Integration and Continuous Delivery (CI/CD).

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

### 1.1 What is Jenkins?

**Jenkins** is an open-source **automation server** written in Java, used primarily to build **CI/CD pipelines**. It automates the parts of software development related to building, testing, and deploying — enabling teams to integrate changes frequently and deliver software reliably. Jenkins started as "Hudson" in 2004/2005 and was forked into Jenkins in 2011. It has a massive ecosystem of **1,800+ plugins**.

### 1.2 CI/CD fundamentals

- **Continuous Integration (CI):** Developers merge code changes frequently into a shared repository; each change automatically triggers a build and automated tests, catching integration issues early.
- **Continuous Delivery (CD):** Every change that passes the pipeline is automatically prepared for release; deployment to production is a manual, one-click decision.
- **Continuous Deployment:** Goes one step further — every passing change is automatically deployed to production with no manual gate.

```
Code Commit → Build → Unit Tests → Integration Tests → Package → Deploy (Staging) → [Approve] → Deploy (Prod)
   └──────────────────── Continuous Integration ──────────────────┘
                                              └──── Continuous Delivery/Deployment ────┘
```

### 1.3 Why Jenkins?

| Reason | Explanation |
|--------|-------------|
| Mature & proven | Battle-tested over 15+ years in production |
| Extensible | 1,800+ plugins integrate virtually any tool |
| Free & open source | No licensing costs; self-hosted control |
| Pipeline-as-code | `Jenkinsfile` versioned alongside source |
| Distributed builds | Controller/agent architecture scales horizontally |
| Platform-agnostic | Runs anywhere the JVM runs |

Modern alternatives include GitHub Actions, GitLab CI/CD, CircleCI, and Argo Workflows; Jenkins remains widely used in enterprises, especially where self-hosting and customization matter.

### 1.4 Architecture

Jenkins uses a **controller/agent** (formerly master/slave) model:

```
                    +-----------------------------+
                    |       Jenkins Controller    |
                    |  - Web UI & REST API        |
                    |  - Job/Pipeline definitions |
                    |  - Scheduling & orchestration|
                    |  - Plugin management        |
                    |  - Stores build results     |
                    +--------------+--------------+
                                   │ dispatches builds
            ┌──────────────────────┼──────────────────────┐
            ▼                      ▼                       ▼
     +-------------+        +-------------+         +-------------+
     |  Agent 1    |        |  Agent 2    |         |  Agent 3    |
     | (Linux)     |        | (Windows)   |         | (Docker)    |
     | executes    |        | executes    |         | executes    |
     | build steps |        | build steps |         | build steps |
     +-------------+        +-------------+         +-------------+
```

- **Controller:** Orchestrates everything — schedules jobs, dispatches work to agents, serves the UI, stores configuration and results. It should ideally *not* run heavy builds itself.
- **Agents (nodes):** Machines (or containers) that actually execute build steps. Connected via SSH, JNLP, or dynamically provisioned (Docker/Kubernetes).
- **Executors:** Slots on a node that run builds; a node with 4 executors can run 4 builds concurrently.

### 1.5 When to use Jenkins

- Self-hosted CI/CD with full control and customization.
- Complex pipelines integrating many heterogeneous tools.
- Environments with on-prem or air-gapped requirements.
- Teams needing distributed builds across many platforms.

When **not** ideal: small projects that fit cloud-native CI (GitHub Actions/GitLab CI) with less maintenance overhead, or teams unwilling to operate/secure a server.

---

## 2. Installation & Setup

### 2.1 Prerequisites

- Java (JDK 17 or 21 for recent Jenkins LTS).
- 2 GB+ RAM (4 GB+ recommended), 10 GB+ disk.

```bash
sudo apt update
sudo apt install -y fontconfig openjdk-17-jre
java -version
```

### 2.2 Install Jenkins (Ubuntu/Debian)

```bash
# Add the Jenkins repository key and source
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

sudo apt update
sudo apt install -y jenkins

# Start and enable the service
sudo systemctl enable --now jenkins
sudo systemctl status jenkins
```

### 2.3 Run Jenkins in Docker (quick start)

```bash
docker run -d --name jenkins \
  -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts-jdk17
```

- Port `8080` — web UI.
- Port `50000` — inbound agent connections.
- `jenkins_home` volume — persists all configuration and build history.

### 2.4 Initial setup wizard

```bash
# Get the initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
# Docker:
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

1. Open `http://localhost:8080`.
2. Paste the initial admin password.
3. Choose **"Install suggested plugins"**.
4. Create your first admin user.
5. Confirm the Jenkins URL.

### 2.5 Essential plugins

- **Pipeline** (Workflow Aggregator) — declarative/scripted pipelines.
- **Git / GitHub / GitLab** — source integration.
- **Credentials Binding** — secrets in pipelines.
- **Docker Pipeline** — build/run containers.
- **Blue Ocean** — modern pipeline visualization.
- **JUnit / Test Results Analyzer** — test reporting.
- **Role-based Authorization Strategy** — fine-grained access control.

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 Jobs and project types

A **Job** (or Project) is a runnable task. Types:

| Type | Use case |
|------|----------|
| **Freestyle** | Simple GUI-configured builds |
| **Pipeline** | Code-defined CI/CD (`Jenkinsfile`) — recommended |
| **Multibranch Pipeline** | Auto-creates pipelines per Git branch |
| **Folder** | Organize jobs |
| **Organization Folder** | Auto-discover repos in a GitHub/GitLab org |

#### 3.1.2 Creating a Freestyle job

1. **New Item** → enter a name → select **Freestyle project**.
2. **Source Code Management** → Git → repository URL + credentials.
3. **Build Triggers** → e.g., *Poll SCM* or *GitHub hook trigger*.
4. **Build Steps** → *Execute shell*:
   ```bash
   echo "Building..."
   ./build.sh
   ```
5. **Post-build Actions** → archive artifacts, publish test results, notify.
6. **Save** → **Build Now**.

#### 3.1.3 The build lifecycle

```
Triggered → Checkout SCM → Build steps → Post-build actions → Result (SUCCESS/UNSTABLE/FAILURE)
```

- **SUCCESS** — everything passed.
- **UNSTABLE** — built but tests failed/quality gate flagged.
- **FAILURE** — a step failed.
- **ABORTED** — manually stopped or timed out.

#### 3.1.4 Build triggers

- **Poll SCM** — Jenkins periodically checks the repo (`H/5 * * * *`).
- **Webhook** — the repo notifies Jenkins on push (preferred — instant, efficient).
- **Build periodically** — cron schedule (`H 2 * * *`).
- **Upstream/downstream** — trigger after another job.

> The `H` ("hash") in Jenkins cron spreads load — `H/15 * * * *` means "every 15 minutes but offset by a hash of the job name."

#### 3.1.5 Workspace and artifacts

- **Workspace** — the directory where Jenkins checks out code and runs the build.
- **Artifacts** — files produced by a build (binaries, reports) that you archive for later use.

### 3.2 INTERMEDIATE

#### 3.2.1 Pipeline-as-code: the Jenkinsfile

A **`Jenkinsfile`** in your repo defines the pipeline as code (versioned, reviewable). Two syntaxes:
- **Declarative** (structured, recommended) — starts with `pipeline { }`.
- **Scripted** (Groovy, flexible) — starts with `node { }`.

#### 3.2.2 Declarative pipeline anatomy

```groovy
pipeline {
    agent any                      // where to run

    environment {                  // environment variables
        APP_NAME = 'myapp'
        REGISTRY = 'docker.io/you'
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    triggers {
        pollSCM('H/5 * * * *')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Build') {
            steps {
                sh 'make build'
            }
        }
        stage('Test') {
            steps {
                sh 'make test'
            }
            post {
                always {
                    junit 'reports/**/*.xml'
                }
            }
        }
        stage('Package') {
            steps {
                sh 'docker build -t $REGISTRY/$APP_NAME:$BUILD_NUMBER .'
            }
        }
        stage('Deploy') {
            when { branch 'main' }
            steps {
                sh './deploy.sh'
            }
        }
    }

    post {
        success { echo 'Pipeline succeeded' }
        failure { echo 'Pipeline failed' }
        always  { cleanWs() }
    }
}
```

#### 3.2.3 Key declarative directives

| Directive | Purpose |
|-----------|---------|
| `agent` | Where the pipeline/stage runs (`any`, `none`, `label`, `docker`) |
| `stages` / `stage` | Logical phases of the pipeline |
| `steps` | The commands within a stage |
| `environment` | Define environment variables |
| `options` | Pipeline-level settings (timeouts, retention) |
| `parameters` | User-supplied build parameters |
| `triggers` | Automatic trigger config |
| `when` | Conditional stage execution |
| `post` | Actions after stages (`always`, `success`, `failure`, `unstable`, `changed`) |
| `parallel` | Run stages concurrently |

#### 3.2.4 Parameters and input

```groovy
pipeline {
    agent any
    parameters {
        string(name: 'VERSION', defaultValue: '1.0.0', description: 'Release version')
        choice(name: 'ENV', choices: ['dev', 'staging', 'prod'], description: 'Target env')
        booleanParam(name: 'RUN_TESTS', defaultValue: true)
    }
    stages {
        stage('Deploy') {
            steps {
                input message: "Deploy ${params.VERSION} to ${params.ENV}?"
                sh "./deploy.sh ${params.ENV} ${params.VERSION}"
            }
        }
    }
}
```

#### 3.2.5 Credentials management

Never hard-code secrets. Add them in **Manage Jenkins → Credentials**, then bind them:

```groovy
stage('Push') {
    steps {
        withCredentials([usernamePassword(
            credentialsId: 'docker-hub',
            usernameVariable: 'USER',
            passwordVariable: 'PASS')]) {
            sh 'echo "$PASS" | docker login -u "$USER" --password-stdin'
            sh 'docker push $REGISTRY/$APP_NAME:$BUILD_NUMBER'
        }
    }
}
```

Credential types: secret text, username/password, SSH key, secret file, certificates.

#### 3.2.6 Parallel stages

```groovy
stage('Tests') {
    parallel {
        stage('Unit')        { steps { sh 'make unit' } }
        stage('Integration') { steps { sh 'make integration' } }
        stage('Lint')        { steps { sh 'make lint' } }
    }
}
```

#### 3.2.7 Running steps inside Docker

```groovy
pipeline {
    agent {
        docker {
            image 'node:20-alpine'
            args '-v $HOME/.npm:/root/.npm'
        }
    }
    stages {
        stage('Build') {
            steps {
                sh 'npm ci && npm run build'
            }
        }
    }
}
```

### 3.3 ADVANCED

#### 3.3.1 Multibranch pipelines

A **Multibranch Pipeline** scans a repository and automatically creates a pipeline for each branch containing a `Jenkinsfile`, and removes them when branches are deleted. Ideal for feature-branch and PR-based workflows.

Setup: **New Item → Multibranch Pipeline → Branch Sources (Git/GitHub) → Build Configuration: by Jenkinsfile**.

#### 3.3.2 Shared Libraries

**Shared Libraries** let you reuse pipeline code across many projects. Structure:

```
(repo root)
├── vars/
│   └── deployApp.groovy      # global function: deployApp()
├── src/
│   └── org/company/Utils.groovy
└── resources/
    └── templates/config.json
```

```groovy
// vars/deployApp.groovy
def call(Map config) {
    sh "kubectl set image deployment/${config.name} app=${config.image}"
}
```

```groovy
// Jenkinsfile
@Library('my-shared-lib') _
pipeline {
    agent any
    stages {
        stage('Deploy') {
            steps {
                deployApp(name: 'web', image: 'myapp:1.2.3')
            }
        }
    }
}
```

Configure under **Manage Jenkins → System → Global Pipeline Libraries**.

#### 3.3.3 Distributed builds and dynamic agents

- **Static agents:** Connect fixed machines via SSH or JNLP.
- **Docker agents:** Spin up a fresh container per build (clean, reproducible).
- **Kubernetes plugin:** Provision ephemeral agent Pods on demand — elastic scaling.

```groovy
pipeline {
    agent {
        kubernetes {
            yaml '''
              apiVersion: v1
              kind: Pod
              spec:
                containers:
                  - name: maven
                    image: maven:3.9-eclipse-temurin-17
                    command: ["cat"]
                    tty: true
            '''
        }
    }
    stages {
        stage('Build') {
            steps {
                container('maven') { sh 'mvn -B package' }
            }
        }
    }
}
```

#### 3.3.4 Blue/Green and Canary deployment patterns

```groovy
stage('Deploy Canary') {
    steps {
        sh './deploy.sh canary 10%'        // route 10% traffic
        sh './smoke-test.sh'
    }
}
stage('Promote') {
    steps {
        input message: 'Promote canary to 100%?'
        sh './deploy.sh production 100%'
    }
}
```

#### 3.3.5 Configuration as Code (JCasC)

The **Jenkins Configuration as Code (JCasC)** plugin defines the entire Jenkins configuration in YAML, enabling reproducible, version-controlled Jenkins instances.

```yaml
jenkins:
  systemMessage: "Managed by JCasC"
  numExecutors: 0
  securityRealm:
    local:
      allowsSignup: false
  authorizationStrategy:
    roleBased: {}
tool:
  git:
    installations:
      - name: Default
        home: git
unclassified:
  location:
    url: https://jenkins.example.com/
```

#### 3.3.6 Security hardening

- Enable security; integrate with LDAP/SSO/OIDC.
- Use **Role-based Authorization Strategy** for least privilege.
- Store all secrets in the credentials store (or external vault via plugin).
- Don't run builds on the controller (`numExecutors: 0`); use agents.
- Keep Jenkins and plugins updated (CVEs are frequent).
- Restrict who can configure jobs/run script consoles.
- Enable CSRF protection and audit logging.

#### 3.3.7 Scaling and reliability

- Back up `JENKINS_HOME` regularly (config, jobs, history).
- Use `buildDiscarder` to prune old builds and control disk usage.
- Externalize agents; autoscale with Kubernetes.
- Monitor with the Prometheus plugin / metrics; set up alerting.
- Consider High Availability via operator/cluster setups (or managed CloudBees).

#### 3.3.8 Error handling and retries

```groovy
stage('Flaky') {
    steps {
        retry(3) {
            sh './sometimes-fails.sh'
        }
        timeout(time: 5, unit: 'MINUTES') {
            sh './long-task.sh'
        }
    }
    post {
        failure {
            mail to: 'team@example.com', subject: "Build ${env.BUILD_NUMBER} failed", body: "${env.BUILD_URL}"
        }
    }
}
```

---

## 4. Hands-on Tasks

### Task 1: Run Jenkins and complete setup

```bash
docker run -d --name jenkins -p 8080:8080 -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts-jdk17
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

**Expected:** UI at `http://localhost:8080`; complete the wizard and install suggested plugins.

### Task 2: Create a Freestyle job

1. New Item → Freestyle → add an *Execute shell* step: `echo "Hello Jenkins"; date`.
2. Build Now → open **Console Output**.

**Expected:** Console shows the greeting, timestamp, and `Finished: SUCCESS`.

### Task 3: First declarative pipeline

Create a Pipeline job with this script:

```groovy
pipeline {
    agent any
    stages {
        stage('Build') { steps { echo 'Building...' } }
        stage('Test')  { steps { echo 'Testing...' } }
        stage('Deploy'){ steps { echo 'Deploying...' } }
    }
}
```

**Expected:** Stage View shows three green stages.

### Task 4: Pipeline from a Jenkinsfile in Git

1. Add a `Jenkinsfile` to a repo.
2. Create a Pipeline job → *Pipeline script from SCM* → point to the repo.
3. Build.

**Expected:** Jenkins checks out the repo and runs the versioned pipeline.

### Task 5: Add build parameters

Use the parameterized pipeline from section 3.2.4, click **Build with Parameters**, choose values.

**Expected:** The build uses the supplied `VERSION` and `ENV`.

### Task 6: Use credentials securely

1. Manage Jenkins → Credentials → add a username/password (`docker-hub`).
2. Use the `withCredentials` block from section 3.2.5.

**Expected:** Secrets are masked in the console (`****`).

### Task 7: Parallel testing

Add the parallel stage from section 3.2.6 and run.

**Expected:** Unit, Integration, and Lint run concurrently in the Stage View.

### Task 8: Publish JUnit test results

```groovy
stage('Test') {
    steps { sh 'pytest --junitxml=results.xml || true' }
    post { always { junit 'results.xml' } }
}
```

**Expected:** A "Test Result" trend graph appears on the job page.

### Task 9: Trigger builds on Git push (webhook)

1. Convert to a Multibranch Pipeline or enable "GitHub hook trigger".
2. Add a webhook in the repo pointing to `http://<jenkins>/github-webhook/`.
3. Push a commit.

**Expected:** A build starts automatically on push.

### Task 10: Build and push a Docker image

```groovy
stage('Image') {
    steps {
        sh 'docker build -t myapp:$BUILD_NUMBER .'
        withCredentials([usernamePassword(credentialsId: 'docker-hub',
            usernameVariable: 'U', passwordVariable: 'P')]) {
            sh 'echo "$P" | docker login -u "$U" --password-stdin'
            sh 'docker push myapp:$BUILD_NUMBER'
        }
    }
}
```

**Expected:** Image built and pushed; build number used as the tag.

---

## 5. Projects

### Project 1 (Beginner): CI Pipeline for a Web App

**Goal:** Automatically build and test an app on every commit.

Steps:
1. Add a `Jenkinsfile` with Checkout → Install deps → Lint → Unit tests.
2. Publish JUnit results and code coverage.
3. Configure a webhook so pushes trigger builds.
4. Add email/Slack notification on failure.
5. Retain only the last 10 builds.

**Skills:** declarative pipeline, SCM triggers, test reporting, notifications.

### Project 2 (Intermediate): Full CI/CD to a Staging Environment

**Goal:** Build, test, containerize, and deploy to staging automatically; production behind manual approval.

Steps:
1. Multibranch pipeline; `main` deploys to staging, tags deploy to prod.
2. Stages: Checkout → Build → Test (parallel unit/integration/lint) → Build Docker image → Push to registry → Deploy to staging.
3. Use credentials for the registry and deploy target.
4. Add an `input` approval gate before production deploy.
5. Implement rollback on failure.

**Skills:** multibranch, parallelism, Docker, credentials, environments, approvals.

### Project 3 (Advanced): Scalable, Reusable Jenkins Platform

**Goal:** A production-grade Jenkins with reusable libraries, dynamic agents, and IaC.

Steps:
1. Provision Jenkins via **JCasC** (config in Git) with role-based security and SSO.
2. Use the **Kubernetes plugin** to provision ephemeral agent Pods per build.
3. Create a **Shared Library** encapsulating build/test/deploy steps used by many repos.
4. Implement a deployment strategy (blue/green or canary) with smoke tests and auto-rollback.
5. Integrate security scanning (SAST/Trivy) and quality gates (SonarQube).
6. Add monitoring (Prometheus plugin) and automated backups of `JENKINS_HOME`.
7. Document onboarding so a new repo gets CI/CD by adding a small `Jenkinsfile`.

**Skills:** JCasC, dynamic agents, shared libraries, advanced deploys, security, observability.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Use pipeline-as-code** (`Jenkinsfile` in Git) — versioned, reviewable, reproducible.
- **Prefer Declarative pipelines** for readability; use Scripted only when needed.
- **Never run builds on the controller** — use agents (set controller executors to 0).
- **Store secrets in the credentials store** (or external Vault); never hard-code.
- **Keep stages small and named** clearly; fail fast.
- **Use Shared Libraries** to avoid copy-pasting pipeline logic.
- **Clean the workspace** (`cleanWs()`) and **discard old builds** to save disk.
- **Pin plugin and tool versions**; update regularly for security.
- **Use ephemeral/containerized agents** for clean, reproducible builds.
- **Back up `JENKINS_HOME`** and manage config with JCasC.
- **Apply least-privilege RBAC** and enable security from day one.

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Configuring jobs only in the UI | Not versioned, hard to reproduce | Pipeline-as-code + JCasC |
| Running builds on the controller | Security & stability risk | Use agents; 0 controller executors |
| Hard-coding secrets in pipelines | Credential leakage | Use credentials store / Vault |
| Never cleaning workspace/builds | Disk fills up, slow Jenkins | `cleanWs()`, `buildDiscarder` |
| Giant monolithic pipeline | Hard to maintain/debug | Modularize with stages + shared libs |
| Ignoring plugin updates | Known CVEs exploited | Patch regularly, audit plugins |
| Polling SCM aggressively | Wasted resources, latency | Use webhooks |
| Snowflake controller (manual config) | Unrecoverable if lost | JCasC + backups |
| No timeouts | Hung builds block executors | `timeout()` in options |
| Flaky tests left unaddressed | Eroded trust in CI | Quarantine/fix; use `retry` judiciously |

---

## 7. Interview Questions

### Beginner

1. **What is Jenkins used for?**
   *Automating CI/CD — building, testing, and deploying software through pipelines.*

2. **What is the difference between CI and CD?**
   *CI integrates and tests code changes frequently; CD automatically prepares (and optionally deploys) every passing change for release.*

3. **What is a Jenkins job?**
   *A configured, runnable task (Freestyle, Pipeline, Multibranch, etc.) that performs build/test/deploy steps.*

4. **What is a Jenkinsfile?**
   *A text file in the repo that defines the pipeline as code (declarative or scripted).*

5. **What is the controller/agent architecture?**
   *The controller orchestrates and serves the UI; agents execute the actual build work on connected nodes.*

### Intermediate

6. **Declarative vs. scripted pipelines — differences?**
   *Declarative uses a structured `pipeline { }` syntax that's easier to read and validate; scripted is full Groovy (`node { }`), more flexible but more complex.*

7. **How do you handle secrets in Jenkins pipelines?**
   *Store them in the Credentials store and bind with `withCredentials`/`environment`; values are masked in logs. Better: integrate an external vault.*

8. **What is a Multibranch Pipeline?**
   *A job that scans a repo and auto-creates/destroys pipelines per branch (and PRs) containing a `Jenkinsfile`.*

9. **How do you run stages in parallel and why?**
   *With the `parallel` directive — to reduce total pipeline time (e.g., running unit, integration, and lint concurrently).*

10. **What does the `post` section do?**
    *Defines actions after stages run, conditioned on result (`always`, `success`, `failure`, `unstable`, `changed`) — e.g., notifications, cleanup, archiving.*

### Advanced

11. **What are Shared Libraries and why use them?**
    *Reusable pipeline code (functions/classes) stored in a separate repo, loaded via `@Library`, eliminating duplication across many pipelines and standardizing CI/CD.*

12. **How do you scale Jenkins for many concurrent builds?**
    *Use distributed agents — static nodes, Docker, or dynamic Kubernetes Pods — and keep the controller for orchestration only; tune executors and queue.*

13. **What is JCasC and what problem does it solve?**
    *Jenkins Configuration as Code defines the whole instance config in YAML, making Jenkins reproducible, version-controlled, and recoverable (no snowflake servers).*

14. **How would you implement a safe production deployment in Jenkins?**
    *Promote through environments, require an `input` approval gate, deploy via canary/blue-green with smoke tests, and automate rollback on failure; gate on quality/security checks.*

15. **How do you secure a Jenkins instance?**
    *Enable security/SSO, role-based authorization (least privilege), credentials store/Vault, no builds on controller, patched plugins, CSRF protection, restricted script console, and audit logging.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** What language is Jenkins written in?
- A) Python  B) Java  C) Go  D) Ruby

**Q2.** Which file defines a pipeline as code?
- A) `pipeline.yml`  B) `Jenkinsfile`  C) `build.gradle`  D) `.jenkins`

**Q3.** Which executes the actual build steps?
- A) Controller  B) Plugin  C) Agent/Node  D) Webhook

**Q4.** Which pipeline syntax is recommended for readability?
- A) Scripted  B) Declarative  C) XML  D) JSON

**Q5.** Which directive runs stages concurrently?
- A) `stages`  B) `parallel`  C) `steps`  D) `matrix`

**Q6.** Where should secrets be stored?
- A) In the Jenkinsfile  B) Environment files on the controller  C) Credentials store  D) Git repo

**Q7.** Which trigger is preferred for instant builds on push?
- A) Poll SCM  B) Build periodically  C) Webhook  D) Manual

**Q8.** What auto-creates pipelines per branch?
- A) Freestyle  B) Folder  C) Multibranch Pipeline  D) Matrix project

**Q9.** Which `post` condition always runs?
- A) `success`  B) `failure`  C) `always`  D) `changed`

**Q10.** What does JCasC provide?
- A) Job scheduling  B) Configuration as Code  C) A test framework  D) A container runtime

### Short Answer

**S1.** Write the `agent` line that runs a pipeline inside a `node:20-alpine` Docker image.

**S2.** Which directive limits how long a pipeline can run before failing?

**S3.** How do you bind a username/password credential with ID `reg` into env vars `U` and `P`?

**S4.** What is the purpose of `buildDiscarder(logRotator(...))`?

**S5.** Name the four common build results.

### Answer Key

**Multiple Choice:** Q1-B, Q2-B, Q3-C, Q4-B, Q5-B, Q6-C, Q7-C, Q8-C, Q9-C, Q10-B

**Short Answer:**
- **S1.** `agent { docker { image 'node:20-alpine' } }`
- **S2.** `timeout` (within `options` or a step).
- **S3.** `withCredentials([usernamePassword(credentialsId: 'reg', usernameVariable: 'U', passwordVariable: 'P')]) { ... }`
- **S4.** It rotates/prunes old builds to control disk usage and history retention.
- **S5.** SUCCESS, UNSTABLE, FAILURE, ABORTED.

---

## 9. Further Resources

### Official Documentation
- Jenkins Docs — https://www.jenkins.io/doc/
- Pipeline syntax reference — https://www.jenkins.io/doc/book/pipeline/syntax/
- Jenkinsfile tutorial — https://www.jenkins.io/doc/book/pipeline/jenkinsfile/
- Plugins index — https://plugins.jenkins.io/

### Learning
- Jenkins "Getting Started" — https://www.jenkins.io/doc/pipeline/tour/getting-started/
- Blue Ocean — https://www.jenkins.io/projects/blueocean/
- JCasC plugin docs — https://www.jenkins.io/projects/jcasc/

### Tools & Ecosystem
- Blue Ocean (visualization), Warnings NG (static analysis)
- Kubernetes plugin (dynamic agents)
- HashiCorp Vault plugin (secrets)
- SonarQube (quality gates), Trivy (security scanning)

### Books
- *Jenkins: The Definitive Guide* — John Ferguson Smart
- *Continuous Delivery* — Jez Humble & David Farley
- *Pipeline as Code* — Mohamed Labouardy

### Comparisons / Alternatives
- GitHub Actions, GitLab CI/CD, CircleCI, Argo Workflows, Tekton

---

*End of Jenkins — Zero to Hero.*
