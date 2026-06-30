# Google Cloud Platform (GCP) — Zero to Hero

> A complete, beginner-to-advanced guide to Google Cloud Platform for DevOps engineers.

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

### 1.1 What is GCP?

**Google Cloud Platform (GCP)** is Google's suite of **cloud computing services**, running on the same global infrastructure that powers Google Search, Gmail, and YouTube. It offers compute, storage, networking, big data, machine learning, and management tools. GCP is one of the three major hyperscale clouds (with AWS and Azure) and is particularly strong in **data analytics, machine learning/AI, Kubernetes** (Google created Kubernetes), and global networking.

### 1.2 Why GCP?

| Strength | Explanation |
|----------|-------------|
| **Kubernetes leadership** | GKE is the most mature managed Kubernetes (Google invented K8s) |
| **Data & analytics** | BigQuery is a best-in-class serverless data warehouse |
| **AI/ML** | Vertex AI, TPUs, and Google's ML heritage |
| **Global network** | Google's private fiber backbone, low latency |
| **Live migration** | VMs migrate during maintenance with no downtime |
| **Per-second billing & sustained-use discounts** | Cost-friendly defaults |
| **Strong networking** | Global VPCs spanning regions |

### 1.3 Global infrastructure

```
   Multi-region (e.g., US, EU, ASIA)
        │ contains
        ▼
   Region (e.g., us-central1)         [independent geographic area]
        │ contains
        ▼
   Zones (us-central1-a, -b, -c)      [isolated deployment areas]
        │ contain
        ▼
   Resources (VMs, disks, etc.)
```

- **Region:** An independent geographic area (e.g., `us-central1`, `europe-west1`).
- **Zone:** An isolated location within a region; deploy across zones for HA.
- **Edge / PoPs:** Hundreds of points of presence for CDN and low-latency entry to Google's network.

### 1.4 The resource hierarchy

GCP organizes resources in a hierarchy that governs **IAM policies and billing**:

```
   Organization        (your company root; tied to Workspace/Cloud Identity)
        │
        ▼
   Folders             (optional; group projects by dept/team/env)
        │
        ▼
   Projects            (the core unit: billing, APIs, resources, IAM)
        │
        ▼
   Resources           (VMs, buckets, databases, etc.)
```

- **Organization:** Root node representing your company.
- **Folders:** Optional grouping (departments, teams, environments) for delegated administration.
- **Project:** The fundamental unit — every resource belongs to exactly one project; it's the boundary for billing, APIs, quotas, and IAM. **Most work starts by creating a project.**
- **Resource:** An individual service instance.

> IAM policies set higher in the hierarchy are **inherited** downward.

### 1.5 Projects in depth

A **project** has:
- A globally unique **Project ID** (immutable), a **Project Name**, and a **Project Number**.
- Enabled **APIs/services** (you must enable an API before using a service).
- A linked **billing account**.
- Its own **IAM policy** and **quotas**.

### 1.6 IAM model

**Cloud IAM** answers "**who** can do **what** on **which resource**":
- **Principal (member):** a user, group, service account, or domain.
- **Role:** a collection of permissions (Basic, Predefined, or Custom).
- **Binding:** assigns a role to a principal on a resource (and its descendants via inheritance).

### 1.7 When to use GCP

- Data analytics and warehousing (BigQuery), streaming (Dataflow/Pub-Sub).
- Kubernetes-centric platforms (GKE) and cloud-native apps.
- AI/ML workloads (Vertex AI, TPUs).
- Global, low-latency applications leveraging Google's network.

When **not** ideal: organizations deeply standardized on another cloud's ecosystem, or with specific service/compliance requirements only another provider meets.

---

## 2. Installation & Setup

### 2.1 Create an account and project

Sign up at https://cloud.google.com (free tier + credits). Create a project in the Console, or via CLI below.

### 2.2 Install the Google Cloud CLI (gcloud)

```bash
# Linux
curl https://sdk.cloud.google.com | bash
exec -l $SHELL

# macOS
brew install --cask google-cloud-sdk

# Verify
gcloud version
```

The SDK includes `gcloud` (general), `gsutil` (Cloud Storage), and `bq` (BigQuery). (`gcloud storage` is the newer unified storage command.)

### 2.3 Initialize and authenticate

```bash
gcloud init                                  # interactive: login + pick project/region
gcloud auth login                            # user auth
gcloud auth application-default login        # creds for local SDKs/Terraform
gcloud config list                           # current config
```

### 2.4 Create a project and set defaults

```bash
gcloud projects create my-demo-proj-123 --name="Demo"
gcloud config set project my-demo-proj-123
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a

# Link billing + enable an API
gcloud services enable compute.googleapis.com
```

**Expected:** Project created, set as default; Compute Engine API enabled.

### 2.5 Cloud Shell (no install)

The browser-based **Cloud Shell** provides a preauthenticated environment with `gcloud`, `kubectl`, Terraform, and an editor — great for quick tasks.

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 Compute Engine (VMs)

```bash
gcloud compute instances create my-vm \
  --machine-type=e2-medium \
  --image-family=debian-12 --image-project=debian-cloud \
  --zone=us-central1-a

gcloud compute instances list
gcloud compute ssh my-vm --zone=us-central1-a
```

**Compute Engine** is IaaS. Key ideas: **machine types** (e2, n2, c3 families; custom types), **images**, **persistent disks**, **preemptible/Spot VMs** (cheap, interruptible), and **live migration** during maintenance.

#### 3.1.2 Compute options overview

| Service | Type | Use |
|---------|------|-----|
| **Compute Engine** | IaaS VMs | Full control, custom workloads |
| **Google Kubernetes Engine (GKE)** | Containers | Managed Kubernetes |
| **Cloud Run** | Serverless containers | Run stateless containers, scale to zero |
| **Cloud Functions** | FaaS | Event-driven functions |
| **App Engine** | PaaS | Managed app hosting (standard/flexible) |

#### 3.1.3 Cloud Storage (object storage)

```bash
gcloud storage buckets create gs://my-unique-bucket --location=US
gcloud storage cp ./file.txt gs://my-unique-bucket/
gcloud storage ls gs://my-unique-bucket/
```

**Cloud Storage** holds objects in globally-unique **buckets**. Storage classes: **Standard**, **Nearline** (30-day), **Coldline** (90-day), **Archive** (365-day) — trade access frequency for cost. Use lifecycle rules to auto-transition.

#### 3.1.4 Networking (VPC)

GCP **VPCs are global** (span all regions) — a notable difference from AWS/Azure:

```bash
gcloud compute networks create my-vpc --subnet-mode=custom
gcloud compute networks subnets create web \
  --network=my-vpc --range=10.0.1.0/24 --region=us-central1
gcloud compute firewall-rules create allow-ssh \
  --network=my-vpc --allow=tcp:22
```

Concepts: **VPC** (global), **subnets** (regional), **firewall rules**, **routes**, **Cloud Load Balancing** (global L7/L4), **Cloud NAT**, **Cloud DNS**.

#### 3.1.5 IAM basics

```bash
gcloud projects add-iam-policy-binding my-demo-proj-123 \
  --member="user:alice@example.com" \
  --role="roles/viewer"
```

Role types:
- **Basic:** Owner, Editor, Viewer (broad — avoid in production).
- **Predefined:** Service-specific, fine-grained (e.g., `roles/storage.objectViewer`).
- **Custom:** Your own permission set.

### 3.2 INTERMEDIATE

#### 3.2.1 Service accounts

```bash
gcloud iam service-accounts create my-sa --display-name="My SA"
gcloud projects add-iam-policy-binding my-proj \
  --member="serviceAccount:my-sa@my-proj.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"
```

A **service account** is a non-human identity for applications/VMs/services. Best practice: attach a service account to a resource (VM/GKE/Cloud Run) and grant it least-privilege roles — **avoid downloading SA keys** (use attached identities or Workload Identity Federation).

#### 3.2.2 Google Kubernetes Engine (GKE)

```bash
gcloud container clusters create-auto my-cluster --region=us-central1   # Autopilot
# or standard:
gcloud container clusters create my-cluster --num-nodes=2 --zone=us-central1-a
gcloud container clusters get-credentials my-cluster --region=us-central1
kubectl get nodes
```

**GKE** is managed Kubernetes. Modes:
- **Autopilot** — Google manages nodes; you pay per pod resource (hands-off).
- **Standard** — you manage node pools.

Integrates with **Workload Identity** (pods authenticate as service accounts), Artifact Registry, and Cloud Load Balancing.

#### 3.2.3 Cloud Run (serverless containers)

```bash
gcloud run deploy myservice \
  --image=us-docker.pkg.dev/proj/repo/app:v1 \
  --region=us-central1 --allow-unauthenticated
```

**Cloud Run** runs stateless containers, **scales to zero**, and bills per request/CPU. Great middle ground between Functions and GKE — bring any container, no cluster to manage.

#### 3.2.4 Databases

| Service | Type |
|---------|------|
| **Cloud SQL** | Managed MySQL/PostgreSQL/SQL Server |
| **Cloud Spanner** | Globally distributed, horizontally scalable relational |
| **Firestore** | Serverless NoSQL document DB |
| **Bigtable** | Wide-column NoSQL for huge analytical/IoT workloads |
| **Memorystore** | Managed Redis/Memcached |
| **BigQuery** | Serverless data warehouse (analytics) |

#### 3.2.5 BigQuery (data warehouse)

```bash
bq mk mydataset
bq query --use_legacy_sql=false \
  'SELECT name, SUM(number) AS total
   FROM `bigquery-public-data.usa_names.usa_1910_2013`
   GROUP BY name ORDER BY total DESC LIMIT 5'
```

**BigQuery** is a serverless, petabyte-scale analytics warehouse — you run SQL without managing infrastructure, billed by data scanned (or via slots/flat-rate). A flagship GCP differentiator.

#### 3.2.6 Pub/Sub (messaging)

```bash
gcloud pubsub topics create my-topic
gcloud pubsub subscriptions create my-sub --topic=my-topic
gcloud pubsub topics publish my-topic --message="hello"
```

**Pub/Sub** is a global, scalable async messaging service for event-driven systems and streaming pipelines (often feeding Dataflow → BigQuery).

#### 3.2.7 Operations suite (monitoring/logging)

Formerly Stackdriver, **Cloud Operations** includes:
- **Cloud Monitoring** — metrics, dashboards, alerting (uptime checks, SLOs).
- **Cloud Logging** — centralized logs with the **Logs Explorer** and log-based metrics.
- **Cloud Trace / Profiler / Error Reporting** — APM and diagnostics.

### 3.3 ADVANCED

#### 3.3.1 Infrastructure as Code

```hcl
# Terraform with the Google provider
resource "google_storage_bucket" "data" {
  name     = "my-tf-bucket-123"
  location = "US"
}
```

Options: **Terraform** (most popular), **Config Connector** (manage GCP via Kubernetes CRDs), and **Infrastructure Manager** (managed Terraform). Deployment Manager is legacy.

#### 3.3.2 Organization, folders, and policy governance

- Use **folders** to mirror org structure and delegate admin.
- **Organization Policies** enforce constraints (e.g., restrict regions, disable SA key creation, require OS Login).
- **IAM inheritance** flows from org → folder → project → resource.
- **Resource hierarchy + least privilege** is the foundation of governance.

#### 3.3.3 Networking deep dive

- **Shared VPC:** A host project shares its VPC with service projects (central network control).
- **VPC Peering** and **Cloud Interconnect/VPN** for hybrid/multi-VPC connectivity.
- **Cloud Load Balancing:** global, anycast, single IP across regions (L7 HTTP(S), L4 TCP/UDP).
- **Cloud Armor** for WAF/DDoS protection; **Cloud CDN** for caching.
- **Private Google Access / Private Service Connect** to reach Google APIs privately.

#### 3.3.4 Workload Identity Federation

Lets external identities (GitHub Actions, AWS, on-prem OIDC) impersonate GCP service accounts **without SA keys** — the recommended keyless way to authenticate CI/CD and external workloads to GCP.

#### 3.3.5 Security: secrets, KMS, and posture

- **Secret Manager** — store/version secrets, accessed via IAM.
- **Cloud KMS** — managed encryption keys (CMEK); envelope encryption.
- **Security Command Center** — posture management, threat detection.
- **VPC Service Controls** — perimeter to prevent data exfiltration.
- **Binary Authorization** — only allow trusted container images in GKE.

```bash
echo -n "s3cr3t" | gcloud secrets create db-pass --data-file=-
gcloud secrets versions access latest --secret=db-pass
```

#### 3.3.6 Cost management

- **Billing accounts, budgets, and alerts**; export billing to **BigQuery** for analysis.
- **Discounts:** **Sustained-use** (automatic for steady VMs), **Committed-use** (1–3 yr commitments), **Spot/preemptible VMs** (deep discounts, interruptible).
- **Per-second billing**; right-size with **Recommender**.
- Labels for cost allocation; quotas to cap spend.

#### 3.3.7 Data & analytics pipelines

```
   Sources ──► Pub/Sub ──► Dataflow (Apache Beam) ──► BigQuery ──► Looker/BI
                              (stream/batch ETL)        (warehouse)
```

GCP excels at analytics: **Dataflow** (managed Beam), **Dataproc** (managed Spark/Hadoop), **Composer** (managed Airflow), **Dataform/dbt**, and **Vertex AI** for ML.

#### 3.3.8 Reliability and SRE

- Deploy across **zones/regions**; use global load balancing and managed instance groups (autoscaling/autohealing).
- Define **SLOs** and error budgets in Cloud Monitoring (Google pioneered SRE).
- Backups, multi-region storage, and DR planning per the **Google Cloud Architecture Framework**.

---

## 4. Hands-on Tasks

### Task 1: Initialize gcloud and create a project

```bash
gcloud init
gcloud projects create my-proj-$RANDOM --name="Demo"
gcloud config set project <project-id>
```

**Expected:** Project created and set as default.

### Task 2: Launch a VM

Create an `e2-medium` Debian VM and SSH in.

**Expected:** VM runs; SSH session opens.

### Task 3: Create a Cloud Storage bucket and upload

```bash
gcloud storage buckets create gs://my-bucket-$RANDOM --location=US
gcloud storage cp ./file.txt gs://<bucket>/
```

**Expected:** Object uploaded and listable.

### Task 4: Grant an IAM role

Bind `roles/viewer` to a user on the project.

**Expected:** The user gains read access; inheritance applies to resources.

### Task 5: Create and use a service account

Create an SA, grant it `roles/storage.objectViewer`, and attach it to a VM.

**Expected:** The VM can read the bucket without keys.

### Task 6: Deploy to Cloud Run

Build/push a container to Artifact Registry and `gcloud run deploy`.

**Expected:** A public URL serves the container; scales to zero when idle.

### Task 7: Create a GKE Autopilot cluster

```bash
gcloud container clusters create-auto demo --region=us-central1
gcloud container clusters get-credentials demo --region=us-central1
kubectl get nodes
```

**Expected:** Cluster is ready; `kubectl` works.

### Task 8: Run a BigQuery query

Query a public dataset (section 3.2.5).

**Expected:** Aggregated results returned with bytes-scanned reported.

### Task 9: Store a secret in Secret Manager

Create a secret and access the latest version.

**Expected:** The secret value is returned (subject to IAM).

### Task 10: Set a budget alert

In Billing, create a budget with threshold alerts.

**Expected:** Alerts configured to notify on spend.

---

## 5. Projects

### Project 1 (Beginner): Deploy a Web App on Cloud Run

**Goal:** Containerize and serve an app serverlessly.

Steps:
1. Create a project; enable required APIs.
2. Build a container, push to **Artifact Registry**.
3. Deploy on **Cloud Run** with a custom domain and HTTPS.
4. Store config secrets in **Secret Manager**; grant the service's SA access.
5. Add **Cloud Monitoring** uptime checks and **Cloud Logging**.

**Skills:** Artifact Registry, Cloud Run, Secret Manager, IAM, monitoring.

### Project 2 (Intermediate): Three-Tier App with Managed Services

**Goal:** Web + API + database with secure identity and IaC.

Steps:
1. Provision a custom **VPC**, subnets, and firewall rules.
2. Host the API on **Cloud Run**/GKE; data in **Cloud SQL** (private IP).
3. Use **service accounts** + **Secret Manager** (no keys).
4. Put a **global HTTPS Load Balancer** + **Cloud Armor** in front.
5. Provision everything with **Terraform**.
6. Add monitoring dashboards, alerts, and SLOs.

**Skills:** VPC, Cloud SQL, IAM/SA, load balancing, Terraform, observability.

### Project 3 (Advanced): Data Platform with GKE and Analytics

**Goal:** A scalable, governed platform combining apps and analytics.

Steps:
1. Set up an **Organization → Folders → Projects** structure with **Org Policies**.
2. Use **Shared VPC** for centralized networking across service projects.
3. Run workloads on **GKE Autopilot** with **Workload Identity** (keyless auth).
4. Build a streaming pipeline: **Pub/Sub → Dataflow → BigQuery**; visualize with Looker.
5. Authenticate CI/CD via **Workload Identity Federation** (no SA keys).
6. Secure with **VPC Service Controls**, **CMEK (Cloud KMS)**, **Binary Authorization**, and **Security Command Center**.
7. Optimize cost (committed-use, Spot, budgets, billing export to BigQuery) and define **SLOs**.

**Skills:** org hierarchy/governance, Shared VPC, GKE/Workload Identity, data pipelines, federation, advanced security, cost/SRE.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Use the resource hierarchy** (org/folders/projects) and **least-privilege IAM**; prefer predefined/custom roles over Basic.
- **Use service accounts attached to resources**; avoid downloading SA keys — use Workload Identity / Federation.
- **One project per app/environment** for isolation and clear billing.
- **Store secrets in Secret Manager**, not in code or images.
- **Adopt IaC (Terraform)** for repeatable infrastructure.
- **Enforce Organization Policies** (region restrictions, disable key creation, OS Login).
- **Design for HA** across zones/regions; use managed instance groups and global LB.
- **Use the right storage class** and lifecycle rules; use BigQuery for analytics.
- **Enable Cloud Monitoring/Logging**, define SLOs and alerts.
- **Control cost** with committed-use/sustained-use discounts, Spot VMs, budgets, and labels.
- **Harden** with VPC Service Controls, CMEK, and Security Command Center.

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Using Basic roles (Owner/Editor) | Over-privileged access | Predefined/custom least-privilege roles |
| Downloading & embedding SA keys | Key leakage | Attached SAs / Workload Identity Federation |
| Secrets in code/images | Credential leak | Secret Manager |
| One giant project for everything | Poor isolation/billing | Project per app/env + folders |
| Single-zone deployment | Outage on zone failure | Multi-zone/region + managed instance groups |
| Forgetting to enable APIs | "API not enabled" errors | `gcloud services enable ...` |
| Ignoring egress/data costs | Surprise bills | Budgets, billing export, design for locality |
| No org policies | Ungoverned sprawl | Enforce Organization Policies |
| Public buckets/resources | Data exposure | IAM least privilege, uniform bucket access |
| Click-ops (no IaC) | Drift, irreproducible | Terraform |

---

## 7. Interview Questions

### Beginner

1. **What is GCP?**
   *Google Cloud Platform — Google's suite of cloud services for compute, storage, networking, data, and ML on its global infrastructure.*

2. **What is a GCP project?**
   *The fundamental organizing unit: the boundary for billing, enabled APIs, quotas, IAM, and resources. Every resource belongs to one project.*

3. **Explain regions and zones.**
   *A region is an independent geographic area; zones are isolated locations within a region. Deploy across zones for high availability.*

4. **What is Compute Engine?**
   *GCP's IaaS virtual machine service offering configurable VMs with persistent disks and images.*

5. **What is Cloud Storage?**
   *Object storage organized in globally-unique buckets, with classes (Standard/Nearline/Coldline/Archive) trading access frequency for cost.*

### Intermediate

6. **What is a service account?**
   *A non-human identity for applications/VMs/services; attach it to resources and grant least-privilege roles instead of using user credentials.*

7. **How does GCP IAM work?**
   *It binds principals (users/groups/SAs) to roles (sets of permissions) on resources; policies are inherited down the resource hierarchy.*

8. **Compare GKE, Cloud Run, and Cloud Functions.**
   *GKE is managed Kubernetes for full orchestration; Cloud Run runs stateless containers serverlessly (scale to zero); Cloud Functions runs event-driven snippets of code.*

9. **What makes GCP VPCs distinctive?**
   *VPCs are global (span all regions), with regional subnets — unlike the regional VPCs of some other clouds.*

10. **What is BigQuery?**
    *A serverless, petabyte-scale data warehouse where you run SQL without managing infrastructure, billed by data scanned or via slots.*

### Advanced

11. **What is Workload Identity Federation and why use it?**
    *It lets external identities (GitHub Actions, AWS, OIDC providers) impersonate GCP service accounts without SA keys, enabling secure keyless authentication for CI/CD and external workloads.*

12. **What is a Shared VPC?**
    *A model where a host project shares its VPC network with service projects, centralizing network administration while letting teams deploy into shared subnets.*

13. **How do Organization Policies differ from IAM?**
    *IAM grants who can do what; Organization Policies set guardrail constraints on resources (e.g., allowed regions, disabling SA key creation) regardless of IAM permissions.*

14. **How would you secure sensitive data on GCP?**
    *Secret Manager for secrets, Cloud KMS/CMEK for encryption keys, VPC Service Controls to prevent exfiltration, least-privilege IAM, and Security Command Center for posture/threats.*

15. **Describe a streaming analytics pipeline on GCP.**
    *Ingest events via Pub/Sub, process with Dataflow (Apache Beam) for stream/batch ETL, load into BigQuery for analytics, and visualize with Looker — all managed/serverless.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** The fundamental organizing unit in GCP is the:
- A) Folder  B) Project  C) Zone  D) Bucket

**Q2.** GCP VPCs are:
- A) Regional  B) Zonal  C) Global  D) Per-subnet

**Q3.** Which is serverless containers that scale to zero?
- A) Compute Engine  B) GKE  C) Cloud Run  D) App Engine flexible

**Q4.** Which is Google's serverless data warehouse?
- A) Bigtable  B) BigQuery  C) Spanner  D) Firestore

**Q5.** A non-human identity for apps is a:
- A) User  B) Group  C) Service account  D) Role

**Q6.** Which provides managed Kubernetes?
- A) Cloud Run  B) GKE  C) Cloud Functions  D) Dataflow

**Q7.** Cheap, interruptible VMs are called:
- A) Reserved  B) Spot/preemptible  C) Sustained  D) Shielded

**Q8.** Which keyless method authenticates GitHub Actions to GCP?
- A) SA key file  B) Workload Identity Federation  C) Basic auth  D) API key

**Q9.** Which stores application secrets?
- A) Cloud KMS only  B) Secret Manager  C) Cloud Storage  D) BigQuery

**Q10.** Isolated locations within a region are:
- A) Folders  B) Zones  C) Projects  D) PoPs

### Short Answer

**S1.** What command enables a service API?

**S2.** Name the global messaging service used for event-driven pipelines.

**S3.** Which discount applies automatically to steadily-running VMs?

**S4.** What hierarchy level sits directly above projects (optional)?

**S5.** Which GKE mode has Google fully manage the nodes?

### Answer Key

**Multiple Choice:** Q1-B, Q2-C, Q3-C, Q4-B, Q5-C, Q6-B, Q7-B, Q8-B, Q9-B, Q10-B

**Short Answer:**
- **S1.** `gcloud services enable <api>`.
- **S2.** Pub/Sub.
- **S3.** Sustained-use discounts.
- **S4.** Folders.
- **S5.** Autopilot.

---

## 9. Further Resources

### Official Documentation
- GCP docs — https://cloud.google.com/docs
- gcloud CLI reference — https://cloud.google.com/sdk/gcloud/reference
- Architecture Framework — https://cloud.google.com/architecture/framework
- Architecture Center — https://cloud.google.com/architecture

### Learning
- Google Cloud Skills Boost — https://www.cloudskillsboost.google/
- Free tier — https://cloud.google.com/free
- Qwiklabs hands-on labs

### Tools
- gcloud, gsutil, bq, Cloud Shell
- Terraform Google provider, Config Connector
- Cloud Code (IDE plugins)

### Books
- *Google Cloud Platform in Action* — JJ Geewax
- *Official Google Cloud Certified Associate Cloud Engineer Study Guide*
- *Site Reliability Engineering* — Google (free online)

### Certifications
- Associate Cloud Engineer
- Professional Cloud Architect
- Professional Cloud DevOps Engineer
- Professional Data Engineer

---

*End of Google Cloud Platform (GCP) — Zero to Hero.*
