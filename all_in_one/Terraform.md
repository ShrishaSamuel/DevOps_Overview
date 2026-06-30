# Terraform — Zero to Hero

> A complete, beginner-to-advanced guide to Terraform and Infrastructure as Code (IaC).

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

### 1.1 What is Terraform?

**Terraform** is an open-source **Infrastructure as Code (IaC)** tool created by HashiCorp (2014). It lets you define, provision, and manage infrastructure — servers, networks, databases, DNS, Kubernetes clusters, and 1000s of other resources — using a declarative configuration language called **HCL (HashiCorp Configuration Language)**. You describe *what* you want, and Terraform figures out *how* to create, update, or destroy it.

> Note: Following HashiCorp's 2023 license change to the BSL, an open-source fork called **OpenTofu** (under the Linux Foundation) exists and is largely compatible with Terraform configurations.

### 1.2 What is Infrastructure as Code?

IaC means managing infrastructure through machine-readable definition files rather than manual processes (clicking in a cloud console). Benefits:

| Benefit | Explanation |
|---------|-------------|
| **Reproducibility** | Spin up identical environments (dev/staging/prod) |
| **Version control** | Infrastructure changes tracked in Git, reviewed via PRs |
| **Automation** | Provision via pipelines, no manual clicking |
| **Documentation** | The code *is* the documentation of your infra |
| **Consistency** | Eliminates configuration drift and human error |
| **Disaster recovery** | Rebuild everything from code |

### 1.3 Declarative vs. imperative

- **Imperative** (e.g., shell scripts): you specify *each step* ("create VM, then attach disk, then...").
- **Declarative** (Terraform): you specify the *desired end state*; Terraform computes the steps to reach it.

Terraform is declarative — you describe the target infrastructure, and it determines the minimal set of changes (create/update/delete) needed.

### 1.4 How Terraform works: the core loop

```
   Configuration (.tf)        State (terraform.tfstate)        Real Infrastructure
        │ desired                  │ last-known                      │ actual
        ▼                          ▼                                 ▼
   +--------------------------------------------------------------------+
   |  terraform plan: compare desired vs state vs real → show diff      |
   +--------------------------------------------------------------------+
        │ approve
        ▼
   +--------------------------------------------------------------------+
   |  terraform apply: call provider APIs to make real == desired       |
   +--------------------------------------------------------------------+
```

1. You write **configuration** describing desired resources.
2. **`terraform plan`** compares the config with the **state** and real infrastructure, producing an execution plan (what will be added/changed/destroyed).
3. **`terraform apply`** executes the plan via **provider** APIs (AWS, Azure, GCP, etc.).
4. The **state file** records what Terraform manages, mapping config to real resource IDs.

### 1.5 Providers and the ecosystem

**Providers** are plugins that let Terraform interact with a platform's API. There are 3,000+ providers in the **Terraform Registry**:
- Cloud: AWS, Azure, Google Cloud, Oracle Cloud, Alibaba.
- Platforms: Kubernetes, Helm, Docker.
- SaaS/Tools: GitHub, Datadog, Cloudflare, PostgreSQL, Vault.

### 1.6 State: the heart of Terraform

The **state file** (`terraform.tfstate`) is Terraform's record of the resources it manages and their current attributes. It:
- Maps your configuration to real-world resources.
- Tracks metadata and dependencies.
- Improves performance (caches attributes).

State is **critical and sensitive** (can contain secrets). In teams it must be stored **remotely** (e.g., S3 + DynamoDB lock, Terraform Cloud) with locking to prevent concurrent corruption.

### 1.7 When to use Terraform

- Provisioning cloud infrastructure across one or many providers.
- Standardizing environments and enabling GitOps for infra.
- Managing complex, interdependent resources declaratively.

When **not** ideal: configuring software *inside* servers (use Ansible/Chef/Puppet — config management), or one-off throwaway resources where IaC overhead isn't justified. Terraform provisions infrastructure; configuration management tools handle in-OS configuration (they're complementary).

---

## 2. Installation & Setup

### 2.1 Install Terraform (Linux)

```bash
# HashiCorp APT repo (Ubuntu/Debian)
wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform

# Or download a binary directly
# https://developer.hashicorp.com/terraform/install

terraform -version
```

```bash
# macOS
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Enable tab completion
terraform -install-autocomplete
```

### 2.2 Configure cloud credentials (AWS example)

```bash
# Option 1: AWS CLI configuration
aws configure          # sets ~/.aws/credentials

# Option 2: Environment variables
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_DEFAULT_REGION="us-east-1"
```

> Best practice: use IAM roles / short-lived credentials / OIDC in CI rather than long-lived keys.

### 2.3 Your first configuration

```hcl
# main.tf
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "demo" {
  bucket = "my-unique-demo-bucket-12345"
  tags = {
    Environment = "learning"
  }
}
```

```bash
terraform init       # download providers, initialize backend
terraform plan       # preview changes
terraform apply      # create the bucket (type 'yes')
terraform destroy    # tear it down
```

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 The core workflow

```bash
terraform init       # initialize: download providers/modules, set up backend
terraform validate   # check syntax/config validity
terraform fmt        # format code to canonical style
terraform plan       # show what will change (dry run)
terraform apply      # apply changes
terraform destroy    # destroy managed infrastructure
terraform show       # inspect current state
terraform output     # show output values
```

#### 3.1.2 HCL syntax basics

```hcl
# A block: <type> <labels...> { arguments }
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"   # argument
  instance_type = "t3.micro"

  tags = {                                   # nested block / map
    Name = "web-server"
  }
}

# Comments: # or // single line, /* */ multi-line
```

#### 3.1.3 Resources

A **resource** is the most important element — it describes one infrastructure object.

```hcl
resource "aws_instance" "app" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.micro"
}
# Reference its attributes elsewhere as: aws_instance.app.id
```

Syntax: `resource "<PROVIDER>_<TYPE>" "<LOCAL_NAME>" { ... }`.

#### 3.1.4 Variables

Parameterize configurations with **input variables**.

```hcl
# variables.tf
variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "instance_count" {
  type    = number
  default = 2
}

variable "tags" {
  type = map(string)
  default = {
    Project = "demo"
  }
}
```

Provide values via:

```bash
terraform apply -var="region=eu-west-1"
terraform apply -var-file="prod.tfvars"
export TF_VAR_region="us-west-2"      # environment variable
```

```hcl
# terraform.tfvars (auto-loaded)
region         = "us-east-1"
instance_count = 3
```

#### 3.1.5 Outputs

Expose values after apply (and for other modules to consume).

```hcl
# outputs.tf
output "instance_ip" {
  description = "Public IP of the instance"
  value       = aws_instance.app.public_ip
}

output "bucket_arn" {
  value     = aws_s3_bucket.demo.arn
  sensitive = false
}
```

```bash
terraform output
terraform output instance_ip
terraform output -json
```

### 3.2 INTERMEDIATE

#### 3.2.1 State management

```bash
terraform state list                       # list managed resources
terraform state show aws_instance.app      # show one resource
terraform state mv old.addr new.addr       # rename/move in state
terraform state rm aws_instance.app        # stop managing (don't destroy)
terraform import aws_instance.app i-12345   # bring existing infra under management
terraform refresh                          # sync state with real infra (deprecated; use -refresh-only)
terraform apply -refresh-only
```

#### 3.2.2 Remote state and locking

Storing state remotely enables team collaboration and locking.

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tf-locks"     # state locking
    encrypt        = true
  }
}
```

> The S3 bucket + DynamoDB table must exist before `terraform init`. Locking prevents two people from applying simultaneously and corrupting state.

#### 3.2.3 Data sources

**Data sources** read information about existing resources (not managed by this config).

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]   # Canonical
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id   # use the looked-up AMI
  instance_type = "t3.micro"
}
```

#### 3.2.4 Resource dependencies

Terraform builds a **dependency graph** automatically from references:

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id          # implicit dependency
  cidr_block = "10.0.1.0/24"
}

resource "aws_instance" "web" {
  # ...
  depends_on = [aws_subnet.public]      # explicit dependency (rarely needed)
}
```

#### 3.2.5 Meta-arguments: count and for_each

```hcl
# count: create N copies
resource "aws_instance" "web" {
  count         = 3
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  tags = { Name = "web-${count.index}" }
}

# for_each: iterate a set/map (more stable than count for keyed resources)
resource "aws_iam_user" "team" {
  for_each = toset(["alice", "bob", "carol"])
  name     = each.key
}

resource "aws_instance" "app" {
  for_each      = var.instances   # map of name => config
  ami           = each.value.ami
  instance_type = each.value.type
  tags          = { Name = each.key }
}
```

> Prefer `for_each` over `count` when resources have stable identities, because removing an item from the middle of a `count` list re-indexes and recreates resources.

#### 3.2.6 Expressions, functions, and conditionals

```hcl
locals {
  name_prefix = "${var.project}-${var.env}"
  is_prod     = var.env == "prod"
}

resource "aws_instance" "web" {
  instance_type = local.is_prod ? "t3.large" : "t3.micro"   # conditional
  tags = merge(var.common_tags, { Name = local.name_prefix })  # functions
}

# Common functions:
# join, split, length, lookup, merge, concat, format, cidrsubnet,
# file, templatefile, jsonencode, toset, tolist, coalesce, try
```

#### 3.2.7 Provisioners (use sparingly)

```hcl
resource "aws_instance" "web" {
  # ...
  provisioner "remote-exec" {
    inline = ["sudo apt update", "sudo apt install -y nginx"]
    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip
    }
  }
}
```

> Provisioners are a **last resort**. Prefer cloud-init/user_data or configuration management (Ansible). Provisioners aren't tracked in state and break idempotency.

### 3.3 ADVANCED

#### 3.3.1 Modules

**Modules** are reusable, encapsulated groups of resources. Every Terraform configuration is technically a module (the "root module"); you compose child modules.

```
modules/
└── vpc/
    ├── main.tf        # resources
    ├── variables.tf   # inputs
    └── outputs.tf     # outputs
```

```hcl
# Using a local module
module "network" {
  source     = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
  env        = var.env
}

# Using a registry module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  name    = "my-vpc"
  cidr    = "10.0.0.0/16"
  azs     = ["us-east-1a", "us-east-1b"]
}

# Reference module outputs
resource "aws_instance" "app" {
  subnet_id = module.network.public_subnet_id
}
```

Module best practices: keep them focused, version them, expose clear inputs/outputs, document, and pin versions.

#### 3.3.2 Workspaces

Workspaces allow multiple state instances from one configuration (e.g., per environment).

```bash
terraform workspace new dev
terraform workspace new prod
terraform workspace list
terraform workspace select prod
terraform workspace show
```

```hcl
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t3.micro"
}
```

> Workspaces are good for lightweight separation but many teams prefer separate directories/backends per environment for stronger isolation.

#### 3.3.3 Provisioning a realistic stack

```hcl
# Dynamic blocks generate repeated nested blocks
resource "aws_security_group" "web" {
  name = "web-sg"
  dynamic "ingress" {
    for_each = var.allowed_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

#### 3.3.4 Managing secrets

- Never commit secrets or state to Git.
- Mark sensitive variables/outputs with `sensitive = true`.
- Integrate **Vault**, AWS Secrets Manager, or SSM Parameter Store via data sources.
- Encrypt remote state; restrict access with IAM.

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db/password"
}
```

#### 3.3.5 Testing, validation, and policy

```bash
terraform validate              # config validity
terraform fmt -check -recursive # formatting check (CI gate)
tflint                          # linter for best practices/provider rules
tfsec  / trivy config .         # security scanning of IaC
checkov -d .                    # policy-as-code scanning
terraform test                  # native test framework (1.6+)
```

Policy-as-code (Sentinel/OPA) can enforce guardrails (e.g., "no public S3 buckets") before apply.

#### 3.3.6 CI/CD for Terraform

A typical pipeline:

```
fmt/validate → tflint/tfsec → terraform plan (post as PR comment) → manual approval → terraform apply
```

```yaml
# Example pipeline stages (pseudo)
- terraform fmt -check -recursive
- terraform init -backend-config=...
- terraform validate
- tflint && tfsec .
- terraform plan -out=tfplan
- # approval gate
- terraform apply tfplan
```

Use remote state with locking, least-privilege CI credentials (OIDC), and store plan artifacts for auditable applies.

#### 3.3.7 Lifecycle meta-arguments

```hcl
resource "aws_instance" "web" {
  # ...
  lifecycle {
    create_before_destroy = true   # avoid downtime on replacement
    prevent_destroy       = true   # guard critical resources
    ignore_changes        = [tags] # ignore drift on certain attributes
  }
}
```

#### 3.3.8 Targeting, importing, and troubleshooting

```bash
terraform plan  -target=aws_instance.web      # operate on a subset (use sparingly)
terraform apply -replace=aws_instance.web     # force recreate (replaces taint)
terraform import aws_instance.web i-0abc123    # adopt existing resource
TF_LOG=DEBUG terraform apply                   # verbose logs for debugging
terraform graph | dot -Tpng > graph.png        # visualize dependency graph
```

---

## 4. Hands-on Tasks

### Task 1: Initialize and create your first resource

Use the `main.tf` from section 2.3 (an S3 bucket):

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply -auto-approve
terraform show
```

**Expected:** Plan shows `1 to add`; apply creates the bucket; `show` lists it.

### Task 2: Add variables and outputs

```hcl
variable "bucket_name" { type = string }
resource "aws_s3_bucket" "b" { bucket = var.bucket_name }
output "bucket_id" { value = aws_s3_bucket.b.id }
```

```bash
terraform apply -var="bucket_name=my-var-bucket-98765"
terraform output bucket_id
```

**Expected:** Bucket created using the variable; output prints its ID.

### Task 3: Use a data source

Add the `aws_ami` data source from section 3.2.3 and reference it in an EC2 instance.

```bash
terraform plan
```

**Expected:** Plan shows the AMI resolved dynamically (no hard-coded ID).

### Task 4: Create multiple resources with for_each

```hcl
resource "aws_iam_user" "team" {
  for_each = toset(["alice", "bob"])
  name     = each.key
}
```

```bash
terraform apply
terraform state list
```

**Expected:** Two IAM users; state shows `aws_iam_user.team["alice"]` and `["bob"]`.

### Task 5: Inspect and manipulate state

```bash
terraform state list
terraform state show aws_s3_bucket.b
terraform state rm aws_s3_bucket.b     # stop managing (resource remains in cloud)
```

**Expected:** Understand the difference between state and real resources.

### Task 6: Configure remote state (S3 backend)

Create the S3 bucket + DynamoDB table, add the `backend "s3"` block, then:

```bash
terraform init -migrate-state
```

**Expected:** State migrated to S3; locking via DynamoDB.

### Task 7: Build and use a module

Create `modules/bucket/` with a parameterized bucket, then:

```hcl
module "logs" {
  source      = "./modules/bucket"
  bucket_name = "my-logs-bucket-1357"
}
```

```bash
terraform init
terraform apply
```

**Expected:** Module-managed bucket created; reusable across environments.

### Task 8: Use workspaces for environments

```bash
terraform workspace new dev
terraform apply
terraform workspace new prod
terraform apply
terraform workspace list
```

**Expected:** Separate state per workspace; different sizing if conditional logic used.

### Task 9: Lint and security-scan

```bash
terraform fmt -check -recursive
tflint
tfsec .          # or: trivy config .
```

**Expected:** Formatting/lint/security findings reported.

### Task 10: Destroy cleanly

```bash
terraform plan -destroy
terraform destroy -auto-approve
terraform state list      # empty
```

**Expected:** All managed resources removed; state empty.

---

## 5. Projects

### Project 1 (Beginner): Provision a Web Server on AWS

**Goal:** Create a single EC2 instance serving a web page, fully via Terraform.

Steps:
1. Define provider, a security group (allow 22/80), and an EC2 instance.
2. Use a data source for the latest Ubuntu AMI.
3. Use `user_data` to install and start nginx.
4. Output the public IP.
5. `apply`, verify in a browser, then `destroy`.

**Skills:** providers, resources, data sources, variables, outputs, user_data.

### Project 2 (Intermediate): Reusable Multi-Environment VPC + App

**Goal:** A modular network + app deployable to dev/staging/prod.

Steps:
1. Create a **VPC module** (VPC, public/private subnets, IGW, route tables, NAT).
2. Create an **app module** (ASG/instances + security groups + load balancer).
3. Use **remote state** (S3 + DynamoDB lock).
4. Parameterize per environment with `*.tfvars` (or workspaces).
5. Use `for_each`/`count` and conditionals for sizing per env.
6. Output the load balancer DNS name.

**Skills:** modules, remote state, environments, meta-arguments, networking.

### Project 3 (Advanced): Full IaC Platform with CI/CD and Policy

**Goal:** Production-grade Terraform with automation and guardrails.

Steps:
1. Structure the repo: reusable modules + per-environment root configs.
2. Remote state per environment with locking and encryption.
3. **CI/CD pipeline:** fmt/validate → tflint/tfsec/checkov → `plan` (posted to PR) → approval → `apply`.
4. **Policy-as-code** (OPA/Sentinel) to block non-compliant changes (e.g., public buckets, missing tags).
5. Manage secrets via Vault/Secrets Manager; mark sensitive values.
6. Provision a realistic stack: VPC + EKS/Kubernetes + RDS + IAM + monitoring.
7. Use the **Kubernetes/Helm providers** to deploy workloads after the cluster exists.
8. Document drift detection (`plan` on schedule) and rollback procedures.

**Skills:** module composition, CI/CD, policy-as-code, secrets, multi-provider, drift management.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Store state remotely** with locking and encryption (S3+DynamoDB, Terraform Cloud) — never commit state to Git.
- **Use modules** to encapsulate and reuse; **version and pin** modules and providers.
- **Run `fmt`, `validate`, `tflint`, `tfsec`** in CI; gate merges.
- **Always review `terraform plan`** before apply; in CI, post the plan to the PR.
- **Use `for_each` over `count`** for keyed resources to avoid re-indexing churn.
- **Parameterize with variables**; separate config per environment.
- **Mark secrets `sensitive`** and source them from a secrets manager.
- **Use `lifecycle`** rules (`prevent_destroy`, `create_before_destroy`) for safety/uptime.
- **Keep modules small and focused**; document inputs/outputs.
- **Pin `required_version` and provider versions** for reproducibility.
- **Tag resources** consistently for cost/ownership tracking.

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Committing state to Git | Secret leakage, corruption | Remote backend + `.gitignore` |
| No state locking | Concurrent applies corrupt state | DynamoDB / Terraform Cloud locking |
| Editing infra manually (out-of-band) | Drift; plan wants to revert | Manage everything in code; import drift |
| Hard-coding secrets/AMIs | Insecure, brittle | Variables, data sources, secret managers |
| Overusing `count` | Resources recreated on list changes | Use `for_each` for stable keys |
| Skipping `plan` | Surprise destroys | Always plan and review |
| Giant monolithic config | Slow, risky applies | Split into modules/stacks |
| Overusing provisioners | Non-idempotent, fragile | Use user_data / Ansible |
| `terraform destroy` on shared state | Wipes prod | Isolate state per env; `prevent_destroy` |
| Unpinned versions | Unexpected breaking changes | Pin provider/module versions |

---

## 7. Interview Questions

### Beginner

1. **What is Terraform?**
   *An open-source IaC tool to declaratively provision and manage infrastructure across providers using HCL.*

2. **What is the difference between `terraform plan` and `terraform apply`?**
   *`plan` previews the changes (dry run); `apply` executes them against real infrastructure.*

3. **What is a provider?**
   *A plugin that lets Terraform interact with a specific platform's API (AWS, Azure, Kubernetes, etc.).*

4. **What does `terraform init` do?**
   *Initializes the working directory: downloads providers/modules and configures the backend.*

5. **What is the state file?**
   *A file mapping your configuration to real resources, tracking their attributes and dependencies.*

### Intermediate

6. **Why store state remotely?**
   *For team collaboration, locking (preventing concurrent corruption), durability, and security (encryption/access control).*

7. **What is the difference between `count` and `for_each`?**
   *`count` creates N indexed copies (list-based; re-indexes on changes); `for_each` iterates a set/map giving stable keys, avoiding unintended recreation.*

8. **What are data sources?**
   *Read-only lookups of information about existing resources/infra not managed by the current config.*

9. **How does Terraform determine resource order?**
   *It builds a dependency graph from references between resources; explicit `depends_on` can force ordering when needed.*

10. **What is a module?**
    *A reusable, encapsulated set of resources with inputs and outputs; the root configuration is itself a module that can call child modules.*

### Advanced

11. **How do you handle secrets in Terraform?**
    *Mark variables/outputs `sensitive`, never commit state/secrets, encrypt remote state, restrict IAM, and source secrets from Vault/Secrets Manager via data sources.*

12. **Explain drift and how to detect/fix it.**
    *Drift is when real infra diverges from state (manual changes). Detect via `terraform plan` (or scheduled `-refresh-only`); fix by reverting via apply or importing/adjusting config to match.*

13. **What does `create_before_destroy` solve?**
    *It creates the replacement resource before destroying the old one, avoiding downtime when a change forces replacement.*

14. **How do you bring existing infrastructure under Terraform management?**
    *Write matching configuration and use `terraform import` (or `import` blocks in 1.5+) to associate real resources with state, then `plan` to verify no diffs.*

15. **How would you design a multi-environment, team-safe Terraform setup?**
    *Reusable versioned modules, isolated remote state per environment with locking, environment-specific tfvars, CI with fmt/validate/lint/security/plan + approval before apply, least-privilege credentials, and policy-as-code guardrails.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** What language does Terraform use for configuration?
- A) YAML  B) JSON  C) HCL  D) Python

**Q2.** Which command previews changes without applying them?
- A) `terraform init`  B) `terraform plan`  C) `terraform show`  D) `terraform apply`

**Q3.** What maps configuration to real resources?
- A) Provider  B) Module  C) State file  D) Backend

**Q4.** Which is a read-only lookup of existing infrastructure?
- A) resource  B) data source  C) variable  D) output

**Q5.** Which meta-argument is best for resources with stable, named identities?
- A) `count`  B) `for_each`  C) `depends_on`  D) `provider`

**Q6.** Where should production state be stored?
- A) In Git  B) Locally only  C) A remote backend with locking  D) In the module

**Q7.** What does `terraform init` download?
- A) State  B) Providers and modules  C) Variables  D) Credentials

**Q8.** Which lifecycle setting guards a resource from deletion?
- A) `create_before_destroy`  B) `ignore_changes`  C) `prevent_destroy`  D) `replace_triggered_by`

**Q9.** A reusable, encapsulated group of resources is a:
- A) Workspace  B) Module  C) Provider  D) Provisioner

**Q10.** Which command adopts an existing resource into state?
- A) `terraform refresh`  B) `terraform import`  C) `terraform state rm`  D) `terraform taint`

### Short Answer

**S1.** Write the command to migrate local state to a newly configured remote backend.

**S2.** How do you mark a variable as sensitive?

**S3.** What is the difference between `terraform state rm` and `terraform destroy` for a single resource?

**S4.** Write a `for_each` block creating IAM users for `["a","b","c"]`.

**S5.** Which command forces the recreation of `aws_instance.web`?

### Answer Key

**Multiple Choice:** Q1-C, Q2-B, Q3-C, Q4-B, Q5-B, Q6-C, Q7-B, Q8-C, Q9-B, Q10-B

**Short Answer:**
- **S1.** `terraform init -migrate-state`
- **S2.** Add `sensitive = true` to the variable block.
- **S3.** `state rm` stops Terraform from managing the resource (it stays in the cloud); `destroy` actually deletes the real resource.
- **S4.** `resource "aws_iam_user" "u" { for_each = toset(["a","b","c"]) name = each.key }`
- **S5.** `terraform apply -replace=aws_instance.web`

---

## 9. Further Resources

### Official Documentation
- Terraform Docs — https://developer.hashicorp.com/terraform/docs
- Terraform Registry (providers & modules) — https://registry.terraform.io
- Language reference (HCL) — https://developer.hashicorp.com/terraform/language
- HashiCorp Learn tutorials — https://developer.hashicorp.com/terraform/tutorials

### Learning
- "Getting Started" tutorials per cloud (AWS/Azure/GCP) on HashiCorp Learn
- Terraform Up & Running examples repo
- OpenTofu docs — https://opentofu.org/docs/ (open-source fork)

### Tools
- tflint — https://github.com/terraform-linters/tflint
- tfsec / Trivy config — IaC security scanning
- Checkov — https://www.checkov.io
- Terragrunt — DRY wrapper for Terraform — https://terragrunt.gruntwork.io
- Infracost — cost estimates from plans — https://www.infracost.io
- pre-commit-terraform hooks

### Books
- *Terraform: Up & Running* — Yevgeniy Brikman
- *Terraform in Action* — Scott Winkler
- *The Terraform Book* — James Turnbull

### Certifications
- HashiCorp Certified: Terraform Associate

---

*End of Terraform — Zero to Hero.*
