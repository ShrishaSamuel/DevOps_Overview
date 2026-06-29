# Terraform Interview Preparation — Hands-On Tasks

> A curated set of real-world Terraform challenges organized by difficulty.  
> Each task mirrors scenarios you will encounter in DevOps interviews and on the job.

---

## Table of Contents

1. [Beginner Tasks](#beginner-tasks)
2. [Intermediate Tasks](#intermediate-tasks)
3. [Advanced Tasks](#advanced-tasks)
4. [Troubleshooting & Debugging Tasks](#troubleshooting--debugging-tasks)
5. [Infrastructure Design Tasks](#infrastructure-design-tasks)

---

## Beginner Tasks

---

### Task B1 — Deploy a Static Website on AWS S3

**Problem Statement**  
A startup wants to host a static website using AWS S3. Provision an S3 bucket, enable static website hosting, set a public-read bucket policy, and upload an `index.html` object using Terraform.

**Expected Outcome**
- An S3 bucket with website hosting enabled.
- A bucket policy that allows public `GetObject` access.
- A `terraform output` that prints the website endpoint URL.

**Hints**
- Use the `aws_s3_bucket`, `aws_s3_bucket_website_configuration`, and `aws_s3_bucket_policy` resources.
- Use `aws_s3_object` to upload the HTML file.
- The bucket name must be globally unique — use a variable with a default or a random suffix via `random_id`.

**Skills Tested**
- Provider configuration (`aws`)
- Core resource types
- Outputs and variables
- Basic IAM/policy JSON inline in Terraform

---

### Task B2 — Parameterize Infrastructure with Variables and `tfvars`

**Problem Statement**  
You have a hard-coded Terraform config that creates an EC2 instance. Refactor it so that instance type, AMI ID, region, and environment tag are all driven by variables. Create separate `dev.tfvars` and `prod.tfvars` files.

**Expected Outcome**
- Running `terraform apply -var-file=dev.tfvars` creates a `t3.micro` instance tagged `env=dev`.
- Running `terraform apply -var-file=prod.tfvars` creates a `t3.medium` instance tagged `env=prod`.
- No literal values remain in `main.tf`.

**Hints**
- Use `variable` blocks with `type`, `default`, and `description`.
- Use `locals` to compose the full Name tag string.
- Validate the `instance_type` variable with a `validation` block.

**Skills Tested**
- Variables, `tfvars`, and variable validation
- Locals
- Separation of configuration from code

---

### Task B3 — Remote State with S3 and DynamoDB Locking

**Problem Statement**  
Your team works on the same Terraform project simultaneously and keeps getting state conflicts. Migrate the local `terraform.tfstate` to a remote backend using S3 for storage and DynamoDB for state locking.

**Expected Outcome**
- `terraform init` migrates state to S3.
- Concurrent `terraform apply` runs are serialized by DynamoDB lock.
- The S3 bucket has versioning enabled so previous states can be recovered.

**Hints**
- Configure the `backend "s3"` block in `terraform.tf`.
- The DynamoDB table needs a `LockID` primary key of type `String`.
- Create the S3 bucket and DynamoDB table before adding the backend (chicken-and-egg problem — think about how to solve it).

**Skills Tested**
- Remote backends
- State locking concepts
- Bootstrap problem for backend infrastructure

---

## Intermediate Tasks

---

### Task I1 — Modularize a Multi-Tier VPC

**Problem Statement**  
You have a flat Terraform project that creates a VPC, public subnets, private subnets, an internet gateway, NAT gateway, and route tables — all in `main.tf`. Refactor it into reusable modules: `modules/vpc`, `modules/subnets`, and `modules/nat_gateway`. The root module should compose them.

**Expected Outcome**
- Each module has its own `variables.tf`, `main.tf`, and `outputs.tf`.
- The root module passes outputs from `vpc` as inputs to `subnets`.
- Running `terraform apply` from the root produces the same infrastructure as before.

**Hints**
- Use `module` blocks with `source = "./modules/vpc"`.
- Export `vpc_id` and `subnet_ids` as outputs so downstream modules can consume them.
- Use `depends_on` only when implicit dependency via output reference is not sufficient.

**Skills Tested**
- Module design and composition
- Inter-module data flow via outputs
- Reusability and DRY principles

---

### Task I2 — Manage Multiple Environments with Workspaces

**Problem Statement**  
You need to manage `dev`, `staging`, and `prod` environments for an RDS PostgreSQL instance without duplicating Terraform code. Use Terraform workspaces to deploy environment-specific configurations from a single codebase.

**Expected Outcome**
- `terraform workspace new dev && terraform apply` creates a `db.t3.micro` RDS instance.
- `terraform workspace new prod && terraform apply` creates a `db.r6g.large` instance with Multi-AZ enabled.
- Instance identifiers include the workspace name (e.g., `myapp-prod-db`).

**Hints**
- Use `terraform.workspace` to drive a `local` or `lookup` map.
- Define an `env_config` local as a map of workspace → settings.
- Understand the tradeoff: workspaces share a backend but use separate state files per workspace.

**Skills Tested**
- Workspaces
- Dynamic configuration via `lookup` and maps
- Understanding workspace limitations vs. separate root modules

---

### Task I3 — Provision an Auto Scaling Group Behind an ALB

**Problem Statement**  
Provision an AWS Auto Scaling Group (ASG) with a Launch Template behind an Application Load Balancer (ALB). The ASG should scale between 2 and 6 instances based on CPU utilization. The ALB should forward traffic on port 80 to the ASG target group.

**Expected Outcome**
- An ALB with a listener on port 80.
- An ASG using a Launch Template with a user-data script that installs nginx.
- A CloudWatch scaling policy that triggers scale-out at 70% CPU.
- `terraform output alb_dns_name` prints the load balancer address.

**Hints**
- Use `aws_launch_template`, `aws_autoscaling_group`, `aws_lb`, `aws_lb_listener`, and `aws_lb_target_group`.
- Attach the target group to the ASG via `target_group_arns`.
- Use `aws_autoscaling_policy` with `policy_type = "TargetTrackingScaling"`.

**Skills Tested**
- Complex multi-resource dependencies
- User data and compute provisioning
- Auto-scaling and load balancing wiring

---

### Task I4 — Import Existing Cloud Resources into State

**Problem Statement**  
A colleague manually created an RDS instance and an S3 bucket via the AWS Console before Terraform was adopted. Import both resources into the existing Terraform state without destroying and recreating them.

**Expected Outcome**
- `terraform import` succeeds for both resources.
- `terraform plan` shows no diff after the import (configuration matches real state).
- The resources are now fully managed by Terraform going forward.

**Hints**
- Write the resource blocks in `main.tf` first, then run `terraform import`.
- The import ID format differs per resource (e.g., S3 uses bucket name; RDS uses `db-identifier`).
- Use `terraform show` after import to see the actual attribute values and reconcile your config.
- As of Terraform 1.5+, you can also use `import` blocks — try both approaches.

**Skills Tested**
- `terraform import` workflow
- State inspection with `terraform show` and `terraform state list`
- Configuration drift detection

---

## Advanced Tasks

---

### Task A1 — Build a Reusable Multi-Account AWS Landing Zone Module

**Problem Statement**  
Your company uses AWS Organizations with multiple accounts (`dev`, `staging`, `prod`). Build a Terraform module that provisions a standardized baseline for each account: a CloudTrail trail, a default VPC deletion, an S3 logging bucket, and a budget alert at $100. The module must be callable for any account using `assume_role`.

**Expected Outcome**
- A `modules/account_baseline` module that accepts `account_id`, `role_name`, and `budget_limit` as inputs.
- The root module calls it once per account using a `for_each` over a map of accounts.
- Each account gets its own isolated provider configuration via `provider` aliasing.

**Hints**
- Use `provider` blocks with `alias` and pass them to modules via `providers` argument.
- Use `for_each` on the module block over a `map(object(...))` variable.
- The `assume_role` block inside the provider takes `role_arn` built from `account_id`.

**Skills Tested**
- Provider aliasing and dynamic providers
- Module `for_each`
- Multi-account AWS patterns
- Complex variable types (`map(object(...))`)

---

### Task A2 — Zero-Downtime Blue/Green Deployment with Terraform

**Problem Statement**  
You manage an EC2-based application behind an ALB. Implement a blue/green deployment pattern using Terraform so that a new version of the application can be deployed without downtime. Traffic should switch only after the new (green) target group passes health checks.

**Expected Outcome**
- Two Launch Templates (blue/green) and two ASGs exist.
- An ALB listener rule is updated by Terraform to shift weight from blue to green.
- After verification, the blue ASG is scaled down to 0 via a variable.
- A single `terraform apply` with changed variables performs the cutover.

**Hints**
- Use `aws_lb_listener_rule` with weighted target groups.
- Drive the weight split with variables: `blue_weight = 0`, `green_weight = 100`.
- Use `lifecycle { create_before_destroy = true }` on ASG resources.
- Consider `aws_autoscaling_group` `min_size`/`max_size` as variables.

**Skills Tested**
- `lifecycle` meta-arguments
- Weighted ALB routing
- Progressive traffic shifting patterns
- Immutable infrastructure thinking

---

### Task A3 — Manage Terraform State Across Teams with Atlantis or CI/CD

**Problem Statement**  
Set up a GitHub Actions CI/CD pipeline that runs `terraform fmt`, `terraform validate`, `terraform plan` on every PR and `terraform apply` only on merge to `main`. Ensure the pipeline uses OIDC authentication to AWS (no static keys) and posts plan output as a PR comment.

**Expected Outcome**
- A `.github/workflows/terraform.yml` that implements the above.
- AWS OIDC trust policy and IAM role provisioned by a bootstrap Terraform config.
- `terraform plan` output is truncated and posted as a GitHub PR comment using the GitHub API.
- `terraform apply` is gated behind branch protection rules.

**Hints**
- Use `aws_iam_openid_connect_provider` and `aws_iam_role` with a federated trust policy for GitHub OIDC.
- The workflow uses `aws-actions/configure-aws-credentials` with `role-to-assume`.
- Use `terraform show -no-color -json` and `jq` to parse and trim the plan output.
- Store the plan artifact with `actions/upload-artifact` and reuse it in the apply step.

**Skills Tested**
- CI/CD integration with Terraform
- OIDC-based cloud authentication (no long-lived credentials)
- Plan artifact management
- Security best practices for Terraform pipelines

---

### Task A4 — Dynamic Resource Creation with `for_each` and `dynamic` Blocks

**Problem Statement**  
You need to provision a variable number of AWS security groups, each with a dynamic set of ingress rules defined in a YAML file. The number of security groups and their rules should be entirely driven by a structured input — no hard-coded resource blocks.

**Expected Outcome**
- A `security_groups.yaml` file defines 3+ groups with 2–5 rules each.
- `terraform apply` creates exactly those groups and rules.
- Adding a new group or rule to the YAML and rerunning `apply` adds only the delta.

**Hints**
- Use `yamldecode(file("security_groups.yaml"))` to load the config.
- Use `for_each` on `aws_security_group` resources over the parsed map.
- Use `dynamic "ingress"` blocks inside the resource with `for_each` over the rules list.
- Keying strategy matters for `for_each` — use meaningful, stable keys.

**Skills Tested**
- `for_each` with complex types
- `dynamic` blocks
- External data loading via `file()` and `yamldecode()`
- State key stability

---

## Troubleshooting & Debugging Tasks

---

### Task T1 — Fix a Corrupted or Drifted State File

**Problem Statement**  
A team member ran `terraform state rm` on several resources by mistake and then manually deleted some cloud resources. The state file no longer reflects reality. Fix the state without running a full `terraform destroy` and re-apply.

**Expected Outcome**
- All real resources are represented in state.
- `terraform plan` shows no unexpected destroy/recreate operations.
- The fix is documented in a runbook you write alongside the Terraform code.

**Hints**
- Use `terraform state list`, `terraform state show`, and `terraform import` to reconcile.
- For resources that no longer exist in the cloud, remove them from state with `terraform state rm`.
- Use `terraform refresh` (or `terraform apply -refresh-only`) to sync state with reality.
- Check for resources that exist in the cloud but not in state — these need `import`.

**Skills Tested**
- State surgery commands (`state list`, `state rm`, `state mv`, `import`)
- Drift detection and remediation
- Incident response mindset

---

### Task T2 — Debug a Dependency Cycle Error

**Problem Statement**  
A colleague's Terraform config throws: `Error: Cycle: aws_security_group.app, aws_instance.web`. Identify the circular dependency and fix it without changing the intended architecture.

**Expected Outcome**
- `terraform validate` and `terraform plan` succeed with no errors.
- The fix does not change the actual AWS resources created.

**Hints**
- Circular dependencies often happen when two resources reference each other's ID.
- Split the security group into two: one for ingress rules (attached to the instance) and one for egress, or use `aws_security_group_rule` separately.
- Use `terraform graph | dot -Tpng > graph.png` to visualize the dependency graph.

**Skills Tested**
- Dependency graph understanding
- `terraform graph` usage
- Refactoring resources to break cycles

---

### Task T3 — Resolve a Provider Version Conflict After Upgrade

**Problem Statement**  
After upgrading from Terraform 1.3 to 1.6 and bumping the AWS provider from `4.x` to `5.x`, `terraform plan` fails with multiple errors about removed arguments and changed resource schemas. Fix all breaking changes.

**Expected Outcome**
- `terraform init -upgrade` completes without error.
- `terraform plan` shows only the expected infrastructure changes, no errors.
- `versions.tf` pins provider versions with `~>` constraints correctly.

**Hints**
- Read the AWS provider v5 migration guide — many `aws_s3_bucket` sub-resources were split out in v4, and v5 removed more deprecated arguments.
- Use `terraform validate` after each fix to catch remaining issues.
- Use `required_providers` with `~> 5.0` (not `>= 5.0`) to allow patch updates but not major upgrades.

**Skills Tested**
- Provider version management and constraints
- Migration between major provider versions
- Reading provider changelogs and migration guides

---

## Infrastructure Design Tasks

---

### Task D1 — Design a Multi-Region Active-Passive DR Setup

**Problem Statement**  
Design and implement a Terraform configuration that provisions a primary AWS region (`us-east-1`) and a DR region (`eu-west-1`). The setup should include: VPCs, RDS with a read replica in the DR region, S3 cross-region replication, and Route 53 health-check-based failover routing.

**Expected Outcome**
- A module that can be instantiated for each region with a different provider alias.
- RDS primary in `us-east-1` with a read replica in `eu-west-1`.
- Route 53 records that automatically fail over to the DR region when the primary health check fails.
- A `README.md` in the module explaining the failover procedure.

**Hints**
- Use provider aliases: `provider "aws" { alias = "primary" }` and `provider "aws" { alias = "dr" }`.
- S3 replication requires versioning enabled on both buckets and an IAM replication role.
- Route 53 failover routing requires `aws_route53_health_check` and `failover_routing_policy`.

**Skills Tested**
- Multi-region provider configuration
- Disaster recovery architecture patterns
- Complex multi-resource dependency chains
- Infrastructure documentation

---

### Task D2 — Implement Policy-as-Code with Sentinel or OPA (Open Policy Agent)

**Problem Statement**  
Your security team mandates that no EC2 instance in production can be of type `t2.*` and all S3 buckets must have encryption enabled. Implement these guardrails as OPA policies that are evaluated against the Terraform plan JSON before `apply`.

**Expected Outcome**
- Two OPA `.rego` policy files enforcing the above rules.
- A shell script (or CI step) that runs `terraform show -json` and pipes it to `opa eval`.
- A failing plan (intentionally violating config) is blocked with a clear error message.
- A passing plan proceeds to `apply`.

**Hints**
- Use `terraform show -json tfplan.binary > plan.json` to get a machine-readable plan.
- OPA policy input is the plan JSON; iterate over `resource_changes` to find violations.
- Check `change.after.server_side_encryption_configuration` for S3 encryption.
- Use `opa exec` or `conftest` (a wrapper around OPA) for simpler policy testing.

**Skills Tested**
- Policy-as-Code concepts
- Terraform plan JSON structure
- Shift-left security practices
- Integration of external tools with Terraform workflow

---

## Quick Reference — Key Terraform Commands

| Command | Purpose |
|---|---|
| `terraform init` | Initialize provider plugins and backend |
| `terraform validate` | Check configuration syntax and logic |
| `terraform fmt -recursive` | Auto-format all `.tf` files |
| `terraform plan -out=tfplan` | Preview changes, save plan artifact |
| `terraform apply tfplan` | Apply a saved plan |
| `terraform show -json tfplan` | Export plan as JSON (for OPA, PR comments) |
| `terraform state list` | List all resources in state |
| `terraform state show <resource>` | Inspect a single resource in state |
| `terraform state mv <src> <dst>` | Rename/move resource in state |
| `terraform state rm <resource>` | Remove resource from state (dangerous) |
| `terraform import <resource> <id>` | Import existing cloud resource into state |
| `terraform workspace list/new/select` | Manage workspaces |
| `terraform graph \| dot -Tpng > g.png` | Visualize dependency graph |
| `terraform apply -refresh-only` | Sync state with real infrastructure |
| `terraform force-unlock <lock-id>` | Release a stuck state lock |

---

## Study Path

```
Week 1  → B1, B2, B3          (Core concepts, providers, state)
Week 2  → I1, I2, I3          (Modules, workspaces, compute)
Week 3  → I4, T1, T2          (Import, state surgery, debugging)
Week 4  → A1, A2, T3          (Multi-account, blue/green, upgrades)
Week 5  → A3, A4, D1, D2      (CI/CD, dynamic resources, design)
```

---

## Interview Cheat Sheet

- **"Explain Terraform state"** → It's the source of truth mapping config to real resources. Remote state enables team collaboration; locking prevents conflicts.
- **"Modules vs workspaces"** → Modules are reusable code packages. Workspaces isolate state for the same code. For truly different environments, separate root modules with separate backends are safer.
- **"What is `terraform refresh`?"** → Syncs state with real-world infrastructure without making changes. Deprecated in favor of `apply -refresh-only`.
- **"How do you handle secrets?"** → Never store in `.tf` files or state if avoidable. Use AWS Secrets Manager / Vault data sources. Mark sensitive outputs with `sensitive = true`.
- **"What is a provider?"** → A plugin that translates Terraform resources into API calls for a specific platform (AWS, Azure, GCP, Kubernetes, etc.).
- **"Difference between `count` and `for_each`?"** → `count` is index-based (fragile on removal); `for_each` uses stable string/map keys and is preferred for dynamic resources.

---

*Good luck with your interviews. The best preparation is running every task in a real AWS free-tier account.*
