# Microsoft Azure — Zero to Hero

> A complete, beginner-to-advanced guide to Microsoft Azure cloud for DevOps engineers.

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

### 1.1 What is Microsoft Azure?

**Microsoft Azure** is a comprehensive **cloud computing platform** offering 200+ services for compute, storage, networking, databases, AI, analytics, and more. It lets organizations build, deploy, and manage applications across a global network of Microsoft-managed data centers, paying only for what they use. Azure is one of the three major hyperscale clouds (alongside AWS and Google Cloud) and is especially common in enterprises invested in the Microsoft ecosystem.

### 1.2 Cloud computing models

| Model | You manage | Provider manages | Example |
|-------|-----------|------------------|---------|
| **IaaS** | OS, apps, data | Hardware, virtualization, network | Azure Virtual Machines |
| **PaaS** | Apps, data | OS, runtime, infra | Azure App Service |
| **SaaS** | Configuration | Everything else | Microsoft 365 |
| **FaaS/Serverless** | Code | Everything else | Azure Functions |

### 1.3 Azure's global infrastructure

```
   Geography (e.g., North America, Europe)
        │ contains
        ▼
   Region (e.g., East US, West Europe)  ── paired with another region (DR)
        │ contains
        ▼
   Availability Zones (physically separate datacenters within a region)
        │ contain
        ▼
   Datacenters (racks, servers)
```

- **Region:** A set of datacenters in a geographic area. You deploy resources to regions.
- **Availability Zones (AZs):** Physically isolated locations within a region; deploy across zones for high availability.
- **Region pairs:** Each region is paired with another (often 300+ miles away) for disaster recovery and data residency.
- **Edge/CDN locations:** For content delivery and low latency.

### 1.4 The resource hierarchy

Azure organizes everything into a strict management hierarchy:

```
   Management Groups        (org-wide policy/governance, optional)
        │
        ▼
   Subscriptions            (billing + access boundary)
        │
        ▼
   Resource Groups          (logical container for related resources)
        │
        ▼
   Resources                (VMs, storage accounts, databases, etc.)
```

- **Management Group:** Groups subscriptions to apply governance (policies, RBAC) at scale.
- **Subscription:** A billing and access-control boundary; resources live in subscriptions.
- **Resource Group (RG):** A logical container holding related resources that share a lifecycle (deploy/delete together).
- **Resource:** An individual service instance (a VM, a database, a storage account).

### 1.5 Azure Resource Manager (ARM)

**Azure Resource Manager (ARM)** is the deployment and management layer — every request (Portal, CLI, PowerShell, SDK, REST) goes through ARM. It provides consistent management, RBAC, tags, locks, and template-based (declarative) deployments via **ARM templates** or **Bicep**.

### 1.6 Microsoft Entra ID (identity)

**Microsoft Entra ID** (formerly **Azure Active Directory / Azure AD**) is Azure's cloud identity and access management service. It handles authentication, users/groups, service principals, **managed identities**, conditional access, and SSO. It is the backbone of access control in Azure.

### 1.7 When to use Azure

- Enterprises already using Microsoft products (Windows Server, Active Directory, Microsoft 365, SQL Server).
- Hybrid cloud scenarios (Azure Arc, Azure Stack).
- .NET/Windows workloads, but also full Linux/open-source support.
- Regulated industries needing broad compliance certifications.

When **not** ideal: when an organization is standardized on another cloud, or for the absolute lowest-cost niche workloads where a smaller provider fits.

---

## 2. Installation & Setup

### 2.1 Create an account

Sign up at https://azure.microsoft.com — new accounts get free credits and an always-free tier for many services.

### 2.2 Install the Azure CLI

```bash
# Linux (Debian/Ubuntu)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# macOS
brew install azure-cli

# Windows
winget install -e --id Microsoft.AzureCLI

az version
```

### 2.3 Sign in and set context

```bash
az login                                   # opens browser
az account list -o table                   # list subscriptions
az account set --subscription "<name|id>"  # choose active subscription
az account show                            # current context
```

### 2.4 Create a resource group and a resource

```bash
az group create --name myRG --location eastus

az storage account create \
  --name mystorage$RANDOM \
  --resource-group myRG \
  --location eastus \
  --sku Standard_LRS

az resource list --resource-group myRG -o table
```

**Expected:** The resource group and storage account are created and listed.

### 2.5 Azure Cloud Shell (no install)

The **Azure Cloud Shell** (browser, in the Portal) provides a preauthenticated Bash/PowerShell environment with `az`, `kubectl`, Terraform, and more — ideal for quick tasks.

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 Compute: Virtual Machines

```bash
az vm create \
  --resource-group myRG \
  --name myVM \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys

az vm list-ip-addresses -g myRG -n myVM -o table
```

VMs (IaaS) give full OS control. Key options: **size** (B/D/E/F series), **image**, **disks** (managed disks), **VM Scale Sets** (autoscaling identical VMs).

#### 3.1.2 Compute options overview

| Service | Type | Use |
|---------|------|-----|
| **Virtual Machines** | IaaS | Full control, lift-and-shift |
| **VM Scale Sets** | IaaS | Autoscaling VM fleets |
| **App Service** | PaaS | Host web apps/APIs without managing servers |
| **Azure Functions** | Serverless | Event-driven code, pay-per-execution |
| **Azure Container Instances (ACI)** | Containers | Run a container without orchestration |
| **Azure Kubernetes Service (AKS)** | Containers | Managed Kubernetes |

#### 3.1.3 Storage accounts

Azure **Storage Account** services:
- **Blob Storage** — object storage (files, images, backups); tiers: Hot/Cool/Cold/Archive.
- **Files** — managed SMB/NFS file shares.
- **Queues** — simple message queuing.
- **Tables** — NoSQL key-value store.

```bash
az storage container create --account-name <acct> --name data --auth-mode login
az storage blob upload --account-name <acct> -c data -f ./file.txt -n file.txt --auth-mode login
```

#### 3.1.4 Networking basics

- **Virtual Network (VNet):** Your private network in Azure.
- **Subnet:** A segment of a VNet.
- **Network Security Group (NSG):** Firewall rules controlling traffic to subnets/NICs.
- **Public IP / Load Balancer / Application Gateway:** Inbound connectivity and load distribution.

```bash
az network vnet create -g myRG -n myVNet --address-prefix 10.0.0.0/16 \
  --subnet-name web --subnet-prefix 10.0.1.0/24
```

#### 3.1.5 Identity and access (RBAC)

```bash
az role assignment create \
  --assignee user@example.com \
  --role "Contributor" \
  --resource-group myRG
```

**Azure RBAC** grants access via **role assignments** = *security principal* (user/group/service principal/managed identity) + *role definition* (set of permissions) + *scope* (management group/subscription/RG/resource). Built-in roles: **Owner**, **Contributor**, **Reader**, plus many service-specific roles.

### 3.2 INTERMEDIATE

#### 3.2.1 App Service (PaaS web hosting)

```bash
az appservice plan create -g myRG -n myPlan --sku B1 --is-linux
az webapp create -g myRG -p myPlan -n myapp$RANDOM --runtime "NODE:20-lts"
az webapp deployment source config-zip -g myRG -n <app> --src app.zip
```

App Service hosts web apps/APIs with built-in scaling, deployment slots, custom domains, TLS, and CI/CD integration — no server management.

#### 3.2.2 Azure Functions (serverless)

Event-driven functions triggered by HTTP, timers, queues, blobs, Event Grid, etc. Billed per execution (Consumption plan) — scale to zero when idle.

```bash
func init MyFuncApp --python
func new --name HttpExample --template "HTTP trigger"
func azure functionapp publish <appname>
```

#### 3.2.3 Azure Kubernetes Service (AKS)

```bash
az aks create -g myRG -n myAKS --node-count 2 --enable-managed-identity --generate-ssh-keys
az aks get-credentials -g myRG -n myAKS
kubectl get nodes
```

AKS is **managed Kubernetes** — Azure runs the control plane (free); you manage/pay for worker nodes. Integrates with Entra ID, Azure CNI, ACR, and the cluster autoscaler.

#### 3.2.4 Databases

| Service | Type |
|---------|------|
| **Azure SQL Database** | Managed SQL Server (PaaS) |
| **Azure Database for PostgreSQL/MySQL** | Managed open-source DBs |
| **Cosmos DB** | Globally distributed multi-model NoSQL |
| **Azure Cache for Redis** | Managed in-memory cache |

```bash
az sql server create -g myRG -n mysqlsrv$RANDOM -l eastus -u admin -p '<pw>'
az sql db create -g myRG -s <server> -n mydb --service-objective S0
```

#### 3.2.5 Storage tiers and redundancy

**Redundancy options:**
- **LRS** (Locally Redundant) — 3 copies in one datacenter.
- **ZRS** (Zone Redundant) — across availability zones.
- **GRS** (Geo Redundant) — replicated to the paired region.
- **GZRS** — zone + geo redundancy.

**Blob access tiers:** Hot (frequent), Cool (infrequent), Cold, Archive (rare, lowest cost). Use **lifecycle policies** to auto-tier.

#### 3.2.6 Managed identities

```bash
az vm identity assign -g myRG -n myVM      # system-assigned managed identity
```

**Managed identities** give Azure resources an automatically-managed Entra identity to access other Azure services **without storing credentials** — the recommended way for app-to-service auth (e.g., a VM/App Service reading from Key Vault).

#### 3.2.7 Azure Key Vault

```bash
az keyvault create -g myRG -n myvault$RANDOM -l eastus
az keyvault secret set --vault-name <vault> --name dbpass --value 's3cr3t'
az keyvault secret show --vault-name <vault> --name dbpass
```

Key Vault stores **secrets, keys, and certificates** with access via RBAC/policies; combine with **managed identities** so apps fetch secrets with no stored credentials.

### 3.3 ADVANCED

#### 3.3.1 Infrastructure as Code: ARM templates and Bicep

**Bicep** is a domain-specific language that compiles to ARM JSON — the recommended native IaC:

```bicep
// main.bicep
param location string = resourceGroup().location

resource sa 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'stg${uniqueString(resourceGroup().id)}'
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}
output storageId string = sa.id
```

```bash
az deployment group create -g myRG --template-file main.bicep
```

Terraform is also widely used with the AzureRM provider.

#### 3.3.2 Governance: Policy, Blueprints, Management Groups

- **Azure Policy:** Enforce rules (e.g., "only allow certain regions," "require tags," "deny public IPs") with audit/deny effects.
- **Management Groups:** Apply policy/RBAC across many subscriptions.
- **Resource Locks:** Prevent accidental delete/modify (`CanNotDelete`, `ReadOnly`).
- **Tags:** Metadata for cost allocation and organization.

```bash
az policy assignment create --name require-tag \
  --policy <policy-def-id> --scope /subscriptions/<id>
```

#### 3.3.3 Networking deep dive

- **VNet Peering** connects VNets (same or cross-region).
- **VPN Gateway / ExpressRoute** for hybrid connectivity (ExpressRoute = private, dedicated).
- **Azure Firewall / NSG / Application Gateway (WAF)** for security and L7 routing.
- **Private Endpoints / Private Link** to reach PaaS services privately.
- **Load Balancer (L4)** vs **Application Gateway (L7)** vs **Front Door / Traffic Manager** (global).

#### 3.3.4 Monitoring: Azure Monitor

```
   Resources ──► Azure Monitor
                   ├── Metrics (numeric, near real-time)
                   ├── Logs → Log Analytics workspace (KQL queries)
                   ├── Alerts (metric/log/activity) → Action Groups
                   └── Application Insights (APM for apps)
```

Query logs with **KQL** (Kusto Query Language); visualize with **Workbooks** and dashboards; alert via **Action Groups** (email/SMS/webhook/Logic App).

#### 3.3.5 Identity at scale

- **Service principals** for app/CI authentication (`az ad sp create-for-rbac`).
- **Managed identities** (system- and user-assigned) — preferred, credential-free.
- **Conditional Access**, MFA, **PIM** (Privileged Identity Management) for just-in-time admin access.
- **Entra ID groups** mapped to RBAC roles.

#### 3.3.6 Cost management

- **Azure Cost Management + Billing:** analyze spend, budgets, alerts.
- **Pricing levers:** Reserved Instances / Savings Plans (1–3 yr commitments), Spot VMs, autoscaling, right-sizing, Hybrid Benefit (reuse Windows/SQL licenses).
- **Tags** for cost allocation; budgets with alerts to avoid surprises.

#### 3.3.7 High availability and disaster recovery

- **Availability Zones** for intra-region HA; **Availability Sets** for fault/update domains.
- **Region pairs** + **geo-redundant storage** for DR.
- **Azure Site Recovery** for VM replication/failover.
- **Backups** via Azure Backup / Recovery Services Vault.
- Design with the **Azure Well-Architected Framework** (reliability, security, cost, operational excellence, performance).

#### 3.3.8 DevOps on Azure

- **Azure DevOps** (Repos, Pipelines, Boards, Artifacts) and/or **GitHub** (now Microsoft-owned).
- **Azure Container Registry (ACR)** for images; integrate with AKS.
- **Deployment slots**, blue-green, and IaC pipelines (Bicep/Terraform).
- **Microsoft Defender for Cloud** for security posture and recommendations.

---

## 4. Hands-on Tasks

### Task 1: Set up CLI and create a resource group

```bash
az login
az group create -n myRG -l eastus
```

**Expected:** Resource group `myRG` is created.

### Task 2: Deploy a virtual machine

Create an Ubuntu VM (section 3.1.1) and retrieve its public IP.

**Expected:** VM provisions; you can SSH in with the generated key.

### Task 3: Create a storage account and upload a blob

Create a storage account, a container, and upload a file.

**Expected:** The blob appears in the container.

### Task 4: Host a web app on App Service

Create a plan + web app and deploy a zip.

**Expected:** The app is reachable at `https://<app>.azurewebsites.net`.

### Task 5: Assign an RBAC role

Grant a user/group the **Reader** role on `myRG`.

**Expected:** The principal can view but not modify resources.

### Task 6: Store and read a Key Vault secret

Create a Key Vault, set a secret, and read it back.

**Expected:** The secret value is returned (subject to access).

### Task 7: Enable a managed identity

Assign a system-assigned managed identity to a VM/App Service and grant it Key Vault access.

**Expected:** The resource reads the secret with no stored credentials.

### Task 8: Deploy with Bicep

Author `main.bicep` (storage account) and deploy with `az deployment group create`.

**Expected:** The resource is created declaratively; output shows its ID.

### Task 9: Create an AKS cluster

Create a 2-node AKS cluster and run `kubectl get nodes`.

**Expected:** Two nodes report `Ready`.

### Task 10: Set a budget alert

In Cost Management, create a budget with an alert threshold.

**Expected:** An alert is configured to notify on spend.

---

## 5. Projects

### Project 1 (Beginner): Host a Static/Dynamic Website

**Goal:** Deploy a website with proper structure and security basics.

Steps:
1. Create a resource group and (for static) a storage account with static website hosting, or an **App Service** for dynamic.
2. Configure a custom domain and **TLS**.
3. Restrict access with **NSG**/App Service settings.
4. Add **Azure Monitor**/Application Insights.
5. Tag resources for cost tracking.

**Skills:** RGs, storage/App Service, networking basics, monitoring, tags.

### Project 2 (Intermediate): Three-Tier App with Managed Services

**Goal:** Web + API + database with secrets and identity done right.

Steps:
1. Host the API on **App Service**/Functions; web on App Service or Static Web Apps.
2. Use **Azure SQL** or **PostgreSQL** (PaaS) for data.
3. Store credentials in **Key Vault**; access via **managed identity** (no secrets in code).
4. Put the app behind **Application Gateway (WAF)** or **Front Door**.
5. Add **autoscaling**, **deployment slots**, and **Azure Monitor** alerts.
6. Provision everything with **Bicep**.

**Skills:** PaaS compute/DB, Key Vault + managed identity, L7 networking, IaC, monitoring.

### Project 3 (Advanced): Production AKS Platform with Governance

**Goal:** A secure, scalable, well-governed container platform.

Steps:
1. Provision **AKS** (with **ACR**, Azure CNI, autoscaler) via **Bicep/Terraform** in CI/CD.
2. Use **managed identities** + **Workload Identity** for pod-to-Azure auth; secrets from **Key Vault** (CSI driver).
3. Enforce **Azure Policy** and **Management Groups** for governance; **resource locks** on critical resources.
4. Set up **Azure Monitor for containers**, Log Analytics (KQL), and alerts.
5. Implement HA across **Availability Zones**, **geo-redundant storage**, and **Azure Backup**/Site Recovery for DR.
6. Harden with **Defender for Cloud**, **Private Endpoints**, and least-privilege RBAC.
7. Optimize cost with **Reserved Instances/Savings Plans**, Spot node pools, and budgets.

**Skills:** AKS, IaC pipelines, workload identity, governance/policy, observability, HA/DR, security, cost optimization.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Use resource groups** to group resources by lifecycle; tag everything for cost/ownership.
- **Apply least-privilege RBAC** at the narrowest scope; prefer groups over individual assignments.
- **Use managed identities** instead of storing credentials/secrets in code.
- **Store secrets in Key Vault**, not in config or source control.
- **Adopt IaC (Bicep/Terraform)** for repeatable, reviewable deployments.
- **Enforce governance** with Azure Policy and Management Groups.
- **Design for HA** with Availability Zones; plan DR with region pairs/backups.
- **Right-size and commit** (Reserved Instances/Savings Plans, Spot) to control cost.
- **Enable Azure Monitor + Defender for Cloud** for observability and security posture.
- **Use Private Endpoints** to keep PaaS traffic off the public internet.
- **Set budgets and cost alerts.**

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Secrets in code/config | Credential leak | Key Vault + managed identity |
| Overly broad RBAC (Owner) | Excessive blast radius | Least privilege, narrow scope |
| No tagging | Untraceable cost/ownership | Enforce tags via Policy |
| Single zone deployment | Outage on zone failure | Spread across Availability Zones |
| No IaC (click-ops) | Drift, irreproducible | Bicep/Terraform |
| Ignoring region pairs/DR | Data loss on region outage | GRS + DR plan/Site Recovery |
| Public PaaS endpoints | Larger attack surface | Private Endpoints/Link |
| No budgets/alerts | Surprise bills | Cost Management budgets |
| Leaving idle resources running | Wasted spend | Autoscale, deallocate, lifecycle policies |
| No resource locks on critical infra | Accidental deletion | `CanNotDelete` locks |

---

## 7. Interview Questions

### Beginner

1. **What is Microsoft Azure?**
   *A cloud platform offering 200+ services for compute, storage, networking, databases, AI, and more across global datacenters.*

2. **Explain the Azure resource hierarchy.**
   *Management Groups → Subscriptions → Resource Groups → Resources, providing nested scopes for billing, access, and governance.*

3. **What is a resource group?**
   *A logical container for related resources that typically share a lifecycle (deployed/deleted together).*

4. **What is the difference between IaaS and PaaS on Azure?**
   *IaaS (e.g., VMs) gives OS-level control; PaaS (e.g., App Service) manages the OS/runtime so you focus on the app.*

5. **What is Microsoft Entra ID?**
   *Azure's cloud identity service (formerly Azure AD) for authentication, users/groups, service principals, and managed identities.*

### Intermediate

6. **How does Azure RBAC work?**
   *Access is granted via role assignments combining a security principal, a role definition (permissions), and a scope (MG/subscription/RG/resource).*

7. **What are managed identities and why use them?**
   *Automatically managed Entra identities for Azure resources, letting them authenticate to services like Key Vault without storing credentials.*

8. **Compare LRS, ZRS, and GRS storage redundancy.**
   *LRS keeps 3 copies in one datacenter; ZRS spreads copies across availability zones; GRS replicates to the paired region for geo-redundancy.*

9. **What is AKS and what does Azure manage?**
   *Azure Kubernetes Service — managed Kubernetes where Azure runs the control plane (free) and you manage/pay for the worker nodes.*

10. **What are Availability Zones and region pairs?**
    *AZs are physically isolated datacenters within a region for intra-region HA; region pairs are linked regions for DR and data residency.*

### Advanced

11. **What is Bicep and how does it relate to ARM?**
    *Bicep is a concise DSL that compiles to ARM JSON templates — the recommended native IaC for declarative, repeatable Azure deployments.*

12. **How do you enforce governance across many subscriptions?**
    *Use Management Groups to apply Azure Policy (audit/deny rules), RBAC, and tagging consistently, plus resource locks on critical resources.*

13. **How would you secure PaaS service connectivity?**
    *Use Private Endpoints/Private Link to access PaaS over the private VNet, disable public access, and restrict with NSGs/firewall.*

14. **What strategies reduce Azure costs?**
    *Reserved Instances/Savings Plans for steady workloads, Spot VMs for interruptible jobs, autoscaling, right-sizing, Hybrid Benefit, lifecycle storage tiering, and budgets/alerts.*

15. **How do you design for high availability and disaster recovery in Azure?**
    *Spread across Availability Zones, use geo-redundant storage and region pairs, configure backups and Azure Site Recovery, and follow the Well-Architected Framework's reliability guidance.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** A logical container for related resources is a:
- A) Subscription  B) Resource Group  C) Region  D) Tenant

**Q2.** Which provides managed Kubernetes?
- A) ACI  B) AKS  C) App Service  D) Functions

**Q3.** Azure's identity service is now called:
- A) Azure AD only  B) Microsoft Entra ID  C) IAM  D) Directory Services

**Q4.** Which gives a resource an identity without stored credentials?
- A) Service principal secret  B) Managed identity  C) SAS token  D) Access key

**Q5.** Which storage redundancy replicates to the paired region?
- A) LRS  B) ZRS  C) GRS  D) None

**Q6.** Which is the recommended native IaC language?
- A) YAML  B) Bicep  C) JSON only  D) HCL

**Q7.** Which enforces org-wide rules like required tags?
- A) RBAC  B) Azure Policy  C) NSG  D) Key Vault

**Q8.** Physically isolated datacenters within a region are:
- A) Region pairs  B) Availability Zones  C) Edge sites  D) Subnets

**Q9.** Where should application secrets be stored?
- A) App settings in code  B) Key Vault  C) Blob storage  D) A VM disk

**Q10.** Which service provides metrics, logs, and alerts?
- A) Azure Monitor  B) Azure Policy  C) Entra ID  D) ARM

### Short Answer

**S1.** Name the top level of the Azure management hierarchy.

**S2.** Which CLI command signs you in?

**S3.** What query language is used in Log Analytics?

**S4.** Name the L7 load balancer with a web application firewall.

**S5.** Which purchase option offers discounts for 1–3 year commitments?

### Answer Key

**Multiple Choice:** Q1-B, Q2-B, Q3-B, Q4-B, Q5-C, Q6-B, Q7-B, Q8-B, Q9-B, Q10-A

**Short Answer:**
- **S1.** Management Groups.
- **S2.** `az login`.
- **S3.** KQL (Kusto Query Language).
- **S4.** Application Gateway (with WAF).
- **S5.** Reserved Instances / Savings Plans.

---

## 9. Further Resources

### Official Documentation
- Azure docs — https://learn.microsoft.com/azure/
- Azure CLI reference — https://learn.microsoft.com/cli/azure/
- Bicep — https://learn.microsoft.com/azure/azure-resource-manager/bicep/
- Well-Architected Framework — https://learn.microsoft.com/azure/well-architected/

### Learning
- Microsoft Learn — https://learn.microsoft.com/training/azure/
- Azure Architecture Center — https://learn.microsoft.com/azure/architecture/
- Azure free account — https://azure.microsoft.com/free/

### Tools
- Azure CLI, Azure PowerShell, Cloud Shell
- Terraform AzureRM provider, Bicep
- Azure DevOps, GitHub Actions, Defender for Cloud

### Books
- *Learning Microsoft Azure* — Jonah Andersson
- *Azure for Architects* — Ritesh Modi
- *Exam Ref AZ-104 / AZ-204 / AZ-305* (Microsoft Press)

### Certifications
- AZ-900 (Azure Fundamentals)
- AZ-104 (Administrator), AZ-204 (Developer)
- AZ-400 (DevOps Engineer), AZ-305 (Solutions Architect)

---

*End of Microsoft Azure — Zero to Hero.*
