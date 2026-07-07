---
title: System Design
nav_order: 61
description: "System Design — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# Infrastructure System Design — DevOps Interview Preparation

## Table of Contents
- [System Design Framework](#system-design-framework)
- [High Availability & Fault Tolerance](#high-availability--fault-tolerance)
- [Scalability Patterns](#scalability-patterns)
- [Caching Strategies](#caching-strategies)
- [Database Scaling](#database-scaling)
- [Message Queues & Event-Driven Architecture](#message-queues--event-driven-architecture)
- [Disaster Recovery](#disaster-recovery)
- [Design Problems](#design-problems)
- [Capacity Planning](#capacity-planning)

---

## System Design Framework

### Q: How to approach infra system design in an interview?

**Step 1: Requirements (2-3 min)**
- Functional: What does the system do?
- Non-functional: Scale, latency, availability, durability
- Constraints: Budget, compliance, tech stack

**Step 2: Back-of-envelope estimation (2-3 min)**
- Users, requests/sec, data volume, bandwidth
- Storage growth per month/year

**Step 3: High-level design (10 min)**
- Draw major components and data flow
- Identify single points of failure

**Step 4: Deep dive (15 min)**
- Scale specific components
- Address failure scenarios
- Discuss trade-offs

**Step 5: Monitoring & Operations (5 min)**
- How do you know it's healthy?
- How do you deploy changes?
- How do you handle incidents?

---

## High Availability & Fault Tolerance

### Q: Explain the nines of availability.

| Availability | Downtime/Year | Downtime/Month | Downtime/Week |
|-------------|---------------|----------------|----------------|
| 99% | 3.65 days | 7.31 hours | 1.68 hours |
| 99.9% | 8.77 hours | 43.8 min | 10.1 min |
| 99.95% | 4.38 hours | 21.9 min | 5.04 min |
| 99.99% | 52.6 min | 4.38 min | 1.01 min |
| 99.999% | 5.26 min | 26.3 sec | 6.05 sec |

**Compound availability:** If Service A (99.9%) depends on Service B (99.9%):
Combined = 99.9% × 99.9% = 99.8%

**With redundancy:** Two instances at 99.9% each:
Combined = 1 - (0.001 × 0.001) = 99.9999%

### Q: HA patterns.

```
Active-Active:
┌────────┐     ┌────────┐
│ App A  │     │ App B  │     Both serving traffic
└───┬────┘     └───┬────┘     Load balanced
    │              │
    └──── LB ──────┘

Active-Passive:
┌────────┐     ┌────────┐
│ App A  │     │ App B  │     B is standby
│(Active)│     │(Standby)│    Takes over on A failure
└───┬────┘     └────────┘
    │
  Traffic

Multi-Region Active-Active:
┌─── US-East ─────┐     ┌─── EU-West ─────┐
│ App + DB (R/W)  │ ←──→ │ App + DB (R/W)  │
└────────┬────────┘     └────────┬────────┘
         │                       │
    Route 53 (latency-based routing)
```

### Q: What are circuit breakers?

Prevent cascading failures when a dependency is down:

```
States:
                    success threshold met
          ┌─────────────────────────────────┐
          ▼                                 │
    ┌──────────┐   failure threshold   ┌────┴─────┐
    │  CLOSED  │ ─────────────────────►│   OPEN   │
    │(normal)  │                       │(fail fast)│
    └──────────┘                       └────┬──────┘
                                            │ timeout
                                       ┌────▼──────┐
                                       │ HALF-OPEN │
                                       │(test one) │
                                       └───────────┘
                                        success → CLOSED
                                        failure → OPEN
```

**Tools:** Hystrix (legacy), Resilience4j, Istio circuit breaking, Envoy

---

## Scalability Patterns

### Q: Horizontal vs Vertical scaling.

| Aspect | Vertical (Scale Up) | Horizontal (Scale Out) |
|--------|-------------------|----------------------|
| Method | Bigger machine | More machines |
| Limit | Hardware ceiling | Nearly unlimited |
| Complexity | Simple | Need load balancing, state management |
| Downtime | Usually required | No downtime |
| Cost | Expensive at scale | Cost-effective |

### Q: Stateless vs Stateful services.

**Stateless (easy to scale):**
- No local state — all state in external stores (DB, cache, object store)
- Any instance can handle any request
- Auto-scaling is straightforward

**Stateful (harder to scale):**
- State tied to specific instance (sessions, connections, data)
- Requires: sticky sessions, or StatefulSets, or distributed consensus
- Examples: databases, WebSocket servers, caches

**Making services stateless:**
```
Before: Session stored in local memory
After:  Session stored in Redis/DynamoDB

Before: File uploads saved to local disk
After:  File uploads to S3

Before: Cache in local process
After:  Cache in ElastiCache/Redis
```

---

## Caching Strategies

### Q: Caching patterns.

| Pattern | How | When to Use |
|---------|-----|-------------|
| **Cache-Aside** | App checks cache → miss → read DB → write cache | General purpose, read-heavy |
| **Write-Through** | App writes to cache → cache writes to DB | Write consistency needed |
| **Write-Behind** | App writes to cache → cache async writes to DB | High write throughput |
| **Read-Through** | Cache loads from DB on miss automatically | Simplify app logic |

**Cache-Aside (most common):**
```
Read:  App → Cache HIT → Return data
       App → Cache MISS → Read DB → Write to Cache → Return data

Write: App → Write DB → Invalidate/Update Cache
```

### Q: Cache invalidation strategies.

| Strategy | Description | Trade-off |
|----------|-------------|-----------|
| **TTL** | Expire after time | Simple but stale data possible |
| **Event-driven** | Invalidate on write | Fresh data but complex |
| **Write-through** | Update cache on write | Fresh data, write latency |
| **Versioning** | Cache key includes version | Simple, good for static assets |

**FAANG Tip:** "There are only two hard things in CS: cache invalidation and naming things." — Phil Karlton

### Q: Multi-level caching.

```
Browser Cache (ms)
    → CDN (ms)
        → Application Cache / Redis (1-5ms)
            → Database Query Cache (5-10ms)
                → Database (10-100ms)
                    → Disk (1-10ms)
```

---

## Database Scaling

### Q: Database scaling strategies.

**Read replicas:**
```
┌───── Primary (R/W) ─────┐
│                           │
│  ──async replication──►   │
│                           │
├── Replica 1 (Read) ──────┤
├── Replica 2 (Read) ──────┤
└── Replica 3 (Read) ──────┘

App: Write → Primary
     Read  → Replicas (via connection pooling / proxy)
```

**Sharding (horizontal partitioning):**
```
┌── Shard 1 ──┐  ┌── Shard 2 ──┐  ┌── Shard 3 ──┐
│ Users A-H   │  │ Users I-P   │  │ Users Q-Z   │
└─────────────┘  └─────────────┘  └─────────────┘

Shard key: Determines which shard holds the data
Challenge: Cross-shard queries, rebalancing, joins
```

**Connection pooling:**
```
Without pooling: 100 app instances × 10 connections = 1000 DB connections (💀)
With PgBouncer:  100 app instances → PgBouncer (50 pooled connections) → DB
```

### Q: SQL vs NoSQL decision matrix.

| Use SQL When | Use NoSQL When |
|-------------|---------------|
| Complex joins and queries | Simple key-value or document access |
| ACID transactions required | Eventually consistent is OK |
| Schema is well-defined | Schema evolves frequently |
| Moderate scale | Massive scale (millions of ops/sec) |
| Reporting/analytics | Real-time, low-latency access |

---

## Message Queues & Event-Driven Architecture

### Q: When to use message queues.

```
Without queue (tight coupling):
Service A ──► Service B     (A waits, B must be up)

With queue (loose coupling):
Service A ──► Queue ──► Service B
              │
              └── Service C (fan-out)

Benefits:
- Decouple producers/consumers
- Buffer traffic spikes
- Retry failed processing
- Enable async processing
```

### Q: Event-driven architecture patterns.

| Pattern | Description | Example |
|---------|-------------|---------|
| **Event notification** | Publish event, consumers react | Order placed → send email |
| **Event-carried state** | Event contains full data | No need to call back to source |
| **Event sourcing** | Store events, not state | Bank transactions, audit logs |
| **CQRS** | Separate read and write models | Write to SQL, read from Elasticsearch |

---

## Disaster Recovery

### Q: DR strategies (RTO and RPO).

| Term | Definition |
|------|-----------|
| **RTO** (Recovery Time Objective) | Max acceptable downtime |
| **RPO** (Recovery Point Objective) | Max acceptable data loss |

| Strategy | RTO | RPO | Cost | Description |
|----------|-----|-----|------|-------------|
| **Backup & Restore** | Hours | Hours | $ | Restore from backups |
| **Pilot Light** | 10s of min | Minutes | $$ | Core services running, scale up on failure |
| **Warm Standby** | Minutes | Seconds | $$$ | Scaled-down copy in DR region |
| **Multi-Site Active-Active** | Near zero | Near zero | $$$$ | Full capacity in both regions |

### Q: Multi-region architecture.

```
┌─── Primary (us-east-1) ──────┐   ┌─── DR (us-west-2) ──────────┐
│                                │   │                               │
│  ALB → ECS/EKS → Aurora       │──►│  ALB → ECS/EKS → Aurora     │
│                   (Writer)     │   │                  (Reader)    │
│  ElastiCache (Redis)           │   │  ElastiCache (Redis)         │
│  S3 (CRR enabled)              │──►│  S3 (replica)                │
│                                │   │                               │
└────────────────────────────────┘   └───────────────────────────────┘
                      │                             │
              Route 53 (failover routing policy)
```

**Key considerations:**
- Database replication: Aurora Global Database, DynamoDB Global Tables
- DNS failover: Route 53 health checks
- S3 Cross-Region Replication (CRR)
- Stateless services: Easy to replicate
- Testing: Regular DR drills (chaos engineering)

---

## Design Problems

### Q: Design a CI/CD platform for 100+ microservices.

```
Source: GitHub (monorepo or multi-repo)
    │
    ▼
CI (GitHub Actions / Jenkins):
    ├── Detect changed services (path filter / Nx affected)
    ├── Parallel builds per service
    ├── Shared pipeline templates (DRY)
    ├── Test → Scan → Build → Push to ECR
    └── Artifact: Docker image tagged with git SHA
    │
    ▼
CD (ArgoCD — GitOps):
    ├── Config repo (K8s manifests / Helm values)
    ├── Image Updater watches ECR → updates config repo
    ├── ArgoCD syncs desired state to clusters
    ├── Canary deployments via Argo Rollouts
    └── Per-environment overlays (Kustomize)
    │
    ▼
Environments:
    dev → staging → production (promotion via Git merges)

Key decisions:
- Monorepo vs multi-repo (monorepo for shared libraries)
- Self-hosted vs managed runners (cost vs convenience)
- Centralized pipeline templates (reduce duplication)
- Service catalog (Backstage) for standardization
```

### Q: Design a logging platform for a large-scale K8s cluster.

```
┌─── K8s Nodes ──────────────────────────────┐
│  DaemonSet: Fluent Bit (lightweight)        │
│  - Tail container logs from /var/log/       │
│  - Parse, enrich with K8s metadata          │
│  - Buffer in memory                         │
└──────────────┬─────────────────────────────┘
               │
┌──────────────▼─────────────────────────────┐
│  Kafka (buffer/decouple)                    │
│  - Handles backpressure                     │
│  - Multiple consumers                       │
└──────────────┬─────────────────────────────┘
               │
     ┌─────────┼──────────┐
     ▼         ▼          ▼
  Loki    Elasticsearch    S3
(recent)  (search/analyze) (archive)

Grafana → Query Loki + ES for dashboards and alerts

Scale considerations:
- Log volume: estimate 1-10 GB/day per service
- Retention: Hot (7 days), Warm (30 days), Cold (1 year in S3)
- Sampling for high-volume services
- Rate limiting to prevent log storms
- Cost: Loki is 10x cheaper than Elasticsearch for storage
```

### Q: Design infrastructure for a global e-commerce platform.

```
Users (worldwide)
    │
    ▼
CloudFlare/CloudFront (CDN + WAF + DDoS)
    │
    ▼
Route 53 (latency-based routing to nearest region)
    │
    ├──► US Region (us-east-1)
    │    ALB → EKS (auto-scaling)
    │    Aurora (writer) + Read Replicas
    │    ElastiCache Redis (sessions + product cache)
    │    S3 (product images, static assets)
    │
    ├──► EU Region (eu-west-1)
    │    ALB → EKS (auto-scaling)
    │    Aurora (Global DB — read replica, promotable)
    │    ElastiCache Redis
    │    S3 (replicated via CRR)
    │
    └──► Asia Region (ap-southeast-1)
         ALB → EKS (auto-scaling)
         Aurora (Global DB — read replica)
         ElastiCache Redis
         S3 (replicated)

Event-driven backend:
  Order Placed → SQS → Order Processor → Payment → Inventory → Shipping

Async Processing:
  SQS (orders, notifications)
  EventBridge (inter-service events)
  Lambda (email, SMS, webhooks)
  
Search: OpenSearch (product search, with replicas per region)
Monitoring: Prometheus + Grafana + CloudWatch + X-Ray
```

---

## Capacity Planning

### Q: How to estimate infrastructure needs.

```
Given: 10M daily active users, avg 20 requests/user/day

Requests/day: 10M × 20 = 200M
Requests/sec (avg): 200M / 86400 ≈ 2,300 RPS
Peak (3x avg): ~7,000 RPS

If each request needs:
  - 50ms compute time → Need ~350 cores at peak (7000 × 0.05)
  - 256KB memory → ~1.8 GB RAM for concurrent requests
  - 2KB response → ~14 MB/s bandwidth at peak

Storage:
  - 1KB per user record × 50M total users = 50 GB for user data
  - 100KB avg per order × 1M orders/day = 100 GB/day
  - Growth: ~3 TB/month for orders

Instance sizing:
  - m6i.xlarge (4 vCPU, 16GB) handles ~400 RPS
  - Need ~18 instances at peak → round up to 20 for headroom
  - Run 8-10 normally, auto-scale to 20+
```

---

## Key Resources

- **Designing Data-Intensive Applications (book)** — Martin Kleppmann (THE system design book)
- **System Design Interview (book)** — Alex Xu (Vol 1 & 2)
- **Google SRE Book** — https://sre.google/sre-book/
- **AWS Architecture Center** — https://aws.amazon.com/architecture/
- **High Scalability Blog** — http://highscalability.com
- **ByteByteGo** — https://bytebytego.com (Alex Xu's visual guides)
