# AWS DevOps Interview Preparation — Hands-On Tasks

> A curated set of real-world AWS challenges organized by difficulty.  
> Each task mirrors scenarios you will encounter in DevOps interviews and on the job.

---

## Table of Contents

1. [Beginner Tasks](#beginner-tasks)
2. [Intermediate Tasks](#intermediate-tasks)
3. [Advanced Tasks](#advanced-tasks)
4. [Troubleshooting & Debugging Tasks](#troubleshooting--debugging-tasks)
5. [Cloud Architecture Design Tasks](#cloud-architecture-design-tasks)

---

## Beginner Tasks

---

### Task B1 — Launch and Harden an EC2 Instance

**Problem Statement**  
Launch an EC2 instance running Amazon Linux 2023 that hosts a simple web server. The instance must be accessible on port 80 from the internet but SSH access must only be allowed from your IP address. No one should be able to SSH using a password — only key-pair authentication. Block all other inbound traffic.

**Expected Outcome**
- EC2 instance is `running` and `curl http://<public-ip>` returns the nginx default page.
- Port 22 is only open to your specific IP (`/32` CIDR), not `0.0.0.0/0`.
- Password-based SSH is disabled in `/etc/ssh/sshd_config`.
- Security group has a minimal rule set — no wildcard inbound rules except port 80.
- Instance has an IAM instance profile attached (even if empty) — never use access keys on EC2.

**Hints**
- Use `User Data` to run `yum install -y nginx && systemctl enable --now nginx` at launch.
- Security group inbound rules: HTTP (`0.0.0.0/0`), SSH (`<your-ip>/32`).
- `PasswordAuthentication no` in `/etc/ssh/sshd_config` — verify with `ssh -o PreferredAuthentications=password`.
- Use `aws ec2 describe-instances --query` to practice filtering CLI output.
- Tag every resource: `Name`, `Environment`, `Owner` — gets asked in every interview.

**Skills Tested**
- EC2 launch configuration and User Data
- Security group rule design (least privilege)
- SSH key-pair vs password authentication
- IAM instance profiles (no hardcoded credentials on EC2)
- Resource tagging best practices

---

### Task B2 — Set Up S3 Bucket Policies and Block Public Access

**Problem Statement**  
Your team accidentally made an S3 bucket containing internal reports publicly accessible. Immediately lock it down. Then create a second bucket correctly from scratch: private by default, versioning enabled, server-side encryption with SSE-S3, access logging to a separate log bucket, and a lifecycle policy that moves objects to Glacier after 90 days.

**Expected Outcome**
- `aws s3api get-bucket-acl` and `get-bucket-policy` confirm the original bucket has no public access.
- Block Public Access is enabled at both bucket and account level.
- New bucket has versioning, SSE-S3 encryption, access logging, and a lifecycle rule all configured.
- `aws s3 cp` to the private bucket without credentials returns `403 Forbidden`.
- Lifecycle rule transitions objects with prefix `reports/` to `GLACIER` after 90 days.

**Hints**
- Enable Block Public Access: `aws s3api put-public-access-block --public-access-block-configuration BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true`.
- Use `aws s3api put-bucket-encryption` with `SSEAlgorithm: AES256` for SSE-S3.
- Lifecycle rules require JSON config — use `put-bucket-lifecycle-configuration`.
- Access logging requires the log bucket to have a bucket ACL granting write to the S3 log delivery group.
- Use `aws s3api get-bucket-versioning` to confirm versioning state.

**Skills Tested**
- S3 security configuration (Block Public Access, bucket policies, ACLs)
- S3 versioning, encryption, access logging
- S3 lifecycle rules and storage class transitions
- AWS CLI proficiency for S3 operations

---

### Task B3 — Create a Least-Privilege IAM Role and Policy

**Problem Statement**  
A Lambda function needs to read objects from a specific S3 bucket (`arn:aws:s3:::my-data-bucket/*`) and write logs to CloudWatch. Create an IAM role with a custom policy granting only these permissions — no wildcards, no `*` on resources, no `AdministratorAccess`. Attach the role to the Lambda and verify it can perform only its intended actions.

**Expected Outcome**
- IAM role with a trust policy allowing `lambda.amazonaws.com` to assume it.
- Custom inline or managed policy with `s3:GetObject` on `arn:aws:s3:::my-data-bucket/*` and `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents` on the specific log group ARN.
- Lambda function can read from the bucket and write to CloudWatch.
- `aws iam simulate-principal-policy` confirms `s3:DeleteObject` is denied.
- Attempting `s3:ListBucket` on a different bucket returns `AccessDenied`.

**Hints**
- Trust policy for Lambda: `{"Service": "lambda.amazonaws.com"}` in the `Principal` field.
- Use `aws iam simulate-principal-policy` to test the policy before deploying.
- Use IAM policy conditions: `"StringEquals": {"s3:prefix": "data/"}` to restrict by prefix.
- Avoid `iam:PassRole` unless explicitly needed — it's a privilege escalation vector.
- Use AWS Policy Generator or IAM Access Analyzer to validate your policy logic.

**Skills Tested**
- IAM role and trust policy structure
- Principle of least privilege in policy design
- Resource-level permissions (not `*`)
- Policy simulation and testing
- Privilege escalation awareness

---

### Task B4 — Build a Custom VPC from Scratch

**Problem Statement**  
Create a custom VPC (`10.0.0.0/16`) with the following layout: 2 public subnets (in different AZs), 2 private subnets (in the same AZs), an Internet Gateway attached to the VPC, a NAT Gateway in one public subnet, and correct route tables for each subnet tier. Launch an EC2 instance in a private subnet and verify it can reach the internet via NAT but cannot be reached directly from the internet.

**Expected Outcome**
- Public subnets route `0.0.0.0/0` → Internet Gateway.
- Private subnets route `0.0.0.0/0` → NAT Gateway.
- An EC2 instance in the private subnet can `curl https://example.com` successfully.
- The private instance has no public IP and is unreachable directly from the internet.
- A bastion host in the public subnet allows SSH tunneling into the private instance.

**Hints**
- Subnet CIDR example: public `10.0.1.0/24`, `10.0.2.0/24`; private `10.0.3.0/24`, `10.0.4.0/24`.
- NAT Gateway requires an Elastic IP and must be in a **public** subnet.
- Each subnet tier needs its own route table — don't reuse the main route table for public subnets.
- Enable `Auto-assign public IPv4` on public subnets: `modify-subnet-attribute --map-public-ip-on-launch`.
- Use a bastion host with SSH agent forwarding (`ssh -A`) to reach private instances without copying private keys.

**Skills Tested**
- VPC design: subnets, AZs, CIDR planning
- Internet Gateway vs NAT Gateway
- Route table configuration
- Public vs private subnet distinction
- Bastion host pattern

---

## Intermediate Tasks

---

### Task I1 — Configure an Auto Scaling Group with a Custom Launch Template

**Problem Statement**  
A web application currently runs on a single EC2 instance. A traffic spike has caused outages twice this month. Replace it with an Auto Scaling Group (ASG) that scales between 2 and 8 instances based on average CPU utilization (target: 60%). Use a Launch Template with a hardened AMI, IMDSv2 enforced, and user data that installs and starts the app. Place the ASG behind an Application Load Balancer.

**Expected Outcome**
- ASG maintains 2 healthy instances at baseline.
- Running a stress test causes the ASG to scale out within 5 minutes.
- ALB health checks remove unhealthy instances and replace them automatically.
- IMDSv2 is enforced (`HttpTokens: required`) in the Launch Template — `curl http://169.254.169.254/latest/meta-data/` without a token returns `401`.
- Scaling events are visible in CloudWatch and the ASG Activity History.

**Hints**
- Create the Launch Template with `--metadata-options HttpTokens=required,HttpEndpoint=enabled`.
- Use `aws autoscaling put-scaling-policy` with `TargetTrackingScaling` and `ASGAverageCPUUtilization`.
- ALB target group health check path must match a real endpoint (e.g., `/health`).
- Set `MinHealthyPercentage` in instance refresh to enable zero-downtime AMI updates.
- Use `aws autoscaling describe-scaling-activities` to see scale-out/in history.

**Skills Tested**
- Launch Template configuration (vs Launch Configuration — deprecated)
- IMDSv2 enforcement (SSRF protection on EC2 metadata)
- Target tracking autoscaling policies
- ALB target group and health check integration
- ASG lifecycle and instance refresh

---

### Task I2 — Set Up RDS with Multi-AZ, Automated Backups, and Parameter Tuning

**Problem Statement**  
Your application needs a production-grade PostgreSQL database. Provision an RDS PostgreSQL instance with Multi-AZ enabled, automated backups retained for 7 days, encryption at rest (KMS), and no public accessibility. Place it in a DB subnet group spanning 3 AZs. Tune the `max_connections` parameter and test a manual failover.

**Expected Outcome**
- RDS instance is in a private subnet group — `PubliclyAccessible: false`.
- Multi-AZ shows a standby in a different AZ (`MultiAZ: true`).
- Automated backups are enabled with a 7-day retention window.
- A custom DB parameter group sets `max_connections` to `200`.
- `aws rds reboot-db-instance --force-failover` triggers a failover and the new primary is in the secondary AZ within 60 seconds.
- Encryption status: `StorageEncrypted: true` with a KMS key ARN.

**Hints**
- DB subnet group requires subnets in at least 2 AZs to enable Multi-AZ.
- Create and attach a custom parameter group before modifying parameters — the default group cannot be modified.
- `max_connections` is a static parameter — it requires a reboot to take effect.
- Monitor failover with `aws rds describe-events --source-type db-instance`.
- The application connection string must use the RDS endpoint (not the IP) — the endpoint is DNS-updated during failover.

**Skills Tested**
- RDS Multi-AZ architecture and failover behavior
- DB subnet groups and private subnet placement
- KMS encryption for RDS
- Parameter groups (static vs dynamic parameters)
- Automated backup and restore concepts

---

### Task I3 — Build a CloudWatch Monitoring and Alerting Stack

**Problem Statement**  
Your production EC2 fleet and RDS instance have no monitoring. Set up a CloudWatch dashboard, custom metrics for memory utilization (not available by default), log groups with metric filters for `ERROR` log lines, alarms that notify an SNS topic, and a composite alarm that fires only when both CPU > 80% AND error rate > 10 per minute simultaneously.

**Expected Outcome**
- CloudWatch Agent installed on EC2 publishing `mem_used_percent` as a custom metric.
- A log group with a metric filter counting lines containing `[ERROR]`.
- Three alarms: `HighCPU` (> 80%), `HighErrorRate` (> 10/min), `CriticalAlert` (composite of both).
- `CriticalAlert` sends an email via SNS only when both conditions are true simultaneously.
- A CloudWatch dashboard with widgets for all metrics in a single view.

**Hints**
- CloudWatch Agent config (`amazon-cloudwatch-agent.json`) defines the `mem_used_percent` metric namespace.
- Use SSM Parameter Store to distribute the agent config: `amazon-cloudwatch-agent-ctl -a fetch-config -s ssm:<parameter-name>`.
- Metric filter pattern for errors: `[timestamp, level=ERROR, ...]` or `{ $.level = "ERROR" }` for JSON logs.
- Composite alarm syntax: `ALARM(HighCPU) AND ALARM(HighErrorRate)`.
- SNS topic subscription confirmation email must be confirmed before alarms can deliver notifications.

**Skills Tested**
- CloudWatch Agent and custom metrics
- Log Groups, metric filters, and log-based alarms
- SNS for alarm notifications
- Composite alarms for correlated incident detection
- CloudWatch dashboard design

---

### Task I4 — Implement a Serverless API with Lambda, API Gateway, and DynamoDB

**Problem Statement**  
Build a serverless REST API for a simple item inventory. `POST /items` creates an item, `GET /items/{id}` retrieves it. Lambda functions handle the logic, API Gateway provides the HTTP interface, and DynamoDB stores the data. The Lambda must use an IAM role (not hardcoded credentials) and API Gateway must require an API key.

**Expected Outcome**
- `curl -X POST -H "x-api-key: <key>" https://<api-id>.execute-api.<region>.amazonaws.com/prod/items -d '{"name":"widget"}'` returns `201` with the new item's ID.
- `curl -H "x-api-key: <key>" https://.../items/<id>` returns the item.
- Requests without the API key return `403 Forbidden`.
- DynamoDB table uses `id` as the partition key (UUID generated in Lambda).
- Lambda logs are visible in CloudWatch with the request ID and response status.

**Hints**
- Lambda IAM role needs `dynamodb:PutItem` and `dynamodb:GetItem` on the specific table ARN only.
- API Gateway usage plan + API key are separate from IAM auth — set method `apiKeyRequired: true`.
- Use `boto3.resource('dynamodb').Table('items')` in Python Lambda; the region is inferred from the execution environment.
- Lambda environment variable `TABLE_NAME` avoids hardcoding — never hardcode resource names.
- Test Lambda locally with `sam local invoke` before deploying (AWS SAM CLI).

**Skills Tested**
- Lambda function + IAM role with DynamoDB permissions
- API Gateway REST API with API key auth
- DynamoDB single-table design basics
- Serverless event-driven architecture
- Environment variables in Lambda

---

### Task I5 — Configure Cross-Account S3 Access with IAM Roles

**Problem Statement**  
Your company has two AWS accounts: `account-a` (data producer) writes logs to an S3 bucket, and `account-b` (data consumer) needs to read those logs. Set up cross-account access so that an EC2 instance in `account-b` can read from the S3 bucket in `account-a` using IAM role assumption — no static credentials.

**Expected Outcome**
- S3 bucket in `account-a` has a bucket policy allowing `sts:AssumeRole` and `s3:GetObject` for the `account-b` role.
- An IAM role in `account-a` (`cross-account-read-role`) has a trust policy allowing `account-b`'s EC2 instance profile to assume it.
- From the EC2 instance in `account-b`: `aws sts assume-role --role-arn arn:aws:iam::<account-a>::role/cross-account-read-role` succeeds.
- `aws s3 ls s3://account-a-bucket --profile assumed` lists objects in the bucket.
- Direct access without assuming the role is denied.

**Hints**
- Trust policy in `account-a` role: `Principal: {"AWS": "arn:aws:iam::<account-b-id>:role/ec2-instance-role"}`.
- The EC2 instance in `account-b` needs `sts:AssumeRole` in its instance profile policy.
- Use `aws sts assume-role` and export `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN` to use temporary credentials.
- Add an `ExternalId` condition in the trust policy as a security best practice for third-party access.
- Bucket policy in `account-a` must explicitly grant access — S3 buckets don't automatically trust cross-account roles.

**Skills Tested**
- Cross-account IAM role assumption
- Trust policies and principal specification
- STS temporary credentials workflow
- S3 bucket policy for cross-account access
- ExternalId for confused deputy prevention

---

## Advanced Tasks

---

### Task A1 — Build a Multi-Region Active-Passive DR Architecture

**Problem Statement**  
Your application runs in `us-east-1`. Design and implement a disaster recovery setup with `eu-west-1` as the standby region. The setup must include: S3 cross-region replication, RDS read replica promotion runbook, Route 53 health-check-based DNS failover, and a target RTO of < 15 minutes.

**Expected Outcome**
- S3 cross-region replication rule configured with status `Enabled`.
- RDS read replica in `eu-west-1` exists and is in `replicating` state.
- Route 53 primary record points to the `us-east-1` ALB with a health check.
- Route 53 secondary record points to the `eu-west-1` ALB with `Failover: SECONDARY`.
- A runbook documents the 5-step failover procedure including promoting the read replica and updating environment config.
- Simulating primary failure (disabling the health check) causes DNS to cut over to `eu-west-1` within 60 seconds.

**Hints**
- S3 CRR requires versioning enabled on both buckets and an IAM replication role.
- Route 53 health check must use the ALB DNS name or an endpoint that reflects actual app health (not just TCP ping).
- RDS read replica promotion is irreversible — the runbook must include creating a new replica after failover.
- Use `aws route53 get-health-check-status` to monitor health check state.
- Test the failover without real traffic using Route 53 Traffic Flow with a test policy.

**Skills Tested**
- Multi-region architecture patterns
- S3 Cross-Region Replication
- RDS read replica and promotion
- Route 53 health checks and DNS failover
- RTO/RPO concepts and DR runbook design

---

### Task A2 — Implement AWS WAF with Custom Rules and Rate Limiting

**Problem Statement**  
Your public API is experiencing scraping attacks, brute-force login attempts, and traffic from known bad IP ranges. Implement AWS WAF in front of your ALB with: an IP block list (your own custom bad IPs), rate limiting at 100 requests per 5 minutes per IP, AWS Managed Rules for common web exploits (OWASP Top 10), and a custom rule that blocks requests with a specific malicious User-Agent string.

**Expected Outcome**
- WAF WebACL associated with the ALB.
- A custom IP set with 5+ blocked IPs — requests from those IPs return `403`.
- Rate-based rule blocks IPs exceeding 100 req/5min — verified with a load test.
- AWS Managed Rule Group `AWSManagedRulesCommonRuleSet` is active with at least one rule in `COUNT` mode for testing before `BLOCK`.
- Custom rule blocks `User-Agent: BadBot/1.0` requests.
- WAF logs are flowing to S3 (or CloudWatch) for forensic analysis.

**Hints**
- WAF association with ALB: `aws wafv2 associate-web-acl --web-acl-arn <arn> --resource-arn <alb-arn>`.
- Start managed rules in `COUNT` mode to see what would be blocked before enabling `BLOCK`.
- Rate-based rules use `RateLimit` and `AggregateKeyType: IP` — minimum rate is 100 per 5 minutes.
- Custom User-Agent rule: `ByteMatchStatement` on `SingleHeader` `user-agent` with `CONTAINS` match.
- WAF logging requires a Kinesis Firehose delivery stream or CloudWatch log group with name starting `aws-waf-logs-`.

**Skills Tested**
- WAF WebACL, rule groups, and rule priority
- IP sets and custom blocking rules
- Rate limiting concepts
- Managed rule groups and COUNT mode evaluation
- WAF logging and forensic analysis

---

### Task A3 — Implement End-to-End CI/CD with CodePipeline, CodeBuild, and ECS

**Problem Statement**  
Build a fully automated CI/CD pipeline: a `git push` to the `main` branch of a CodeCommit repository triggers CodePipeline, which runs CodeBuild to build and push a Docker image to ECR, then deploys it to an ECS Fargate service with a blue/green deployment strategy using CodeDeploy. Failed health checks must automatically trigger a rollback.

**Expected Outcome**
- Pushing to `main` automatically starts the pipeline (no manual trigger).
- CodeBuild produces a Docker image tagged with `$CODEBUILD_RESOLVED_SOURCE_VERSION` and pushes to ECR.
- ECS Fargate service updates with zero downtime via CodeDeploy blue/green.
- A failing health check during deployment triggers automatic rollback within 5 minutes.
- The full pipeline (source → build → deploy) completes in under 10 minutes.
- All pipeline stages and build logs are visible in the AWS Console.

**Hints**
- CodeBuild IAM role needs `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:PutImage`.
- `buildspec.yml` phases: `pre_build` (ECR login), `build` (docker build + tag), `post_build` (docker push + write `imagedefinitions.json`).
- `imagedefinitions.json` format: `[{"name": "<container-name>", "imageUri": "<ecr-uri>:<tag>"}]` — this is what CodePipeline passes to ECS.
- Blue/green requires an ALB with two target groups (Blue + Green) and a CodeDeploy ECS deployment group.
- Set `terminationWaitTimeInMinutes` in the CodeDeploy config to control how long the old (blue) tasks run before termination.

**Skills Tested**
- CodePipeline, CodeBuild, CodeDeploy integration
- ECR image lifecycle and tagging strategy
- ECS Fargate blue/green deployment
- `buildspec.yml` authoring
- Automated rollback on health check failure

---

### Task A4 — Design a Cost Optimization Strategy for an Overprovisioned AWS Account

**Problem Statement**  
An AWS account is spending $15,000/month. After reviewing with AWS Cost Explorer, you identify: EC2 instances running 24/7 that only need business hours, over-provisioned RDS instances, S3 buckets with no lifecycle policies, unused Elastic IPs, and no Reserved Instances or Savings Plans. Implement a cost optimization plan and demonstrate savings.

**Expected Outcome**
- EC2 instances scheduled to stop at 6pm and start at 8am on weekdays via EventBridge + Lambda.
- RDS instance right-sized (e.g., `db.r5.2xlarge` → `db.r5.large`) after analyzing CloudWatch CPU/memory metrics.
- S3 lifecycle policies applied to move old objects to Glacier and expire after 365 days.
- Unused Elastic IPs released (each unused EIP costs ~$3.60/month).
- A Savings Plans purchase recommendation reviewed in Cost Explorer and committed.
- AWS Budgets alert configured at 80% and 100% of the monthly target.

**Hints**
- EventBridge scheduler: `cron(0 18 ? * MON-FRI *)` for 6pm stop, `cron(0 8 ? * MON-FRI *)` for 8am start.
- Use `aws ce get-rightsizing-recommendation` for EC2 rightsizing suggestions.
- `aws ec2 describe-addresses --filters "Name=association-id,Values="` finds unassociated (unused) Elastic IPs.
- Compute Savings Plans are more flexible than EC2 Instance Savings Plans — understand the trade-off.
- Use S3 Storage Lens for cross-account, cross-region S3 cost visibility.

**Skills Tested**
- AWS cost optimization pillars (right-sizing, scheduling, savings plans, lifecycle policies)
- EventBridge + Lambda for scheduled automation
- Cost Explorer and Budgets
- Reserved Instances vs Savings Plans vs On-Demand trade-offs
- FinOps mindset

---

## Troubleshooting & Debugging Tasks

---

### Task T1 — Debug EC2 Instance Connectivity Issues

**Problem Statement**  
An EC2 instance is running but you cannot SSH into it and the web application is unreachable. The instance was working yesterday. Systematically diagnose and fix the issue using only AWS Console and CLI — no direct server access yet.

**Expected Outcome**
- Root cause identified from a checklist of possible causes.
- Connectivity restored — `ssh ec2-user@<ip>` and `curl http://<ip>` both succeed.
- Root cause documented (e.g., security group rule removed, route table detached, instance ran out of disk space causing the process to crash).

**Hints**
- Work outward-in: internet → VPC → subnet → security group → instance.
- Check security group inbound rules: were rules accidentally removed?
- Check route tables: is the public subnet still associated with a route to the Internet Gateway?
- Check Network ACL: NACLs are stateless — both inbound and outbound rules must allow traffic.
- Use EC2 Instance Connect (browser-based SSH) if key is lost — bypasses network entirely.
- Use `aws ec2 get-console-output --instance-id` and `aws ec2 get-console-screenshot` to see what the instance is doing without SSH.
- Check instance status checks: `System status check` failure = AWS infrastructure issue (stop/start moves to healthy hardware).

**Skills Tested**
- Layered network troubleshooting methodology
- Security groups vs Network ACLs (stateful vs stateless)
- Route table and Internet Gateway associations
- EC2 status checks and recovery options
- `ec2-instance-connect` and console output as alternative access methods

---

### Task T2 — Investigate and Fix an IAM Permission Denied Error

**Problem Statement**  
A Lambda function is failing with `AccessDeniedException: User is not authorized to perform: s3:GetObject on resource arn:aws:s3:::prod-data/sensitive/report.csv`. The Lambda's IAM role appears to have an `s3:GetObject` policy attached. Find all the reasons this could still fail and fix the actual cause.

**Expected Outcome**
- Root cause identified — it is one (or more) of: wrong resource ARN in the policy, S3 bucket policy explicitly denying the role, SCP blocking the action, missing `s3:GetObject` on the bucket itself (vs just the prefix), or KMS key policy not granting decrypt permission.
- After fix, Lambda successfully reads the file.
- `aws iam simulate-principal-policy` used to validate the fix before deploying.

**Hints**
- IAM evaluation logic: explicit `Deny` always wins. Check bucket policy for explicit denies.
- S3 resource ARN in IAM policies: `arn:aws:s3:::bucket-name` (bucket) vs `arn:aws:s3:::bucket-name/*` (objects) — you need **both** for `ListBucket` + `GetObject`.
- If the object is encrypted with a customer-managed KMS key, the Lambda role also needs `kms:Decrypt` on that key.
- Check if an SCP at the AWS Organization level blocks the action: `aws organizations list-policies-for-target`.
- Use CloudTrail to see the exact failed API call and reason: filter `eventName=GetObject, errorCode=AccessDenied`.

**Skills Tested**
- IAM policy evaluation order (Deny > Allow)
- S3 + KMS combined permission requirements
- SCP and permission boundary interaction
- CloudTrail for access investigation
- `simulate-principal-policy` for pre-flight permission testing

---

### Task T3 — Diagnose High RDS CPU and Slow Query Performance

**Problem Statement**  
The RDS PostgreSQL instance CPU has been at 95%+ for 6 hours. Application response times have degraded from 200ms to 8 seconds. The on-call engineer needs to identify the slow queries, determine whether to scale or optimize, and restore normal performance without data loss.

**Expected Outcome**
- Slow queries identified using Performance Insights or `pg_stat_activity`.
- At least one query fixed with a missing index (verified with `EXPLAIN ANALYZE`).
- RDS CPU returns below 40% after optimization.
- `pg_stat_statements` enabled via a custom parameter group to track query statistics.
- A decision made on read replica vs vertical scaling based on workload type (read-heavy vs write-heavy).

**Hints**
- Enable Performance Insights on the RDS instance — it shows top SQL queries by load with no app changes needed.
- Connect to RDS via a bastion host and run: `SELECT query, calls, total_time, mean_time FROM pg_stat_statements ORDER BY mean_time DESC LIMIT 10;`
- `EXPLAIN ANALYZE SELECT ...` shows the query plan — look for `Seq Scan` on large tables (missing index).
- Create the index with `CREATE INDEX CONCURRENTLY` to avoid locking the table.
- If the workload is read-heavy, adding an RDS Read Replica and updating the app's read connection string offloads query load without downtime.

**Skills Tested**
- RDS Performance Insights and slow query diagnosis
- PostgreSQL `pg_stat_statements` and `EXPLAIN ANALYZE`
- Index creation without table locks
- Vertical scaling vs read replica decision logic
- Database performance tuning fundamentals

---

### Task T4 — Fix a Lambda Function That Is Timing Out

**Problem Statement**  
A Lambda function that processes S3 events is consistently timing out at the 3-second default timeout. The function downloads a file from S3, transforms it, and writes the result to DynamoDB. It works fine for small files but fails for files > 5MB. Fix the performance and configuration issues.

**Expected Outcome**
- Lambda timeout increased appropriately (not just set to max — explain the right value).
- Memory increased from the default 128MB to a value that also increases CPU allocation.
- Cold start time reduced by moving the S3 client initialization outside the handler function.
- Lambda processes a 50MB file successfully within the new timeout.
- X-Ray tracing enabled and the trace shows which segment (S3 download, transform, DynamoDB write) was slowest.

**Hints**
- Lambda CPU is proportional to memory — increasing from 128MB to 1024MB also gives 8× more CPU.
- Move `boto3.client('s3')` outside the handler to reuse the connection across warm invocations.
- Use `boto3` streaming: `s3.get_object()['Body']` returns a stream — process it without loading the entire file into memory.
- Enable X-Ray: `aws lambda update-function-configuration --tracing-config Mode=Active`.
- Set timeout to `(expected_max_file_size / average_throughput) * 1.5` — not just the maximum.
- Check Lambda's `Max Memory Used` in CloudWatch Logs to right-size memory.

**Skills Tested**
- Lambda timeout and memory configuration
- Cold start optimization (connection reuse outside handler)
- Streaming large file processing in Lambda
- AWS X-Ray distributed tracing
- Lambda performance tuning methodology

---

### Task T5 — Investigate a Cost Spike Using CloudTrail and Cost Explorer

**Problem Statement**  
Your AWS bill jumped 40% unexpectedly this month. Cost Explorer shows the spike is in EC2 and data transfer costs in `ap-southeast-1` — a region your team never uses. Investigate whether this is unauthorized usage, a misconfiguration, or a compromised credential. Contain the threat and remediate.

**Expected Outcome**
- CloudTrail investigation identifies the IAM principal that created resources in `ap-southeast-1`.
- If credentials are compromised: the access key is rotated immediately, the IAM user is disabled, and active sessions are revoked.
- All unauthorized EC2 instances are terminated.
- GuardDuty findings reviewed for `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration`.
- An SCP is created to deny resource creation in non-approved regions going forward.
- Incident report written with timeline and remediation steps.

**Hints**
- CloudTrail: filter by `eventSource=ec2.amazonaws.com`, `awsRegion=ap-southeast-1`, `eventName=RunInstances`.
- `aws iam list-access-keys --user-name <user>` shows key creation date — compare to CloudTrail event time.
- Revoke all active sessions: `aws iam delete-role-policy` and detach all policies, then rotate the key.
- GuardDuty finding `CryptoCurrency:EC2/BitcoinTool.B` indicates cryptomining — a common attack pattern.
- SCP to restrict regions: `"Condition": {"StringNotEquals": {"aws:RequestedRegion": ["us-east-1", "eu-west-1"]}}`.

**Skills Tested**
- CloudTrail forensic investigation
- IAM incident response (key rotation, session revocation)
- GuardDuty threat detection
- SCP for preventive controls
- Cloud security incident response process

---

## Cloud Architecture Design Tasks

---

### Task D1 — Design a Secure Three-Tier Web Architecture

**Problem Statement**  
Design and deploy a production-ready three-tier architecture: a public-facing ALB serving a web tier (EC2 ASG), an application tier (EC2 ASG, private subnets), and a data tier (RDS Multi-AZ, isolated subnets). Each tier must only accept traffic from the tier above it. No tier should have internet access except the web tier. Implement VPC flow logs for security monitoring.

**Expected Outcome**
- ALB in public subnets accepts HTTPS (port 443) from `0.0.0.0/0`.
- Web tier security group: only allows `443` from ALB security group (not CIDR).
- App tier security group: only allows `8080` from web tier security group.
- RDS security group: only allows `5432` from app tier security group.
- No security group uses `0.0.0.0/0` on any port except the ALB on port 443.
- VPC flow logs enabled and flowing to CloudWatch Logs.

**Hints**
- Reference security groups as sources (not CIDRs) in inbound rules — this is more maintainable and doesn't break when IPs change.
- App tier and DB tier instances need NAT Gateway to download updates — place them in private subnets with route to NAT.
- Use AWS Secrets Manager (not env vars) to pass the DB password to the app tier.
- VPC flow logs: create a CloudWatch log group and IAM role, then `aws ec2 create-flow-logs`.
- Enable HTTPS on the ALB using ACM (AWS Certificate Manager) — never terminate SSL on EC2 in production.

**Skills Tested**
- Three-tier architecture design
- Security group chaining (SG-to-SG references)
- Network isolation per tier
- ACM and HTTPS termination at ALB
- VPC Flow Logs for security visibility

---

### Task D2 — Build an Event-Driven Data Pipeline with S3, Lambda, SQS, and SNS

**Problem Statement**  
Design and implement an event-driven pipeline: CSV files uploaded to S3 trigger a Lambda that validates and transforms the data, publishes results to an SQS queue, and a second Lambda consumes the queue to write records to DynamoDB. Failed validations must publish to an SNS topic for alerting. The pipeline must be idempotent and handle duplicate S3 events.

**Expected Outcome**
- S3 `ObjectCreated` event triggers Lambda 1 within 10 seconds of upload.
- Lambda 1 validates the CSV schema, transforms rows to JSON, and sends each row as an SQS message.
- Lambda 2 is triggered by SQS, processes each message, and writes to DynamoDB with a `PutItem` using the row's natural key (idempotent — same key doesn't create duplicates).
- Invalid CSVs publish to SNS, sending an alert email.
- SQS Dead Letter Queue captures messages that fail Lambda 2 processing after 3 retries.

**Hints**
- S3 → Lambda trigger: `aws lambda add-permission` + S3 event notification (`put-bucket-notification-configuration`).
- Idempotency: DynamoDB `PutItem` with a condition expression (`attribute_not_exists(id)`) or accept that `PutItem` overwrites with the same data (idempotent for same-content records).
- SQS Lambda trigger with `batchSize: 10` and `bisectBatchOnFunctionError: true` for DLQ routing of specific failing messages.
- DLQ: set `maxReceiveCount: 3` on the SQS queue's redrive policy pointing to a `-dlq` queue.
- Use SQS FIFO queue if ordering of records from the same CSV matters.

**Skills Tested**
- Event-driven architecture patterns
- S3 event notifications and Lambda triggers
- SQS + Lambda event source mapping
- Dead Letter Queue for error handling
- Idempotency design in serverless pipelines

---

### Task D3 — Implement Zero-Trust Network Access with AWS PrivateLink and VPC Endpoints

**Problem Statement**  
Your application in a private VPC currently calls AWS services (S3, DynamoDB, SQS) via the public internet through a NAT Gateway, incurring data transfer costs and exposing traffic to the internet. Migrate all AWS service calls to use VPC Endpoints (Gateway for S3/DynamoDB, Interface for SQS) and block internet access for the Lambda functions and EC2 instances entirely.

**Expected Outcome**
- Gateway VPC endpoints for S3 and DynamoDB created and route tables updated.
- Interface VPC endpoints for SQS, CloudWatch, and Secrets Manager created in private subnets.
- NAT Gateway removed (or confirmed unused for AWS service traffic) — data transfer costs from NAT drop to near zero.
- Lambda functions in the VPC can call S3, DynamoDB, and SQS without NAT.
- `aws ec2 describe-vpc-endpoints` confirms all endpoints are `available`.
- Security groups on interface endpoints restrict access to only the application security group.

**Hints**
- Gateway endpoints (S3, DynamoDB) are free and modify route tables — no ENI, no security group needed.
- Interface endpoints (SQS, etc.) cost ~$7/month/AZ but keep traffic on AWS network.
- Lambda in a VPC: attach the Lambda to private subnets — remove any NAT association once VPC endpoints are in place.
- Test: from a Lambda in the VPC, `boto3.client('s3', endpoint_url='https://bucket.vpce-<id>.s3.<region>.vpce.amazonaws.com')` — or just confirm NAT traffic drops to 0 in VPC flow logs.
- VPC endpoint policies can restrict which S3 buckets can be accessed from the VPC — an extra security layer.

**Skills Tested**
- VPC Endpoints (Gateway vs Interface distinction)
- Cost reduction via private AWS service connectivity
- Zero-trust network design
- Lambda VPC configuration
- VPC endpoint policies for service-level access control

---

## Quick Reference — Key AWS CLI Commands

| Command | Purpose |
|---|---|
| `aws ec2 describe-instances --query 'Reservations[].Instances[].[InstanceId,State.Name,PublicIpAddress]' --output table` | List instances with state and IP |
| `aws ec2 get-console-output --instance-id <id>` | Get EC2 serial console output |
| `aws s3api get-bucket-policy --bucket <name>` | Read S3 bucket policy |
| `aws iam simulate-principal-policy --policy-source-arn <role-arn> --action-names s3:GetObject --resource-arns <s3-arn>` | Test IAM permissions |
| `aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances` | Find who launched EC2 instances |
| `aws rds describe-events --source-type db-instance --duration 60` | Recent RDS events |
| `aws logs filter-log-events --log-group-name <name> --filter-pattern ERROR` | Search CloudWatch logs for errors |
| `aws ce get-cost-and-usage --time-period Start=2026-06-01,End=2026-06-30 --granularity MONTHLY --metrics BlendedCost` | Monthly cost breakdown |
| `aws sts get-caller-identity` | Show current AWS identity |
| `aws ec2 describe-security-groups --filters Name=group-name,Values=<name>` | Inspect security group rules |
| `aws autoscaling describe-scaling-activities --auto-scaling-group-name <name>` | View ASG scaling history |
| `aws wafv2 list-web-acls --scope REGIONAL` | List WAF WebACLs |
| `aws lambda invoke --function-name <name> --payload '{}' output.json` | Invoke Lambda manually |
| `aws route53 list-health-checks` | List Route 53 health checks |
| `aws organizations list-policies --filter SERVICE_CONTROL_POLICY` | List SCPs |

---

## Interview Cheat Sheet

- **"What is the difference between a security group and a Network ACL?"**  
  Security groups are stateful (return traffic is automatically allowed), attached to ENIs, and only support allow rules. NACLs are stateless (both directions must be explicitly allowed), attached to subnets, support both allow and deny rules, and are evaluated in rule-number order.

- **"Explain IAM roles vs IAM users."**  
  IAM users have permanent credentials (access key + secret). IAM roles are assumed temporarily via STS — they issue short-lived tokens. Always use roles for EC2, Lambda, and cross-account access. Never use IAM users for programmatic access on AWS services.

- **"What is the difference between a NAT Gateway and an Internet Gateway?"**  
  Internet Gateway enables two-way internet access for resources with public IPs. NAT Gateway enables outbound-only internet access for resources in private subnets — they can initiate connections out but cannot receive inbound connections from the internet.

- **"How does S3 versioning help with accidental deletion?"**  
  With versioning enabled, `DELETE` adds a delete marker rather than removing the object. Previous versions remain and can be restored by deleting the delete marker. Use MFA Delete for extra protection on critical buckets.

- **"Explain RDS Multi-AZ vs Read Replicas."**  
  Multi-AZ is for high availability (failover) — the standby is synchronously replicated but cannot serve read traffic. Read Replicas are for read scaling — they're asynchronously replicated and can serve read queries, but lag behind the primary. Multi-AZ failover is automatic; Read Replica promotion is manual.

- **"What is an SCP and how does it differ from an IAM policy?"**  
  SCPs (Service Control Policies) are applied at the AWS Organizations level and set the maximum permissions boundary for all accounts in an OU. IAM policies grant permissions within that boundary. SCPs don't grant permissions — they only restrict. An action requires both an SCP allow and an IAM allow.

- **"When would you use SQS vs SNS vs EventBridge?"**  
  SQS: point-to-point async messaging, decoupling, backpressure handling. SNS: fan-out pub/sub to multiple subscribers. EventBridge: event routing based on rules, integrates 200+ AWS services and SaaS events, schema registry. For complex routing logic, use EventBridge. For simple fan-out, use SNS. For work queues with retries, use SQS.

- **"How do you rotate IAM access keys without downtime?"**  
  1. Create new access key. 2. Update the application to use the new key. 3. Verify the new key works. 4. Deactivate the old key (not delete — allows reverting if needed). 5. Monitor for any access denied errors. 6. Delete the old key after 24-48 hours.

---

## Study Path

```
Week 1  → B1, B2, B3, B4         (EC2, S3, IAM, VPC fundamentals)
Week 2  → I1, I2, I3             (ASG/ALB, RDS Multi-AZ, CloudWatch)
Week 3  → I4, I5, T1, T2         (Serverless, cross-account, connectivity debugging)
Week 4  → T3, T4, T5             (RDS performance, Lambda tuning, security incident)
Week 5  → A1, A2, A3             (DR, WAF, CI/CD pipeline)
Week 6  → A4, D1, D2, D3         (Cost optimization, architecture design tasks)
```

---

*Use the AWS Free Tier for every task where possible. For tasks requiring paid services (NAT Gateway, RDS Multi-AZ), use `terraform destroy` or stop/delete resources immediately after the task to minimize cost. Always set a Budget Alert at $20/month before starting.*
