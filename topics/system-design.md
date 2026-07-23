---
title: System Design
nav_order: 61
description: "System Design — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# Infrastructure System Design — DevOps Interview Preparation

> **Question frequency:** 🔥 very frequently asked · ⭐ important · 💡 good to know

## Table of Contents
- [How to Approach an SRE System Design Interview](#how-to-approach-an-sre-system-design-interview)
- [System Design Framework](#system-design-framework)
- [High Availability & Fault Tolerance](#high-availability--fault-tolerance)
- [Scalability Patterns](#scalability-patterns)
- [Load Balancing](#load-balancing)
- [Caching Strategies](#caching-strategies)
- [Database Scaling](#database-scaling)
- [Message Queues & Event-Driven Architecture](#message-queues--event-driven-architecture)
- [Rate Limiting & Backpressure](#rate-limiting--backpressure)
- [Resilience Patterns](#resilience-patterns)
- [SRE Fundamentals](#sre-fundamentals)
- [Disaster Recovery](#disaster-recovery)
- [Design Problems](#design-problems)
- [Capacity Planning](#capacity-planning)

---

## How to Approach an SRE System Design Interview

### ⭐ Q: What makes SRE/DevOps system design different from product system design?

**Focus areas unique to SRE/DevOps interviews:**
- **Reliability first:** Availability targets, failure modes, blast radius, graceful degradation
- **Operations at scale:** Monitoring, alerting, incident response, capacity planning
- **Infrastructure primitives:** Load balancing, caching, queuing, replication patterns
- **Trade-offs:** Cost vs reliability, consistency vs availability (CAP), latency vs throughput
- **Real-world constraints:** Budget limits, on-call burden, technical debt

**Common mistakes:**
- Jumping to solutions before clarifying requirements
- Over-designing for scale you don't need yet
- Ignoring failure modes and recovery paths
- Not discussing monitoring/observability
- Forgetting about cost

**Interview structure (45 min typical):**
```
Minutes 0-5:   Clarify requirements (functional + non-functional)
Minutes 5-10:  Back-of-envelope estimation (scale, storage, bandwidth)
Minutes 10-20: High-level architecture (draw components, data flow)
Minutes 20-35: Deep dives (scale bottlenecks, failure modes, trade-offs)
Minutes 35-40: Monitoring & operations (SLIs, alerts, deployment)
Minutes 40-45: Q&A, alternative approaches
```

**Communicating effectively:**
- Think out loud — interviewers want to see your thought process
- Start broad, then zoom into details when prompted
- Ask clarifying questions: "Should I optimize for cost or reliability?"
- Acknowledge trade-offs: "This approach gives us X but sacrifices Y"
- Draw ASCII diagrams liberally — visual > words
- State assumptions explicitly

---

## System Design Framework

### 🔥 Q: How to approach infra system design in an interview?

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

### 🔥 Q: Explain the nines of availability.

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

### 🔥 Q: HA patterns.

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

### ⭐ Q: What is quorum and why does it matter?

**Quorum:** Minimum number of nodes that must agree for an operation to succeed.

```
3-node cluster with quorum = 2:
┌──────┐  ┌──────┐  ┌──────┐
│ Node │  │ Node │  │ Node │
│  A   │  │  B   │  │  C   │
└──────┘  └──────┘  └──────┘

Write succeeds if 2/3 nodes acknowledge
Read succeeds if 2/3 nodes respond

Network partition:
┌──────┐  ┌──────┐     │     ┌──────┐
│ Node │  │ Node │     │     │ Node │
│  A   │  │  B   │     │     │  C   │
└──────┘  └──────┘     │     └──────┘
   Majority partition   │   Minority partition
   (can elect leader)   │   (cannot serve writes)
```

**Why it matters:**
- Prevents split-brain: Two groups can't both think they're in charge
- Ensures consistency: Reads see most recent acknowledged writes
- Used in: etcd, Consul, Zookeeper, Cassandra, MongoDB replica sets

**Common quorum formulas:**
- Simple majority: `floor(N/2) + 1`
- For 3 nodes: 2 must agree
- For 5 nodes: 3 must agree
- For 7 nodes: 4 must agree

### ⭐ Q: Failover strategies.

| Strategy | Detection Time | Failover Time | Complexity |
|----------|---------------|---------------|------------|
| **DNS failover** | Health check interval (30-60s) | TTL (60-300s) | Low |
| **Load balancer health checks** | 5-30s | Instant | Medium |
| **Database auto-failover** | 10-30s | 30-60s | High |
| **Kubernetes pod restart** | 1-10s | 5-30s | Medium |
| **Manual failover** | Human detection time | Minutes | Low (to implement) |

**Best practices:**
- Test failover regularly (chaos engineering)
- Automate detection and promotion
- Monitor failover system itself (who watches the watchers?)
- Document runbooks for manual override

### 🔥 Q: What are circuit breakers?

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

### 🔥 Q: Horizontal vs Vertical scaling.

| Aspect | Vertical (Scale Up) | Horizontal (Scale Out) |
|--------|-------------------|----------------------|
| Method | Bigger machine | More machines |
| Limit | Hardware ceiling | Nearly unlimited |
| Complexity | Simple | Need load balancing, state management |
| Downtime | Usually required | No downtime |
| Cost | Expensive at scale | Cost-effective |

### 🔥 Q: Stateless vs Stateful services.

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

### 🔥 Q: Sharding and partitioning strategies.

**Sharding key selection:**
```
Good shard key: user_id (even distribution, co-locates user data)
Bad shard key:  timestamp (hot shard for recent data)
Bad shard key:  country (uneven distribution)

Range-based sharding:
Shard 1: user_id 0-999999
Shard 2: user_id 1000000-1999999
Shard 3: user_id 2000000-2999999
- Simple but risk of hot shards
- Rebalancing requires data movement

Hash-based sharding:
Shard = hash(user_id) % num_shards
- Even distribution
- Adding shards requires rehashing (use consistent hashing!)

Geo-based sharding:
Shard by region (US, EU, APAC)
- Data locality (low latency)
- Compliance (GDPR data residency)
```

**Consistent hashing:**
```
Without consistent hashing (modulo):
- Add/remove server → ~80% keys move

With consistent hashing:
- Add/remove server → ~20% keys move (1/N only)

Ring with virtual nodes:
    Server A (vnodes: A1, A2, A3)
         ↓
    ●───●───●───●───●
   A1  B1  A2  C1  A3
   ↑       ↑       ↑
Server B  Server C

Key "user123" → hash → lands between A2 and C1 → stored on C1
```

**Used in:** DynamoDB, Cassandra, Redis Cluster, CDNs, load balancers

---

## Load Balancing

### 🔥 Q: L4 vs L7 load balancing.

| Layer | OSI Model | Operates On | Use Case | Examples |
|-------|-----------|-------------|----------|----------|
| **L4** | Transport | IP + Port | Fast, protocol-agnostic | AWS NLB, HAProxy (TCP mode) |
| **L7** | Application | HTTP headers, path, cookies | Content-based routing, SSL termination | AWS ALB, Nginx, Envoy |

**L4 (Network Load Balancer):**
```
Client → NLB (IP:Port) → Backend servers
         - Low latency (~100μs)
         - High throughput (millions of RPS)
         - No header inspection
         - Preserves client IP
```

**L7 (Application Load Balancer):**
```
Client → ALB → Path /api/* → API servers
             → Path /static/* → Static file servers
             → Host: v2.example.com → V2 servers
         - Header-based routing
         - SSL termination
         - WebSocket support
         - Higher latency (~ms) but more features
```

### ⭐ Q: Load balancing algorithms.

| Algorithm | Description | Best For |
|-----------|-------------|----------|
| **Round Robin** | Each server in turn | Equal capacity servers |
| **Least Connections** | Server with fewest active connections | Variable request duration |
| **Weighted Round Robin** | Distribute based on server capacity | Heterogeneous servers |
| **IP Hash** | Hash client IP → sticky sessions | Stateful apps |
| **Least Response Time** | Server with lowest latency | Mixed workloads |
| **Random** | Pick randomly | Simple, surprisingly effective |

**Consistent hashing for load balancers:**
```
Problem: Server added/removed → all sessions break (with IP hash)
Solution: Consistent hashing → only affected users remap

Ring with servers:
    ●───S1───●───S2───●───S3───●
    User1    User2    User3

Add S4:
    ●───S1───●───S2───●─S4─●───S3───●
    User1    User2  User4  User3

Only users between S2 and S3 remap to S4
```

### ⭐ Q: Health checks and circuit breaking at the LB level.

**Health check types:**
```
TCP health check:
  - Connect to port → success if connection established
  - Fast but shallow

HTTP health check:
  - GET /health → expect 200 OK
  - Can check dependencies (DB, cache)
  - Configure: interval (5s), timeout (2s), threshold (3 failures)

Deep health check:
  - Test actual functionality (query DB, read cache)
  - Risk: health check itself can fail during outage
```

**Circuit breaking at LB:**
```
ALB Target Group:
  Unhealthy threshold: 3 consecutive failures
  Healthy threshold:   2 consecutive successes
  Interval:            30s
  Timeout:             5s

ELB → [ Server A (healthy) ]
      [ Server B (unhealthy) ] ← no traffic sent
      [ Server C (healthy) ]
```

---

## Caching Strategies

### 🔥 Q: Caching patterns.

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

### 🔥 Q: Cache invalidation strategies.

| Strategy | Description | Trade-off |
|----------|-------------|-----------|
| **TTL** | Expire after time | Simple but stale data possible |
| **Event-driven** | Invalidate on write | Fresh data but complex |
| **Write-through** | Update cache on write | Fresh data, write latency |
| **Versioning** | Cache key includes version | Simple, good for static assets |

**FAANG Tip:** "There are only two hard things in CS: cache invalidation and naming things." — Phil Karlton

### ⭐ Q: Multi-level caching.

```
Browser Cache (ms)
    → CDN (ms)
        → Application Cache / Redis (1-5ms)
            → Database Query Cache (5-10ms)
                → Database (10-100ms)
                    → Disk (1-10ms)
```

### ⭐ Q: Cache eviction policies.

| Policy | Description | Best For |
|--------|-------------|----------|
| **LRU** (Least Recently Used) | Evict oldest accessed | General purpose |
| **LFU** (Least Frequently Used) | Evict least accessed over time | Long-lived popular content |
| **FIFO** (First In First Out) | Evict oldest added | Simple, predictable |
| **TTL** (Time To Live) | Evict after fixed duration | Time-sensitive data |
| **Random** | Evict randomly | Simple, low overhead |

**Redis eviction policies:**
```
noeviction:        Return error when max memory reached
allkeys-lru:       Evict any key, LRU
volatile-lru:      Evict keys with TTL set, LRU
allkeys-lfu:       Evict any key, LFU
volatile-lfu:      Evict keys with TTL set, LFU
allkeys-random:    Evict any key, randomly
volatile-random:   Evict keys with TTL, randomly
volatile-ttl:      Evict keys with TTL, shortest TTL first
```

### ⭐ Q: Cache stampede / thundering herd problem.

**Problem:**
```
Cache miss at t=0 → 1000 concurrent requests hit DB
                  → DB overload
                  → Cascading failure

Timeline:
t=0:    Cache expires
t=0-5s: 1000 requests → all miss → all query DB
```

**Solutions:**

**1. Lock-based (request coalescing):**
```python
def get_from_cache(key):
    val = cache.get(key)
    if val:
        return val
    
    lock_key = f"lock:{key}"
    if cache.set(lock_key, "1", nx=True, ex=10):  # Only one request gets lock
        val = expensive_db_query()
        cache.set(key, val, ex=300)
        cache.delete(lock_key)
        return val
    else:
        time.sleep(0.1)  # Other requests wait and retry
        return get_from_cache(key)
```

**2. Probabilistic early expiration:**
```python
def get_from_cache(key):
    val, expiry = cache.get_with_expiry(key)
    if not val:
        return refresh_cache(key)
    
    # Refresh probabilistically before expiry
    time_left = expiry - now()
    if random() < (1.0 - time_left / TTL):
        background_refresh(key)  # Async refresh
    
    return val
```

**3. Background refresh:**
```
Cache worker refreshes top 1000 keys every 5 minutes
Before they expire → no stampede
```

### ⭐ Q: CDN strategies.

**CDN cache behaviors:**
```
Static assets (images, CSS, JS):
  Cache-Control: public, max-age=31536000, immutable
  → Cache for 1 year, never revalidate

Dynamic content with ETags:
  Cache-Control: public, max-age=0, must-revalidate
  ETag: "abc123"
  → CDN revalidates, returns 304 Not Modified if unchanged

Personalized content:
  Cache-Control: private, no-cache
  → Skip CDN, cache in browser only

API responses (short-lived):
  Cache-Control: public, max-age=60, s-maxage=60
  → CDN caches for 60s
```

**Cache key strategy:**
```
Default:  example.com/api/users?id=123
Custom:   example.com/api/users?id=123&version=v2&region=us

Include in cache key:
  - URL path + query params
  - Selected headers (Authorization for per-user cache)
  - Cookies (session-based caching)
  - Geo (region-specific content)
```

**Purging strategies:**
```
1. Purge by URL:     DELETE /api/users/123
2. Purge by tag:     DELETE cache-tag:users
3. Purge by prefix:  DELETE /api/users/*
4. Full purge:       DELETE /* (rare, slow)
```

---

## Database Scaling

### 🔥 Q: Database scaling strategies.

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

### 🔥 Q: SQL vs NoSQL decision matrix.

| Use SQL When | Use NoSQL When |
|-------------|---------------|
| Complex joins and queries | Simple key-value or document access |
| ACID transactions required | Eventually consistent is OK |
| Schema is well-defined | Schema evolves frequently |
| Moderate scale | Massive scale (millions of ops/sec) |
| Reporting/analytics | Real-time, low-latency access |

### 🔥 Q: CAP theorem in practice.

**CAP Theorem:** In a distributed system, you can only guarantee 2 of 3:
- **C** (Consistency): All nodes see the same data at the same time
- **A** (Availability): Every request receives a response (success or failure)
- **P** (Partition tolerance): System continues despite network partitions

```
Network partition occurs:
┌─── Partition 1 ───┐    │    ┌─── Partition 2 ───┐
│  Node A    Node B  │    │    │  Node C    Node D  │
└────────────────────┘    │    └────────────────────┘

CP (Consistency + Partition tolerance):
  - Reject writes until partition heals
  - Banking systems, inventory

AP (Availability + Partition tolerance):
  - Accept writes, resolve conflicts later
  - Social media feeds, shopping carts
```

**Real-world systems:**
```
CP Systems:
  - HBase, MongoDB (with majority writes), etcd, Consul
  - Trade-off: May reject requests during partition

AP Systems:
  - Cassandra, DynamoDB, Riak, CouchDB
  - Trade-off: Temporary inconsistency, conflict resolution needed

CA Systems (not partition-tolerant):
  - Single-node RDBMS
  - Not distributed (not realistic in practice)
```

**Consistency models:**
```
Strong consistency (CP):
  Read always returns most recent write
  All replicas synchronized before ACK

Eventual consistency (AP):
  Replicas eventually converge
  Reads may return stale data temporarily

Causal consistency:
  Related operations are seen in order
  Independent operations can be seen differently
```

### ⭐ Q: Connection pooling patterns.

**Without pooling:**
```
Request → Open connection → Query → Close connection
Overhead: Connection setup ~10-50ms per request
Problem:  Exhausts DB connection limit (typical: 100-500)
```

**With pooling (PgBouncer, HikariCP):**
```
Application pool (size: 10)
    ↓
PgBouncer (pool size: 50)
    ↓
PostgreSQL (max connections: 500)

100 app instances × 10 connections = 1000 needed
PgBouncer multiplexes → only 50 active DB connections

Pool modes:
- Session pooling:     Connection held for client session
- Transaction pooling: Connection held for transaction only (most common)
- Statement pooling:   Connection held for single query
```

**Pool sizing formula:**
```
connections = ((core_count × 2) + effective_spindle_count)

For SSD: effective_spindle_count ≈ 1
For 8-core DB: ~17 connections optimal

Too small:  Requests queue, high latency
Too large:  Context switching overhead, memory waste
```

---

## Message Queues & Event-Driven Architecture

### 🔥 Q: When to use message queues.

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

### ⭐ Q: Event-driven architecture patterns.

| Pattern | Description | Example |
|---------|-------------|---------|
| **Event notification** | Publish event, consumers react | Order placed → send email |
| **Event-carried state** | Event contains full data | No need to call back to source |
| **Event sourcing** | Store events, not state | Bank transactions, audit logs |
| **CQRS** | Separate read and write models | Write to SQL, read from Elasticsearch |

### 🔥 Q: Kafka vs SQS vs RabbitMQ.

| Feature | Kafka | SQS | RabbitMQ |
|---------|-------|-----|----------|
| **Throughput** | Very high (1M+ msg/s) | High (10K msg/s per queue) | Medium (10K msg/s) |
| **Ordering** | Per partition | FIFO queues only | Per queue |
| **Retention** | Days to weeks | 1-14 days | Until consumed |
| **Replay** | Yes (offset-based) | No | No |
| **Latency** | Low (2-10ms) | Medium (10-100ms) | Low (1-10ms) |
| **Use case** | Event streaming, logs | Simple queues, decoupling | Complex routing, RPC |

**Kafka architecture:**
```
┌─── Topic: orders (partitions: 3) ────┐
│                                        │
│  Partition 0:  [M1] [M2] [M3] ─────►  │  Consumer Group A
│  Partition 1:  [M4] [M5] [M6] ─────►  │    ├─ Consumer 1 (reads P0, P1)
│  Partition 2:  [M7] [M8] [M9] ─────►  │    └─ Consumer 2 (reads P2)
│                                        │
└────────────────────────────────────────┘

Key concepts:
- Partitions for parallelism (consumer per partition)
- Offsets for replay (consumer tracks offset)
- Consumer groups for scale-out
- Retention independent of consumption
```

### ⭐ Q: At-least-once vs exactly-once vs at-most-once delivery.

| Guarantee | Description | Implementation |
|-----------|-------------|----------------|
| **At-most-once** | Message may be lost | Send and forget (no retry) |
| **At-least-once** | Message may be duplicated | Retry on failure (most common) |
| **Exactly-once** | Message delivered exactly once | Idempotent consumer + transactional producer |

**At-least-once with idempotency (standard pattern):**
```python
# Producer: Retry on failure
def send_message(msg):
    while not queue.send(msg):
        retry_with_backoff()

# Consumer: Idempotent processing
def process_message(msg):
    if already_processed(msg.id):  # Check DB/cache
        return  # Skip duplicate
    
    do_work(msg)
    mark_processed(msg.id)
    queue.delete(msg)  # ACK
```

**Idempotency key patterns:**
```
1. Natural ID:        Use order_id, user_id, etc.
2. Message ID:        UUID from producer
3. Hash:              Hash(message_content + timestamp)
4. Composite key:     user_id + action + date
```

### ⭐ Q: Backpressure handling.

**Problem:**
```
Producer: 10,000 msg/s
Consumer:  1,000 msg/s
Queue grows unbounded → memory/disk exhaustion
```

**Solutions:**

**1. Rate limiting at producer:**
```
if queue_depth > threshold:
    slow_down_producer()  # Sleep, reduce batch size
```

**2. Bounded queue with reject:**
```
queue.put(msg, timeout=1s)
if timeout:
    return 503 Service Unavailable to client
```

**3. Autoscaling consumers:**
```
CloudWatch Alarm: ApproximateNumberOfMessages > 1000
  → Trigger: ECS auto-scaling
  → Scale consumers 2 → 10
```

**4. Batch processing:**
```
Consumer pulls 100 messages at once
Process in parallel (thread pool)
Throughput: 1K → 10K msg/s
```

**5. Circuit breaker:**
```
If consumer keeps failing:
  Stop consuming → let queue build → alert on-call
Better than infinite retries causing DB overload
```

---

## Rate Limiting & Backpressure

### 🔥 Q: Rate limiting algorithms.

**Token Bucket:**
```
Bucket capacity: 100 tokens
Refill rate:     10 tokens/second

Request arrives:
  if bucket has tokens:
      bucket -= 1
      allow request
  else:
      reject (429 Too Many Requests)

Allows bursts up to bucket size
Smooth refill rate
```

**Leaky Bucket:**
```
Queue requests → Process at fixed rate

Incoming: [R1] [R2] [R3] [R4] ...
           ↓
        [ Queue ]
           ↓
       Fixed rate (10/s)

Smooths bursts
Rejects when queue full
```

**Fixed Window:**
```
Minute 0: 100 requests allowed
Minute 1: 100 requests allowed

Simple but allows bursts at window boundaries
(99 requests at 0:59, 101 at 1:00 = 200 in 1 second)
```

**Sliding Window Log:**
```
Track timestamp of each request
Count requests in last N seconds
Accurate but memory-intensive
```

**Sliding Window Counter:**
```
Estimate based on previous + current window
Memory-efficient
Good balance of accuracy and cost
```

**Comparison:**

| Algorithm | Burst Handling | Accuracy | Memory | Use Case |
|-----------|---------------|----------|---------|----------|
| **Token Bucket** | Allows bursts | High | O(1) | API rate limiting |
| **Leaky Bucket** | Smooths bursts | High | O(N) | Traffic shaping |
| **Fixed Window** | Allows boundary bursts | Medium | O(1) | Simple quotas |
| **Sliding Window** | Smooth | Very high | O(N) | Strict rate limiting |

### ⭐ Q: Distributed rate limiting.

**Problem:**
```
3 API servers, rate limit: 100 req/s per user
User sends 100 req/s to each server = 300 req/s total (over limit!)
```

**Solutions:**

**1. Shared Redis counter:**
```python
def is_rate_limited(user_id):
    key = f"rate_limit:{user_id}:{current_minute}"
    count = redis.incr(key)
    redis.expire(key, 60)
    return count > 100

Pros: Accurate
Cons: Redis is single point of failure, added latency
```

**2. Gossip protocol (Envoy, Istio):**
```
Each server tracks local rate
Periodically gossip to peers: "I've seen 30 req/s from user123"
Sum across servers → enforce limit

Pros: No central store
Cons: Eventually consistent, complex
```

**3. Sticky sessions:**
```
Route same user to same server (via consistent hashing)
Each server enforces limit independently

Pros: Simple, no coordination
Cons: Uneven load, failover breaks limit
```

**4. Pre-allocated quotas:**
```
User quota: 100 req/min
Allocate to servers: 33 / 33 / 34
Each server enforces its quota

Pros: Simple, no coordination
Cons: Wastes quota if load uneven
```

---

## Resilience Patterns

### ⭐ Q: Circuit breaker pattern in depth.

**States:**
```
CLOSED (normal):
  - All requests pass through
  - Track failures (sliding window)
  - If failure_rate > threshold → OPEN

OPEN (fail fast):
  - Immediately reject requests (no backend call)
  - After timeout → HALF_OPEN

HALF_OPEN (testing):
  - Allow limited requests through
  - If success → CLOSED
  - If failure → OPEN
```

**Configuration example:**
```yaml
circuit_breaker:
  failure_threshold: 50%           # Open if >50% fail
  request_volume_threshold: 20     # Need 20 requests before triggering
  sleep_window: 30s                # Time in OPEN before trying HALF_OPEN
  half_open_requests: 3            # Test with 3 requests in HALF_OPEN
```

**Implementation (pseudo-code):**
```python
class CircuitBreaker:
    def call(self, func):
        if self.state == OPEN:
            if now() > self.open_until:
                self.state = HALF_OPEN
            else:
                raise CircuitBreakerOpen()
        
        try:
            result = func()
            self.on_success()
            return result
        except Exception:
            self.on_failure()
            raise
    
    def on_failure(self):
        self.failures += 1
        if self.failures / self.requests > 0.5:
            self.state = OPEN
            self.open_until = now() + 30s
```

### 💡 Q: Bulkhead pattern.

**Concept:** Isolate resources so failure in one area doesn't sink the whole ship.

```
Without bulkhead:
  Shared thread pool (50 threads)
  ├─ API calls to Service A (slow, timing out)
  ├─ API calls to Service B
  └─ API calls to Service C
  
  Service A times out → exhausts all threads → everything fails

With bulkhead:
  Thread pool A (20 threads) → Service A
  Thread pool B (20 threads) → Service B
  Thread pool C (10 threads) → Service C
  
  Service A times out → only pool A exhausted → B and C still work
```

**Implementations:**
```
1. Thread pools:        Separate thread pools per dependency
2. Connection pools:    Separate DB connection pools per tenant
3. Semaphores:          Limit concurrent requests per resource
4. Process isolation:   Separate processes/containers per service
```

**Example (Kubernetes resource quotas):**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: api-quota
spec:
  hard:
    requests.cpu: "10"       # Limit total CPU
    requests.memory: 20Gi
    pods: "50"               # Limit pod count
```

### 🔥 Q: Retry strategies with backoff and jitter.

**Exponential backoff:**
```
Retry 1: wait 1s
Retry 2: wait 2s
Retry 3: wait 4s
Retry 4: wait 8s

Formula: wait_time = base_delay * (2 ^ attempt)
```

**Exponential backoff + jitter:**
```
Retry 1: wait random(0.5s, 1.5s)
Retry 2: wait random(1s, 3s)
Retry 3: wait random(2s, 6s)

Formula: wait_time = random(0.5 * delay, 1.5 * delay)

Why jitter?
- Without: All clients retry at same time (thundering herd)
- With:    Retries spread out over time
```

**Full jitter (AWS recommended):**
```
wait_time = random(0, min(max_delay, base_delay * (2 ^ attempt)))

Retry 1: wait random(0, 1s)
Retry 2: wait random(0, 2s)
Retry 3: wait random(0, 4s)
```

**Retry policy example:**
```python
def call_with_retry(func, max_attempts=3):
    for attempt in range(max_attempts):
        try:
            return func()
        except RetryableError as e:
            if attempt == max_attempts - 1:
                raise
            
            base_delay = 1.0
            max_delay = 30.0
            delay = min(max_delay, base_delay * (2 ** attempt))
            jittered_delay = random.uniform(0, delay)
            time.sleep(jittered_delay)
```

**When to retry:**
```
Retry on:
  - Network timeouts
  - 429 Rate Limit Exceeded
  - 500, 502, 503, 504 (transient server errors)
  - Connection refused (service restart)

Do NOT retry on:
  - 400 Bad Request (client error)
  - 401 Unauthorized (auth issue)
  - 404 Not Found (wrong endpoint)
  - 409 Conflict (data conflict)
```

---

## SRE Fundamentals

### 🔥 Q: SLI, SLO, SLA, and error budgets.

**SLI (Service Level Indicator):**
Quantitative measure of service level.
```
Examples:
- Request latency:       p99 latency < 200ms
- Availability:          % of requests returning 200 OK
- Throughput:            requests/second
- Error rate:            % of requests returning 5xx
```

**SLO (Service Level Objective):**
Target value or range for an SLI.
```
Examples:
- 99.9% of requests complete in < 200ms
- 99.95% of requests return non-5xx
- API uptime: 99.9% per month

SLO = SLI + target
```

**SLA (Service Level Agreement):**
Contract with consequences (money, credits) if SLO not met.
```
Example:
- SLA: 99.9% uptime per month
- If breached: 10% credit for 99.5-99.9%, 25% for 99.0-99.5%, 100% for <99.0%
```

**Error Budget:**
```
SLO: 99.9% uptime = 0.1% downtime allowed
Per month: 43.2 minutes allowed downtime

Error budget = (1 - SLO) × time_period

If error budget exhausted:
  - Freeze feature launches
  - Focus on reliability
  - Improve monitoring/alerting

If error budget healthy:
  - Launch features faster
  - Take risks (canary deploys, chaos engineering)
```

**Example dashboard:**
```
SLO: 99.9% of requests succeed (monthly)
Current: 99.92% success rate
Error budget remaining: 60% (25 minutes of downtime left this month)

Actions:
- Green (>50% budget):  Ship features
- Yellow (20-50%):      Review reliability
- Red (<20%):           Freeze launches, focus on incidents
```

### 🔥 Q: Four golden signals (Google SRE).

| Signal | Description | Example |
|--------|-------------|---------|
| **Latency** | Time to service a request | p50, p95, p99 latency |
| **Traffic** | Demand on the system | Requests/second, bytes/second |
| **Errors** | Rate of failed requests | % 5xx, % timeouts |
| **Saturation** | How "full" the system is | CPU%, memory%, queue depth |

**Latency:**
```
Alert on: p99 latency > 500ms for 5 minutes
Dashboard: Show p50, p95, p99 latency over time

Why p99, not average?
- Average hides outliers (99% users at 100ms, 1% at 10s → avg 190ms looks fine!)
- p99 represents worst 1% of user experience
```

**Traffic:**
```
Alert on: Sudden drop (>50% in 5 min) → service down?
          Sudden spike (>2x normal) → attack? viral event?
Dashboard: Requests/second, breakdown by endpoint
```

**Errors:**
```
Alert on: Error rate > 1% for 5 minutes
Dashboard: % 2xx, 4xx, 5xx over time
```

**Saturation:**
```
Alert on: CPU > 80% for 10 minutes
          Memory > 90%
          Disk > 85%
Dashboard: Resource utilization over time
```

### ⭐ Q: Graceful degradation strategies.

**Concept:** When a dependency fails, degrade service rather than fail completely.

```
Full functionality:
  Show personalized feed (requires ML service)
  
ML service down → Graceful degradation:
  Show generic feed (cached popular posts)
  
Cache down → Further degradation:
  Show static "Service unavailable" page
```

**Patterns:**

**1. Feature flags:**
```python
if feature_enabled("personalization") and ml_service.healthy():
    return personalized_feed()
else:
    return generic_feed()
```

**2. Fallback responses:**
```python
try:
    return recommendation_service.get()
except ServiceUnavailable:
    return cached_recommendations()  # Stale but better than nothing
```

**3. Timeout and fallback:**
```python
try:
    return external_api.call(timeout=200ms)
except Timeout:
    return default_response()  # Fast failure, graceful response
```

**4. Static fallback:**
```python
try:
    return db.query_user_prefs()
except:
    return DEFAULT_PREFS  # Reasonable defaults
```

**Real-world example (Netflix):**
```
Full experience:
  ├─ Personalized recommendations (ML)
  ├─ Custom artwork (A/B testing service)
  ├─ User ratings (DB query)
  └─ Continue watching (session service)

ML down → Degraded:
  ├─ Generic popular titles
  ├─ Default artwork
  ├─ User ratings (DB query)
  └─ Continue watching (session service)

DB slow → Further degraded:
  ├─ Generic popular titles
  ├─ Default artwork
  └─ Cached continue watching

Critical failure → Minimal:
  ├─ Static list of popular titles
  └─ User can still browse and play
```

---

## Disaster Recovery

### 🔥 Q: DR strategies (RTO and RPO).

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

### ⭐ Q: Multi-region architecture.

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

### 🔥 Q: Design a URL shortener (infrastructure focus).

**Requirements:**
- Functional: Short URL → redirect to long URL, custom aliases, expiration
- Scale: 100M URLs created/month, 10B redirects/month
- Non-functional: Low latency (<50ms p99), high availability (99.99%)

**Back-of-envelope:**
```
Writes (URL creation):
  100M/month = ~40 writes/second (peak: ~200/s)

Reads (redirects):
  10B/month = ~4K reads/second (peak: ~20K/s)
  Read-heavy: 100:1 read/write ratio

Storage:
  100M URLs/month × 500 bytes/URL = 50 GB/month
  5 years: ~3 TB total

Bandwidth:
  20K redirects/s × 500 bytes = 10 MB/s
```

**High-level design:**
```
┌─────────────────────────────────────────────────────────┐
│                       Client                             │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│              CloudFront (CDN)                            │
│  - Cache GET requests (redirects)                        │
│  - TTL: 1 hour                                           │
│  - Geo-distribution                                      │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│              ALB (Application Load Balancer)             │
└────────┬───────────────────────────┬────────────────────┘
         │                           │
         ▼                           ▼
┌────────────────┐          ┌────────────────┐
│  API Servers   │          │  API Servers   │
│  (ECS/EKS)     │          │  (ECS/EKS)     │
│                │          │                │
│  /create       │          │  /create       │
│  /:shortURL    │          │  /:shortURL    │
└────────┬───────┘          └────────┬───────┘
         │                           │
         └───────────┬───────────────┘
                     │
         ┌───────────┼──────────────┐
         ▼           ▼              ▼
    ┌─────────┐ ┌─────────┐   ┌──────────┐
    │ Redis   │ │  Aurora  │   │    S3    │
    │(cache)  │ │(DB)      │   │(archive) │
    │         │ │          │   │          │
    │ Hot:    │ │ All URLs │   │ Old URLs │
    │ 1M URLs │ │          │   │ (cold)   │
    └─────────┘ └─────────┘   └──────────┘
```

**Deep dives:**

**1. URL shortening algorithm:**
```
Base62 encoding (a-z, A-Z, 0-9):
  7 characters = 62^7 = 3.5 trillion unique URLs

Generate short code:
  Option 1: Auto-increment counter (simple but needs distributed counter)
  Option 2: Hash long URL + collision handling
  Option 3: Random + collision check (preferred)

random_id = generate_random(7_chars)
while db.exists(random_id):
    random_id = generate_random(7_chars)
db.insert(short_url=random_id, long_url=long_url)
```

**2. Database schema:**
```sql
CREATE TABLE urls (
    short_url VARCHAR(10) PRIMARY KEY,
    long_url TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,
    user_id INT,
    click_count BIGINT DEFAULT 0,
    INDEX idx_user_id (user_id),
    INDEX idx_expires_at (expires_at)
);
```

**3. Caching strategy:**
```
Cache-Aside pattern:
  1. Check Redis for short_url
  2. Cache HIT → return long_url (redirect)
  3. Cache MISS → query DB → write to Redis (TTL: 1 hour) → redirect

Eviction: LRU (keep hot URLs in cache)
Cache size: 1M URLs × 500 bytes = 500 MB (fits in memory)
```

**4. Redirect flow:**
```
GET /abc123
  ↓
CloudFront cache HIT (90% of requests) → 301/302 redirect (cached)
  ↓ (miss)
API server
  ↓
Redis cache HIT (9% of requests) → 301/302 redirect
  ↓ (miss)
DB query (1% of requests) → 301/302 redirect + update Redis
```

**5. Analytics (click tracking):**
```
Option 1: Update DB on each click (slow, write-heavy)
Option 2: Async write to Kafka → batch process → write to DB
  GET /abc123
    → Redirect immediately (low latency)
    → Background: Log click event to Kafka
    → Stream processor aggregates clicks per minute
    → Batch insert to DB every minute
```

**6. Custom aliases:**
```
User requests: short.ly/google → https://google.com
Check if "google" is available
If taken → return 409 Conflict
If free → insert into DB
```

**7. Expiration handling:**
```
Cron job (daily):
  DELETE FROM urls WHERE expires_at < NOW()

Or lazy deletion:
  On redirect, check expires_at
  If expired → return 404 + async delete
```

**Failure modes:**
- DB down: Serve from Redis cache (stale data OK for redirects)
- Redis down: Serve from DB directly (slower but works)
- Region failure: Multi-region Aurora + Route 53 failover

**Cost optimization:**
- Archive old URLs (>1 year) to S3 (lazy load if accessed)
- Use CloudFront to cache 90% of redirects (reduce API/DB load)

---

### 🔥 Q: Design a rate limiter service.

**Requirements:**
- Protect APIs from abuse (per user, per IP, per API key)
- Distributed (multiple API servers)
- Configurable limits (100 req/min, 10K req/day)
- Low latency overhead (<10ms)
- High availability (don't become SPOF)

**High-level design:**
```
┌──────────┐
│  Client  │
└─────┬────┘
      │
      ▼
┌─────────────────────────────────────┐
│      API Gateway / Envoy Proxy       │
│  - First line rate limiting          │
│  - Per-IP limits (coarse)            │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│      Application Load Balancer       │
└────────┬────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│       API Servers (rate limit logic)  │
│  - Per-user, per-API-key limits       │
│  - Call Rate Limiter Service          │
└────────┬─────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────┐
│    Rate Limiter Service (Redis)       │
│  - Sliding window counter              │
│  - Token bucket                        │
│  - Distributed coordination            │
└────────────────────────────────────────┘
```

**Redis-based implementation (Token Bucket):**
```lua
-- Lua script (atomic)
local key = KEYS[1]               -- rate_limit:user123:api_call
local capacity = tonumber(ARGV[1])  -- 100 tokens
local rate = tonumber(ARGV[2])      -- 10 tokens/second
local now = tonumber(ARGV[3])       -- current timestamp

local tokens = redis.call('HGET', key, 'tokens')
local last_refill = redis.call('HGET', key, 'last_refill')

if tokens == false then
    tokens = capacity
    last_refill = now
else
    tokens = tonumber(tokens)
    last_refill = tonumber(last_refill)
    
    -- Refill tokens
    local elapsed = now - last_refill
    local refill = math.floor(elapsed * rate)
    tokens = math.min(capacity, tokens + refill)
    last_refill = now
end

if tokens >= 1 then
    tokens = tokens - 1
    redis.call('HSET', key, 'tokens', tokens)
    redis.call('HSET', key, 'last_refill', last_refill)
    redis.call('EXPIRE', key, 3600)
    return 1  -- Allow
else
    return 0  -- Deny
end
```

**API:**
```python
def is_rate_limited(user_id, limit_key, capacity, rate):
    key = f"rate_limit:{user_id}:{limit_key}"
    result = redis.eval(lua_script, key, capacity, rate, time.time())
    return result == 0  # 0 = denied, 1 = allowed

# Usage
if is_rate_limited(user_id, "api_call", capacity=100, rate=10):
    return 429, {"error": "Rate limit exceeded"}
```

**Handling distributed scenarios:**
```
Problem: 3 API servers, Redis is shared
Solution: Redis atomic operations (Lua script)

Each server:
  1. Call Redis with EVAL (atomic token bucket check)
  2. Redis returns allow/deny
  3. Server enforces decision

Race condition: Two servers call Redis simultaneously
  → Lua script is atomic → Redis serializes requests → safe
```

**Multi-tier rate limiting:**
```
Tier 1: API Gateway (Envoy)
  - Per-IP: 1000 req/s (coarse, protect against DDoS)
  - Local rate limiting (no Redis call)

Tier 2: Application (per-user, per-API-key)
  - Per-user: 100 req/min
  - Call Redis for distributed coordination

Tier 3: Database
  - Connection pooling limits concurrent queries
```

**Failure modes:**
```
Redis down:
  Option 1: Fail open (allow all requests, log warning)
  Option 2: Fail closed (deny all requests, safer)
  Option 3: Local rate limiting (in-memory, approximate)

Preferred: Fail open with monitoring alert
  → Don't let rate limiter take down entire API
```

**Monitoring:**
```
Metrics:
  - Rate limit denials per user/key (track abuse)
  - Redis latency (ensure <10ms)
  - Redis availability
  - Rate limiter service latency

Alerts:
  - Rate limit denials spike (attack?)
  - Redis down
  - Rate limiter latency >50ms
```

---

### 🔥 Q: Design a monitoring/alerting system for microservices.

**Requirements:**
- 1000 microservices, 10K pods across 20 K8s clusters
- Metrics, logs, traces, alerts
- Query latency <1s for dashboards
- Alert latency <1 minute
- Retention: Metrics (30 days), Logs (7 days hot, 90 days cold), Traces (7 days)

**High-level architecture:**
```
┌──────────────────────────────────────────────────────────┐
│                   Microservices (K8s Pods)                │
│  - Expose /metrics (Prometheus format)                    │
│  - Log to stdout (12-factor app)                          │
│  - Traces via OpenTelemetry SDK                           │
└─────────┬────────────────────────────────────────────────┘
          │
  ┌───────┼──────────────────────────────┐
  │       │                              │
  ▼       ▼                              ▼
┌────────────┐   ┌────────────────┐   ┌──────────────┐
│ Prometheus │   │ Fluent Bit     │   │ OTel         │
│ (scrape)   │   │ (DaemonSet)    │   │ Collector    │
│            │   │ - Parse logs   │   │ (sidecar)    │
│ - Metrics  │   │ - Enrich K8s   │   │ - Sample     │
│ - 15s scrape│   │   metadata     │   │ - Buffer     │
└──────┬─────┘   └──────┬─────────┘   └──────┬───────┘
       │                │                    │
       ▼                ▼                    ▼
┌────────────┐   ┌────────────────┐   ┌──────────────┐
│ Thanos /   │   │ Kafka          │   │ Tempo        │
│ Cortex     │   │ (buffer)       │   │ (traces)     │
│ (long-term)│   └───────┬────────┘   │              │
│            │           │             │ - Trace ID   │
│ - Aggregate│           │             │   index      │
│ - Downsample│          ▼             └──────────────┘
└──────┬─────┘   ┌────────────────┐
       │         │ Loki           │
       │         │ (logs)         │
       │         │ - Index labels │
       │         │ - Store chunks │
       │         └────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│                Grafana (Unified UI)                   │
│  - Dashboards (metrics from Prometheus/Thanos)        │
│  - Log explorer (Loki)                                │
│  - Trace viewer (Tempo)                               │
│  - Correlate metrics → logs → traces                  │
└──────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│            Alertmanager (Prometheus)                  │
│  - Route alerts by severity/team                      │
│  - Deduplication, grouping, silencing                 │
│  - Integrations: PagerDuty, Slack, Email              │
└──────────────────────────────────────────────────────┘
```

**Metrics pipeline:**
```
1. Prometheus scrapes /metrics every 15s
   - Per-pod metrics: CPU, memory, request rate, latency, errors
   - Custom business metrics: orders/sec, queue depth

2. Local Prometheus (per cluster)
   - Stores 2 days locally
   - Query for real-time dashboards

3. Thanos/Cortex (global)
   - Aggregates all Prometheus instances
   - Downsamples: 15s → 5m → 1h (reduce storage)
   - Stores 30 days
   - Query for historical dashboards
```

**Logs pipeline:**
```
1. Pods log to stdout (JSON format):
   {"level":"error", "msg":"DB timeout", "trace_id":"abc123", "user_id":456}

2. Fluent Bit DaemonSet:
   - Tail /var/log/containers/*.log
   - Parse JSON, enrich with K8s metadata (pod, namespace, labels)
   - Send to Kafka (buffer)

3. Kafka:
   - Decouple producers/consumers
   - Handle backpressure (Loki ingest spikes)

4. Loki:
   - Index: labels (namespace, pod, level, service)
   - Store: log chunks in S3
   - Retention: 7 days hot (query-optimized), 90 days cold (S3)
```

**Traces pipeline:**
```
1. Service instruments with OpenTelemetry SDK:
   span = tracer.start_span("api_call")
   span.set_attribute("user_id", 123)
   span.end()

2. OTel Collector (sidecar or DaemonSet):
   - Receives traces via gRPC
   - Samples: 1% of successful requests, 100% of errors
   - Batches and sends to Tempo

3. Tempo:
   - Stores traces in object storage (S3, GCS)
   - Indexes trace IDs
   - Queries: "Find trace by ID", "Find slow traces"
```

**Unified observability (RED method):**
```
For each service, track:
  - Rate:   requests/second (metric)
  - Errors: error rate % (metric)
  - Duration: latency p50/p95/p99 (metric)

Correlation:
  1. Dashboard shows error rate spike
  2. Click → Loki query for error logs in that time range
  3. Click log → Trace ID → Tempo trace view
  4. Trace shows slow DB query → root cause
```

**Alerting rules (Prometheus):**
```yaml
groups:
- name: sre_alerts
  rules:
  - alert: HighErrorRate
    expr: |
      rate(http_requests_total{status=~"5.."}[5m]) 
      / rate(http_requests_total[5m]) > 0.01
    for: 5m
    labels:
      severity: critical
      team: backend
    annotations:
      summary: "High error rate on {{ $labels.service }}"
      dashboard: "https://grafana/d/service/{{ $labels.service }}"

  - alert: HighLatency
    expr: |
      histogram_quantile(0.99, 
        rate(http_request_duration_seconds_bucket[5m])
      ) > 1.0
    for: 5m
    labels:
      severity: warning
      team: backend
```

**Alertmanager routing:**
```yaml
route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s          # Wait for grouped alerts
  group_interval: 5m       # Send grouped alerts every 5m
  repeat_interval: 12h     # Re-send if still firing
  receiver: default

  routes:
  - match:
      severity: critical
    receiver: pagerduty

  - match:
      team: backend
    receiver: slack-backend

receivers:
- name: pagerduty
  pagerduty_configs:
  - service_key: '<key>'

- name: slack-backend
  slack_configs:
  - channel: '#alerts-backend'
```

**Scale considerations:**
```
Metrics:
  - 10K pods × 1000 metrics/pod × 8 bytes/sample × 4 samples/min × 60 min
    = ~20 GB/hour per cluster
  - Downsampling: 15s → 5m (12x reduction after 2 days)

Logs:
  - 10K pods × 100 log lines/sec × 500 bytes/line = 500 MB/s
  - Sampling: Debug logs only for flagged requests

Traces:
  - 1% sampling for successful requests (reduce volume)
  - 100% sampling for errors (keep critical data)
```

---

### ⭐ Q: Design a CI/CD platform for 1000 engineers and 100+ microservices.

**Requirements:**
- 1000 engineers, 100+ services, 500+ deploys/day
- Fast feedback (<10 min build, <5 min test)
- Safe deployments (canary, rollback)
- Multi-environment (dev → staging → prod)

**High-level design:**
```
GitHub (monorepo) → CI (GH Actions) → ECR → GitOps repo → ArgoCD → K8s (dev/staging/prod)
```

**Key implementation details:**
- Monorepo with Nx for incremental builds (only changed services)
- Self-hosted runners on EKS (cheaper, faster for 1000 engineers)
- Shared pipeline templates (80% code reuse)
- Auto-deploy to dev/staging, manual approval for prod
- Canary deployments via Argo Rollouts (10% → 25% → 50% → 100% with automatic rollback on high error rate)
- Backstage service catalog for standardization

### ⭐ Q: Design a logging platform for a large-scale K8s cluster.

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

### 🔥 Q: Design infrastructure for a global e-commerce platform.

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

### ⭐ Q: Design a multi-region active-active architecture.

**Requirements:**
- Service available in US and EU simultaneously
- Both regions serve live traffic (active-active)
- Data replicated across regions
- Failover time <1 minute
- No single point of failure

**Architecture:**
```
┌────────────────────────────────────────────────────────────┐
│                       Route 53                              │
│  Geoproximity routing:                                      │
│    - US users → us-east-1                                   │
│    - EU users → eu-west-1                                   │
│  Health checks:                                             │
│    - Failover if primary region down                        │
└────────────┬───────────────────────────┬───────────────────┘
             │                           │
             ▼                           ▼
┌──────────────────────────┐  ┌──────────────────────────┐
│   US Region (us-east-1)   │  │   EU Region (eu-west-1)   │
│                           │  │                           │
│  ALB                      │  │  ALB                      │
│   ↓                       │  │   ↓                       │
│  EKS (app pods)           │  │  EKS (app pods)           │
│   ↓                       │  │   ↓                       │
│  Aurora Global DB         │←→│  Aurora Global DB         │
│   - Writer (primary)      │  │   - Reader (can promote)  │
│   - Read replicas         │  │   - <1s replication lag   │
│                           │  │                           │
│  DynamoDB Global Tables   │←→│  DynamoDB Global Tables   │
│   - Multi-master writes   │  │   - Multi-master writes   │
│   - Conflict resolution   │  │   - Eventual consistency  │
│                           │  │                           │
│  ElastiCache (Redis)      │  │  ElastiCache (Redis)      │
│   - Local cache           │  │   - Local cache           │
│   - Warm on startup       │  │   - Warm on startup       │
│                           │  │                           │
│  S3 + CRR enabled         │→ │  S3 (replica)             │
└───────────────────────────┘  └───────────────────────────┘
```

**Key design decisions:**

**1. Database strategy (write patterns):**
```
Option A: Single-writer (Aurora Global DB)
  - Primary writes to us-east-1
  - Replicate to eu-west-1 (async, <1s lag)
  - On primary failure: Promote eu-west-1 to writer (~1 min)
  - Pros: No write conflicts, simple
  - Cons: Cross-region latency for EU writes

Option B: Multi-writer (DynamoDB Global Tables)
  - Both regions accept writes
  - Conflict resolution: Last-writer-wins
  - Pros: Low latency everywhere
  - Cons: Eventual consistency, conflict handling needed

Hybrid approach:
  - User profiles, orders: Single-writer (Aurora)
  - Sessions, carts: Multi-writer (DynamoDB)
```

**2. Failover automation:**
```
Detection:
  - Route 53 health checks (every 30s)
  - Custom health endpoint: /health
    → Check: DB reachable, cache reachable, downstream services OK

Failover:
  1. Route 53 marks us-east-1 unhealthy
  2. All traffic routed to eu-west-1 (DNS TTL: 60s)
  3. Auto-promote eu-west-1 Aurora to writer (if needed)
  4. Alert on-call team

Testing:
  - Monthly chaos engineering drill
  - Trigger failover in staging
  - Measure: Detection time, failover time, data loss
```

**3. Data consistency:**
```
Strong consistency (synchronous replication):
  - Use for: Financial transactions, inventory
  - Write to both regions synchronously
  - Latency: +50-100ms cross-region
  - Availability: Lower (both must be up)

Eventual consistency (asynchronous replication):
  - Use for: User profiles, product catalog
  - Write to local region, replicate async
  - Latency: Low (local writes)
  - Availability: High (regions independent)

Design per use case:
  - Cart: Eventual (OK if stale by 1s)
  - Checkout: Strong (no overselling)
```

**4. Session management:**
```
Problem: User session in us-east-1, failover to eu-west-1 → session lost

Solution A: Sticky sessions with replication
  - Store session in DynamoDB Global Tables
  - Both regions can read/write
  - User transparently fails over

Solution B: JWT tokens
  - Stateless sessions (no server-side storage)
  - Token contains user ID, permissions
  - Works in any region
```

**5. Cache warming:**
```
Problem: Failover to cold eu-west-1 cache → slow performance

Solution:
  - Pre-warm cache on startup (top 1000 products)
  - Lazy-load on miss (acceptable latency)
  - Monitor cache hit rate (alert if <80%)
```

---

### 💡 Q: Design a distributed logging pipeline at scale.

**Requirements:**
- 10K services, 100 GB/s log volume
- Search logs by trace ID, user ID, service name
- Retention: 7 days hot (search), 90 days cold (archive)
- Query latency <1s
- Cost-effective

**Architecture:**
```
┌─── Services (10K pods) ────────────────────────────────┐
│  Log to stdout (structured JSON):                       │
│  {"ts":"2025-01-01T12:00:00Z", "level":"error",         │
│   "msg":"DB timeout", "trace_id":"abc123",              │
│   "user_id":456, "service":"api"}                       │
└──────────────────┬─────────────────────────────────────┘
                   │ (Fluent Bit DaemonSet tails logs)
                   ▼
┌─────────────────────────────────────────────────────────┐
│  Fluent Bit (lightweight log shipper)                    │
│  - Parse JSON                                            │
│  - Enrich with K8s metadata (pod, namespace)             │
│  - Sample high-volume services (keep errors, sample 1%)  │
│  - Buffer to Kafka                                       │
└──────────────────┬──────────────────────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────────────────────┐
│  Kafka (multi-broker, partitioned by service)            │
│  - Decouple ingest from processing                       │
│  - Handle backpressure (Loki slow → buffer in Kafka)     │
│  - Multiple consumers (Loki, S3 archiver)                │
└──────┬──────────────────────────────────┬───────────────┘
       │                                  │
       ▼                                  ▼
┌─────────────────┐              ┌─────────────────┐
│  Loki (hot)     │              │  S3 (cold)      │
│  - 7 days       │              │  - 90 days      │
│  - Index labels │              │  - Parquet fmt  │
│  - Store chunks │              │  - Athena query │
│    in S3        │              │  - Cost: $0.02  │
│  - Cost: $0.20  │              │    /GB/month    │
│    /GB/month    │              │                 │
└────────┬────────┘              └─────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Grafana (query UI)                                      │
│  - LogQL queries: {service="api", level="error"}         │
│  - Trace ID correlation                                  │
│  - Alerts on log patterns (e.g. "DB timeout" >10/min)    │
└─────────────────────────────────────────────────────────┘
```

**Key optimizations:**

**1. Sampling strategy:**
```
High-volume services (e.g., health checks):
  - Sample 1% of INFO logs
  - Keep 100% of WARN/ERROR logs
  - Result: 100x volume reduction, no critical data lost

Trace-based sampling:
  - If trace marked for sampling (1%), keep all logs for that trace
  - Enables end-to-end debugging of sampled requests
```

**2. Cost optimization:**
```
Loki vs Elasticsearch:
  - Loki: Index labels only, store raw logs in S3
  - ES: Index full text (expensive)
  - Cost: Loki is 10x cheaper at scale

Retention tiers:
  - Hot (7 days): Loki (fast queries)
  - Cold (90 days): S3 + Athena (batch queries, rare access)
  - Archive (1+ years): S3 Glacier (compliance, $0.004/GB/month)
```

**3. Query performance:**
```
Loki label best practices:
  - Index: service, namespace, level, pod (cardinality <1000)
  - Don't index: trace_id, user_id, message (high cardinality)
  - Query: {service="api"} |= "trace_id=abc123" (grep after index)

Elasticsearch alternative (for full-text search):
  - Use for: Security logs, compliance (need full-text search)
  - Cost: 10x more expensive than Loki
```

---

## Capacity Planning

### 🔥 Q: How to estimate infrastructure needs.

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

**Books:**
- **Designing Data-Intensive Applications** — Martin Kleppmann (THE system design bible)
- **System Design Interview (Vol 1 & 2)** — Alex Xu (visual explanations, product-focused but overlaps SRE)
- **The Site Reliability Workbook** — Google SRE team (practical SRE patterns)
- **Database Internals** — Alex Petrov (deep dive on storage, replication, consensus)

**Online:**
- **Google SRE Book** — https://sre.google/sre-book/ (free, foundational)
- **AWS Architecture Center** — https://aws.amazon.com/architecture/ (real-world reference architectures)
- **High Scalability Blog** — http://highscalability.com (case studies: Netflix, Uber, etc.)
- **ByteByteGo** — https://bytebytego.com (Alex Xu's visual system design guides)
- **Reliability Engineering at Google** — https://sre.google/resources/ (SLO, error budgets, monitoring)

**Practice:**
- Draw architectures on a whiteboard (or Excalidraw) — visual thinking is key
- Estimate back-of-envelope (QPS, storage, bandwidth) for every design
- Study post-mortems: AWS outages, GitHub incidents, Cloudflare failures
- Chaos engineering: Try breaking your designs (what if DB goes down? Region fails?)
