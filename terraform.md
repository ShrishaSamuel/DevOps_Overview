# 🚀 Terraform Mastery with AWS — From Zero to Hero

> **Your Personal Terraform Trainer & AWS DevOps Mentor**
> A complete, real-time, hands-on guide. No prior Terraform knowledge required.

---

## 📚 Table of Contents

1. [What is Terraform & Why It Matters](#1-what-is-terraform--why-it-matters)
2. [Infrastructure as Code (IaC) Concepts](#2-infrastructure-as-code-iac-concepts)
3. [Installation & Setup (AWS + Terraform)](#3-installation--setup-aws--terraform)
4. [Terraform Core Workflow & Commands](#4-terraform-core-workflow--commands)
5. [Your First Terraform Project (Hello AWS)](#5-your-first-terraform-project-hello-aws)
6. [Terraform Language Building Blocks](#6-terraform-language-building-blocks)
7. [Variables, Outputs & Data Sources](#7-variables-outputs--data-sources)
8. [Terraform State Deep Dive](#8-terraform-state-deep-dive)
9. [Remote State with S3 + DynamoDB Locking](#9-remote-state-with-s3--dynamodb-locking)
10. [Provisioners, Meta-Arguments & Expressions](#10-provisioners-meta-arguments--expressions)
11. [Modules — Reusable Infrastructure](#11-modules--reusable-infrastructure)
12. [Workspaces & Multi-Environment Strategy](#12-workspaces--multi-environment-strategy)
13. [Real-Time Project: 3-Tier AWS Architecture](#13-real-time-project-3-tier-aws-architecture)
14. [CI/CD Integration & Real Workflow](#14-cicd-integration--real-workflow)
15. [Troubleshooting & Bug Fixing Playbook](#15-troubleshooting--bug-fixing-playbook)
16. [Best Practices (Production-Grade)](#16-best-practices-production-grade)
17. [Assignments (Practice Like a Pro)](#17-assignments-practice-like-a-pro)
18. [Interview Questions & Answers](#18-interview-questions--answers)
19. [Cheat Sheet & Quick Reference](#19-cheat-sheet--quick-reference)

---

## 1. What is Terraform & Why It Matters

**Terraform** is an open-source **Infrastructure as Code (IaC)** tool created by HashiCorp. It lets you define cloud and on-prem resources in human-readable configuration files that you can version, reuse, and share.

### 🤔 The Problem It Solves
Imagine you manually click through the AWS Console to create:
- A VPC
- 4 subnets
- An EC2 instance
- A security group
- A load balancer

Now do that again for **dev**, **staging**, and **production**. Then do it again after a teammate accidentally deletes something. 😱

Terraform solves this: **you write the desired state once, and Terraform makes the cloud match it.**

### Key Benefits
| Benefit | Explanation |
|---------|-------------|
| **Declarative** | You describe *what* you want, not *how* to build it. |
| **Cloud-agnostic** | Works with AWS, Azure, GCP, Kubernetes, and 1000+ providers. |
| **Version controlled** | Infra lives in Git like application code. |
| **Repeatable** | Spin up identical environments in minutes. |
| **Plan before apply** | Preview changes before they happen. |

### Terraform vs Other Tools
- **Terraform vs CloudFormation**: Terraform is multi-cloud; CloudFormation is AWS-only.
- **Terraform vs Ansible**: Terraform is for *provisioning* (creating infra); Ansible is for *configuration management* (installing software). They are often used together.

---

## 2. Infrastructure as Code (IaC) Concepts

### Declarative vs Imperative
- **Imperative** (bash/Ansible): "Run these steps in this order."
- **Declarative** (Terraform): "Here is the final state I want. Figure out the steps."

### Idempotency
Running `terraform apply` multiple times with the same config produces the **same result**. Terraform only changes what's different — it won't recreate resources that already match.

### Desired State vs Actual State
Terraform constantly compares:
- **Desired State** = your `.tf` files
- **Actual State** = what's really in AWS (tracked in the *state file*)
- **Plan** = the diff between them

---

## 3. Installation & Setup (AWS + Terraform)

### Step 1: Install Terraform (Windows)

```powershell
# Using Chocolatey (recommended)
choco install terraform

# Verify installation
terraform version
```

Or download the binary from https://developer.hashicorp.com/terraform/downloads and add it to your PATH.

### Step 2: Install AWS CLI

```powershell
choco install awscli
aws --version
```

### Step 3: Configure AWS Credentials

```powershell
aws configure
```

You'll be prompted for:
```
AWS Access Key ID:     AKIAxxxxxxxxxxxx
AWS Secret Access Key: xxxxxxxxxxxxxxxxxxxx
Default region name:   us-east-1
Default output format: json
```

> 🔐 **Best Practice**: Create an IAM user with programmatic access and least-privilege permissions. **Never** use root account keys. For production, use IAM Roles / SSO instead of long-lived keys.

### Step 4: Verify

```powershell
aws sts get-caller-identity
```

This confirms Terraform can authenticate to AWS using your credentials.

---

## 4. Terraform Core Workflow & Commands

The **core workflow** is: **Write → Init → Plan → Apply → Destroy**

```
┌─────────┐   ┌──────┐   ┌──────┐   ┌───────┐   ┌─────────┐
│  Write  │──▶│ Init │──▶│ Plan │──▶│ Apply │──▶│ Destroy │
└─────────┘   └──────┘   └──────┘   └───────┘   └─────────┘
```

| Command | What It Does |
|---------|--------------|
| `terraform init` | Initializes the working directory, downloads providers & modules. |
| `terraform fmt` | Auto-formats your code to canonical style. |
| `terraform validate` | Checks syntax and internal consistency. |
| `terraform plan` | Shows what Terraform *will* do (dry run). |
| `terraform apply` | Creates/updates infrastructure. |
| `terraform destroy` | Deletes all managed infrastructure. |
| `terraform show` | Displays current state in readable form. |
| `terraform state list` | Lists resources tracked in state. |
| `terraform output` | Shows output values. |
| `terraform refresh` | Syncs state with real-world resources. |

> 💡 Useful flags: `terraform plan -out=tfplan` then `terraform apply tfplan` (apply exactly what you planned), `terraform apply -auto-approve` (skip confirmation — use carefully!), `terraform destroy -target=aws_instance.web` (target a single resource).

---

## 5. Your First Terraform Project (Hello AWS)

Let's create a single EC2 instance. Create a folder and a file `main.tf`:

```hcl
# main.tf

# 1️⃣ Terraform block: configure required providers and versions
terraform {
  required_version = ">= 1.5.0"        # Minimum Terraform CLI version

  required_providers {
    aws = {
      source  = "hashicorp/aws"        # Where to download the provider from
      version = "~> 5.0"               # Allow 5.x but not 6.0 (pessimistic constraint)
    }
  }
}

# 2️⃣ Provider block: tells Terraform WHICH cloud and region
provider "aws" {
  region = "us-east-1"                 # All resources go to N. Virginia
}

# 3️⃣ Resource block: the actual infrastructure to create
resource "aws_instance" "my_first_server" {
  ami           = "ami-0c02fb55956c7d316"   # Amazon Linux 2 AMI (region-specific!)
  instance_type = "t2.micro"                # Free-tier eligible

  tags = {
    Name = "MyFirstTerraformServer"    # A friendly name tag in AWS Console
  }
}
```

### 📖 Line-by-Line Explanation
- **`terraform { ... }`** — Global settings. `required_providers` declares dependencies.
- **`source = "hashicorp/aws"`** — Short for `registry.terraform.io/hashicorp/aws`.
- **`version = "~> 5.0"`** — The `~>` is the "pessimistic operator": allows `5.1`, `5.99` but **not** `6.0`. Prevents breaking changes.
- **`provider "aws"`** — Authentication + region. Picks up credentials from `aws configure`.
- **`resource "aws_instance" "my_first_server"`** — `aws_instance` is the *type*; `my_first_server` is your *local name* (used to reference it elsewhere).
- **`ami`** — The machine image. ⚠️ AMIs differ per region.
- **`instance_type`** — Hardware size.
- **`tags`** — Key-value metadata. Always tag everything!

### Run It

```powershell
terraform init      # Downloads AWS provider
terraform fmt       # Tidy formatting
terraform validate  # Confirm it's valid
terraform plan      # Preview: "1 to add, 0 to change, 0 to destroy"
terraform apply     # Type 'yes' to create the EC2 instance
```

When done experimenting:

```powershell
terraform destroy   # Clean up to avoid charges
```

---

## 6. Terraform Language Building Blocks

HCL (HashiCorp Configuration Language) has these core blocks:

| Block | Purpose |
|-------|---------|
| `terraform` | Settings, backend, required providers. |
| `provider` | Configure a plugin (AWS, Azure...). |
| `resource` | A piece of infrastructure to manage. |
| `data` | Read existing/external infrastructure. |
| `variable` | Input parameters. |
| `output` | Return values. |
| `module` | Group of resources reused together. |
| `locals` | Named local values (like constants). |

### Resource Addressing & References
You reference attributes using `<TYPE>.<NAME>.<ATTRIBUTE>`:

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id      # 👈 reference VPC's id — creates dependency!
  cidr_block = "10.0.1.0/24"
}
```

Terraform automatically builds a **dependency graph** — it knows the VPC must exist before the subnet.

---

## 7. Variables, Outputs & Data Sources

### Input Variables (`variables.tf`)

```hcl
variable "instance_type" {
  description = "EC2 instance size"
  type        = string
  default     = "t2.micro"
}

variable "region" {
  description = "AWS region to deploy into"
  type        = string
  default     = "us-east-1"
}

variable "tags" {
  description = "Common tags for all resources"
  type        = map(string)
  default = {
    Environment = "dev"
    Owner       = "platform-team"
  }
}
```

### Using Variables

```hcl
provider "aws" {
  region = var.region
}

resource "aws_instance" "web" {
  ami           = "ami-0c02fb55956c7d316"
  instance_type = var.instance_type
  tags          = var.tags
}
```

### Ways to Set Variable Values (precedence: lowest → highest)
1. `default` in the variable block
2. Environment variables: `TF_VAR_instance_type=t3.micro`
3. `terraform.tfvars` file (auto-loaded)
4. `*.auto.tfvars` files (auto-loaded)
5. `-var` and `-var-file` on the command line (highest priority)

**`terraform.tfvars`:**
```hcl
instance_type = "t3.small"
region        = "us-west-2"
```

### Outputs (`outputs.tf`)

```hcl
output "instance_public_ip" {
  description = "Public IP of the web server"
  value       = aws_instance.web.public_ip
}

output "instance_id" {
  value = aws_instance.web.id
}
```

```powershell
terraform output                    # show all
terraform output instance_public_ip # show one
```

### Data Sources — Read Existing Resources

```hcl
# Fetch the latest Amazon Linux 2 AMI dynamically — no hardcoded AMI ID!
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id   # 👈 always-latest AMI
  instance_type = var.instance_type
}
```

> 💡 Data sources are **read-only**. Terraform queries AWS but never modifies these resources.

---

## 8. Terraform State Deep Dive

### What is State?
Terraform records everything it manages in a file called **`terraform.tfstate`** (JSON). It maps your config to real AWS resource IDs.

### Why State Matters
- It's how Terraform knows what exists vs. what to create/destroy.
- It stores resource metadata and dependency info.
- It enables `plan` to compute diffs quickly.

### ⚠️ State File Warnings
- **Contains secrets** (passwords, keys) in plaintext → never commit to Git!
- **Single source of truth** → corruption = trouble.
- **Concurrency issues** → two people running apply simultaneously can corrupt state.

### Common State Commands

```powershell
terraform state list                          # List all tracked resources
terraform state show aws_instance.web         # Detailed view of one resource
terraform state mv old_name new_name          # Rename without destroying
terraform state rm aws_instance.web           # Stop tracking (won't delete in AWS)
terraform import aws_instance.web i-1234567890 # Import existing AWS resource into state
```

> 🛡️ Add `*.tfstate*` and `.terraform/` to your `.gitignore`!

---

## 9. Remote State with S3 + DynamoDB Locking

For teams, store state remotely so everyone shares one source of truth, with **locking** to prevent concurrent edits.

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-company-terraform-state"   # S3 bucket (create first!)
    key            = "prod/network/terraform.tfstate" # Path within bucket
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"              # For state locking
    encrypt        = true                            # Encrypt state at rest
  }
}
```

### Bootstrap the Backend (chicken-and-egg: create these first)

```hcl
resource "aws_s3_bucket" "tf_state" {
  bucket = "my-company-terraform-state"
}

resource "aws_s3_bucket_versioning" "tf_state" {
  bucket = aws_s3_bucket.tf_state.id
  versioning_configuration {
    status = "Enabled"   # Keep history of state to recover from mistakes
  }
}

resource "aws_dynamodb_table" "tf_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

```powershell
terraform init   # Terraform asks to migrate local state to S3 — type 'yes'
```

> 🔒 **Locking** means when you run `apply`, DynamoDB holds a lock so no one else can apply at the same time.

---

## 10. Provisioners, Meta-Arguments & Expressions

### Meta-Arguments (work on any resource)

**`count` — create N copies:**
```hcl
resource "aws_instance" "web" {
  count         = 3                          # Creates web[0], web[1], web[2]
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t2.micro"
  tags = {
    Name = "web-${count.index}"              # web-0, web-1, web-2
  }
}
```

**`for_each` — create from a map/set (preferred over count):**
```hcl
resource "aws_instance" "app" {
  for_each      = toset(["api", "worker", "cache"])
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t2.micro"
  tags = {
    Name = "app-${each.key}"                 # app-api, app-worker, app-cache
  }
}
```

> 💡 Use `for_each` over `count` when items can be added/removed — it avoids destroying/recreating unrelated resources.

**`depends_on` — explicit dependency:**
```hcl
resource "aws_instance" "web" {
  # ...
  depends_on = [aws_s3_bucket.assets]   # Force ordering when not auto-detected
}
```

**`lifecycle` — control behavior:**
```hcl
resource "aws_instance" "web" {
  # ...
  lifecycle {
    create_before_destroy = true    # Zero-downtime replacement
    prevent_destroy       = true    # Safety: block accidental deletion
    ignore_changes        = [tags]  # Ignore drift on these attributes
  }
}
```

### Expressions & Functions
```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"   # String interpolation
  azs         = ["us-east-1a", "us-east-1b"]
  bucket_name = lower(replace(var.project, " ", "-"))  # Built-in functions
}

# Conditional (ternary)
instance_type = var.environment == "prod" ? "t3.large" : "t2.micro"
```

### Provisioners (use sparingly — last resort!)
```hcl
resource "aws_instance" "web" {
  # ...
  provisioner "remote-exec" {
    inline = [
      "sudo yum update -y",
      "sudo yum install -y httpd",
      "sudo systemctl start httpd"
    ]
    connection {
      type        = "ssh"
      user        = "ec2-user"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip
    }
  }
}
```
> ⚠️ Prefer `user_data` scripts or tools like Ansible/Packer over provisioners. HashiCorp recommends provisioners as a last resort.

---

## 11. Modules — Reusable Infrastructure

Modules are reusable packages of Terraform code. Think of them as **functions** for infrastructure.

### Module Structure
```
modules/
└── ec2-instance/
    ├── main.tf        # Resources
    ├── variables.tf   # Inputs
    └── outputs.tf     # Outputs
```

**`modules/ec2-instance/variables.tf`:**
```hcl
variable "instance_type" { type = string }
variable "ami_id"        { type = string }
variable "name"          { type = string }
```

**`modules/ec2-instance/main.tf`:**
```hcl
resource "aws_instance" "this" {
  ami           = var.ami_id
  instance_type = var.instance_type
  tags = { Name = var.name }
}
```

**`modules/ec2-instance/outputs.tf`:**
```hcl
output "instance_id" { value = aws_instance.this.id }
output "public_ip"   { value = aws_instance.this.public_ip }
```

### Using the Module (root `main.tf`)
```hcl
module "web_server" {
  source        = "./modules/ec2-instance"   # Local path
  name          = "web-prod"
  ami_id        = data.aws_ami.amazon_linux.id
  instance_type = "t3.micro"
}

output "web_ip" {
  value = module.web_server.public_ip   # Access module output
}
```

### Using Public Registry Modules
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"
  azs             = ["us-east-1a", "us-east-1b"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
}
```

---

## 12. Workspaces & Multi-Environment Strategy

### Terraform Workspaces
```powershell
terraform workspace new dev
terraform workspace new prod
terraform workspace list
terraform workspace select prod
```

Use `terraform.workspace` in code:
```hcl
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "prod" ? "t3.large" : "t2.micro"
  tags = { Environment = terraform.workspace }
}
```

### 🏆 Better Approach: Directory-per-Environment
Workspaces share the same backend config which can be risky. Many teams prefer:
```
environments/
├── dev/
│   ├── main.tf
│   └── terraform.tfvars
├── staging/
│   └── ...
└── prod/
    ├── main.tf
    └── terraform.tfvars
```
This gives **full isolation** of state and config per environment.

---

## 13. Real-Time Project: 3-Tier AWS Architecture

Let's build a production-style **VPC → Public/Private Subnets → ALB → EC2 → Security Groups**.

```
                    Internet
                       │
                  ┌────▼─────┐
                  │   IGW    │
                  └────┬─────┘
              ┌────────▼────────┐
              │  Public Subnets │  ── ALB
              └────────┬────────┘
              ┌────────▼────────┐
              │ Private Subnets │  ── EC2 (web servers)
              └─────────────────┘
```

**`main.tf`:**
```hcl
# ---------- VPC ----------
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = "${var.project}-vpc" }
}

# ---------- Internet Gateway ----------
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.main.id
  tags   = { Name = "${var.project}-igw" }
}

# ---------- Public Subnets (one per AZ) ----------
resource "aws_subnet" "public" {
  count                   = length(var.azs)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.${count.index + 1}.0/24"
  availability_zone       = var.azs[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "${var.project}-public-${count.index}" }
}

# ---------- Route Table for Public Subnets ----------
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }
  tags = { Name = "${var.project}-public-rt" }
}

resource "aws_route_table_association" "public" {
  count          = length(aws_subnet.public)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# ---------- Security Group ----------
resource "aws_security_group" "web" {
  name        = "${var.project}-web-sg"
  description = "Allow HTTP and SSH"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.my_ip]   # Restrict SSH to YOUR IP only!
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = { Name = "${var.project}-web-sg" }
}

# ---------- EC2 Instances ----------
resource "aws_instance" "web" {
  count                  = 2
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.public[count.index].id
  vpc_security_group_ids = [aws_security_group.web.id]

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Hello from $(hostname)</h1>" > /var/www/html/index.html
              EOF

  tags = { Name = "${var.project}-web-${count.index}" }
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

**`variables.tf`:**
```hcl
variable "project"       { default = "demo-app" }
variable "instance_type" { default = "t2.micro" }
variable "azs"           { default = ["us-east-1a", "us-east-1b"] }
variable "my_ip"         { description = "Your public IP in CIDR, e.g. 1.2.3.4/32" }
```

**`outputs.tf`:**
```hcl
output "web_public_ips" {
  value = aws_instance.web[*].public_ip   # Splat: all IPs as a list
}
```

### Deploy
```powershell
terraform init
terraform plan -var="my_ip=$(curl -s ifconfig.me)/32"
terraform apply -var="my_ip=$(curl -s ifconfig.me)/32"
```

---

## 14. CI/CD Integration & Real Workflow

### Real-Time Team Workflow
```
Developer → Git branch → Pull Request
                              │
                    ┌─────────▼──────────┐
                    │  CI Pipeline runs: │
                    │  terraform fmt     │
                    │  terraform validate│
                    │  terraform plan    │  ← posted as PR comment
                    └─────────┬──────────┘
                       Review & Approve
                              │
                    ┌─────────▼──────────┐
                    │  Merge to main     │
                    │  terraform apply   │  ← automated or manual gate
                    └────────────────────┘
```

### Example GitHub Actions Pipeline (`.github/workflows/terraform.yml`)
```yaml
name: Terraform CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.0

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - run: terraform init
      - run: terraform fmt -check
      - run: terraform validate
      - run: terraform plan -no-color

      - name: Apply (only on main)
        if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve
```

> 🔐 Store credentials in GitHub **Secrets** — never in code. Better: use OIDC to assume an IAM role (no long-lived keys).

---

## 15. Troubleshooting & Bug Fixing Playbook

### 🐞 Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Error: No valid credential sources found` | AWS creds not set | Run `aws configure` or set `AWS_PROFILE`. |
| `Error: Provider configuration not present` | Provider removed but still in state | Re-add provider or `terraform state rm`. |
| `Error acquiring the state lock` | Previous run crashed; lock stuck | `terraform force-unlock <LOCK_ID>`. |
| `InvalidAMIID.NotFound` | AMI doesn't exist in that region | Use a `data "aws_ami"` source instead of hardcoding. |
| `Resource already exists` | Resource created outside Terraform | `terraform import` it into state. |
| `Cycle: a → b → a` | Circular dependency | Break the loop; remove unnecessary `depends_on`. |
| `Error: value depends on resource attributes that cannot be determined until apply` | Using computed value in `count`/`for_each` | Use known values or a 2-stage apply. |

### Debugging Techniques
```powershell
# Enable detailed logs
$env:TF_LOG = "DEBUG"        # TRACE, DEBUG, INFO, WARN, ERROR
$env:TF_LOG_PATH = "tf.log"
terraform apply

# Inspect state when confused
terraform state list
terraform state show aws_instance.web

# Detect drift (manual changes in AWS)
terraform plan -refresh-only

# Recover from a stuck lock (get LOCK_ID from the error message)
terraform force-unlock 1234abcd-5678-...
```

### Handling State Drift
If someone changes infra manually in the AWS Console:
```powershell
terraform plan         # Shows the drift
terraform apply        # Reverts AWS to match your code (desired state wins)
# OR update your code to match, then:
terraform apply -refresh-only   # Accept reality into state
```

### Fixing a Bad Apply (Targeted Operations)
```powershell
terraform apply -target=aws_security_group.web   # Fix just one resource
terraform taint aws_instance.web                 # Mark for recreation (older syntax)
terraform apply -replace="aws_instance.web"      # Modern way to force recreation
```

---

## 16. Best Practices (Production-Grade)

### 📁 Code Organization
- Split files: `main.tf`, `variables.tf`, `outputs.tf`, `providers.tf`, `backend.tf`.
- Use **modules** for reusable components.
- One state file per environment/component (blast radius isolation).

### 🔐 Security
- **Never** commit `*.tfstate` or `.tfvars` with secrets. Use a `.gitignore`.
- Store secrets in **AWS Secrets Manager** / **SSM Parameter Store**, not in code.
- Use IAM roles + OIDC instead of static keys in CI/CD.
- Encrypt remote state (`encrypt = true`).
- Restrict security group ingress — avoid `0.0.0.0/0` on SSH.

### 🏗️ State Management
- Always use **remote state** (S3 + DynamoDB) for teams.
- Enable S3 **versioning** to recover from corruption.
- Lock state to prevent concurrent applies.

### ✅ Quality & Workflow
- Run `terraform fmt` and `terraform validate` in CI.
- Always review `terraform plan` before apply.
- Pin provider & module versions (`~>` constraints).
- Use `lifecycle { prevent_destroy = true }` on critical resources (databases!).
- Tag every resource consistently (cost tracking, ownership).
- Use tools: **tflint** (linting), **checkov/tfsec** (security scanning), **terraform-docs** (auto docs), **Terragrunt** (DRY config).

### Example `.gitignore`
```gitignore
.terraform/
*.tfstate
*.tfstate.*
crash.log
*.tfvars
override.tf
.terraformrc
terraform.rc
```

---

## 17. Assignments (Practice Like a Pro)

### 🟢 Beginner
1. Create a single S3 bucket with versioning enabled and a tag `Environment=dev`.
2. Launch a `t2.micro` EC2 instance using a `data` source for the latest Amazon Linux 2 AMI.
3. Add input variables for region and instance type, and output the instance's public IP.

### 🟡 Intermediate
4. Build a VPC with 2 public and 2 private subnets across 2 AZs using `count`.
5. Create a security group allowing HTTP from anywhere and SSH only from your IP.
6. Convert your EC2 setup into a reusable **module** and call it twice (web + app).
7. Configure **remote state** in S3 with DynamoDB locking.

### 🔴 Advanced
8. Deploy the full 3-tier architecture (ALB + Auto Scaling Group + RDS in private subnets).
9. Set up 3 environments (dev/staging/prod) using a directory-per-environment structure.
10. Build a CI/CD pipeline (GitHub Actions/GitLab CI) that runs plan on PR and apply on merge.
11. Use `for_each` to create IAM users from a map, each with different policies.

---

## 18. Interview Questions & Answers

**Q1: What is Terraform state and why is it important?**
> State is a JSON file mapping your configuration to real-world resource IDs. It lets Terraform track what it manages, compute diffs during `plan`, and know what to create/update/destroy. Without it, Terraform couldn't tell the difference between resources to create vs. ones that already exist.

**Q2: Difference between `count` and `for_each`?**
> `count` creates resources by index (list-based) — removing a middle item shifts indices and can recreate resources. `for_each` uses a map/set with stable keys, so adding/removing items only affects that specific item. Prefer `for_each` for stable, named collections.

**Q3: How do you manage secrets in Terraform?**
> Never hardcode them. Use AWS Secrets Manager / SSM Parameter Store via data sources, environment variables, or `-var` at runtime. Mark variables `sensitive = true`, encrypt remote state, and keep `.tfvars` out of Git.

**Q4: What is a Terraform module?**
> A reusable, self-contained package of Terraform configuration with inputs (variables) and outputs. It promotes DRY code and consistency. The root configuration is itself a module that can call child modules.

**Q5: Explain `terraform plan` vs `terraform apply`.**
> `plan` is a dry-run that shows the diff between desired and actual state without making changes. `apply` executes the plan, actually creating/modifying/destroying resources.

**Q6: What is state locking and how does it work with S3 backend?**
> Locking prevents concurrent operations that could corrupt state. With the S3 backend, a DynamoDB table holds a lock record (`LockID`) during operations. If someone else tries to apply, they get a lock error until the first operation finishes.

**Q7: How do you handle state drift?**
> Run `terraform plan` (or `plan -refresh-only`) to detect manual changes. Then either `apply` to revert AWS to match code, or update code and accept reality. Prevent drift by enforcing all changes through Terraform.

**Q8: What is `terraform import` used for?**
> To bring existing resources (created manually or by another tool) under Terraform management without recreating them. You write the resource block, then `terraform import <address> <id>`.

**Q9: Difference between provisioners and `user_data`?**
> `user_data` runs at instance boot (cloud-init) and is the recommended way to bootstrap EC2. Provisioners run from the machine executing Terraform via SSH/WinRM and are a "last resort" — fragile and not idempotent.

**Q10: How do you achieve zero-downtime deployments?**
> Use `lifecycle { create_before_destroy = true }`, combine with Auto Scaling Groups and load balancers, and use rolling/blue-green strategies so new instances are healthy before old ones are removed.

**Q11: What is the difference between Terraform and Ansible?**
> Terraform is declarative IaC focused on *provisioning* infrastructure. Ansible is procedural and focused on *configuration management* (installing/configuring software). They complement each other.

**Q12: What does `terraform taint` / `-replace` do?**
> Forces a resource to be destroyed and recreated on the next apply. `taint` is the older command; `terraform apply -replace="addr"` is the modern approach.

---

## 19. Cheat Sheet & Quick Reference

```powershell
# --- Lifecycle ---
terraform init                 # Initialize
terraform plan                 # Preview changes
terraform apply                # Apply changes
terraform destroy              # Tear down

# --- Code Quality ---
terraform fmt                  # Format code
terraform validate             # Validate syntax

# --- State ---
terraform state list           # List resources
terraform state show <addr>    # Inspect resource
terraform state rm <addr>      # Remove from state
terraform state mv <a> <b>     # Move/rename
terraform import <addr> <id>   # Import existing resource

# --- Targeting / Replacement ---
terraform apply -target=<addr>      # Apply one resource
terraform apply -replace=<addr>     # Force recreate
terraform force-unlock <LOCK_ID>    # Release stuck lock

# --- Output & Debug ---
terraform output                    # Show outputs
terraform show                      # Show full state
$env:TF_LOG="DEBUG"; terraform apply # Verbose logs

# --- Workspaces ---
terraform workspace new <name>
terraform workspace select <name>
terraform workspace list
```

### Version Constraint Operators
| Operator | Meaning |
|----------|---------|
| `= 1.2.0` | Exactly 1.2.0 |
| `>= 1.2.0` | 1.2.0 or higher |
| `~> 1.2.0` | >= 1.2.0, < 1.3.0 (patch updates only) |
| `~> 1.2` | >= 1.2.0, < 2.0.0 (minor updates) |

---

## 🎓 Your Learning Path

```
Week 1: Sections 1-7   → Basics, first deploy, variables
Week 2: Sections 8-12  → State, modules, environments
Week 3: Section 13-14  → Real project + CI/CD
Week 4: Sections 15-18 → Troubleshooting, best practices, interview prep
```

> 💪 **Practice daily.** Build it, break it, fix it, destroy it. That's how you master Terraform.

**Happy Terraforming! 🚀**

