---
title: Monitoring
nav_order: 60
description: "Monitoring — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# Monitoring & Observability — DevOps Interview Preparation

## Table of Contents
- [Observability Fundamentals](#observability-fundamentals)
- [Prometheus](#prometheus)
- [Grafana](#grafana)
- [ELK / EFK Stack](#elk--efk-stack)
- [Loki](#loki)
- [Alerting Best Practices](#alerting-best-practices)
- [Distributed Tracing](#distributed-tracing)
- [SLIs, SLOs, SLAs](#slis-slos-slas)
- [Incident Management](#incident-management)
- [Scenario-Based Questions](#scenario-based-questions)

---

## Observability Fundamentals

### Q: Three Pillars of Observability.

| Pillar | What | Tools |
|--------|------|-------|
| **Metrics** | Numeric measurements over time | Prometheus, CloudWatch, Datadog |
| **Logs** | Discrete events with context | ELK, Loki, CloudWatch Logs, Splunk |
| **Traces** | Request flow across services | Jaeger, Tempo, X-Ray, Zipkin |

### Q: Monitoring methodologies.

**USE Method (for infrastructure)** — Brendan Gregg:
| Signal | Questions |
|--------|-----------|
| **Utilization** | How busy is the resource? (CPU %, disk %) |
| **Saturation** | How much work is queued? (load average, queue depth) |
| **Errors** | How many errors? (disk errors, network errors) |

**RED Method (for services)** — Tom Wilkie:
| Signal | Questions |
|--------|-----------|
| **Rate** | Requests per second |
| **Errors** | Errors per second |
| **Duration** | Latency distribution (p50, p95, p99) |

**Four Golden Signals (Google SRE):**
| Signal | Description |
|--------|-------------|
| **Latency** | Time to service a request |
| **Traffic** | Demand on the system (rps) |
| **Errors** | Rate of failed requests |
| **Saturation** | How full the service is |

---

## Prometheus

### Q: Prometheus architecture.

```
┌─── Targets ──────────────────────────────┐
│  App /metrics  │  Node Exporter  │  cAdvisor │
└──────┬──────────────┬──────────────┬─────┘
       │              │              │
       │    Pull (HTTP scrape)       │
       ▼              ▼              ▼
┌──────────────────────────────────────────┐
│              Prometheus Server             │
│  ┌────────┐  ┌──────────┐  ┌──────────┐ │
│  │ Scraper │  │  TSDB    │  │ PromQL   │ │
│  │         │  │ (storage)│  │ (query)  │ │
│  └────────┘  └──────────┘  └──────────┘ │
│  ┌────────────────────────────────────┐  │
│  │      Alert Rules Engine            │  │
│  └──────────────┬─────────────────────┘  │
└─────────────────┼────────────────────────┘
                  │
         ┌────────▼────────┐
         │  Alertmanager    │ → Slack, PagerDuty, Email
         └─────────────────┘
                  │
         ┌────────▼────────┐
         │  Grafana         │ (Visualization)
         └─────────────────┘
```

**Key characteristics:**
- **Pull-based** — Prometheus scrapes targets (vs push-based like StatsD)
- **TSDB** — Time-series database optimized for metric storage
- **PromQL** — Powerful query language
- **Service discovery** — Kubernetes, Consul, EC2, file-based

### Q: Prometheus metric types.

| Type | Description | Example |
|------|-------------|---------|
| **Counter** | Only goes up (resets on restart) | Total requests, total errors |
| **Gauge** | Can go up or down | Temperature, memory usage, active connections |
| **Histogram** | Samples in configurable buckets | Request latency distribution |
| **Summary** | Client-side quantiles | Request latency percentiles |

```
# Counter
http_requests_total{method="GET", status="200"} 12345

# Gauge
node_memory_available_bytes 8589934592

# Histogram (multiple series per metric)
http_request_duration_seconds_bucket{le="0.1"} 50000
http_request_duration_seconds_bucket{le="0.5"} 80000
http_request_duration_seconds_bucket{le="1.0"} 95000
http_request_duration_seconds_bucket{le="+Inf"} 100000
http_request_duration_seconds_count 100000
http_request_duration_seconds_sum 45000
```

### Q: Essential PromQL queries.

```promql
# Request rate (per second over 5 minutes)
rate(http_requests_total[5m])

# Error rate percentage
sum(rate(http_requests_total{status=~"5.."}[5m])) /
sum(rate(http_requests_total[5m])) * 100

# 95th percentile latency
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# CPU usage percentage
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory usage percentage
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Top 5 pods by memory usage
topk(5, container_memory_usage_bytes{namespace="production"})

# Disk space remaining
node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100

# Pod restart rate
rate(kube_pod_container_status_restarts_total[1h]) > 0

# Aggregation by label
sum by(namespace) (kube_pod_info)
```

### Q: Prometheus in Kubernetes (ServiceMonitor).

```yaml
# ServiceMonitor (Prometheus Operator CRD)
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  labels:
    release: prometheus       # Must match Prometheus selector
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    interval: 15s
    path: /metrics

---
# PrometheusRule (alerting)
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-alerts
spec:
  groups:
  - name: my-app
    rules:
    - alert: HighErrorRate
      expr: |
        sum(rate(http_requests_total{status=~"5.."}[5m])) /
        sum(rate(http_requests_total[5m])) > 0.05
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "High error rate ({{ $value | humanizePercentage }})"
        runbook: "https://wiki.example.com/runbooks/high-error-rate"
```

### Q: Prometheus long-term storage.

Prometheus is designed for **short-term storage** (15-30 days). For long-term:

| Solution | Type | Key Feature |
|----------|------|-------------|
| **Thanos** | Sidecar + object storage | Global query view, deduplication |
| **Cortex/Mimir** | Remote write + object storage | Multi-tenant, horizontally scalable |
| **VictoriaMetrics** | Drop-in replacement | Better compression, faster queries |

---

## Grafana

### Q: Grafana dashboard best practices.

- **USE/RED structure** — Organize dashboards by methodology
- **Variables/templates** — Allow filtering by namespace, service, pod
- **Consistent layout** — Golden signals at top, details below
- **Link dashboards** — Drill-down from overview → detail → logs
- **Annotations** — Mark deployments, incidents on graphs
- **Alert integration** — Visual alert states on panels

**Dashboard hierarchy:**
```
Executive Overview (SLOs, business metrics)
    └── Service Overview (RED metrics per service)
        └── Service Detail (per-instance, per-pod)
            └── Infrastructure (node CPU, memory, disk, network)
                └── Debug (traces, logs)
```

---

## ELK / EFK Stack

### Q: ELK Stack architecture.

```
┌─── Log Sources ──────────────────────────────┐
│  Application  │  System  │  Infrastructure    │
└──────┬──────────────┬──────────────┬─────────┘
       │              │              │
       ▼              ▼              ▼
┌──────────────────────────────────────────┐
│  Beats / Fluentd / Fluent Bit             │  (Collection & shipping)
│  (Filebeat, Metricbeat, etc.)            │
└──────────────────┬───────────────────────┘
                   │
┌──────────────────▼───────────────────────┐
│  Logstash / Fluentd                       │  (Processing, parsing, enriching)
│  (Grok patterns, filters, transforms)    │
└──────────────────┬───────────────────────┘
                   │
┌──────────────────▼───────────────────────┐
│  Elasticsearch                            │  (Storage, indexing, search)
│  (Distributed, full-text search engine)  │
└──────────────────┬───────────────────────┘
                   │
┌──────────────────▼───────────────────────┐
│  Kibana                                   │  (Visualization, dashboards)
└──────────────────────────────────────────┘
```

**EFK variant:** Replace Logstash with **Fluentd** (lighter, K8s-native)

### Q: Structured logging best practices.

```json
{
  "timestamp": "2024-01-15T10:30:00.123Z",
  "level": "ERROR",
  "service": "api-gateway",
  "trace_id": "abc123def456",
  "span_id": "789ghi",
  "message": "Failed to process payment",
  "error": "Connection refused",
  "user_id": "u-12345",
  "request_id": "req-67890",
  "duration_ms": 2500,
  "method": "POST",
  "path": "/api/v1/payments",
  "status_code": 500
}
```

**Rules:**
- Use **JSON format** (machine-parseable)
- Include **correlation IDs** (trace_id, request_id)
- Use **standard severity levels** (DEBUG, INFO, WARN, ERROR, FATAL)
- **Don't log sensitive data** (passwords, tokens, PII)
- Include **context** (user_id, request_path, service name)

---

## Loki

### Q: Loki vs Elasticsearch for logs.

| Feature | Loki | Elasticsearch |
|---------|------|---------------|
| Index strategy | **Labels only** (not full-text) | Full-text indexing |
| Storage cost | Much lower | Higher (indexes everything) |
| Query speed | Slower for full-text | Faster for complex queries |
| Resource usage | Low | High (RAM-hungry) |
| K8s integration | Native (Grafana) | Via Kibana |
| Best for | Cloud-native, K8s logs | Complex log analytics |

**LogQL example:**
```logql
# Filter by label and content
{namespace="production", app="api"} |= "error"

# Parse and aggregate
{app="nginx"} | json | status >= 500 | line_format "{{.method}} {{.path}} {{.status}}"

# Count errors per minute
sum(rate({app="api"} |= "error" [1m])) by (level)
```

---

## Alerting Best Practices

### Q: How to design effective alerting.

**Alert on symptoms, not causes:**
```
Bad:  Alert when CPU > 90%     (cause — maybe it's fine under load)
Good: Alert when latency p99 > 500ms  (symptom — users are affected)
```

**Alert hierarchy:**
| Severity | Response | Example |
|----------|----------|---------|
| **Critical/P1** | Wake someone up | Service down, data loss risk |
| **Warning/P2** | Investigate in hours | Error rate elevated, disk 85% |
| **Info/P3** | Investigate in days | Certificate expiring in 30 days |

**Alertmanager routing:**
```yaml
route:
  receiver: 'default-slack'
  group_by: ['alertname', 'namespace']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
  - match:
      severity: critical
    receiver: 'pagerduty-critical'
    repeat_interval: 15m
  - match:
      severity: warning
    receiver: 'slack-warnings'

receivers:
- name: 'pagerduty-critical'
  pagerduty_configs:
  - service_key: '<key>'
- name: 'slack-warnings'
  slack_configs:
  - channel: '#alerts'
    send_resolved: true
- name: 'default-slack'
  slack_configs:
  - channel: '#monitoring'
```

**Anti-patterns:**
- Alert fatigue (too many non-actionable alerts)
- Not having runbooks linked to alerts
- Alerting on causes instead of symptoms
- No escalation path
- Duplicate alerts from multiple systems

---

## Distributed Tracing

### Q: How does distributed tracing work?

```
User Request
    │
    ▼
┌─── API Gateway ──── Trace ID: abc123 ─────────────────┐
│  Span 1: gateway (50ms total)                          │
│    │                                                    │
│    ├──► Span 2: auth-service (10ms)                    │
│    │                                                    │
│    ├──► Span 3: user-service (25ms)                    │
│    │      │                                             │
│    │      └──► Span 4: postgres query (15ms)           │
│    │                                                    │
│    └──► Span 5: cache-lookup (5ms)                     │
└─────────────────────────────────────────────────────────┘
```

**Key concepts:**
- **Trace** — Full journey of a request (collection of spans)
- **Span** — Single operation within a trace (has start time, duration, tags)
- **Context Propagation** — Passing trace ID across service boundaries (headers)
- **Sampling** — Collect subset of traces (head-based, tail-based)

**OpenTelemetry** — The standard for instrumentation:
```
Application → OTEL SDK → OTEL Collector → Backend (Jaeger/Tempo/X-Ray)
                              │
                         ┌────┴────┐
                         │         │
                      Metrics    Logs
                    (Prometheus) (Loki)
```

---

## SLIs, SLOs, SLAs

### Q: Define SLI, SLO, SLA with examples.

| Term | Definition | Example |
|------|-----------|---------|
| **SLI** (Indicator) | Metric that measures service quality | 99.2% of requests < 200ms |
| **SLO** (Objective) | Target value for an SLI | 99.9% of requests should be < 200ms |
| **SLA** (Agreement) | Contract with consequences | 99.95% uptime or customer gets credits |

**SLI → SLO → SLA:** Measure → Target → Contract

### Q: How to calculate error budgets.

```
SLO: 99.9% availability per month

Error Budget = 100% - 99.9% = 0.1%

In a 30-day month:
  30 days × 24 hours × 60 minutes = 43,200 minutes
  43,200 × 0.001 = 43.2 minutes of allowed downtime

If you've used 30 minutes of downtime this month:
  Remaining budget = 43.2 - 30 = 13.2 minutes
  Budget consumed = 30/43.2 = 69.4%
```

**Error budget policies:**
- Budget remaining → Deploy freely, experiment
- Budget < 25% → Slow down deployments, prioritize reliability
- Budget exhausted → Freeze deployments, focus on stability

### Q: Common SLIs to track.

| Service Type | SLI | Measurement |
|-------------|-----|-------------|
| **API** | Availability | % of successful requests (non-5xx) |
| **API** | Latency | % of requests under threshold (p99 < 500ms) |
| **Pipeline** | Freshness | % of data processed within time window |
| **Storage** | Durability | % of data retained without loss |
| **Batch** | Throughput | % of jobs completing within deadline |

---

## Incident Management

### Q: Incident response process.

```
1. DETECT
   - Automated alerts (Prometheus, CloudWatch)
   - Customer reports
   - Status page monitoring

2. TRIAGE
   - Severity classification (P1-P4)
   - Assign incident commander
   - Open incident channel (Slack/Teams)

3. MITIGATE
   - Restore service ASAP (rollback, scale, failover)
   - Fix != Root cause — just stop the bleeding

4. COMMUNICATE
   - Internal: Engineering, support, leadership
   - External: Status page, customer notification

5. RESOLVE
   - Confirm service restored
   - Monitor for recurrence

6. POST-MORTEM (Blameless)
   - Timeline of events
   - Root cause analysis (5 Whys)
   - Impact assessment
   - Action items with owners and deadlines
   - Share learnings broadly
```

### Q: What are DORA metrics?

| Metric | Elite | High | Medium | Low |
|--------|-------|------|--------|-----|
| **Deployment Frequency** | On-demand (multiple/day) | Weekly-Monthly | Monthly-Biannual | <1/6months |
| **Lead Time for Changes** | <1 hour | 1 day-1 week | 1-6 months | >6 months |
| **Mean Time to Recovery** | <1 hour | <1 day | 1 day-1 week | >6 months |
| **Change Failure Rate** | 0-15% | 16-30% | 16-30% | >30% |

---

## Scenario-Based Questions

### Q: Application is slow but infrastructure looks fine. How to investigate?

```
1. Confirm "slow" — Is it latency, throughput, or specific endpoints?
   Check: Grafana dashboards, p50/p95/p99 latency

2. Check application metrics:
   - Request rate, error rate, latency per endpoint
   - Database query duration
   - External API call duration
   - Cache hit/miss ratio

3. Distributed tracing:
   - Find slow traces in Jaeger/Tempo
   - Identify which span is the bottleneck
   - Is it DB? External API? Application logic?

4. Check dependencies:
   - Database: slow queries, connection pool exhaustion
   - Cache: miss rate increased? eviction rate?
   - External APIs: increased latency?

5. Application profiling:
   - CPU profiling (flame graphs)
   - Memory profiling (GC pauses in Java/Go)
   - Connection pool metrics
   - Thread/goroutine counts

6. Recent changes:
   - New deployment? Config change? Traffic spike?
   - Check Git history and deployment logs
```

### Q: You're on-call and get paged at 3am. Error rate spiked. What do you do?

```
1. Acknowledge page (within SLA, e.g., 5 min)

2. Quick assessment (2 min):
   - Dashboard: Which service? How severe?
   - Recent deployments?
   - Similar past incidents?

3. Mitigate (goal: <15 min):
   - If recent deploy → ROLLBACK immediately
   - If traffic spike → Scale up
   - If dependency down → Failover / circuit breaker

4. Communicate:
   - Update incident channel
   - If customer-facing → status page

5. After mitigation:
   - Confirm error rate returned to normal
   - Note timeline for post-mortem
   - If root cause clear → fix now or create ticket
   
6. Next business day:
   - Blameless post-mortem
   - Action items to prevent recurrence
```

---

## Key Resources

- **Google SRE Book** — https://sre.google/sre-book/table-of-contents/ (free online)
- **Google SRE Workbook** — https://sre.google/workbook/table-of-contents/
- **Prometheus Documentation** — https://prometheus.io/docs/
- **Grafana Documentation** — https://grafana.com/docs/
- **OpenTelemetry** — https://opentelemetry.io/docs/
- **Awesome Prometheus Alerts** — https://samber.github.io/awesome-prometheus-alerts/
- **Brendan Gregg's Performance** — https://www.brendangregg.com
