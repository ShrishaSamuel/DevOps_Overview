# Helm — Zero to Hero

> A complete, beginner-to-advanced guide to Helm, the package manager for Kubernetes.

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

### 1.1 What is Helm?

**Helm** is the **package manager for Kubernetes** — often called "the apt/yum/homebrew of Kubernetes." It bundles Kubernetes manifests into reusable, versioned, configurable packages called **charts**. Instead of manually writing and `kubectl apply`-ing dozens of YAML files, you install an application with a single command and customize it through values. Helm is a CNCF Graduated project.

### 1.2 The problem Helm solves

Deploying a real application to Kubernetes means managing many related objects: Deployments, Services, ConfigMaps, Secrets, Ingresses, ServiceAccounts, RBAC, HPAs, and more. Doing this with raw YAML is:
- **Repetitive** — the same boilerplate across environments.
- **Hard to parameterize** — different values for dev/staging/prod require copy-paste.
- **Hard to version and roll back** — no built-in release tracking.
- **Hard to share** — no standard distribution format.

Helm addresses all of these with **templated, parameterized, versioned packages**.

```
   Raw YAML approach                    Helm approach
   ─────────────────                    ─────────────
   deployment-dev.yaml                  chart/
   deployment-staging.yaml      vs.       templates/deployment.yaml  (templated)
   deployment-prod.yaml                   values.yaml                (defaults)
   service-dev.yaml                       values-prod.yaml           (overrides)
   ...dozens more...                    → helm install -f values-prod.yaml
```

### 1.3 Key terminology

| Term | Meaning |
|------|---------|
| **Chart** | A Helm package: templates + default values + metadata |
| **Release** | An instance of a chart installed in a cluster (named) |
| **Repository** | A place charts are stored and shared (HTTP or OCI registry) |
| **Values** | Configuration that parameterizes templates |
| **Template** | A Go-templated Kubernetes manifest |
| **Revision** | A versioned snapshot of a release (enables rollback) |

### 1.4 How Helm works (Helm 3 architecture)

```
   helm CLI ──► renders templates + values ──► Kubernetes manifests
        │                                            │
        │ stores release metadata as a Secret        ▼
        └──────────────────────► Kubernetes API ──► creates/updates objects
                                       │
                              Release history (revisions) in-cluster
```

- **Helm 3 is client-only** — there is **no Tiller** (the old server-side component removed in v3 for security).
- The CLI renders charts locally and talks directly to the Kubernetes API using your kubeconfig.
- **Release state** is stored in-cluster as Secrets (by default) in the release's namespace.

### 1.5 Helm 2 vs. Helm 3

| Aspect | Helm 2 | Helm 3 |
|--------|--------|--------|
| Tiller (server component) | Required | **Removed** (more secure) |
| Release storage | ConfigMaps | Secrets (per namespace) |
| Security | Tiller had broad cluster access | Uses your kubeconfig/RBAC |
| Namespaces | Global releases | Namespace-scoped releases |
| OCI registries | No | Yes (charts as OCI artifacts) |

### 1.6 When to use Helm

- Packaging and distributing Kubernetes applications.
- Deploying the same app across multiple environments with different config.
- Installing third-party software (databases, ingress controllers, monitoring stacks) from public charts.
- Versioned releases with rollback.

When **not** ideal: very simple single-manifest apps (plain YAML or Kustomize may suffice); teams preferring pure declarative GitOps may use Kustomize or Helm *rendered* by Argo CD/Flux.

### 1.7 Helm vs. Kustomize

| | Helm | Kustomize |
|--|------|-----------|
| Approach | Templating (Go templates) + packaging | Overlay/patch plain YAML |
| Parameterization | Rich (values, functions, conditionals) | Patches, no logic |
| Packaging/sharing | Yes (charts, repos) | No native packaging |
| Learning curve | Higher (templating) | Lower |
| Best for | Complex, distributable apps | Environment overlays of own manifests |

They're often combined (Helm for packaging, Kustomize for last-mile patches).

---

## 2. Installation & Setup

### 2.1 Install Helm

```bash
# Script
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Or package managers
brew install helm                 # macOS
choco install kubernetes-helm     # Windows
sudo snap install helm --classic  # Linux snap

helm version
```

**Expected:** Prints the Helm client version (v3.x).

### 2.2 Add a repository and install a chart

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo nginx
helm install my-nginx bitnami/nginx
helm list
```

**Expected:** A release named `my-nginx` is deployed; `helm list` shows it as `deployed`.

### 2.3 Create your own chart

```bash
helm create mychart
```

This scaffolds:

```
mychart/
  Chart.yaml          # chart metadata (name, version, appVersion)
  values.yaml         # default configuration values
  charts/             # subchart dependencies
  templates/          # templated Kubernetes manifests
    deployment.yaml
    service.yaml
    ingress.yaml
    _helpers.tpl      # reusable template snippets
    NOTES.txt         # post-install message
    tests/            # chart tests
  .helmignore
```

### 2.4 Install your chart

```bash
helm install myrelease ./mychart
helm install myrelease ./mychart --set replicaCount=3
helm install myrelease ./mychart -f custom-values.yaml
```

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 Chart.yaml

```yaml
apiVersion: v2
name: mychart
description: A Helm chart for my app
type: application          # or "library"
version: 0.1.0             # chart version (SemVer)
appVersion: "1.16.0"       # version of the app being deployed
dependencies:
  - name: postgresql
    version: "15.x.x"
    repository: https://charts.bitnami.com/bitnami
```

> **`version` vs `appVersion`:** `version` is the chart's own version; `appVersion` is the version of the application the chart deploys. They change independently.

#### 3.1.2 values.yaml

```yaml
replicaCount: 2
image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
resources:
  limits:
    cpu: 200m
    memory: 256Mi
```

Values are the **public API** of your chart — what users customize.

#### 3.1.3 Templates and the `.Values` object

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-app
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

Built-in objects: `.Values`, `.Release` (Name, Namespace, Revision), `.Chart`, `.Capabilities`, `.Files`, `.Template`.

#### 3.1.4 Core CLI commands

```bash
helm install <release> <chart>      # install
helm upgrade <release> <chart>      # upgrade
helm uninstall <release>            # remove
helm list                           # list releases
helm status <release>               # release status
helm rollback <release> <revision>  # roll back
helm history <release>              # revision history
```

#### 3.1.5 Dry-run and template rendering

```bash
helm install myrelease ./mychart --dry-run --debug   # preview without applying
helm template myrelease ./mychart                    # render manifests to stdout
helm lint ./mychart                                  # validate chart
```

### 3.2 INTERMEDIATE

#### 3.2.1 Template functions and pipelines

```yaml
metadata:
  name: {{ .Values.name | default "app" | quote }}
  labels:
    app: {{ .Values.name | lower }}
data:
  config: {{ .Values.config | toYaml | nindent 4 }}
  upper: {{ .Values.text | upper | trunc 10 }}
```

Helm includes the **Sprig** function library: `default`, `quote`, `upper/lower`, `trim`, `indent/nindent`, `toYaml`, `b64enc`, `sha256sum`, `printf`, and many more. Pipelines (`|`) chain them.

#### 3.2.2 Conditionals, loops, and `with`

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
# ...
{{- end }}

# range over a list/map
env:
{{- range $key, $val := .Values.env }}
  - name: {{ $key }}
    value: {{ $val | quote }}
{{- end }}

# with: change scope
{{- with .Values.resources }}
resources:
  {{- toYaml . | nindent 2 }}
{{- end }}
```

> `{{-` and `-}}` trim whitespace — essential for clean YAML output.

#### 3.2.3 Named templates and _helpers.tpl

```yaml
# templates/_helpers.tpl
{{- define "mychart.fullname" -}}
{{- printf "%s-%s" .Release.Name .Chart.Name | trunc 63 | trimSuffix "-" -}}
{{- end -}}

{{- define "mychart.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
{{- end -}}
```

```yaml
# use them
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
    {{- include "mychart.labels" . | nindent 4 }}
```

> Use `include` (not `template`) so output can be piped into functions like `nindent`.

#### 3.2.4 Values precedence and overrides

```bash
helm install r ./chart \
  -f values.yaml \           # base
  -f values-prod.yaml \      # overrides base
  --set image.tag=v2 \       # overrides files
  --set-string foo=123       # force string
```

Precedence (low → high): chart `values.yaml` → `-f` files (in order) → `--set`. Subchart values can be set under the subchart's key.

#### 3.2.5 Dependencies (subcharts)

```yaml
# Chart.yaml
dependencies:
  - name: redis
    version: "18.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled    # toggle via values
```

```bash
helm dependency update ./mychart   # downloads into charts/
helm dependency build ./mychart
```

Parent charts can override subchart values and use `condition`/`tags` to enable/disable them.

#### 3.2.6 Hooks

```yaml
metadata:
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
```

Hook phases: `pre-install`, `post-install`, `pre-upgrade`, `post-upgrade`, `pre-delete`, `post-delete`, `test`. Use for migrations, backups, or readiness gates.

#### 3.2.7 Chart tests

```yaml
# templates/tests/test-connection.yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: wget
      image: busybox
      command: ['wget']
      args: ['{{ include "mychart.fullname" . }}:{{ .Values.service.port }}']
  restartPolicy: Never
```

```bash
helm test myrelease
```

### 3.3 ADVANCED

#### 3.3.1 Library charts

A chart of `type: library` provides reusable named templates (no installable resources) that application charts import — DRY across many charts in an organization.

#### 3.3.2 OCI registries

Helm 3 can store charts as OCI artifacts in container registries:

```bash
helm package ./mychart                       # produces mychart-0.1.0.tgz
helm push mychart-0.1.0.tgz oci://registry.example.com/charts
helm install myrel oci://registry.example.com/charts/mychart --version 0.1.0
```

OCI is now the recommended distribution mechanism (works with Docker Hub, GHCR, ECR, etc.).

#### 3.3.3 Schema validation

```json
// values.schema.json — validates user-supplied values
{
  "$schema": "https://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["replicaCount"],
  "properties": {
    "replicaCount": { "type": "integer", "minimum": 1 }
  }
}
```

Helm validates `--set`/`-f` values against this schema on install/upgrade, catching misconfiguration early.

#### 3.3.4 Advanced functions: lookup, required, tpl

```yaml
# required: fail with a message if a value is missing
image: {{ required "image.repository is required!" .Values.image.repository }}

# lookup: read existing cluster objects at render time
{{- $existing := lookup "v1" "Secret" .Release.Namespace "my-secret" }}

# tpl: render a string that itself contains templates
value: {{ tpl .Values.templateString . }}
```

#### 3.3.5 Upgrade strategies and rollback

```bash
helm upgrade myrel ./chart --atomic --timeout 5m   # auto-rollback on failure
helm upgrade myrel ./chart --install               # install if not present
helm upgrade myrel ./chart --wait                   # wait for resources ready
helm rollback myrel 2                                # revert to revision 2
```

- `--atomic` rolls back automatically if the upgrade fails.
- `--wait` blocks until resources are ready.
- Every upgrade creates a new **revision** for rollback.

#### 3.3.6 Managing secrets

Helm doesn't encrypt values. Patterns:
- **helm-secrets** plugin (SOPS) — encrypt values files in Git.
- **External Secrets Operator** — reference external stores.
- **Sealed Secrets** for the rendered secret objects.

> Never commit plaintext secrets in `values.yaml`.

#### 3.3.7 Helm with GitOps (Argo CD / Flux)

Argo CD and Flux can consume Helm charts but typically **render them** (no in-cluster Helm release object) so Git remains the source of truth. You provide the chart + values; the GitOps controller applies the rendered manifests and manages drift.

#### 3.3.8 Plugins and the ecosystem

```bash
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade myrel ./chart     # preview changes before upgrading
```

Popular plugins: **helm-diff** (preview changes), **helm-secrets** (SOPS), **helm-unittest** (unit tests for templates).

---

## 4. Hands-on Tasks

### Task 1: Install a public chart

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami && helm repo update
helm install web bitnami/nginx
helm list
```

**Expected:** A `web` release is `deployed`.

### Task 2: Scaffold and inspect a chart

```bash
helm create demo
helm lint ./demo
helm template demo ./demo | head -40
```

**Expected:** Lint passes; rendered manifests print.

### Task 3: Install with custom values

```bash
helm install demo ./demo --set replicaCount=3 --set image.tag=1.25
kubectl get deploy
```

**Expected:** Deployment has 3 replicas with the specified image tag.

### Task 4: Upgrade and roll back

```bash
helm upgrade demo ./demo --set replicaCount=5
helm history demo
helm rollback demo 1
```

**Expected:** Replicas change to 5, then revert to revision 1's value.

### Task 5: Preview changes (dry-run / diff)

```bash
helm upgrade demo ./demo --set image.tag=1.26 --dry-run --debug
```

**Expected:** Rendered diff/output without applying.

### Task 6: Use a custom values file

Create `values-prod.yaml` and install with `-f`.

**Expected:** Overrides from the file take effect.

### Task 7: Add a named template

Add a helper in `_helpers.tpl` and `include` it in a template.

**Expected:** `helm template` shows the helper's output rendered.

### Task 8: Add a chart dependency

Add a subchart (e.g., redis) in `Chart.yaml`, run `helm dependency update`.

**Expected:** The subchart appears under `charts/` and deploys with the release.

### Task 9: Add and run a chart test

Add a `tests/` pod with the `helm.sh/hook: test` annotation; run `helm test`.

**Expected:** The test pod runs and reports success.

### Task 10: Package and push to OCI

```bash
helm package ./demo
helm push demo-0.1.0.tgz oci://<registry>/charts
```

**Expected:** The chart is pushed as an OCI artifact.

---

## 5. Projects

### Project 1 (Beginner): Package a Web App as a Chart

**Goal:** Turn a simple app's manifests into a reusable chart.

Steps:
1. `helm create` a chart; replace templates with your Deployment/Service/Ingress.
2. Parameterize image, replicas, ports, and resources via `values.yaml`.
3. Add a `NOTES.txt` with access instructions.
4. Lint, dry-run, and install.
5. Practice upgrade and rollback.

**Skills:** chart structure, values, templates, lifecycle commands.

### Project 2 (Intermediate): Multi-Environment Chart with Dependencies

**Goal:** Deploy the same app to dev/staging/prod with a database subchart.

Steps:
1. Add a `postgresql` (or redis) **dependency** with a `condition`.
2. Create `values-dev.yaml`, `values-staging.yaml`, `values-prod.yaml`.
3. Use **named templates** (`_helpers.tpl`) for consistent labels/names.
4. Add **conditionals** (enable ingress/HPA per env) and a `values.schema.json`.
5. Add **hooks** for a DB migration job.
6. Add chart **tests**.

**Skills:** dependencies, environment values, helpers, schema validation, hooks, tests.

### Project 3 (Advanced): Chart Library, OCI Distribution, and GitOps

**Goal:** Standardize and distribute charts across an organization.

Steps:
1. Build a **library chart** with shared templates; consume it from app charts.
2. Add `values.schema.json` and robust `required`/`default` guards.
3. **Package and push** charts to an **OCI registry** with CI.
4. Use **helm-diff** in CI to preview changes; `--atomic --wait` upgrades.
5. Integrate with **Argo CD/Flux** (chart rendered, Git as source of truth).
6. Manage secrets via **helm-secrets (SOPS)** or External Secrets.

**Skills:** library charts, OCI, CI/CD for charts, GitOps integration, secrets.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Use `helm lint`, `--dry-run`, and `helm template`** before applying.
- **Document every value** in `values.yaml` and validate with `values.schema.json`.
- **Use named templates (`_helpers.tpl`)** for consistent names/labels (follow `app.kubernetes.io/*` conventions).
- **Use `include` + `nindent`** for clean, correctly indented YAML.
- **Pin chart and dependency versions** (SemVer) for reproducibility.
- **Use `--atomic` and `--wait`** for safe upgrades with auto-rollback.
- **Keep secrets out of plaintext values** — use SOPS/External Secrets.
- **Use `required`** for mandatory values to fail fast with clear messages.
- **Distribute via OCI registries.**
- **Test charts** with `helm test` and `helm-unittest`.
- **Bump `version` on every change**; update `appVersion` when the app changes.

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Plaintext secrets in `values.yaml` | Credential leak | helm-secrets/SOPS, External Secrets |
| Wrong whitespace/indentation in templates | Invalid YAML | Use `{{-`/`-}}` and `nindent` |
| Using `template` instead of `include` | Can't pipe to functions | Use `include` |
| Not pinning versions | Non-reproducible deploys | Pin chart/dependency versions |
| Upgrading without preview | Surprise changes/outages | `--dry-run`, helm-diff |
| No `--atomic` on risky upgrades | Stuck failed release | `--atomic --wait` |
| Confusing `version` vs `appVersion` | Misleading metadata | Treat them separately |
| Giant unparameterized templates | Hard to reuse | Extract helpers, use values |
| Missing required values silently | Broken render | Use `required` |
| Editing released objects with kubectl | Drift vs. Helm state | Manage through Helm/GitOps |

---

## 7. Interview Questions

### Beginner

1. **What is Helm?**
   *The package manager for Kubernetes that bundles manifests into versioned, configurable charts.*

2. **What is a chart vs. a release?**
   *A chart is the package (templates + values); a release is a named instance of that chart installed in a cluster.*

3. **What is `values.yaml`?**
   *The file holding default configuration values that parameterize a chart's templates.*

4. **What's the difference between `helm install` and `helm upgrade`?**
   *`install` creates a new release; `upgrade` changes an existing one (and creates a new revision).*

5. **How do you roll back a release?**
   *`helm rollback <release> <revision>`.*

### Intermediate

6. **What is the difference between chart `version` and `appVersion`?**
   *`version` is the chart's own SemVer; `appVersion` is the version of the application the chart deploys — they change independently.*

7. **Explain values precedence.**
   *Lowest to highest: chart `values.yaml` → `-f` files (in order) → `--set`. Later sources override earlier ones.*

8. **What is `_helpers.tpl` used for?**
   *Defining reusable named templates (e.g., names, labels) shared across the chart via `include`.*

9. **Difference between `include` and `template`?**
   *`include` returns a string that can be piped into functions (e.g., `nindent`); `template` only emits output and can't be piped.*

10. **What are Helm hooks?**
    *Annotated resources that run at lifecycle phases (pre/post install/upgrade/delete, test) for tasks like migrations or backups.*

### Advanced

11. **Why was Tiller removed in Helm 3?**
    *Tiller ran server-side with broad cluster permissions, a security risk. Helm 3 is client-only and uses your kubeconfig/RBAC; release state is stored as Secrets.*

12. **How do you manage chart dependencies?**
    *Declare them in `Chart.yaml` (name/version/repository, optional `condition`/`tags`) and run `helm dependency update` to vendor them into `charts/`.*

13. **How should secrets be handled with Helm?**
    *Never store plaintext in values; use helm-secrets (SOPS), External Secrets Operator, or Sealed Secrets so only encrypted/referenced data lives in Git.*

14. **What does `--atomic` do on upgrade?**
    *If the upgrade fails, it automatically rolls back to the previous successful revision, avoiding a stuck/broken release.*

15. **How do OCI registries change chart distribution?**
    *Charts can be packaged and pushed as OCI artifacts to container registries (GHCR/ECR/Docker Hub), unifying image and chart distribution; it's now the recommended method.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** Helm is best described as:
- A) A container runtime  B) The Kubernetes package manager  C) A CI server  D) A service mesh

**Q2.** A named, installed instance of a chart is a:
- A) Repository  B) Release  C) Template  D) Revision

**Q3.** Which file holds default configuration?
- A) `Chart.yaml`  B) `values.yaml`  C) `_helpers.tpl`  D) `NOTES.txt`

**Q4.** Helm 3 removed which component?
- A) Redis  B) Tiller  C) Kubelet  D) Repo server

**Q5.** Highest precedence values source is:
- A) `values.yaml`  B) `-f` file  C) `--set`  D) schema

**Q6.** Which renders templates without installing?
- A) `helm install`  B) `helm template`  C) `helm list`  D) `helm repo`

**Q7.** Which should you use to pipe output into `nindent`?
- A) `template`  B) `include`  C) `define`  D) `range`

**Q8.** What auto-rolls back a failed upgrade?
- A) `--wait`  B) `--atomic`  C) `--dry-run`  D) `--debug`

**Q9.** Where does Helm 3 store release state?
- A) ConfigMaps  B) Secrets  C) etcd directly  D) A file

**Q10.** Which distributes charts via container registries?
- A) HTTP repo  B) OCI registry  C) Git  D) NFS

### Short Answer

**S1.** What command scaffolds a new chart?

**S2.** Name the field for the deployed application's version in `Chart.yaml`.

**S3.** Which annotation marks a resource as a chart test?

**S4.** What function fails the render if a value is missing?

**S5.** Name one tool to safely store Helm secrets in Git.

### Answer Key

**Multiple Choice:** Q1-B, Q2-B, Q3-B, Q4-B, Q5-C, Q6-B, Q7-B, Q8-B, Q9-B, Q10-B

**Short Answer:**
- **S1.** `helm create <name>`.
- **S2.** `appVersion`.
- **S3.** `"helm.sh/hook": test`.
- **S4.** `required`.
- **S5.** helm-secrets (SOPS) / External Secrets / Sealed Secrets (any one).

---

## 9. Further Resources

### Official Documentation
- Helm docs — https://helm.sh/docs/
- Chart template guide — https://helm.sh/docs/chart_template_guide/
- Best practices — https://helm.sh/docs/chart_best_practices/
- Helm Hub / Artifact Hub — https://artifacthub.io/

### Learning
- Helm quickstart — https://helm.sh/docs/intro/quickstart/
- Sprig function docs — https://masterminds.github.io/sprig/
- Go template docs — https://pkg.go.dev/text/template

### Tools & Plugins
- helm-diff — preview upgrades
- helm-secrets (SOPS) — encrypted values
- helm-unittest — template unit tests
- Artifact Hub — discover charts

### Books & Guides
- *Learning Helm* — Matt Butcher, Matt Farina, Josh Dolitsky (O'Reilly)
- *The Kubernetes Book* — Nigel Poulton (Helm chapters)

### Certifications
- CKA / CKAD (Kubernetes foundation; Helm appears in practice)

---

*End of Helm — Zero to Hero.*
