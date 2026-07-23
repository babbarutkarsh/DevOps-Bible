---
title: AWS
nav_order: 40
description: "AWS — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# AWS — DevOps Interview Preparation

> **Question frequency:** 🔥 very frequently asked · ⭐ important · 💡 good to know

## Table of Contents
- [Core Services Overview](#core-services-overview)
- [IAM & Security](#iam--security)
- [VPC & Networking](#vpc--networking)
- [EC2 & Compute](#ec2--compute)
- [ECS & EKS (Container Services)](#ecs--eks)
- [S3 & Storage](#s3--storage)
- [RDS & Databases](#rds--databases)
- [Route 53 (DNS)](#route-53)
- [Load Balancing (ELB)](#load-balancing)
- [Auto Scaling](#auto-scaling)
- [Lambda & Serverless](#lambda--serverless)
- [CloudWatch & Monitoring](#cloudwatch--monitoring)
- [SQS, SNS, EventBridge](#sqs-sns-eventbridge)
- [Secrets Manager & Parameter Store](#secrets-manager--parameter-store)
- [CloudFormation & IaC](#cloudformation--iac)
- [Cost Optimization & FinOps](#cost-optimization--finops)
- [Multi-Account & Organizations](#multi-account--organizations)
- [Well-Architected Framework](#well-architected-framework)
- [Scenario-Based Questions](#scenario-based-questions)
- [Key Resources](#key-resources)

---

## Core Services Overview

### 🔥 Q: Map AWS services to DevOps needs.

| Need | AWS Service |
|------|------------|
| **Compute** | EC2, ECS, EKS, Lambda, Fargate |
| **Networking** | VPC, ALB/NLB, Route 53, CloudFront, API Gateway |
| **Storage** | S3, EBS, EFS, FSx |
| **Database** | RDS, Aurora, DynamoDB, ElastiCache, DocumentDB |
| **CI/CD** | CodePipeline, CodeBuild, CodeDeploy |
| **IaC** | CloudFormation, CDK, Terraform |
| **Monitoring** | CloudWatch, X-Ray, CloudTrail |
| **Security** | IAM, KMS, Secrets Manager, WAF, GuardDuty, Security Hub |
| **Messaging** | SQS, SNS, EventBridge, Kinesis |
| **Containers** | ECR, ECS, EKS, Fargate, App Runner |

---

## IAM & Security

### 🔥 Q: Explain IAM components.

```
AWS Account
├── Users          (people — long-term credentials)
├── Groups         (collection of users)
├── Roles          (assumed by services/users — temporary credentials)
├── Policies       (JSON documents defining permissions)
│   ├── AWS Managed   (predefined by AWS)
│   ├── Customer Managed (you create, reusable)
│   └── Inline         (embedded in user/group/role)
└── Identity Providers (SSO, SAML, OIDC)
```

### 🔥 Q: Explain IAM policy evaluation logic.

```
Request comes in
    │
    ▼
1. Is there an explicit DENY?  ──► YES ──► DENIED
    │
    NO
    ▼
2. Is there an explicit ALLOW? ──► YES ──► ALLOWED
    │
    NO
    ▼
3. DENIED (implicit deny — default)
```

**Policy example:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-bucket/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "10.0.0.0/8"
        }
      }
    }
  ]
}
```

### ⭐ Q: Roles vs Users vs Groups — when to use which?

| Type | Use Case | Credentials | Best For |
|------|----------|-------------|----------|
| **Users** | Humans (console/CLI access) | Long-term (access keys) | Individual employees |
| **Groups** | Collection of users | Inherited from attached policies | Organizing permissions by team |
| **Roles** | Services, cross-account, federated | **Temporary (STS)** | EC2, Lambda, cross-account, SSO |

**Key:** Roles = temporary credentials = safer. Services should NEVER use IAM users.

### ⭐ Q: How does AssumeRole work?

```
1. Principal (user/service) calls sts:AssumeRole on target role
2. Checks trust policy on role — does it allow this principal?
3. STS returns temporary credentials (access key, secret, session token)
4. Principal uses temp creds (valid 15min–12hr)
5. Temp creds expire automatically
```

**Trust policy (who can assume):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Service": "ec2.amazonaws.com"
    },
    "Action": "sts:AssumeRole"
  }]
}
```

**Permissions policy (what the role can do):** Attached separately as managed/inline policy.

### 💡 Q: IAM Roles for Service Accounts (IRSA) in EKS — how does it work?

```
EKS Pod → ServiceAccount (annotated with IAM role ARN)
    │
    ▼
Pod gets env vars: AWS_ROLE_ARN, AWS_WEB_IDENTITY_TOKEN_FILE
    │
    ▼
AWS SDK reads token → calls sts:AssumeRoleWithWebIdentity
    │
    ▼
Returns temp credentials scoped to pod
```

**ServiceAccount manifest:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/my-app-role
```

**Why IRSA > node IAM role:** Fine-grained per-pod permissions, no shared credentials, follows least privilege.

### 🔥 Q: IAM best practices.

- **Root account:** MFA, no access keys, use only for billing/emergency
- **Least privilege:** Start with minimal permissions, add as needed
- **Use roles, not users** for services (EC2, Lambda, ECS, EKS pods via IRSA)
- **Use OIDC/SAML** for SSO (avoid IAM users for employees)
- **Use OIDC for CI/CD** (GitHub Actions → AssumeRoleWithWebIdentity, no long-lived keys)
- **Enable CloudTrail** for auditing all API calls
- **Use SCPs** (Service Control Policies) in AWS Organizations for guardrails
- **Rotate credentials** regularly (automated via Secrets Manager for DB/API keys)
- **Use Permission Boundaries** to delegate admin safely
- **Enforce MFA** via Condition in policies: `"Condition": {"Bool": {"aws:MultiFactorAuthPresent": "true"}}`
- **Use IMDSv2** on EC2 (token-based, mitigates SSRF)

### ⭐ Q: What is the difference between SCP, Permission Boundary, and IAM Policy?

| Type | Scope | Purpose |
|------|-------|---------|
| **SCP** | Organization/OU/Account | Maximum permissions for an account |
| **Permission Boundary** | User/Role | Maximum permissions for a specific principal |
| **IAM Policy** | User/Group/Role | Actual granted permissions |

Effective permissions = SCP ∩ Permission Boundary ∩ IAM Policy

---

## VPC & Networking

### 🔥 Q: Design a production VPC.

```
┌──────────────────── VPC (10.0.0.0/16) ────────────────────┐
│                                                             │
│  ┌─── AZ-a ──────────────┐  ┌─── AZ-b ──────────────┐    │
│  │  Public  10.0.1.0/24  │  │  Public  10.0.3.0/24  │    │
│  │  (NAT GW, Bastion,ALB)│  │  (NAT GW, ALB)        │    │
│  ├────────────────────────┤  ├────────────────────────┤    │
│  │  Private 10.0.2.0/24  │  │  Private 10.0.4.0/24  │    │
│  │  (App servers, ECS)    │  │  (App servers, ECS)    │    │
│  ├────────────────────────┤  ├────────────────────────┤    │
│  │  Data    10.0.10.0/24 │  │  Data    10.0.11.0/24 │    │
│  │  (RDS, ElastiCache)    │  │  (RDS replica)         │    │
│  └────────────────────────┘  └────────────────────────┘    │
│                                                             │
│  IGW ←→ Public Route Table (0.0.0.0/0 → IGW)              │
│  NAT ←→ Private Route Table (0.0.0.0/0 → NAT GW)         │
│  Data subnets: No internet route                            │
└─────────────────────────────────────────────────────────────┘
```

### 🔥 Q: Security Groups vs NACLs.

| Feature | Security Groups | NACLs |
|---------|----------------|-------|
| Level | ENI (instance) | Subnet |
| State | **Stateful** | **Stateless** |
| Rules | Allow only | Allow & Deny |
| Evaluation | All rules | Ordered by rule number |
| Default | Deny all in, allow all out | Allow all |

### ⭐ Q: VPC Endpoints (Interface vs Gateway).

| Type | How It Works | Services | Cost |
|------|-------------|----------|------|
| **Gateway Endpoint** | Route table entry, no ENI | S3, DynamoDB | Free |
| **Interface Endpoint (PrivateLink)** | ENI in subnet, private IP | Most AWS services, SaaS | $0.01/hr/AZ + data |

**Why use VPC endpoints?** Avoid NAT Gateway data processing charges ($0.045/GB), improve security (traffic stays in AWS network), reduce latency.

### ⭐ Q: VPC Peering vs Transit Gateway vs PrivateLink?

| Feature | VPC Peering | Transit Gateway | PrivateLink |
|---------|------------|-----------------|-------------|
| Topology | 1:1 | Hub-and-spoke (centralized) | Service endpoint (provider-consumer) |
| Transitive | **No** | **Yes** | N/A |
| Cross-account | Yes | Yes | Yes |
| Cross-region | Yes | Yes (Inter-Region TGW Peering) | Yes |
| Use case | Few VPCs | Many VPCs (>10), on-prem | Expose service privately (SaaS, microservices) |
| Cost | Data transfer only | Per attachment ($36/mo) + data | Per endpoint-hour ($7.3/mo/AZ) + data |
| Max VPCs | 125 peerings per VPC | 5000 attachments | Unlimited |

**Interview tip:** "We have 50 VPCs" → Transit Gateway. "Expose API to partners without internet" → PrivateLink.

### 🔥 Q: How do private subnets access the internet?

**NAT Gateway** (managed) or **NAT Instance** (self-managed EC2).

```
Private Subnet Instance → NAT Gateway (in public subnet) → Internet Gateway → Internet
```

- NAT GW: Managed, HA within AZ, auto-scales, ~$32/month + $0.045/GB data processing
- Deploy one NAT GW per AZ for high availability
- For cost savings in non-prod: single NAT GW or NAT Instance

**NAT Gateway vs IGW:**
- **IGW** = bidirectional, for public subnets, free (data transfer charges apply)
- **NAT GW** = unidirectional outbound, for private subnets, $32/mo + $0.045/GB

### ⭐ Q: Debugging VPC connectivity — systematic approach.

```
Problem: Service A in VPC-1 can't reach Service B in VPC-2

1. Network path exists?
   - VPC Peering / Transit Gateway / VPN established?
   - Route tables have routes to destination CIDR?

2. Security Groups (stateful, allow-only)
   - Source SG allows outbound to destination?
   - Destination SG allows inbound from source?

3. NACLs (stateless, ordered rules)
   - Outbound NACL on source subnet allows traffic?
   - Inbound NACL on destination subnet allows traffic?
   - Return traffic allowed (both directions for stateless)?

4. Destination service
   - Application listening on expected port?
   - Check CloudWatch Logs, VPC Flow Logs

Tool: VPC Reachability Analyzer (built-in troubleshooting tool)
```

---

## EC2 & Compute

### ⭐ Q: EC2 instance types to know.

| Family | Optimized For | Use Case |
|--------|--------------|----------|
| **t3/t4g** | Burstable | Dev/test, light workloads |
| **m5/m6i/m7i** | General purpose | Web servers, app servers |
| **c5/c6i/c7i** | Compute | Batch processing, ML inference |
| **r5/r6i/r7i** | Memory | Databases, caching |
| **i3/i4i** | Storage | NoSQL, data warehousing |
| **g5/p5** | GPU | ML training, video processing |

### 🔥 Q: EC2 purchasing options.

| Option | Discount | Commitment | Use Case |
|--------|----------|------------|----------|
| **On-Demand** | None | None | Unpredictable workloads |
| **Reserved (RI)** | Up to 72% | 1 or 3 years | Steady-state workloads |
| **Savings Plans** | Up to 72% | 1 or 3 years ($ commitment) | Flexible across instance types |
| **Spot** | Up to 90% | None (can be interrupted) | Batch, CI/CD, fault-tolerant |
| **Dedicated** | Varies | Per host | Compliance, licensing |

### ⭐ Q: What is EC2 instance metadata?

```bash
# IMDSv2 (recommended — requires token)
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id

# Available metadata: instance-id, ami-id, hostname, public-ipv4,
# iam/security-credentials/<role-name>, placement/availability-zone
```

**Security:** Always enforce IMDSv2 (token-based) — IMDSv1 is vulnerable to SSRF attacks.

---

## ECS & EKS

### 🔥 Q: ECS vs EKS vs Fargate — when to use which?

| Feature | ECS | EKS | Fargate (launch type) |
|---------|-----|-----|-----------------------|
| Orchestrator | AWS-proprietary | Kubernetes | Serverless (works with both ECS/EKS) |
| Learning curve | Lower | Higher | Lowest |
| Portability | AWS only | Multi-cloud | AWS only |
| Ecosystem | AWS integrations | K8s ecosystem (Helm, operators) | Limited (no daemonsets, host volumes) |
| Control plane cost | Free | $0.10/hr ($73/month) | Free (ECS) / $73/mo (EKS) |
| Server management | EC2 launch type: yes | Node groups: yes | **None** |
| Best for | Simpler apps, AWS-native | Complex microservices, K8s skills | Quick wins, no infra overhead |

### ⭐ Q: ECS architecture.

```
┌─── ECS Cluster ──────────────────────────────────┐
│                                                    │
│  ┌─── Service ─────────────────────────────────┐  │
│  │  Task Definition (container image, CPU, mem) │  │
│  │                                               │  │
│  │  ┌── Task ──┐ ┌── Task ──┐ ┌── Task ──┐    │  │
│  │  │Container │ │Container │ │Container │    │  │
│  │  │(Fargate) │ │(Fargate) │ │(EC2)     │    │  │
│  │  └──────────┘ └──────────┘ └──────────┘    │  │
│  └───────────────────────────────────────────────┘  │
│                                                    │
│  Launch Types: Fargate (serverless) or EC2          │
└────────────────────────────────────────────────────┘
```

**Fargate vs EC2 launch type:**

| Feature | Fargate | EC2 |
|---------|---------|-----|
| Server management | None (serverless) | You manage instances |
| Pricing | Per vCPU/memory/second | EC2 instance pricing |
| GPU support | No | Yes |
| Customization | Limited | Full control |
| Best for | Most workloads | GPU, specific instance types, cost optimization |

### ⭐ Q: EKS deep dive — common interview topics.

**EKS architecture:**
```
┌─── EKS Control Plane (AWS-managed, Multi-AZ) ───┐
│  API Server, etcd, Controller Manager, Scheduler │
└────────────────┬──────────────────────────────────┘
                 │
        ┌────────┴────────┐
        │                 │
  ┌─── Managed Node Groups ───┐   ┌─── Fargate ───┐
  │  EC2 Auto Scaling Groups  │   │  Serverless   │
  │  (m5.large, Spot, etc.)   │   │  pods         │
  └───────────────────────────┘   └───────────────┘
```

**Node types:**
- **Managed Node Groups** — AWS provisions/updates EC2 ASGs, you pay EC2 cost
- **Self-Managed Nodes** — Full control, use Spot, custom AMI, kops-like
- **Fargate** — Serverless, no node management, per-pod pricing

**IRSA (IAM Roles for Service Accounts)** — Covered above; enables pod-level IAM.

**Karpenter (2024-2026 trend)** — Open-source K8s autoscaler from AWS, replaces Cluster Autoscaler:
- **Faster scaling** (provisions nodes in seconds, not minutes)
- **Cost-aware** (mixes Spot, on-demand, instance types automatically)
- **Bin packing** (consolidates underutilized nodes)
- **No node groups** needed (Karpenter provisions directly)

**EKS + ALB Ingress Controller** — Provisions ALB from K8s Ingress resources.

**Interview Q:** "We need to scale EKS nodes faster and reduce cost" → **Karpenter**.

---

## S3 & Storage

### 🔥 Q: S3 storage classes.

| Class | Availability | Min Duration | Use Case |
|-------|-------------|-------------|----------|
| **Standard** | 99.99% | None | Frequently accessed |
| **Intelligent-Tiering** | 99.9% | None | Unknown access patterns |
| **Standard-IA** | 99.9% | 30 days | Infrequent access |
| **One Zone-IA** | 99.5% | 30 days | Re-creatable data |
| **Glacier Instant** | 99.9% | 90 days | Archive, ms retrieval |
| **Glacier Flexible** | 99.99% | 90 days | Archive, min-hr retrieval |
| **Glacier Deep Archive** | 99.99% | 180 days | Long-term, 12-48hr retrieval |

### 🔥 Q: S3 security.

```
Block Public Access (account + bucket level)
├── Bucket Policy (resource-based)
├── IAM Policy (identity-based)
├── ACLs (legacy — avoid)
├── Encryption:
│   ├── SSE-S3 (default, AWS-managed keys)
│   ├── SSE-KMS (customer-managed KMS keys, audit trail)
│   └── SSE-C (customer-provided keys)
├── VPC Endpoints (access from VPC without internet)
├── Object Lock (WORM compliance)
└── Versioning (protect against accidental deletion)
```

### ⭐ Q: S3 consistency model.

- **Strong read-after-write consistency** (since Dec 2020) for all operations
- PUTs/DELETEs are immediately consistent — no eventual consistency lag
- Overwrite or delete → subsequent read reflects change immediately

### ⭐ Q: S3 performance optimization.

- **Multipart upload** for files >100MB (required >5GB)
- **S3 Transfer Acceleration** for cross-region uploads (uses CloudFront edge locations)
- **Prefix distribution** — S3 supports 3,500 PUT/5,500 GET per prefix/sec
- **S3 Select / Glacier Select** — Query in-place (SQL over CSV/JSON) without downloading entire object
- **Byte-range fetches** — Download specific portions of objects (parallel downloads)

### 🔥 Q: How would you secure an S3 bucket?

```
1. Block Public Access — Enable at account + bucket level (default since 2023)
2. Bucket Policy — Restrict to specific IAM roles/VPCs
   Example: Allow access only from VPC endpoint
   {
     "Condition": {
       "StringEquals": {
         "aws:SourceVpce": "vpce-12345"
       }
     }
   }
3. Encryption at rest — SSE-S3 (default), SSE-KMS (audit trail), or SSE-C
4. Encryption in transit — Enforce HTTPS:
   "Condition": {"Bool": {"aws:SecureTransport": "false"}}, "Effect": "Deny"
5. Versioning — Protect against accidental deletion
6. Object Lock — WORM (compliance/governance mode)
7. MFA Delete — Require MFA to delete versions
8. Logging — Enable S3 access logs or CloudTrail data events
9. VPC Endpoint — Access S3 from VPC without internet (gateway endpoint, free)
```

---

## RDS & Databases

### 🔥 Q: RDS Multi-AZ vs Read Replicas.

| Feature | Multi-AZ | Read Replicas |
|---------|----------|---------------|
| Purpose | **High availability** | **Read scaling** |
| Replication | Synchronous | Asynchronous |
| Failover | Automatic (DNS) | Manual promotion |
| Read traffic | Standby not readable | Readable |
| Cross-region | No (within region) | Yes |
| Engines | All | MySQL, PostgreSQL, MariaDB, Aurora |

### ⭐ Q: RDS vs Aurora.

| Feature | RDS | Aurora |
|---------|-----|--------|
| Storage | EBS-based | Custom distributed storage (6 copies across 3 AZs) |
| Replication lag | Seconds | Milliseconds |
| Max read replicas | 5 | 15 |
| Failover time | 60-120s | ~30s (DNS update) |
| Auto-scaling storage | Manual/limited | Automatic (up to 128TB) |
| Cost | Lower base | ~20% more, but better performance |
| Serverless option | No | Yes (Aurora Serverless v2) |
| Backtrack | No | Yes (rewind to point-in-time without restore) |

### ⭐ Q: DynamoDB — key concepts for interviews.

| Feature | Detail |
|---------|--------|
| **Data model** | Key-value & document (NoSQL) |
| **Primary key** | Partition key (required) + Sort key (optional) |
| **Capacity modes** | **On-Demand** (pay per request) vs **Provisioned** (RCU/WCU, cheaper at scale) |
| **Indexes** | GSI (different partition key, eventually consistent) vs LSI (same partition key, strongly consistent) |
| **Consistency** | **Eventually consistent** (default, 50% cheaper reads) vs **Strongly consistent** |
| **Transactions** | ACID across multiple items (up to 100 items, 4MB) |
| **Streams** | Change data capture (CDC) → Lambda triggers |
| **TTL** | Auto-delete expired items (free) |
| **Global Tables** | Multi-region active-active replication |

**When to use DynamoDB:** High-scale key-value, single-digit ms latency, serverless, unpredictable traffic (on-demand).  
**When NOT to:** Complex queries, joins, OLAP, strong relational model → Use RDS/Aurora.

---

## Route 53

### ⭐ Q: Route 53 routing policies.

| Policy | Description | Use Case |
|--------|-------------|----------|
| **Simple** | Single resource | Single server |
| **Weighted** | Distribute by weight (70/30) | A/B testing, gradual migration |
| **Latency** | Route to lowest-latency region | Multi-region apps |
| **Failover** | Active-passive | Disaster recovery |
| **Geolocation** | Route by user location | Compliance, localization |
| **Geoproximity** | Route by distance + bias | Shift traffic between regions |
| **Multivalue** | Multiple healthy records | Simple load distribution |

---

## Load Balancing

### 🔥 Q: ALB vs NLB vs CLB.

| Feature | ALB (L7) | NLB (L4) | CLB (Legacy) |
|---------|----------|----------|-------------|
| Layer | HTTP/HTTPS | TCP/UDP/TLS | Both |
| Routing | Path, host, headers, query | Port, protocol | Basic |
| WebSockets | Yes | Yes | No |
| Static IP | No (use Global Accelerator) | Yes | No |
| Performance | Good | Best (millions of rps) | Moderate |
| Target types | Instance, IP, Lambda | Instance, IP, ALB | Instance |
| Use case | Web apps, microservices | Gaming, IoT, extreme perf | Don't use (legacy) |

**ALB path-based routing:**
```
app.example.com/api/*    → API target group
app.example.com/web/*    → Web target group
app.example.com/ws       → WebSocket target group
api.example.com/*        → API target group (host-based)
```

---

## Auto Scaling

### ⭐ Q: Auto Scaling policies.

| Policy Type | Trigger | Example |
|-------------|---------|---------|
| **Target Tracking** | Maintain a metric at target | CPU at 70% |
| **Step Scaling** | Scale based on alarm thresholds | CPU >80% add 2, >90% add 4 |
| **Scheduled** | Time-based | Scale up at 9am, down at 6pm |
| **Predictive** | ML-based forecasting | Anticipate traffic patterns |

```
Scaling Process:
1. CloudWatch alarm triggers (CPU > 70%)
2. Auto Scaling evaluates policy
3. Desired capacity increases
4. New instances launch
5. Health check passes
6. Instances registered with ALB target group
7. Traffic routed to new instances

Cooldown: 300s default (prevents rapid scaling oscillation)
```

---

## Lambda & Serverless

### 🔥 Q: Lambda key concepts.

| Feature | Detail |
|---------|--------|
| Max memory | 10,240 MB |
| Max timeout | 15 minutes |
| Max package size | 50MB (zip), 250MB (unzipped), 10GB (container) |
| Concurrency | 1000 default (soft limit) |
| Cold start | 100ms-10s depending on runtime, VPC, size |
| Pricing | Per request + per GB-second of compute |

**Cold start mitigation:**
- **Provisioned Concurrency** — Pre-warmed instances
- **SnapStart** (Java only) — Snapshot of initialized function
- Smaller packages, fewer dependencies
- Avoid VPC unless necessary (VPC adds ~1s cold start)

---

## CloudWatch & Monitoring

### 🔥 Q: CloudWatch components.

| Component | Purpose |
|-----------|---------|
| **Metrics** | Numeric data points (CPU, memory, custom) |
| **Alarms** | Trigger actions based on metric thresholds |
| **Logs** | Centralized log collection and querying |
| **Log Insights** | SQL-like query language for logs |
| **Dashboards** | Visualize metrics and logs |
| **Events/EventBridge** | React to AWS events |
| **Contributor Insights** | Top-N contributors to a metric |
| **ServiceLens** | End-to-end observability with X-Ray |

**CloudWatch Agent** — Required for OS-level metrics (memory, disk, custom metrics).

### ⭐ Q: CloudWatch vs X-Ray.

| Feature | CloudWatch | X-Ray |
|---------|-----------|-------|
| Purpose | Metrics & logs | Distributed tracing |
| Use case | "Is the service up?" | "Why is this request slow?" |
| Data | Time-series metrics, log lines | Request traces, service map |
| Typical signals | CPU, 5xx rate, latency p99 | End-to-end trace across Lambda/ECS/API Gateway |

**Interview:** "Microservices app is slow" → X-Ray to trace request flow. "High error rate" → CloudWatch Alarms.

---

## SQS, SNS, EventBridge

### 🔥 Q: SQS vs SNS vs EventBridge.

| Feature | SQS | SNS | EventBridge |
|---------|-----|-----|-------------|
| Pattern | Queue (pull) | Pub/Sub (push) | Event bus (rules) |
| Delivery | Consumer polls | Push to subscribers | Rules route to targets |
| Ordering | FIFO or Standard | FIFO or Standard | Ordered per source |
| Persistence | Up to 14 days | No persistence | Archive/replay |
| Targets | Consumer app | HTTP, SQS, Lambda, email | 20+ AWS services |
| Use case | Decouple producers/consumers | Fan-out notifications | Event-driven architectures |

### ⭐ Q: SQS Standard vs FIFO.

| Feature | Standard | FIFO |
|---------|----------|------|
| Throughput | Unlimited | 3,000 msg/s (with batching) |
| Ordering | Best-effort | Strict FIFO |
| Delivery | At-least-once | Exactly-once |
| Deduplication | No | 5-min dedup window |

---

## Secrets Manager & Parameter Store

### ⭐ Q: Secrets Manager vs Parameter Store.

| Feature | Secrets Manager | Parameter Store |
|---------|----------------|-----------------|
| Auto rotation | Yes (Lambda) | No |
| Cross-account | Yes | Limited |
| Pricing | $0.40/secret/month | Free (standard), $0.05/advanced |
| Max size | 64KB | 8KB (standard), 8KB (advanced) |
| Use case | DB passwords, API keys | Configuration, feature flags |
| KMS encryption | Mandatory | Optional |

---

## CloudFormation & IaC

### 🔥 Q: CloudFormation vs Terraform on AWS.

| Feature | CloudFormation | Terraform |
|---------|---------------|-----------|
| Provider | AWS-native | HashiCorp (multi-cloud) |
| State | Managed by AWS | You manage (S3 + DynamoDB lock) |
| Drift detection | Built-in | terraform plan |
| Rollback | Automatic on failure | Manual (`terraform apply` again) |
| Ecosystem | AWS-only | Multi-cloud, rich provider ecosystem |
| Change preview | Change sets | terraform plan |
| Best for | AWS-only, native integrations | Multi-cloud, existing TF workflows |

**CDK (Cloud Development Kit)** — Write IaC in TypeScript/Python/Java → generates CloudFormation.

---

## Cost Optimization & FinOps

**FinOps is heavily tested in 2025-2026 — expect deep questions on reducing AWS spend.**

### 🔥 Q: Reserved Instances vs Savings Plans vs Spot.

| Type | Commitment | Discount | Flexibility | Use Case |
|------|-----------|----------|-------------|----------|
| **Reserved Instances (RI)** | 1 or 3 years, specific instance type/region | Up to 72% | Low (locked to family/region) | Steady workloads, predictable |
| **Compute Savings Plans** | 1 or 3 years, $/hour commitment | Up to 66% | **High** (any instance type, region, Fargate, Lambda) | Steady workloads, flexible |
| **EC2 Instance Savings Plans** | 1 or 3 years, instance family | Up to 72% | Medium (family, any size/AZ/region) | Steady, family-locked |
| **Spot Instances** | None (2-min interruption notice) | Up to 90% | Highest (can be reclaimed) | Fault-tolerant (CI/CD, batch, dev) |

**Interview:** "Reduce cost for steady production workload, but we change instance types often" → **Compute Savings Plan**.

### 🔥 Q: AWS cost optimization strategies (FinOps best practices).

1. **Right-sizing** — Use Compute Optimizer, downsize over-provisioned instances (common 30-50% savings)
2. **Reserved Instances / Savings Plans** — Commit for 1-3 years for steady workloads
3. **Spot Instances** — Use for fault-tolerant workloads (CI/CD, batch, dev/test, Spark/EMR)
4. **Graviton instances** — ARM-based (t4g, m7g, c7g), 20-40% better price-performance vs Intel/AMD
5. **S3 Lifecycle Policies** — Auto-transition to Intelligent-Tiering or Glacier (80-90% savings for archives)
6. **Delete unused resources**:
   - Unattached EBS volumes
   - Unused Elastic IPs ($3.6/month each!)
   - Old EBS snapshots (incremental, but add up)
   - Idle load balancers, NAT Gateways
7. **NAT Gateway optimization** — Biggest hidden cost ($32/mo + $0.045/GB):
   - Use VPC Gateway Endpoints for S3/DynamoDB (free)
   - Use Interface Endpoints (PrivateLink) for other services ($7/mo/AZ, no data processing fee)
   - Consolidate NAT Gateways in non-prod (single AZ)
8. **Data transfer**:
   - Use VPC endpoints (avoid internet/NAT Gateway)
   - Same-AZ placement (cross-AZ = $0.01/GB each way)
   - CloudFront for static assets (caching at edge)
   - Direct Connect for on-prem (cheaper than VPN at scale)
9. **Lambda optimization**:
   - Right-size memory (Compute Optimizer, Lambda Power Tuning)
   - Graviton2 (arm64 runtime, 20% cheaper + faster)
   - Reduce cold starts (Provisioned Concurrency only if needed — expensive)
10. **DynamoDB**:
    - Use On-Demand for spiky traffic, Provisioned for steady (70% cheaper at scale)
    - Auto-scaling for Provisioned capacity
11. **Tags** — Tag everything (cost allocation tags) for chargeback by team/project
12. **Cost governance**:
    - AWS Budgets + alerts
    - Cost Anomaly Detection (ML-based spike detection)
    - Trusted Advisor (free basic checks, full checks with Enterprise Support)
    - Cost Explorer + Rightsizing Recommendations
13. **Use AWS Cost Explorer API** to build custom dashboards/alerts

**Interview scenario:** "Our AWS bill doubled last month — how do you investigate?"

```
1. Cost Explorer → Group by Service, then by Usage Type
2. Look for:
   - NAT Gateway data processing spike (check VPC Flow Logs)
   - EC2 instance type changes (larger instances launched?)
   - Data transfer spikes (cross-region, cross-AZ, internet egress)
   - New resources (RDS, EBS volumes, Load Balancers)
   - DynamoDB on-demand capacity spikes
   - Lambda concurrency spike (check invocations)
   - S3 requests (PUT/GET API calls, not just storage)
3. Set up Cost Anomaly Detection for future
4. Enable detailed billing reports (export to S3, query with Athena)
5. Tag audit — untagged resources often = shadow IT/waste
```

---

## Multi-Account & Organizations

### ⭐ Q: Why use AWS Organizations?

- **Centralized billing** — Consolidated billing, volume discounts
- **Security guardrails** — Service Control Policies (SCPs) enforce org-wide rules
- **Account isolation** — Dev/staging/prod in separate accounts (blast radius containment)
- **Compliance** — Separate accounts for PCI, HIPAA, etc.

### ⭐ Q: Service Control Policies (SCPs) — how do they work?

```
SCP = Maximum allowed permissions (guardrail)

Effective permissions = SCP ∩ IAM Policy

Example: SCP denies ec2:TerminateInstances in production account
  → Even an admin with "ec2:*" cannot terminate instances
```

**Common SCP use cases:**
- Deny leaving the organization
- Deny disabling CloudTrail / GuardDuty
- Deny creating resources in unauthorized regions
- Deny creating IAM users (force SSO)

### ⭐ Q: Multi-account strategy (common patterns).

```
Root (Management Account — billing, SCPs only)
├── Security OU
│   ├── Security Tooling (GuardDuty, Security Hub)
│   └── Log Archive (centralized CloudTrail, VPC Flow Logs)
├── Shared Services OU
│   ├── Networking (Transit Gateway, VPN)
│   └── CI/CD (CodePipeline, Artifact Registry)
├── Workloads OU
│   ├── Dev Account
│   ├── Staging Account
│   └── Production Account
└── Sandbox OU (engineers can experiment, restricted SCPs)
```

### ⭐ Q: Cross-account access — how to enable?

1. **IAM Roles (AssumeRole)** — Most common:
   - Account A creates role with trust policy allowing Account B principals
   - User in Account B calls `sts:AssumeRole` → gets temp creds
2. **Resource-based policies** — S3, SNS, SQS, KMS can grant cross-account access directly
3. **AWS Resource Access Manager (RAM)** — Share VPCs, Transit Gateways, subnets

**Interview:** "Allow CI/CD account to deploy to production account" → IAM Role with trust policy.

### 💡 Q: What is AWS Control Tower?

- **Automated multi-account setup** using Organizations best practices
- **Landing Zone** — Pre-configured account structure (Security, Log Archive, shared services)
- **Guardrails** — Pre-packaged SCPs + AWS Config rules (detective & preventive)
- **Account Factory** — Vend new accounts via Service Catalog
- **Dashboard** — Compliance overview across all accounts

**When to use:** Setting up new AWS environments from scratch, need governance at scale.

---

## Well-Architected Framework

### ⭐ Q: Six Pillars.

| Pillar | Key Principle |
|--------|--------------|
| **Operational Excellence** | Automate, monitor, iterate, IaC |
| **Security** | Least privilege, encryption, traceability |
| **Reliability** | Auto-recover, scale horizontally, test recovery |
| **Performance Efficiency** | Right resource type, monitor, experiment |
| **Cost Optimization** | Pay for what you use, right-size, reserve |
| **Sustainability** | Minimize environmental impact |

---

## Scenario-Based Questions

**These are heavily tested in 2025-2026 — practice drawing architectures and talking through trade-offs.**

### 🔥 Q: Design a highly available, scalable 3-tier web application on AWS.

```
┌─────────────────────────────────────────────────────────────┐
│ Users (Global)                                               │
└────────┬────────────────────────────────────────────────────┘
         │
    Route 53 (DNS, health checks, latency-based routing)
         │
    CloudFront (CDN, DDoS protection via Shield Standard)
         │
    WAF (SQL injection, XSS, rate limiting)
         │
    ┌────┴─── Multi-AZ VPC ────────┐
    │                               │
    │  ┌─── Public Subnets ──┐     │
    │  │  ALB (cross-AZ)      │     │
    │  └──────┬───────────────┘     │
    │         │                     │
    │  ┌─── Private Subnets ──┐    │
    │  │  Auto Scaling Group   │    │
    │  │  (EC2 or ECS Fargate) │    │
    │  │  AZ-a, AZ-b, AZ-c     │    │
    │  └──────┬───────────────┘     │
    │         │                     │
    │  ElastiCache (Redis)         │
    │  (session store, caching)     │
    │         │                     │
    │  ┌─── Data Subnets ──┐       │
    │  │  Aurora Multi-AZ   │       │
    │  │  + 2 Read Replicas │       │
    │  └────────────────────┘       │
    └───────────────────────────────┘
         │
    S3 (static assets, user uploads, backups)
         │
    CloudWatch + X-Ray (monitoring, tracing)
    Secrets Manager (DB creds, API keys)
    GuardDuty (threat detection)
    CloudTrail (audit log)

IaC: Terraform (or CloudFormation)
CI/CD: GitHub Actions + OIDC → AssumeRole → deploy via CodeDeploy or ECS
```

**Key decisions:**
- **Multi-AZ** for all layers (ALB, ASG, Aurora)
- **CloudFront** for global performance + DDoS protection
- **Auto Scaling** with target tracking (CPU 70%)
- **IRSA** for EKS pods (or IAM instance roles for EC2)
- **Aurora** over RDS for better failover + read replica performance
- **S3 + CloudFront** for static assets (cheaper, faster than serving from app)
- **VPC endpoints** for S3/DynamoDB (avoid NAT Gateway costs)

### 🔥 Q: An EC2 instance in a private subnet can't reach the internet. Troubleshoot.

```bash
# 1. Check route table
#    Does private subnet route table have 0.0.0.0/0 → NAT Gateway?
aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=subnet-xyz"

# 2. Is NAT Gateway healthy?
#    Check NAT GW in console — is it in "Available" state?
#    Is it in a PUBLIC subnet with route 0.0.0.0/0 → IGW?
aws ec2 describe-nat-gateways --nat-gateway-ids nat-abc123

# 3. Security Group
#    Does SG allow outbound traffic? (default: all outbound allowed)
#    Check if custom rule blocks outbound
aws ec2 describe-security-groups --group-ids sg-xyz

# 4. NACL
#    Does NACL allow outbound AND return traffic?
#    (Stateless — need both directions!)
#    Check rules 100 (outbound 0.0.0.0/0 ALLOW) and inbound ephemeral ports (1024-65535)
aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=subnet-xyz"

# 5. Check the instance
#    Does it have a private IP? Is the route table associated to its subnet?
aws ec2 describe-instances --instance-ids i-xyz

# 6. DNS
#    Can it resolve DNS? Check VPC DNS settings
#    enableDnsSupport = true, enableDnsHostnames = true
aws ec2 describe-vpc-attribute --vpc-id vpc-abc --attribute enableDnsSupport

# 7. VPC Flow Logs
#    Enable and check for REJECT logs
#    Shows SG/NACL denials
```

### ⭐ Q: S3 bucket suddenly became public — respond immediately.

```bash
# 1. Identify scope
aws s3api get-bucket-acl --bucket my-bucket
aws s3api get-bucket-policy --bucket my-bucket
aws s3api get-public-access-block --bucket my-bucket

# 2. Block public access immediately
aws s3api put-public-access-block --bucket my-bucket \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# 3. Check CloudTrail for who made the change
aws cloudtrail lookup-events --lookup-attributes \
  AttributeKey=ResourceName,AttributeValue=my-bucket \
  --max-results 50

# 4. Review S3 access logs (if enabled)
# 5. Rotate IAM credentials if compromised
# 6. Check for data exfiltration (CloudTrail GetObject events, GuardDuty alerts)
# 7. Incident report: timeline, root cause, remediation, prevention (SCPs to deny s3:PutBucketPolicy)
```

### ⭐ Q: Design cross-account access for CI/CD pipeline to deploy to production.

```
CI/CD Account (123456789012)
    │
    ▼
GitHub Actions workflow
    │
    ▼
OIDC Provider in CI/CD Account
    │
    ▼
AssumeRole → arn:aws:iam::987654321098:role/CICD-Deploy-Role (Production Account)
    │
    ▼
Deploy to Production (987654321098)
```

**Setup:**

1. **Production account** (987654321098) — Create role with trust policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::123456789012:root"
    },
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {
        "sts:ExternalId": "unique-secret-string"
      }
    }
  }]
}
```

2. **Permissions policy** on CICD-Deploy-Role (what it can do in Production):
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ecs:UpdateService",
      "ecs:DescribeServices",
      "ecr:GetAuthorizationToken",
      "ecr:BatchCheckLayerAvailability"
    ],
    "Resource": "*"
  }]
}
```

3. **CI/CD account** — IAM user/role in GitHub Actions workflow:
```bash
aws sts assume-role \
  --role-arn arn:aws:iam::987654321098:role/CICD-Deploy-Role \
  --role-session-name github-actions-deploy \
  --external-id unique-secret-string
```

**Best practice:** Use OIDC (GitHub OIDC → AWS) to avoid long-lived IAM user credentials.

### ⭐ Q: Lambda function timing out at 15 minutes. How to re-architect?

```
Problem: Lambda max timeout = 15 minutes
          Long-running batch job / video processing / large ETL

Solutions:

1. Step Functions + multiple Lambda invocations
   - Break job into chunks
   - State machine orchestrates
   - Each step < 15 min

2. ECS Fargate task
   - No time limit
   - Triggered by EventBridge or Lambda
   - Lambda = orchestrator, Fargate = worker

3. Batch job in AWS Batch
   - Submit job to Batch queue
   - Runs on EC2/Fargate
   - Auto-scales, retries

4. SQS + Lambda (for parallelizable work)
   - SQS queue with messages (chunks of work)
   - Lambda polls SQS, processes each message < 15min
   - Lambda concurrency scales

Best choice: If truly long-running (hours) → ECS Fargate or Batch.
            If divisible into chunks → Step Functions + Lambda or SQS + Lambda.
```

### 🔥 Q: How would you reduce our AWS bill by 30%?

```
Phase 1: Quick wins (week 1)
1. Enable Cost Explorer, identify top 5 services by spend
2. Right-size EC2 instances (Compute Optimizer recommendations)
3. Delete unused resources:
   - Unattached EBS volumes
   - Old EBS snapshots (keep last 7 days)
   - Unused Elastic IPs
   - Idle load balancers
4. Stop/terminate non-production EC2 instances outside business hours
   - Use Instance Scheduler or Lambda + EventBridge
5. Review NAT Gateway usage — replace with VPC endpoints for S3/DynamoDB (free)

Phase 2: Commitments (week 2-3)
6. Purchase Compute Savings Plans for steady workloads (66% savings)
7. Convert dev/test to Spot instances (90% savings, use Spot Fleet or Karpenter)
8. Switch RDS/EC2 to RIs if instance types won't change

Phase 3: Architecture optimizations (month 1-2)
9. S3 Lifecycle → Intelligent-Tiering or Glacier (80% savings for old data)
10. Lambda right-sizing (memory tuning → lower GB-sec cost)
11. Switch to Graviton instances (20-40% cheaper)
12. Consolidate small workloads onto larger instances (better utilization)
13. Use Auto Scaling aggressively (scale down to 0 in non-prod)

Phase 4: Governance (ongoing)
14. Enforce tagging for cost allocation (chargeback to teams)
15. Set up AWS Budgets + Cost Anomaly Detection
16. Monthly cost review meetings with engineering teams
17. SCPs to prevent expensive resource creation (e.g., deny p4d.24xlarge)
```

### ⭐ Q: Design disaster recovery (DR) for a critical application.

**DR strategies (RTO/RPO trade-off):**

| Strategy | RTO | RPO | Cost | Description |
|----------|-----|-----|------|-------------|
| **Backup & Restore** | Hours | Hours | Lowest | Snapshots/backups to S3, restore on failure |
| **Pilot Light** | 10-30 min | Minutes | Low | Minimal always-on (DB), scale up on DR |
| **Warm Standby** | Minutes | Seconds | Medium | Scaled-down replica running, scale up on DR |
| **Multi-Site Active-Active** | Seconds | None | Highest | Full capacity in 2+ regions, Route 53 routes traffic |

**Example: Warm Standby for RTO < 15min, RPO < 5min**

```
Primary Region (us-east-1)          DR Region (us-west-2)
    │                                   │
Route 53 (failover routing, health checks)
    │                                   │
Production (full capacity)      Standby (20% capacity)
    │                                   │
Aurora Global Database (replication lag < 1s)
    │                                   │
S3 Cross-Region Replication (CRR)
    │                                   │
CloudWatch Alarm detects failure → Lambda triggers scale-up in DR region
```

**Steps:**
1. Aurora Global Database (primary → DR, <1s lag)
2. S3 CRR for static assets
3. AMIs / container images replicated to DR region
4. Minimal ASG (2 instances) in DR region
5. CloudWatch Alarm → SNS → Lambda → scale DR ASG to production capacity
6. Route 53 health check fails → switches to DR region

### ⭐ Q: How would you debug high latency in a microservices app?

```
1. Identify the slow service
   - X-Ray service map → which service has high latency?
   - CloudWatch Metrics → p99 latency by service

2. Drill into the slow service
   - X-Ray trace details → which downstream call is slow?
     - Database query? → Slow query log, RDS Performance Insights
     - External API? → Check API provider status, CloudWatch Logs
     - Internal service? → Recurse to step 1

3. Common causes
   - Database: Missing index, N+1 queries, connection pool exhaustion
   - Network: Cross-region calls, large payloads, no connection reuse
   - Code: Synchronous blocking calls (should be async)
   - Resource limits: CPU/memory throttling (CloudWatch metrics)
   - Cold starts (Lambda): Provisioned Concurrency or SnapStart

4. Immediate mitigation
   - Scale horizontally (increase ASG capacity)
   - Add caching (ElastiCache, CloudFront)
   - Enable connection pooling
   - Circuit breaker to failing dependency

5. Long-term fix
   - Optimize queries, add indexes
   - Async processing (SQS for non-critical paths)
   - CDN for static assets
   - Database read replicas for read-heavy workloads
```

---

## Key Resources

- **AWS Well-Architected Labs** — https://www.wellarchitectedlabs.com
- **AWS Docs** — https://docs.aws.amazon.com
- **AWS Solutions Architect Associate** — Great cert for breadth (foundational)
- **AWS Solutions Architect Professional** — Deep architecture, multi-account, DR
- **AWS DevOps Professional** — Deep CI/CD, IaC, monitoring (heavily weighted toward DevOps interviews)
- **Last Week in AWS (newsletter)** — https://www.lastweekinaws.com (FinOps, cloud economics)
- **Open Guide to AWS** — https://github.com/open-guides/og-aws (practical, opinionated)
- **Adrian Cantrill Courses** — https://learn.cantrill.io (best video courses for AWS certs)
- **AWS re:Invent YouTube** — Latest features, best practices from AWS engineers
- **AWS This Week** — https://aweeklyaws.com (curated updates)
- **AWS Cost Optimization** — https://aws.amazon.com/aws-cost-management/
- **EKS Best Practices Guide** — https://aws.github.io/aws-eks-best-practices/
