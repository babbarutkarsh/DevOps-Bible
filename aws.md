# AWS — DevOps Interview Preparation

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
- [Cost Optimization](#cost-optimization)
- [Well-Architected Framework](#well-architected-framework)
- [Scenario-Based Questions](#scenario-based-questions)

---

## Core Services Overview

### Q: Map AWS services to DevOps needs.

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

### Q: Explain IAM components.

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

### Q: Explain IAM policy evaluation logic.

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

### Q: IAM best practices.

- **Root account:** MFA, no access keys, use only for billing
- **Least privilege:** Start with minimal permissions, add as needed
- **Use roles, not users** for services (EC2, Lambda, ECS)
- **Use OIDC** for CI/CD (GitHub Actions → AssumeRoleWithWebIdentity)
- **Enable CloudTrail** for auditing all API calls
- **Use SCPs** (Service Control Policies) in AWS Organizations
- **Rotate credentials** regularly
- **Use Permission Boundaries** to delegate admin safely

### Q: What is the difference between SCP, Permission Boundary, and IAM Policy?

| Type | Scope | Purpose |
|------|-------|---------|
| **SCP** | Organization/OU/Account | Maximum permissions for an account |
| **Permission Boundary** | User/Role | Maximum permissions for a specific principal |
| **IAM Policy** | User/Group/Role | Actual granted permissions |

Effective permissions = SCP ∩ Permission Boundary ∩ IAM Policy

---

## VPC & Networking

### Q: Design a production VPC.

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

### Q: Security Groups vs NACLs.

| Feature | Security Groups | NACLs |
|---------|----------------|-------|
| Level | ENI (instance) | Subnet |
| State | **Stateful** | **Stateless** |
| Rules | Allow only | Allow & Deny |
| Evaluation | All rules | Ordered by rule number |
| Default | Deny all in, allow all out | Allow all |

### Q: VPC Peering vs Transit Gateway vs PrivateLink?

| Feature | VPC Peering | Transit Gateway | PrivateLink |
|---------|------------|-----------------|-------------|
| Topology | 1:1 | Hub-and-spoke | Service endpoint |
| Transitive | No | Yes | N/A |
| Cross-account | Yes | Yes | Yes |
| Cross-region | Yes | Yes | Yes |
| Use case | Few VPCs | Many VPCs | Expose service privately |
| Cost | Data transfer only | Per attachment + data | Per endpoint + data |

### Q: How do private subnets access the internet?

**NAT Gateway** (managed) or **NAT Instance** (self-managed EC2).

```
Private Subnet Instance → NAT Gateway (in public subnet) → Internet Gateway → Internet
```

- NAT GW: Managed, HA within AZ, auto-scales, ~$32/month + data
- Deploy one NAT GW per AZ for high availability
- For cost savings in non-prod: single NAT GW or NAT Instance

---

## EC2 & Compute

### Q: EC2 instance types to know.

| Family | Optimized For | Use Case |
|--------|--------------|----------|
| **t3/t4g** | Burstable | Dev/test, light workloads |
| **m5/m6i/m7i** | General purpose | Web servers, app servers |
| **c5/c6i/c7i** | Compute | Batch processing, ML inference |
| **r5/r6i/r7i** | Memory | Databases, caching |
| **i3/i4i** | Storage | NoSQL, data warehousing |
| **g5/p5** | GPU | ML training, video processing |

### Q: EC2 purchasing options.

| Option | Discount | Commitment | Use Case |
|--------|----------|------------|----------|
| **On-Demand** | None | None | Unpredictable workloads |
| **Reserved (RI)** | Up to 72% | 1 or 3 years | Steady-state workloads |
| **Savings Plans** | Up to 72% | 1 or 3 years ($ commitment) | Flexible across instance types |
| **Spot** | Up to 90% | None (can be interrupted) | Batch, CI/CD, fault-tolerant |
| **Dedicated** | Varies | Per host | Compliance, licensing |

### Q: What is EC2 instance metadata?

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

### Q: ECS vs EKS — when to use which?

| Feature | ECS | EKS |
|---------|-----|-----|
| Orchestrator | AWS-proprietary | Kubernetes |
| Learning curve | Lower | Higher |
| Portability | AWS only | Multi-cloud |
| Ecosystem | AWS integrations | K8s ecosystem (Helm, operators) |
| Cost | No control plane cost | $0.10/hr per cluster ($73/month) |
| Best for | Simpler apps, AWS-native | Complex microservices, multi-cloud |

### Q: ECS architecture.

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

---

## S3 & Storage

### Q: S3 storage classes.

| Class | Availability | Min Duration | Use Case |
|-------|-------------|-------------|----------|
| **Standard** | 99.99% | None | Frequently accessed |
| **Intelligent-Tiering** | 99.9% | None | Unknown access patterns |
| **Standard-IA** | 99.9% | 30 days | Infrequent access |
| **One Zone-IA** | 99.5% | 30 days | Re-creatable data |
| **Glacier Instant** | 99.9% | 90 days | Archive, ms retrieval |
| **Glacier Flexible** | 99.99% | 90 days | Archive, min-hr retrieval |
| **Glacier Deep Archive** | 99.99% | 180 days | Long-term, 12-48hr retrieval |

### Q: S3 security.

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

### Q: S3 performance optimization.

- **Multipart upload** for files >100MB (required >5GB)
- **S3 Transfer Acceleration** for cross-region uploads
- **Prefix distribution** — S3 supports 3,500 PUT/5,500 GET per prefix/sec
- **S3 Select / Glacier Select** — Query in-place without downloading entire object
- **Byte-range fetches** — Download specific portions of objects

---

## RDS & Databases

### Q: RDS Multi-AZ vs Read Replicas.

| Feature | Multi-AZ | Read Replicas |
|---------|----------|---------------|
| Purpose | **High availability** | **Read scaling** |
| Replication | Synchronous | Asynchronous |
| Failover | Automatic (DNS) | Manual promotion |
| Read traffic | Standby not readable | Readable |
| Cross-region | No (within region) | Yes |
| Engines | All | MySQL, PostgreSQL, MariaDB, Aurora |

### Q: RDS vs Aurora.

| Feature | RDS | Aurora |
|---------|-----|--------|
| Storage | EBS-based | Custom distributed storage |
| Replication lag | Seconds | Milliseconds |
| Max read replicas | 5 | 15 |
| Failover time | 60-120s | ~30s |
| Auto-scaling storage | Manual/limited | Automatic (up to 128TB) |
| Cost | Lower base | ~20% more, but better performance |
| Serverless option | No | Yes (Aurora Serverless v2) |

---

## Route 53

### Q: Route 53 routing policies.

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

### Q: ALB vs NLB vs CLB.

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

### Q: Auto Scaling policies.

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

### Q: Lambda key concepts.

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

### Q: CloudWatch components.

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

---

## SQS, SNS, EventBridge

### Q: SQS vs SNS vs EventBridge.

| Feature | SQS | SNS | EventBridge |
|---------|-----|-----|-------------|
| Pattern | Queue (pull) | Pub/Sub (push) | Event bus (rules) |
| Delivery | Consumer polls | Push to subscribers | Rules route to targets |
| Ordering | FIFO or Standard | FIFO or Standard | Ordered per source |
| Persistence | Up to 14 days | No persistence | Archive/replay |
| Targets | Consumer app | HTTP, SQS, Lambda, email | 20+ AWS services |
| Use case | Decouple producers/consumers | Fan-out notifications | Event-driven architectures |

### Q: SQS Standard vs FIFO.

| Feature | Standard | FIFO |
|---------|----------|------|
| Throughput | Unlimited | 3,000 msg/s (with batching) |
| Ordering | Best-effort | Strict FIFO |
| Delivery | At-least-once | Exactly-once |
| Deduplication | No | 5-min dedup window |

---

## Secrets Manager & Parameter Store

### Q: Secrets Manager vs Parameter Store.

| Feature | Secrets Manager | Parameter Store |
|---------|----------------|-----------------|
| Auto rotation | Yes (Lambda) | No |
| Cross-account | Yes | Limited |
| Pricing | $0.40/secret/month | Free (standard), $0.05/advanced |
| Max size | 64KB | 8KB (standard), 8KB (advanced) |
| Use case | DB passwords, API keys | Configuration, feature flags |
| KMS encryption | Mandatory | Optional |

---

## Cost Optimization

### Q: AWS cost optimization strategies.

1. **Right-sizing** — Use Compute Optimizer, downsize over-provisioned instances
2. **Reserved Instances / Savings Plans** — Commit for 1-3 years for steady workloads
3. **Spot Instances** — Use for fault-tolerant workloads (CI/CD, batch, dev)
4. **S3 Lifecycle Policies** — Auto-transition to cheaper storage classes
5. **NAT Gateway** — Consider VPC endpoints instead ($0.045/GB data processing!)
6. **Unused resources** — Delete unattached EBS, unused EIPs, old snapshots
7. **Data transfer** — Use VPC endpoints, CloudFront, same-AZ placement
8. **Tags** — Tag everything for cost allocation and accountability
9. **Graviton instances** — ARM-based, 20-40% better price-performance

---

## Well-Architected Framework

### Q: Six Pillars.

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

### Q: Design a highly available web application on AWS.

```
Route 53 (DNS, health checks, failover)
    │
CloudFront (CDN, DDoS protection via Shield)
    │
WAF (SQL injection, XSS protection)
    │
ALB (cross-AZ, path-based routing)
    │
┌───┴────── Auto Scaling Group ──────┐
│  AZ-a: EC2/ECS      AZ-b: EC2/ECS │
└───┬────────────────────────────────┘
    │
ElastiCache (Redis — session store, caching)
    │
Aurora (Multi-AZ, auto-failover, read replicas)
    │
S3 (static assets, backups)

Monitoring: CloudWatch + X-Ray + CloudTrail
Secrets: Secrets Manager
IaC: Terraform
CI/CD: GitHub Actions + CodeDeploy
```

### Q: An EC2 instance in a private subnet can't reach the internet. Troubleshoot.

```bash
# 1. Check route table
#    Does private subnet route table have 0.0.0.0/0 → NAT Gateway?

# 2. Is NAT Gateway healthy?
#    Check NAT GW in console — is it in "Available" state?
#    Is it in a PUBLIC subnet with route to IGW?

# 3. Security Group
#    Does SG allow outbound traffic? (default: all outbound allowed)

# 4. NACL
#    Does NACL allow outbound AND return traffic?
#    (Stateless — need both directions!)

# 5. Check the instance
#    Does it have an IP? Is the route table associated to its subnet?

# 6. DNS
#    Can it resolve DNS? Check VPC DNS settings
#    enableDnsSupport = true, enableDnsHostnames = true
```

### Q: Your AWS bill spiked unexpectedly. Investigate.

```
1. Cost Explorer → Filter by service, region, usage type
2. Look for:
   - NAT Gateway data processing (very common!)
   - Data transfer between AZs/regions
   - Forgotten resources (large EC2 instances, GP3 volumes)
   - Unattached EBS volumes, unused Elastic IPs
   - CloudWatch Logs ingestion
   - Lambda concurrency spike
   - DynamoDB on-demand capacity spikes

3. Set up:
   - AWS Budgets with alerts
   - Cost Anomaly Detection
   - Tags for cost allocation
   - Trusted Advisor checks
```

---

## Key Resources

- **AWS Well-Architected Labs** — https://www.wellarchitectedlabs.com
- **AWS Docs** — https://docs.aws.amazon.com
- **AWS Solutions Architect Associate** — Great cert for breadth
- **AWS DevOps Professional** — Deep CI/CD and automation
- **Last Week in AWS (newsletter)** — https://www.lastweekinaws.com
- **Open Guide to AWS** — https://github.com/open-guides/og-aws
- **Adrian Cantrill Courses** — https://learn.cantrill.io
