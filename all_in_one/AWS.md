# AWS — Zero to Hero

> A complete, beginner-to-advanced guide to Amazon Web Services (AWS) cloud computing for DevOps engineers and cloud practitioners.

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

### 1.1 What is AWS?

**Amazon Web Services (AWS)** is the world's most comprehensive and widely adopted **cloud computing platform**, launched by Amazon in 2006. It offers 200+ fully featured services from data centers globally — compute, storage, databases, networking, machine learning, analytics, security, and more — on a **pay-as-you-go** basis. Instead of buying and maintaining physical servers, you rent computing resources on demand.

### 1.2 What is cloud computing?

Cloud computing is the on-demand delivery of IT resources over the internet with pay-as-you-go pricing. Key characteristics (per NIST):
- **On-demand self-service** — provision resources instantly.
- **Broad network access** — available over the internet.
- **Resource pooling** — shared infrastructure (multi-tenant).
- **Rapid elasticity** — scale up/down quickly.
- **Measured service** — pay only for what you use.

### 1.3 Cloud service models

```
   On-Premises   │     IaaS      │     PaaS      │     SaaS
 ───────────────┼───────────────┼───────────────┼───────────────
  You manage     │ You manage    │ You manage    │ Provider
  everything     │ OS & up       │ apps & data   │ manages all
                 │ (EC2)         │ (Beanstalk)   │ (e.g., email)
```

| Model | You manage | AWS example |
|-------|-----------|-------------|
| **IaaS** (Infrastructure) | OS, runtime, apps, data | EC2, VPC, EBS |
| **PaaS** (Platform) | Apps and data | Elastic Beanstalk, RDS, Lambda |
| **SaaS** (Software) | Just usage/config | WorkMail, Chime |

### 1.4 Why AWS / benefits

| Benefit | Explanation |
|---------|-------------|
| **No upfront cost** | Pay-as-you-go; convert CapEx to OpEx |
| **Elasticity** | Scale to demand automatically |
| **Global reach** | Deploy worldwide in minutes |
| **Reliability** | Multiple AZs/Regions for high availability |
| **Security** | Compliance certifications, deep security tooling |
| **Breadth** | 200+ services covering nearly every workload |
| **Speed/agility** | Provision resources in seconds |

### 1.5 Global infrastructure

```
        AWS Global Infrastructure
   ┌──────────────────────────────────────┐
   │  Regions (e.g., us-east-1, eu-west-1) │  geographic areas
   │   ┌────────────────────────────────┐ │
   │   │ Availability Zones (AZs)        │ │  isolated data centers
   │   │  AZ-a   AZ-b   AZ-c             │ │  within a region
   │   └────────────────────────────────┘ │
   │  Edge Locations (CloudFront/CDN)      │  100s worldwide for low latency
   └──────────────────────────────────────┘
```

- **Region:** A physical geographic location (e.g., `us-east-1` = N. Virginia) containing multiple AZs. Choose based on latency, cost, compliance.
- **Availability Zone (AZ):** One or more discrete data centers with redundant power/networking, isolated from failures in other AZs. Deploy across multiple AZs for high availability.
- **Edge Location:** Sites used by CloudFront (CDN) and other edge services to cache content close to users.

### 1.6 The shared responsibility model

Security is shared:
- **AWS is responsible for security *of* the cloud:** hardware, software, networking, and facilities that run AWS services.
- **You are responsible for security *in* the cloud:** your data, IAM configuration, OS patching (for EC2), network/firewall config, encryption, application security.

### 1.7 Pricing fundamentals

- **Pay-as-you-go** — pay for what you use, no long-term commitment by default.
- **Save when you commit** — Reserved Instances / Savings Plans for steady workloads.
- **Pay less by using more** — volume discounts (e.g., S3 tiers).
- **Free Tier** — 12-month free tier + always-free services for learning.
- Use the **AWS Pricing Calculator** and **Cost Explorer**; set **Budgets** and alerts.

### 1.8 When to use AWS

- Hosting web/mobile apps, APIs, and microservices.
- Big data, analytics, ML, and data lakes.
- Disaster recovery and backup.
- Elastic workloads with variable demand.
- Global applications needing low latency.

When to reconsider: strict data-residency/regulatory constraints not met by available regions, very predictable fixed workloads where owned hardware is cheaper long-term, or vendor lock-in concerns (mitigate with portable/IaC patterns).

---

## 2. Installation & Setup

### 2.1 Create an AWS account

1. Go to https://aws.amazon.com and **Create an AWS Account**.
2. Provide email, payment method (required even for Free Tier), and verify identity.
3. Choose the **Basic (free) support plan**.
4. **Immediately secure the root user:** enable **MFA** and avoid using root for daily tasks.

### 2.2 Create an IAM admin user (don't use root)

1. Open the **IAM** console.
2. Create a user with **AdministratorAccess** (for learning) — better: scoped roles.
3. Enable MFA for that user.
4. Create **access keys** only if you need CLI/programmatic access (prefer roles/SSO).

### 2.3 Install and configure the AWS CLI

```bash
# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version

# macOS: brew install awscli
```

```bash
aws configure
# AWS Access Key ID:     <your key>
# AWS Secret Access Key: <your secret>
# Default region name:   us-east-1
# Default output format: json

aws sts get-caller-identity        # verify credentials
```

> Best practice: use **IAM Identity Center (SSO)** or named profiles + short-lived credentials instead of long-lived keys. `aws configure sso` sets up SSO-based access.

### 2.4 Useful CLI basics

```bash
aws s3 ls                          # list buckets
aws ec2 describe-instances         # list EC2 instances
aws iam list-users
aws configure list-profiles
aws s3 ls --profile dev            # use a named profile
```

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 IAM (Identity and Access Management)

IAM controls **who** can do **what** in your account. It's the foundation of AWS security.

| Entity | Description |
|--------|-------------|
| **User** | A person or app with long-term credentials |
| **Group** | A collection of users sharing permissions |
| **Role** | Temporary credentials assumed by users/services/resources |
| **Policy** | JSON document defining permissions (allow/deny) |

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

Principles: **least privilege**, prefer **roles** over long-lived keys, always enable **MFA**.

#### 3.1.2 EC2 (Elastic Compute Cloud)

EC2 provides resizable **virtual servers (instances)** in the cloud.

Key concepts:
- **AMI (Amazon Machine Image):** template (OS + software) for an instance.
- **Instance type:** size/family (e.g., `t3.micro` general purpose, `c7g` compute, `r6` memory).
- **Key pair:** SSH credentials for access.
- **Security group:** virtual firewall controlling inbound/outbound traffic.
- **EBS volume:** persistent block storage attached to instances.
- **User data:** startup script run at first boot.

```bash
aws ec2 run-instances \
  --image-id ami-0abcdef1234567890 \
  --instance-type t3.micro \
  --key-name my-key \
  --security-group-ids sg-0123456789 \
  --subnet-id subnet-0123456789 \
  --count 1
```

#### 3.1.3 S3 (Simple Storage Service)

S3 is **object storage** — store and retrieve any amount of data (files/objects) in **buckets**.

- **Bucket:** globally unique container for objects.
- **Object:** a file + metadata, identified by a key.
- **Durability:** 99.999999999% ("11 nines").
- **Storage classes:** Standard, Intelligent-Tiering, Standard-IA, One Zone-IA, Glacier (archive), Glacier Deep Archive.

```bash
aws s3 mb s3://my-unique-bucket-12345        # make bucket
aws s3 cp file.txt s3://my-unique-bucket-12345/
aws s3 sync ./local-dir s3://my-unique-bucket-12345/dir/
aws s3 ls s3://my-unique-bucket-12345/
aws s3 rb s3://my-unique-bucket-12345 --force # remove bucket
```

> By default buckets are **private**. Block Public Access is on by default — keep it on unless you truly need public objects (use CloudFront for public content).

#### 3.1.4 Regions and AZ selection

Choose a region close to users (latency), where required services exist, that meets compliance, and that's cost-effective. Deploy across **multiple AZs** for resilience.

### 3.2 INTERMEDIATE

#### 3.2.1 VPC (Virtual Private Cloud)

A **VPC** is your own isolated virtual network in AWS.

```
VPC (10.0.0.0/16)
├── Public Subnet  (10.0.1.0/24)  ── route to Internet Gateway → internet
│     └── Web servers, NAT Gateway, Load Balancer
├── Private Subnet (10.0.2.0/24)  ── route to NAT Gateway → outbound only
│     └── App servers, databases
└── Components: Route Tables, Security Groups, NACLs, IGW, NAT GW
```

Components:
- **Subnet:** a range of IPs in an AZ; public (route to IGW) or private.
- **Internet Gateway (IGW):** allows internet access for public subnets.
- **NAT Gateway:** lets private instances reach the internet outbound (no inbound).
- **Route Table:** rules directing traffic.
- **Security Group:** stateful instance-level firewall (allow rules only).
- **Network ACL (NACL):** stateless subnet-level firewall (allow + deny).

**Security Group vs. NACL:**

| | Security Group | NACL |
|--|---------------|------|
| Level | Instance | Subnet |
| State | Stateful (return traffic auto-allowed) | Stateless (must allow both ways) |
| Rules | Allow only | Allow and Deny |
| Evaluation | All rules | In order by rule number |

#### 3.2.2 Databases

| Service | Type | Use case |
|---------|------|----------|
| **RDS** | Managed relational (MySQL, PostgreSQL, etc.) | Traditional SQL apps |
| **Aurora** | High-performance MySQL/PostgreSQL-compatible | Scalable relational |
| **DynamoDB** | Managed NoSQL key-value/document | High-scale, low-latency |
| **ElastiCache** | In-memory (Redis/Memcached) | Caching, sessions |
| **Redshift** | Data warehouse | Analytics/BI |

RDS features: automated backups, multi-AZ failover, read replicas, patching.

```bash
aws rds create-db-instance \
  --db-instance-identifier mydb \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --master-username admin \
  --master-user-password '<secret>' \
  --allocated-storage 20
```

#### 3.2.3 Load Balancing and Auto Scaling

- **Elastic Load Balancer (ELB):** distributes traffic across targets.
  - **ALB** (Application LB) — HTTP/HTTPS, layer 7, path/host routing.
  - **NLB** (Network LB) — TCP/UDP, layer 4, ultra-low latency.
  - **GWLB** (Gateway LB) — for network appliances.
- **Auto Scaling Group (ASG):** automatically adjusts the number of EC2 instances based on demand/health, across AZs.

```
        Internet
           │
        [ ALB ]
       /   │   \
   [EC2] [EC2] [EC2]   ← Auto Scaling Group (scales 2→10 on CPU)
   spread across AZ-a, AZ-b, AZ-c
```

#### 3.2.4 Serverless: Lambda

**AWS Lambda** runs code without provisioning servers; you pay per invocation/duration. Event-driven (triggered by S3, API Gateway, DynamoDB, schedules, etc.).

```python
def lambda_handler(event, context):
    name = event.get("name", "World")
    return {"statusCode": 200, "body": f"Hello, {name}!"}
```

Common serverless stack: **API Gateway → Lambda → DynamoDB**.

#### 3.2.5 Messaging and integration

- **SQS** — managed message queue (decouple components).
- **SNS** — pub/sub notifications (fan-out to many subscribers).
- **EventBridge** — event bus connecting AWS services and SaaS.
- **Step Functions** — orchestrate workflows across services.

#### 3.2.6 Monitoring: CloudWatch & CloudTrail

- **CloudWatch:** metrics, logs, alarms, dashboards (operational monitoring).
- **CloudTrail:** records API calls / who did what (auditing/governance).
- **Config:** tracks resource configuration changes/compliance.

```bash
aws logs tail /aws/lambda/my-func --follow
aws cloudwatch put-metric-alarm --alarm-name HighCPU \
  --metric-name CPUUtilization --namespace AWS/EC2 \
  --threshold 80 --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 --period 300 --statistic Average
```

### 3.3 ADVANCED

#### 3.3.1 Infrastructure as Code on AWS

- **CloudFormation** — native IaC using YAML/JSON templates.
- **CDK (Cloud Development Kit)** — define infra in real languages (TypeScript, Python).
- **Terraform** — multi-cloud IaC (very popular for AWS).

```yaml
# CloudFormation snippet
Resources:
  MyBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-cfn-bucket-12345
      VersioningConfiguration:
        Status: Enabled
```

```bash
aws cloudformation deploy --template-file template.yml --stack-name my-stack
```

#### 3.3.2 Containers and orchestration

| Service | Description |
|---------|-------------|
| **ECR** | Private container registry |
| **ECS** | AWS-native container orchestration |
| **Fargate** | Serverless containers (no EC2 to manage) |
| **EKS** | Managed Kubernetes |

A typical pipeline: build image → push to **ECR** → deploy to **ECS/Fargate** or **EKS**.

#### 3.3.3 High availability & resilience design

- Spread across **multiple AZs** (and Regions for DR).
- Use **ASG + ELB** for self-healing, scalable compute.
- **RDS Multi-AZ** for database failover; read replicas for scale.
- **S3 + CloudFront** for durable, low-latency content.
- **Route 53** for DNS, health checks, and failover routing.
- Decouple with **SQS/SNS** so component failures don't cascade.

#### 3.3.4 Security in depth

- **IAM least privilege**, roles over keys, MFA, permission boundaries, SCPs (Organizations).
- **Encryption:** KMS-managed keys; encrypt EBS, S3, RDS at rest; TLS in transit.
- **Secrets Manager / SSM Parameter Store** for credentials.
- **GuardDuty** (threat detection), **Security Hub** (posture), **Inspector** (vuln scanning), **WAF/Shield** (web/DDoS protection).
- **VPC design:** private subnets, security groups, flow logs, no public DBs.
- **CloudTrail** enabled in all regions; **Config** rules for compliance.

#### 3.3.5 Cost optimization

- Right-size instances; use **Savings Plans / Reserved Instances** for steady load.
- **Spot Instances** for fault-tolerant batch workloads (up to ~90% off).
- **S3 lifecycle policies** to tier/expire data (→ IA → Glacier).
- **Auto Scaling** to match demand; stop idle dev resources.
- Monitor with **Cost Explorer**, **Budgets**, and the **Pricing Calculator**.
- Tag resources for cost allocation.

#### 3.3.6 The Well-Architected Framework

AWS's design guidance across six pillars:

1. **Operational Excellence** — run and monitor systems, improve processes.
2. **Security** — protect data, systems, and assets.
3. **Reliability** — recover from failures, scale to meet demand.
4. **Performance Efficiency** — use resources efficiently.
5. **Cost Optimization** — avoid unnecessary costs.
6. **Sustainability** — minimize environmental impact.

#### 3.3.7 Networking advanced

- **VPC Peering / Transit Gateway** — connect VPCs.
- **VPC Endpoints (PrivateLink)** — private access to AWS services without internet.
- **Direct Connect / VPN** — hybrid connectivity to on-prem.
- **Route 53** — DNS, latency/geo/weighted/failover routing policies.
- **CloudFront** — global CDN with edge caching, WAF integration.

#### 3.3.8 DevOps on AWS

- **CodeCommit / CodeBuild / CodeDeploy / CodePipeline** — native CI/CD suite.
- **CloudWatch + X-Ray** — monitoring and distributed tracing.
- **Systems Manager (SSM)** — fleet management, patching, run command, Session Manager (SSH-less access).

---

## 4. Hands-on Tasks

### Task 1: Configure the CLI and verify identity

```bash
aws configure
aws sts get-caller-identity
```

**Expected:** JSON with your account ID, user ARN, and user ID.

### Task 2: Create and use an S3 bucket

```bash
aws s3 mb s3://zth-demo-$RANDOM
BUCKET=$(aws s3 ls | awk '/zth-demo/{print $3}' | head -1)
echo "hello aws" > hello.txt
aws s3 cp hello.txt s3://$BUCKET/
aws s3 ls s3://$BUCKET/
aws s3 rm s3://$BUCKET/hello.txt
aws s3 rb s3://$BUCKET
```

**Expected:** Upload, list, and cleanup succeed.

### Task 3: Launch an EC2 instance with a web server

Use a security group allowing 22 and 80, and user data:

```bash
#!/bin/bash
yum -y install httpd || apt-get -y install apache2
echo "<h1>Hello from EC2</h1>" > /var/www/html/index.html
systemctl enable --now httpd 2>/dev/null || systemctl enable --now apache2
```

```bash
aws ec2 run-instances --image-id <ami> --instance-type t3.micro \
  --key-name my-key --security-group-ids <sg> --subnet-id <subnet> \
  --user-data file://userdata.sh --count 1
```

**Expected:** Instance launches; the public IP serves the web page.

### Task 4: Create an IAM user with a scoped policy

```bash
aws iam create-user --user-name app-reader
aws iam attach-user-policy --user-name app-reader \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam list-attached-user-policies --user-name app-reader
```

**Expected:** User has read-only S3 access (least privilege demo).

### Task 5: Enable S3 versioning and lifecycle

```bash
aws s3api put-bucket-versioning --bucket "$BUCKET" \
  --versioning-configuration Status=Enabled
```

**Expected:** Object versions retained; configure a lifecycle rule to transition to Glacier.

### Task 6: Create a security group

```bash
aws ec2 create-security-group --group-name web-sg --description "web"
aws ec2 authorize-security-group-ingress --group-name web-sg \
  --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-name web-sg \
  --protocol tcp --port 22 --cidr <your-ip>/32
```

**Expected:** SG allows HTTP from anywhere and SSH from your IP only.

### Task 7: Deploy a Lambda function

```bash
zip function.zip lambda_function.py
aws lambda create-function --function-name hello \
  --runtime python3.12 --handler lambda_function.lambda_handler \
  --role arn:aws:iam::<acct>:role/lambda-role \
  --zip-file fileb://function.zip
aws lambda invoke --function-name hello --payload '{"name":"AWS"}' out.json
cat out.json
```

**Expected:** `out.json` contains `Hello, AWS!`.

### Task 8: Create an RDS instance (Free Tier)

```bash
aws rds create-db-instance --db-instance-identifier zth-db \
  --db-instance-class db.t3.micro --engine postgres \
  --master-username admin --master-user-password '<secret>' \
  --allocated-storage 20 --no-multi-az
aws rds describe-db-instances --db-instance-identifier zth-db \
  --query 'DBInstances[0].DBInstanceStatus'
```

**Expected:** Status transitions `creating` → `available`.

### Task 9: Set a billing budget alarm

In the Billing console → **Budgets** → create a monthly cost budget (e.g., $5) with an email alert at 80%.

**Expected:** You get notified before unexpected charges accrue.

### Task 10: Deploy infra with CloudFormation

```bash
aws cloudformation deploy --template-file template.yml --stack-name zth-stack
aws cloudformation describe-stacks --stack-name zth-stack \
  --query 'Stacks[0].StackStatus'
aws cloudformation delete-stack --stack-name zth-stack
```

**Expected:** Stack reaches `CREATE_COMPLETE`; deletion cleans up resources.

---

## 5. Projects

### Project 1 (Beginner): Static Website on S3 + CloudFront

**Goal:** Host a static site cheaply, globally, and securely.

Steps:
1. Create an S3 bucket and upload site files.
2. Create a **CloudFront** distribution in front of the bucket (Origin Access Control), keeping the bucket private.
3. Add an SSL certificate via **ACM**.
4. Point a domain with **Route 53**.
5. Set cache behaviors and test global delivery.

**Skills:** S3, CloudFront, ACM, Route 53, security (private bucket).

### Project 2 (Intermediate): Highly Available Web Application

**Goal:** A resilient, auto-scaling web app across multiple AZs.

Architecture:
```
Route 53 → ALB → Auto Scaling Group (EC2 in 2+ AZs) → RDS (Multi-AZ)
                                   └── ElastiCache (sessions/cache)
Static assets → S3 + CloudFront
```

Steps:
1. Build a **VPC** with public/private subnets across 2 AZs (+ NAT GW).
2. Launch EC2 via an **Auto Scaling Group** behind an **ALB**.
3. Use **RDS Multi-AZ** in private subnets; lock down with security groups.
4. Add **ElastiCache** for sessions; store assets in S3+CloudFront.
5. Configure scaling policies and health checks; test failover.
6. Add **CloudWatch** alarms and a billing budget.

**Skills:** VPC design, ASG, ALB, RDS HA, caching, monitoring, security groups.

### Project 3 (Advanced): Serverless Microservices with IaC & CI/CD

**Goal:** A fully serverless, automated, secure application.

Architecture:
```
Client → API Gateway → Lambda (multiple functions) → DynamoDB
                                  ├── S3 (uploads, triggers Lambda)
                                  ├── SQS/SNS (async/decoupling)
                                  └── Cognito (auth)
Observability: CloudWatch + X-Ray   |   IaC: CDK/SAM/Terraform
```

Steps:
1. Define everything as **IaC** (AWS SAM / CDK / Terraform).
2. Build REST APIs with **API Gateway + Lambda + DynamoDB**.
3. Add **Cognito** for authentication/authorization.
4. Use **S3 event triggers** and **SQS/SNS** for async processing.
5. Apply **least-privilege IAM** per function; secrets in **Secrets Manager**.
6. Build a **CI/CD pipeline** (CodePipeline/GitHub Actions): test → build → deploy.
7. Add **CloudWatch dashboards/alarms** and **X-Ray** tracing.
8. Enforce cost controls (Budgets) and security (GuardDuty, least privilege).

**Skills:** serverless, IaC, API design, auth, event-driven architecture, CI/CD, observability, security, cost management.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Never use the root account** for daily work; enable MFA everywhere; apply **least privilege** with IAM roles.
- **Use roles, not long-lived access keys**; rotate any keys; prefer SSO/short-lived credentials.
- **Deploy across multiple AZs** for high availability.
- **Encrypt data** at rest (KMS) and in transit (TLS); keep S3 Block Public Access on.
- **Use Infrastructure as Code** (CloudFormation/CDK/Terraform) — no manual console drift.
- **Tag resources** for cost allocation and ownership.
- **Set budgets and billing alarms**; right-size and use Savings Plans/Spot.
- **Enable CloudTrail (all regions)** and CloudWatch monitoring/alarms.
- **Design for failure**: decouple with queues, use ASG/Multi-AZ, automate recovery.
- **Follow the Well-Architected Framework**.
- **Keep databases in private subnets**; restrict security groups tightly.

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Using root for everything | Catastrophic if compromised | IAM users/roles + MFA |
| Public S3 buckets by mistake | Data leaks | Keep Block Public Access on; use CloudFront |
| Hard-coding access keys in code | Credential theft | IAM roles, Secrets Manager, env/SSM |
| Single-AZ deployments | Outage on AZ failure | Multi-AZ design |
| No billing alarms | Surprise bills ("bill shock") | Budgets + Cost Explorer |
| Forgetting to stop/terminate resources | Ongoing charges | Tag + automate cleanup; stop idle dev |
| Overly broad IAM (`*:*`) | Privilege escalation risk | Least privilege, scoped policies |
| Click-ops (manual console) | Drift, irreproducible | IaC for everything |
| Unencrypted data | Compliance/security risk | KMS encryption at rest + TLS |
| Ignoring service quotas/limits | Throttling, failed launches | Monitor and request quota increases |

---

## 7. Interview Questions

### Beginner

1. **What is the difference between a Region and an Availability Zone?**
   *A Region is a geographic area; an AZ is one or more isolated data centers within a Region. Use multiple AZs for high availability.*

2. **What is EC2?**
   *Elastic Compute Cloud — resizable virtual servers (instances) you launch from AMIs.*

3. **What is S3 and what is it used for?**
   *Simple Storage Service — highly durable object storage organized in buckets, used for files, backups, static assets, data lakes.*

4. **What is IAM?**
   *Identity and Access Management — controls authentication and authorization (users, groups, roles, policies) in AWS.*

5. **What does the shared responsibility model mean?**
   *AWS secures the cloud infrastructure; the customer secures what they put in the cloud (data, IAM, OS, app, network config).*

### Intermediate

6. **Security Group vs. NACL?**
   *Security groups are stateful, instance-level, allow-only; NACLs are stateless, subnet-level, support allow and deny rules evaluated in order.*

7. **What is a VPC and its main components?**
   *A logically isolated virtual network; components include subnets, route tables, internet/NAT gateways, security groups, and NACLs.*

8. **Compare RDS and DynamoDB.**
   *RDS is managed relational (SQL, fixed schema, vertical scaling, joins); DynamoDB is managed NoSQL (key-value/document, horizontal scale, single-digit ms latency).*

9. **What is Auto Scaling and why use it?**
   *Automatically adjusts the number of instances based on demand/health across AZs — improving availability and cost efficiency.*

10. **What is AWS Lambda?**
    *Serverless compute that runs event-driven code without managing servers; you pay per invocation and duration.*

### Advanced

11. **How would you design a highly available, scalable web application on AWS?**
    *Multi-AZ VPC, ALB + Auto Scaling Group across AZs, RDS Multi-AZ with read replicas, ElastiCache, S3+CloudFront for static content, Route 53 with health checks, decoupling via SQS/SNS, and CloudWatch monitoring.*

12. **How do you secure an AWS environment?**
    *Least-privilege IAM with roles/MFA, no root usage, KMS encryption, private subnets, tight security groups, Secrets Manager, GuardDuty/Security Hub/Inspector, CloudTrail + Config, and SCPs via Organizations.*

13. **Explain strategies to optimize AWS costs.**
    *Right-sizing, Savings Plans/Reserved Instances for steady load, Spot for fault-tolerant batch, S3 lifecycle tiering, Auto Scaling, stopping idle resources, tagging + Cost Explorer/Budgets.*

14. **What are the pillars of the Well-Architected Framework?**
    *Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, and Sustainability.*

15. **How do you connect a private subnet's instances to the internet, and how does that differ from inbound access?**
    *Outbound: route private subnet traffic through a NAT Gateway in a public subnet (no inbound from the internet). Inbound public access requires resources in public subnets behind an Internet Gateway (e.g., a load balancer), keeping app/db tiers private.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** Which service provides object storage?
- A) EBS  B) S3  C) EFS  D) RDS

**Q2.** What is an isolated data center grouping within a region called?
- A) Edge Location  B) Availability Zone  C) VPC  D) Subnet

**Q3.** Which controls who can do what in AWS?
- A) VPC  B) IAM  C) S3  D) CloudWatch

**Q4.** Which is a stateful, instance-level firewall?
- A) NACL  B) Security Group  C) WAF  D) Route Table

**Q5.** Which service runs code without managing servers?
- A) EC2  B) ECS  C) Lambda  D) EKS

**Q6.** Which database is NoSQL?
- A) RDS  B) Aurora  C) DynamoDB  D) Redshift

**Q7.** Which distributes incoming traffic across instances?
- A) ASG  B) ELB  C) Route 53  D) CloudFront

**Q8.** Which records API calls for auditing?
- A) CloudWatch  B) CloudTrail  C) Config  D) Inspector

**Q9.** Which service is a global CDN?
- A) S3  B) CloudFront  C) VPC  D) Direct Connect

**Q10.** What lets private instances reach the internet outbound only?
- A) Internet Gateway  B) NAT Gateway  C) VPC Endpoint  D) Transit Gateway

### Short Answer

**S1.** What command verifies which AWS identity your CLI is using?

**S2.** Name the four core IAM entities.

**S3.** Which S3 setting should stay enabled to prevent accidental public exposure?

**S4.** Which compute purchasing option offers the deepest discount for fault-tolerant workloads?

**S5.** List three pillars of the Well-Architected Framework.

### Answer Key

**Multiple Choice:** Q1-B, Q2-B, Q3-B, Q4-B, Q5-C, Q6-C, Q7-B, Q8-B, Q9-B, Q10-B

**Short Answer:**
- **S1.** `aws sts get-caller-identity`
- **S2.** Users, Groups, Roles, Policies.
- **S3.** S3 **Block Public Access**.
- **S4.** **Spot Instances**.
- **S5.** Any three of: Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, Sustainability.

---

## 9. Further Resources

### Official Documentation
- AWS Documentation — https://docs.aws.amazon.com
- AWS Well-Architected Framework — https://aws.amazon.com/architecture/well-architected/
- AWS CLI reference — https://docs.aws.amazon.com/cli/
- AWS Free Tier — https://aws.amazon.com/free/

### Learning
- AWS Skill Builder — https://skillbuilder.aws
- AWS Workshops — https://workshops.aws
- AWS Hands-on Tutorials — https://aws.amazon.com/getting-started/hands-on/
- freeCodeCamp / Adrian Cantrill / A Cloud Guru courses

### Tools
- AWS CLI, AWS CloudShell (browser shell)
- AWS SAM / CDK — serverless & IaC
- Terraform AWS provider
- AWS Pricing Calculator — https://calculator.aws

### Books
- *AWS Certified Solutions Architect Study Guide* — Ben Piper & David Clinton
- *Amazon Web Services in Action* — Wittig & Wittig
- *The Well-Architected Framework* whitepapers (free)

### Certifications
- AWS Certified Cloud Practitioner (foundational)
- AWS Certified Solutions Architect – Associate
- AWS Certified Developer / SysOps Administrator – Associate
- AWS Certified DevOps Engineer / Solutions Architect – Professional

---

*End of AWS — Zero to Hero.*
