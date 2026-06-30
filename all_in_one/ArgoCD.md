# Argo CD — Zero to Hero

> A complete, beginner-to-advanced guide to Argo CD for declarative GitOps continuous delivery on Kubernetes.

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

### 1.1 What is Argo CD?

**Argo CD** is a **declarative, GitOps continuous delivery tool for Kubernetes**. It is a CNCF Graduated project that continuously monitors a Git repository (the source of truth) and ensures that the live state of a Kubernetes cluster matches the desired state defined in Git. When the two diverge, Argo CD reports the application as **OutOfSync** and can automatically (or manually) reconcile them.

### 1.2 What is GitOps?

**GitOps** is an operational model where:
1. **Git is the single source of truth** for declarative infrastructure and applications.
2. **Desired state is versioned** in Git (auditable, reviewable, revertible).
3. **An agent continuously reconciles** the live cluster to match Git.
4. Changes are made by **committing to Git**, not by running imperative `kubectl` commands.

```
   Developer ──► git push ──► Git repo (desired state, manifests/Helm/Kustomize)
                                   │
                                   ▼
                              Argo CD (agent in cluster)
                                   │ continuously compares
                       ┌───────────┴───────────┐
                       ▼                        ▼
                 Desired state            Live cluster state
                 (from Git)               (from Kubernetes API)
                       └──────── reconcile ─────┘
                         (sync to match Git)
```

### 1.3 Push vs. Pull-based CD

| Model | How it works | Examples |
|-------|--------------|----------|
| **Push** | CI pipeline pushes changes into the cluster (has cluster credentials) | Jenkins, GitHub Actions `kubectl apply` |
| **Pull (GitOps)** | An in-cluster agent pulls desired state from Git and applies it | **Argo CD**, Flux |

Pull-based is more **secure** (cluster credentials never leave the cluster; CI doesn't need cluster access), **self-healing** (agent fixes drift), and **auditable** (everything via Git).

### 1.4 Why Argo CD?

| Feature | Explanation |
|---------|-------------|
| **Declarative GitOps** | Git is the source of truth; automatic reconciliation |
| **Powerful UI** | Visualize app topology, sync status, diffs, health |
| **Multi-cluster** | Manage many clusters from one Argo CD |
| **Multiple config tools** | Helm, Kustomize, plain YAML, Jsonnet |
| **Self-healing & drift detection** | Detects and corrects manual changes |
| **SSO & RBAC** | Enterprise auth, fine-grained permissions |
| **App-of-apps & ApplicationSets** | Manage many apps/clusters at scale |

### 1.5 Architecture

Argo CD runs as several components inside the cluster:

```
   +-----------------------------------------------------------+
   |                      Argo CD                              |
   |                                                          |
   |  +------------------+   +------------------------------+ |
   |  | API Server       |   | Repo Server                  | |
   |  | (UI, CLI, gRPC,  |   | (clones Git, renders         | |
   |  |  RBAC, SSO)      |   |  Helm/Kustomize → manifests) | |
   |  +------------------+   +------------------------------+ |
   |                                                          |
   |  +------------------------------+   +------------------+ |
   |  | Application Controller       |   | Redis (cache)    | |
   |  | (compares desired vs live,   |   |                  | |
   |  |  syncs, reports health)      |   +------------------+ |
   |  +------------------------------+                        |
   +-----------------------------------------------------------+
              │ watches Git + Kubernetes API
              ▼
        Target cluster(s) / namespaces
```

- **API Server:** Serves the UI/CLI, handles auth (SSO), RBAC, and the gRPC/REST API.
- **Repository Server:** Clones Git repos and renders manifests (running Helm/Kustomize/etc.).
- **Application Controller:** The core reconciler — compares desired vs. live state, performs syncs, and reports health.
- **Redis:** Caching layer.
- **Dex (optional):** SSO/identity broker.

### 1.6 When to use Argo CD

- Deploying and managing applications on Kubernetes.
- Teams adopting GitOps for auditability and self-healing.
- Multi-cluster / multi-environment deployments at scale.
- Progressive delivery (with Argo Rollouts).

When **not** ideal: non-Kubernetes deployments, very small single-app setups where plain CI `kubectl apply` suffices, or teams not ready to make Git the source of truth.

---

## 2. Installation & Setup

### 2.1 Prerequisites

- A running Kubernetes cluster (`kubectl` access). For local practice: kind, minikube, or k3d.

### 2.2 Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Verify:

```bash
kubectl get pods -n argocd
```

**Expected:** Pods like `argocd-server`, `argocd-repo-server`, `argocd-application-controller`, `argocd-redis`, `argocd-dex-server` become `Running`.

### 2.3 Access the UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open https://localhost:8080
```

### 2.4 Get the initial admin password and log in (CLI)

```bash
# Install the CLI (example for Linux)
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

# Initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo

# Login
argocd login localhost:8080 --username admin --password <password> --insecure
argocd account update-password    # change it!
```

### 2.5 Create your first Application (declarative)

```yaml
# guestbook-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f guestbook-app.yaml
argocd app get guestbook
argocd app sync guestbook
```

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 The Application resource

The **`Application`** is the core Argo CD CRD. It declares:
- **source** — where the manifests live (Git repo URL, path, revision).
- **destination** — the target cluster and namespace.
- **project** — which AppProject it belongs to.
- **syncPolicy** — manual or automated reconciliation.

#### 3.1.2 Sync status

| Status | Meaning |
|--------|---------|
| **Synced** | Live state matches Git (desired state). |
| **OutOfSync** | Live state differs from Git. |
| **Unknown** | Argo CD can't determine the state. |

#### 3.1.3 Health status

Argo CD reports application **health** based on resource conditions:
- **Healthy** — all resources are running as expected.
- **Progressing** — still rolling out.
- **Degraded** — something is failing.
- **Missing** — resource doesn't exist yet.
- **Suspended** — paused (e.g., a suspended CronJob).

#### 3.1.4 Syncing

A **sync** applies the manifests from Git to the cluster.

```bash
argocd app sync guestbook          # manual sync
argocd app get guestbook           # status, sync, health
argocd app diff guestbook          # what differs from Git
argocd app history guestbook       # deployment history
```

#### 3.1.5 Manual vs. automated sync

```yaml
syncPolicy:
  automated:
    prune: true        # delete resources removed from Git
    selfHeal: true     # revert manual cluster changes (drift)
```

- **Manual:** You click/run sync to apply changes.
- **Automated:** Argo CD syncs automatically when Git changes.
- **selfHeal:** Reverts out-of-band `kubectl` edits.
- **prune:** Removes resources deleted from Git.

### 3.2 INTERMEDIATE

#### 3.2.1 Config management tools

Argo CD renders manifests from your source using:

```yaml
# Helm
source:
  repoURL: https://github.com/org/repo
  path: charts/myapp
  helm:
    valueFiles: [ values-prod.yaml ]
    parameters:
      - name: image.tag
        value: v1.2.3

# Kustomize
source:
  path: overlays/production    # Argo auto-detects kustomization.yaml
```

Supported: plain YAML/directory, **Helm**, **Kustomize**, **Jsonnet**, and custom **config management plugins**.

#### 3.2.2 Projects (AppProjects)

**AppProjects** group applications and enforce guardrails:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-a
  namespace: argocd
spec:
  description: Team A apps
  sourceRepos:
    - https://github.com/org/team-a-*      # allowed repos
  destinations:
    - namespace: 'team-a-*'                # allowed namespaces
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:                # allowed cluster-scoped kinds
    - group: ''
      kind: Namespace
```

Projects restrict which repos, clusters, namespaces, and resource kinds apps may use — essential for multi-tenancy.

#### 3.2.3 Sync waves and hooks

Control ordering during a sync:

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"          # lower waves apply first
```

```yaml
# Resource hooks
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync            # PreSync, Sync, PostSync, SyncFail
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

Use **sync waves** to order resources (e.g., DB migration before app), and **hooks** for pre/post-sync jobs.

#### 3.2.4 Diff and drift detection

```bash
argocd app diff myapp
```

Argo CD continuously detects **drift** (live ≠ Git). With `selfHeal: true`, it automatically reverts manual changes — enforcing Git as the source of truth.

#### 3.2.5 Rollback

```bash
argocd app history myapp
argocd app rollback myapp <HISTORY_ID>
```

Because every state is in Git, rollback is reliable. You can also roll back simply by reverting the Git commit.

#### 3.2.6 RBAC and SSO

```csv
# argocd-rbac-cm policy.csv
p, role:dev, applications, get, team-a/*, allow
p, role:dev, applications, sync, team-a/*, allow
g, dev-team, role:dev
```

Argo CD integrates with OIDC/SAML SSO (via Dex or directly) and offers fine-grained RBAC by project/app/action.

### 3.3 ADVANCED

#### 3.3.1 App-of-Apps pattern

A parent Application whose source is a directory of **other Application manifests** — bootstrapping many apps from one:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bootstrap
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/org/gitops
    path: apps            # contains many Application YAMLs
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```

#### 3.3.2 ApplicationSets

The **ApplicationSet** controller generates many Applications from a template + a generator — ideal for multi-cluster/multi-env at scale:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: guestbook
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: staging
            url: https://staging-api:6443
          - cluster: prod
            url: https://prod-api:6443
  template:
    metadata:
      name: 'guestbook-{{cluster}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/repo
        targetRevision: HEAD
        path: 'guestbook/{{cluster}}'
      destination:
        server: '{{url}}'
        namespace: guestbook
      syncPolicy:
        automated: { prune: true, selfHeal: true }
```

Generators: **list**, **cluster**, **git** (directories/files), **matrix**, **merge**, **pull request**, **SCM provider**.

#### 3.3.3 Multi-cluster management

```bash
argocd cluster add <kube-context>     # register an external cluster
argocd cluster list
```

One Argo CD instance can deploy to many registered clusters; ApplicationSets template across them.

#### 3.3.4 Secrets management

Argo CD does **not** manage secrets natively. Common patterns:
- **Sealed Secrets** (Bitnami) — encrypt secrets safe to store in Git.
- **External Secrets Operator** — sync from Vault/AWS Secrets Manager/etc.
- **SOPS** (with KSOPS plugin) — encrypted files in Git.
- **HashiCorp Vault** integration.

> Never commit plaintext secrets to Git. Use one of the above.

#### 3.3.5 Progressive delivery with Argo Rollouts

**Argo Rollouts** (a sister project) replaces the Deployment with a `Rollout` resource supporting **canary** and **blue-green** strategies, automated analysis, and rollback:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
        - setWeight: 20
        - pause: { duration: 5m }
        - setWeight: 50
        - pause: {}
```

Argo CD deploys the Rollout; Rollouts manages safe, gradual traffic shifting.

#### 3.3.6 Notifications and observability

- **Argo CD Notifications** — alert on sync/health changes (Slack, email, webhooks).
- **Metrics** — Prometheus metrics for sync status, reconciliation, controller performance.
- Integrate with Grafana dashboards for fleet visibility.

#### 3.3.7 High availability and scaling

- Run components in **HA mode** (multiple replicas, HA Redis).
- **Shard** the application controller across replicas for many clusters/apps.
- Tune reconciliation timeouts and resource limits.

#### 3.3.8 Declarative everything (GitOps for Argo itself)

Manage Argo CD's own configuration (Applications, Projects, repos, clusters) declaratively in Git — even Argo CD bootstraps itself via the app-of-apps pattern, so the entire platform is reproducible.

---

## 4. Hands-on Tasks

### Task 1: Install Argo CD and access the UI

Apply the install manifest, port-forward, and log in with the initial admin password.

**Expected:** Argo CD UI loads; pods are Running.

### Task 2: Deploy the guestbook app

Apply the `Application` from section 2.5 and sync.

**Expected:** App becomes **Synced** and **Healthy**; guestbook pods run.

### Task 3: Observe drift and self-heal

With `selfHeal: true`, manually scale the deployment: `kubectl scale deploy/guestbook-ui --replicas=5`.

**Expected:** Argo CD detects drift and reverts replicas to the Git-defined value.

### Task 4: Make a Git change and watch auto-sync

Edit the manifest in Git (e.g., change replicas) and push.

**Expected:** App goes OutOfSync, then auto-syncs to the new desired state.

### Task 5: Use the diff command

```bash
argocd app diff guestbook
```

**Expected:** Shows differences between Git and the live cluster (empty if Synced).

### Task 6: Roll back

```bash
argocd app history guestbook
argocd app rollback guestbook <id>
```

**Expected:** The app reverts to the chosen previous revision.

### Task 7: Deploy a Helm chart

Create an Application with a Helm `source` and parameter overrides.

**Expected:** Argo renders the chart and deploys it; values are applied.

### Task 8: Create an AppProject with restrictions

Define an AppProject limiting `sourceRepos` and `destinations`; assign an app to it.

**Expected:** Apps outside the allowed repos/namespaces are rejected.

### Task 9: Use sync waves

Annotate two resources with different `sync-wave` values.

**Expected:** Resources apply in wave order (lower first).

### Task 10: Create an ApplicationSet

Use a list generator to deploy the same app to two "clusters"/paths.

**Expected:** Two Applications are generated automatically from one ApplicationSet.

---

## 5. Projects

### Project 1 (Beginner): GitOps Deploy of a Web App

**Goal:** Manage a simple app entirely through Git.

Steps:
1. Put Kubernetes manifests in a Git repo.
2. Create an Argo CD `Application` pointing at it with automated sync.
3. Change the image tag via a Git commit; watch auto-sync.
4. Manually edit the cluster and observe **selfHeal** revert it.
5. Practice rollback via `argocd app rollback` and via Git revert.

**Skills:** Application CRD, auto-sync, self-heal, rollback, GitOps basics.

### Project 2 (Intermediate): Multi-Environment with Kustomize/Helm

**Goal:** Deploy the same app to dev/staging/prod with environment-specific config.

Steps:
1. Structure the repo with Kustomize overlays (or Helm value files) per environment.
2. Create one Application per environment (or use app-of-apps).
3. Use **AppProjects** to restrict repos/namespaces per team.
4. Add **sync waves/hooks** (e.g., run DB migration before app).
5. Integrate **Sealed Secrets** or External Secrets for sensitive data.
6. Set up **notifications** to Slack on sync/health changes.

**Skills:** Kustomize/Helm, projects, sync waves/hooks, secrets, notifications.

### Project 3 (Advanced): Multi-Cluster Platform with ApplicationSets and Progressive Delivery

**Goal:** A scalable GitOps platform managing many apps across clusters with safe rollouts.

Steps:
1. Register multiple clusters with Argo CD.
2. Use **ApplicationSets** (git/cluster generators) to template apps across clusters/envs.
3. Bootstrap everything with the **app-of-apps** pattern (Argo manages itself).
4. Adopt **Argo Rollouts** for canary/blue-green with automated analysis.
5. Configure **SSO + RBAC** per team/project.
6. Run Argo CD in **HA**, shard the controller, and export **Prometheus metrics** to Grafana.
7. Integrate secrets via Vault/External Secrets.

**Skills:** multi-cluster, ApplicationSets, app-of-apps, Argo Rollouts, SSO/RBAC, HA/scaling, observability.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Make Git the single source of truth** — no manual `kubectl apply` to managed namespaces.
- **Enable `selfHeal` and `prune`** for true GitOps reconciliation.
- **Separate config from code** — keep manifests in a dedicated GitOps repo.
- **Use Kustomize/Helm** for environment-specific configuration, not copy-paste.
- **Use AppProjects** to enforce multi-tenant guardrails (repos/namespaces/kinds).
- **Never store plaintext secrets in Git** — use Sealed Secrets/External Secrets/Vault.
- **Use app-of-apps / ApplicationSets** to manage many apps/clusters declaratively.
- **Pin `targetRevision`** to tags/commits for production (avoid `HEAD` drift surprises).
- **Use sync waves/hooks** for ordering (migrations, dependencies).
- **Run in HA and monitor** with Prometheus metrics + notifications.
- **Integrate Argo Rollouts** for safe progressive delivery.

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Manual `kubectl` edits to managed apps | Drift, surprises | Enable selfHeal; change via Git |
| Plaintext secrets in Git | Credential leak | Sealed Secrets / External Secrets / Vault |
| Tracking `HEAD` in prod | Unintended deploys | Pin to tags/commits |
| No prune | Orphaned resources linger | Enable `prune: true` |
| One giant repo, no projects | No isolation/multi-tenancy | Use AppProjects + structure |
| Copy-pasted env manifests | Drift, errors | Kustomize overlays / Helm values |
| No ordering for migrations | Broken deploys | Sync waves + hooks |
| Single-replica Argo CD | Outage risk | Run HA, shard controller |
| Giving CI cluster credentials | Larger attack surface | Pull-based GitOps (Argo agent) |
| Ignoring health/sync alerts | Silent failures | Notifications + metrics |

---

## 7. Interview Questions

### Beginner

1. **What is Argo CD?**
   *A declarative GitOps continuous delivery tool for Kubernetes that keeps cluster state in sync with Git.*

2. **What is GitOps?**
   *An operational model where Git is the single source of truth and an agent continuously reconciles the cluster to match the declared state.*

3. **What does an Argo CD `Application` define?**
   *A source (Git repo/path/revision), a destination (cluster/namespace), a project, and a sync policy.*

4. **What do Synced and OutOfSync mean?**
   *Synced means live state matches Git; OutOfSync means they differ.*

5. **What is the difference between manual and automated sync?**
   *Manual requires you to trigger a sync; automated applies Git changes automatically.*

### Intermediate

6. **What is the difference between push and pull-based CD?**
   *Push: CI applies changes into the cluster (needs cluster creds). Pull (GitOps): an in-cluster agent pulls from Git and applies — more secure and self-healing.*

7. **What do `selfHeal` and `prune` do?**
   *selfHeal reverts manual/out-of-band changes (drift); prune deletes resources removed from Git.*

8. **What config tools does Argo CD support?**
   *Plain YAML/directories, Helm, Kustomize, Jsonnet, and custom config management plugins.*

9. **What are AppProjects for?**
   *Grouping applications and enforcing guardrails — allowed repos, destinations, and resource kinds — for multi-tenancy.*

10. **How do sync waves and hooks work?**
    *Sync waves order resource application (lower waves first); hooks (PreSync/Sync/PostSync) run jobs at phases of a sync.*

### Advanced

11. **Explain the app-of-apps pattern.**
    *A parent Application whose source contains other Application manifests, enabling you to bootstrap and manage many apps declaratively from one root app.*

12. **What are ApplicationSets and when do you use them?**
    *A controller that templates many Applications from generators (list/cluster/git/matrix), used for multi-cluster/multi-environment deployments at scale.*

13. **How should secrets be handled with Argo CD?**
    *Never store plaintext in Git; use Sealed Secrets, External Secrets Operator, SOPS, or Vault integration so only encrypted/referenced secrets live in Git.*

14. **How does Argo CD enable safe rollbacks?**
    *Every desired state is versioned in Git, so you can `argocd app rollback` to a prior revision or simply revert the Git commit; the controller reconciles to it.*

15. **How do you achieve progressive delivery with Argo?**
    *Use Argo Rollouts' `Rollout` resource for canary/blue-green strategies with automated analysis and rollback, deployed and managed via Argo CD.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** Argo CD is a CD tool for:
- A) VMs  B) Kubernetes  C) Serverless  D) Bare metal only

**Q2.** In GitOps, the source of truth is:
- A) The cluster  B) Git  C) The CI server  D) A database

**Q3.** What is Argo CD's core custom resource?
- A) Deployment  B) Application  C) Pod  D) Pipeline

**Q4.** "OutOfSync" means:
- A) Git is unreachable  B) Live state differs from Git  C) App is healthy  D) App is deleted

**Q5.** Which setting reverts manual cluster changes?
- A) `prune`  B) `selfHeal`  C) `automated`  D) `hook`

**Q6.** Which deletes resources removed from Git?
- A) `selfHeal`  B) `prune`  C) `diff`  D) `wave`

**Q7.** Which pattern bootstraps many apps from one root app?
- A) ApplicationSet  B) App-of-apps  C) Helm  D) Kustomize

**Q8.** Which generates many Applications from a template?
- A) AppProject  B) ApplicationSet  C) Rollout  D) Sync wave

**Q9.** Which component renders Helm/Kustomize manifests?
- A) API Server  B) Repo Server  C) Redis  D) Dex

**Q10.** Progressive delivery (canary/blue-green) is provided by:
- A) Argo Workflows  B) Argo Rollouts  C) Argo Events  D) Dex

### Short Answer

**S1.** Name the two main fields every Application needs besides `project`.

**S2.** Which Argo CD component performs reconciliation?

**S3.** What annotation orders resources during sync?

**S4.** Name one secure way to store secrets for GitOps.

**S5.** What command rolls an app back to a prior revision?

### Answer Key

**Multiple Choice:** Q1-B, Q2-B, Q3-B, Q4-B, Q5-B, Q6-B, Q7-B, Q8-B, Q9-B, Q10-B

**Short Answer:**
- **S1.** `source` and `destination`.
- **S2.** The **Application Controller**.
- **S3.** `argocd.argoproj.io/sync-wave`.
- **S4.** Sealed Secrets / External Secrets Operator / SOPS / Vault (any one).
- **S5.** `argocd app rollback <app> <id>`.

---

## 9. Further Resources

### Official Documentation
- Argo CD docs — https://argo-cd.readthedocs.io/
- Getting started — https://argo-cd.readthedocs.io/en/stable/getting_started/
- ApplicationSet — https://argo-cd.readthedocs.io/en/stable/user-guide/application-set/
- Argo Rollouts — https://argo-rollouts.readthedocs.io/

### Learning
- Argo Project — https://argoproj.github.io/
- OpenGitOps principles — https://opengitops.dev/
- Example apps — https://github.com/argoproj/argocd-example-apps

### Tools & Ecosystem
- Argo Rollouts (progressive delivery)
- Sealed Secrets, External Secrets Operator, SOPS (secrets)
- Flux CD (alternative GitOps tool)

### Books & Guides
- *GitOps and Kubernetes* — Yuen, Matyushentsev, Ekenstam, Suen (Manning)
- CNCF GitOps Working Group resources

### Certifications
- GitOps Fundamentals (Codefresh/Akuity courses)
- CKA/CKAD (Kubernetes foundation)

---

*End of Argo CD — Zero to Hero.*
