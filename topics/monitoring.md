---
title: Monitoring
nav_order: 60
description: "Monitoring — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# Monitoring & Observability — DevOps Interview Preparation

> **Question frequency:** 🔥 very frequently asked · ⭐ important · 💡 good to know

## Table of Contents
- [Observability Fundamentals](#observability-fundamentals)
- [Prometheus](#prometheus)
- [Grafana](#grafana)
- [ELK / EFK Stack](#elk--efk-stack)
- [Loki](#loki)
- [Alerting Best Practices](#alerting-best-practices)
- [Distributed Tracing](#distributed-tracing)
- [OpenTelemetry (OTEL)](#opentelemetry-otel)
- [SLIs, SLOs, SLAs](#slis-slos-slas)
- [Continuous Profiling](#continuous-profiling)
- [eBPF-Based Observability](#ebpf-based-observability)
- [Incident Management](#incident-management)
- [Observability Cost Optimization](#observability-cost-optimization)
- [Scenario-Based Questions](#scenario-based-questions)
- [Key Resources](#key-resources)

---

## Observability Fundamentals

### 🔥 Q: Three Pillars of Observability (+ Fourth Pillar).

| Pillar | What | Tools |
|--------|------|-------|
| **Metrics** | Numeric measurements over time | Prometheus, CloudWatch, Datadog |
| **Logs** | Discrete events with context | ELK, Loki, CloudWatch Logs, Splunk |
| **Traces** | Request flow across services | Jaeger, Tempo, X-Ray, Zipkin |
| **Profiling** (4th pillar) | Continuous runtime performance data | Pyroscope, Parca, Grafana Faro, Google Cloud Profiler |

**Note:** Profiling is increasingly called the "fourth pillar" — flame graphs, CPU/memory profiles, and continuous profiling are critical for performance optimization.

### 🔥 Q: Monitoring methodologies.

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

### 🔥 Q: Prometheus architecture.

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

### 🔥 Q: Prometheus metric types.

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

### ⭐ Q: Essential PromQL queries.

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

### ⭐ Q: Prometheus in Kubernetes (ServiceMonitor).

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

### ⭐ Q: Prometheus long-term storage.

Prometheus is designed for **short-term storage** (15-30 days). For long-term:

| Solution | Type | Key Feature |
|----------|------|-------------|
| **Thanos** | Sidecar + object storage | Global query view, deduplication |
| **Cortex/Mimir** | Remote write + object storage | Multi-tenant, horizontally scalable |
| **VictoriaMetrics** | Drop-in replacement | Better compression, faster queries |

### 🔥 Q: High cardinality problem in Prometheus.

**High cardinality** = too many unique label combinations → memory explosion, query slowness.

**Bad examples:**
```promql
# user_id has millions of values
http_requests_total{user_id="u-12345"}

# request_id is unique per request
http_requests_total{request_id="req-67890"}

# timestamp as label
http_requests_total{timestamp="2025-01-15T10:30:00Z"}
```

**Solutions:**
- Drop high-cardinality labels via `metric_relabel_configs`
- Use recording rules to pre-aggregate
- Switch to exemplars (store trace IDs without indexing)
- Move user-level data to traces/logs

```yaml
# Drop high-cardinality labels at scrape time
metric_relabel_configs:
- source_labels: [user_id]
  action: labeldrop
  regex: user_id
```

### ⭐ Q: Recording rules vs alerting rules.

| Type | Purpose | When Evaluated | Output |
|------|---------|----------------|--------|
| **Recording** | Pre-compute expensive queries | Every eval interval (e.g., 1m) | New time series |
| **Alerting** | Fire alerts when condition met | Every eval interval | Alert state |

**Recording rule example:**
```yaml
groups:
- name: api_slos
  interval: 30s
  rules:
  - record: job:api_request_duration_seconds:p99
    expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
  
  - record: job:api_error_rate:5m
    expr: |
      sum(rate(http_requests_total{status=~"5.."}[5m])) /
      sum(rate(http_requests_total[5m]))
```

### ⭐ Q: Prometheus scraping vs push (Pushgateway).

| Model | When | Pros | Cons |
|-------|------|------|------|
| **Pull (scrape)** | Long-running services | Service discovery, health checks | NAT/firewall issues, batch jobs |
| **Push (Pushgateway)** | Batch jobs, lambdas | Works behind NAT | Single point of failure, stale data |

**Pushgateway use case:**
```bash
# Batch job pushes metrics before exit
cat <<EOF | curl --data-binary @- http://pushgateway:9091/metrics/job/backup_job/instance/prod
# TYPE backup_duration_seconds gauge
backup_duration_seconds 120.5
# TYPE backup_files_total counter
backup_files_total 1234
EOF
```

### 💡 Q: Prometheus federation.

**Federation** = Aggregate metrics from multiple Prometheus servers.

```
┌─── Cluster A ───┐     ┌─── Cluster B ───┐
│  Prometheus     │     │  Prometheus     │
│  (scrapes pods) │     │  (scrapes pods) │
└────────┬────────┘     └────────┬────────┘
         │                       │
         │  /federate endpoint   │
         └───────────┬───────────┘
                     ▼
          ┌─── Global Prometheus ───┐
          │  (scrapes /federate)    │
          │  (long-term storage)    │
          └─────────────────────────┘
```

**Federation config:**
```yaml
scrape_configs:
- job_name: 'federate'
  honor_labels: true
  metrics_path: '/federate'
  params:
    match[]:
    - '{__name__=~"job:.*"}'  # Only aggregate recording rules
  static_configs:
  - targets:
    - 'prometheus-a:9090'
    - 'prometheus-b:9090'
```

### ⭐ Q: Prometheus Operator vs vanilla Prometheus in Kubernetes.

| Feature | Vanilla Prometheus | Prometheus Operator |
|---------|-------------------|---------------------|
| Config | ConfigMap | ServiceMonitor, PrometheusRule CRDs |
| Service discovery | Manual scrape configs | Automatic via label selectors |
| Alerting | Alertmanager config in YAML | AlertmanagerConfig CRD |
| Updates | Restart pod | Operator reconciles automatically |
| Multi-tenancy | Manual namespacing | Built-in via CRDs |

**Prometheus Operator advantages:** GitOps-friendly, namespace isolation, auto-reload on config changes.

---

## Grafana

### ⭐ Q: Grafana dashboard best practices.

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

### 💡 Q: Grafana templating/variables.

**Variables enable dynamic dashboards** — filter by namespace, environment, service.

```
# Variable definition in dashboard JSON
{
  "templating": {
    "list": [
      {
        "name": "namespace",
        "type": "query",
        "datasource": "Prometheus",
        "query": "label_values(kube_pod_info, namespace)",
        "refresh": 1
      },
      {
        "name": "service",
        "type": "query",
        "query": "label_values(kube_service_info{namespace=\"$namespace\"}, service)"
      }
    ]
  }
}
```

**Use in queries:**
```promql
rate(http_requests_total{namespace="$namespace", service="$service"}[5m])
```

### ⭐ Q: Grafana unified alerting (new in Grafana 9+).

**Unified alerting** replaces legacy alerting + integrates with Prometheus, Loki, Tempo.

**Alert rule anatomy:**
```yaml
# Alert definition
- name: HighErrorRate
  condition: B
  data:
  - refId: A
    queryType: prometheus
    expr: sum(rate(http_requests_total{status=~"5.."}[5m]))
  - refId: B
    queryType: reduce
    reducer: last
    conditions:
    - evaluator:
        type: gt
        params: [100]
  labels:
    severity: critical
  annotations:
    summary: "Error rate > 100/s"
  for: 5m
```

**Routing via contact points:** Slack, PagerDty, email, webhooks.

### ⭐ Q: LGTM Stack (Grafana's unified observability).

**LGTM = Loki (logs) + Grafana (viz) + Tempo (traces) + Mimir (metrics)**

| Component | Replaces | Key Feature |
|-----------|----------|-------------|
| **Loki** | ELK | Label-indexed logs, low cost |
| **Grafana** | Kibana | Unified dashboards for all signals |
| **Tempo** | Jaeger | Scalable tracing, no indexing |
| **Mimir** | Thanos/Cortex | Prometheus long-term storage |

**Correlation:** Click from metric spike → traces → logs (all in Grafana).

---

## ELK / EFK Stack

### 🔥 Q: ELK Stack architecture.

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

### 🔥 Q: Structured logging best practices.

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

### ⭐ Q: Loki vs Elasticsearch for logs.

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

### ⭐ Q: Log sampling and retention strategies.

**Sampling** = Reduce log volume while preserving signal.

| Strategy | When | Trade-off |
|----------|------|-----------|
| **Head sampling** | Sample at ingestion (1% of requests) | Cheap, but miss rare errors |
| **Tail sampling** | Sample after analyzing (keep errors/slow) | Expensive, but keeps important logs |
| **Adaptive sampling** | Higher rate for errors, lower for success | Best balance |

**Retention tiers:**
```
Hot (fast query): 7 days  → SSD/Elasticsearch
Warm (slower):   30 days  → S3/cheaper storage
Cold (archive): 365 days  → Glacier (compliance)
```

**Cost optimization:**
- Sample verbose logs (DEBUG)
- Drop health-check logs
- Aggregate metrics from logs instead of storing raw logs

```yaml
# Promtail config: drop k8s health checks
- drop:
    source: path
    expression: '.*healthz.*'
```

---

## Alerting Best Practices

### 🔥 Q: How to design effective alerting.

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

### 🔥 Q: Reducing alert fatigue.

**Alert fatigue** = Too many alerts → people ignore them → miss real incidents.

**Fixes:**
1. **Alert on SLO burn rate** (not raw thresholds)
2. **Use alert grouping** (one Slack message for 10 failing pods)
3. **Silence during maintenance windows**
4. **Tune thresholds** (if alert fires weekly but no action → raise threshold)
5. **Auto-resolve** (don't page if service auto-recovers in <5 min)
6. **Runbooks** (every alert links to a runbook with mitigation steps)

**Multi-window burn-rate alerts:**
```yaml
# Fast burn (5% budget in 1 hour) → page immediately
- alert: ErrorBudgetBurnFast
  expr: |
    (
      sum(rate(http_requests_total{status=~"5.."}[1h])) /
      sum(rate(http_requests_total[1h]))
    ) > (14.4 * 0.001)  # 14.4x SLO (99.9%) in 1h
  for: 2m
  labels:
    severity: critical

# Slow burn (5% budget in 6 hours) → warn
- alert: ErrorBudgetBurnSlow
  expr: |
    (
      sum(rate(http_requests_total{status=~"5.."}[6h])) /
      sum(rate(http_requests_total[6h]))
    ) > (6 * 0.001)
  for: 15m
  labels:
    severity: warning
```

---

## Distributed Tracing

### 🔥 Q: How does distributed tracing work?

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

**Trace sampling strategies:**
| Type | When Decided | Pros | Cons |
|------|-------------|------|------|
| **Head sampling** | At trace start | Low overhead | Miss rare errors |
| **Tail sampling** | After trace completes | Keep errors/slow | High overhead |
| **Probabilistic** | Random % (e.g., 1%) | Simple | Not adaptive |

---

## OpenTelemetry (OTEL)

### 🔥 Q: What is OpenTelemetry and why is it the standard?

**OpenTelemetry (OTEL)** = Unified, vendor-neutral instrumentation standard for **metrics, logs, traces, and profiling**.

**Replaces:** Jaeger client, Zipkin client, Prometheus client libs, proprietary SDKs.

**Architecture:**
```
Application (instrumented with OTEL SDK)
    │
    ├──► Metrics  ──┐
    ├──► Logs     ──┼───► OTEL Collector ───► Backends
    ├──► Traces   ──┘         │                 │
    └──► Profiles ────────────┘                 │
                                                 ├──► Prometheus (metrics)
                                                 ├──► Loki (logs)
                                                 ├──► Tempo (traces)
                                                 └──► Pyroscope (profiles)
```

**Why OTEL?**
- **Vendor-neutral** — Switch backends without reinstrumenting
- **Single SDK** — One library for metrics/logs/traces
- **Auto-instrumentation** — Zero-code instrumentation for common frameworks
- **Industry standard** — CNCF graduated project (like Kubernetes)

### ⭐ Q: OTEL Collector pipeline.

**OTEL Collector** = Agent/gateway that receives, processes, and exports telemetry.

```
┌─── Receivers ────────────────────────────────┐
│  OTLP, Jaeger, Prometheus scrape, Zipkin    │
└──────────────────┬───────────────────────────┘
                   ▼
┌─── Processors ───────────────────────────────┐
│  batch, filter, attributes, tail_sampling   │
└──────────────────┬───────────────────────────┘
                   ▼
┌─── Exporters ────────────────────────────────┐
│  Prometheus, Loki, Tempo, Jaeger, S3, ...   │
└──────────────────────────────────────────────┘
```

**Config example:**
```yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:
  prometheus:
    config:
      scrape_configs:
      - job_name: 'otel-collector'
        static_configs:
        - targets: ['localhost:8888']

processors:
  batch:
    timeout: 10s
    send_batch_size: 1024
  attributes:
    actions:
    - key: environment
      value: production
      action: insert

exporters:
  prometheus:
    endpoint: "0.0.0.0:9090"
  otlp/tempo:
    endpoint: tempo:4317
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, attributes]
      exporters: [otlp/tempo]
    metrics:
      receivers: [otlp, prometheus]
      processors: [batch]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [loki]
```

### ⭐ Q: OTEL auto-instrumentation vs manual.

| Approach | When | Pros | Cons |
|----------|------|------|------|
| **Auto-instrumentation** | Common frameworks (Express, Django, Spring) | Zero code changes | Limited control |
| **Manual instrumentation** | Custom code, specific spans | Full control | Requires code changes |

**Auto-instrumentation example (Node.js):**
```javascript
// Just import before your app
require('@opentelemetry/auto-instrumentations-node/register');

// Your app code — no changes needed
const express = require('express');
const app = express();
app.get('/', (req, res) => res.send('Hello'));
```

**Manual span:**
```javascript
const { trace } = require('@opentelemetry/api');

async function processPayment(userId, amount) {
  const tracer = trace.getTracer('payment-service');
  const span = tracer.startSpan('process_payment');
  span.setAttribute('user_id', userId);
  span.setAttribute('amount', amount);
  
  try {
    await chargeCard(userId, amount);
    span.setStatus({ code: 0 }); // OK
  } catch (err) {
    span.recordException(err);
    span.setStatus({ code: 2, message: err.message }); // ERROR
    throw err;
  } finally {
    span.end();
  }
}
```

### ⭐ Q: Context propagation in distributed tracing.

**Context propagation** = Passing trace ID + span ID across service boundaries (HTTP headers, message queues).

**W3C Trace Context standard:**
```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
             │  │                                │                │
             │  trace-id (128-bit)               span-id (64-bit) flags
             version
```

**HTTP example:**
```bash
# Service A → Service B
curl -H "traceparent: 00-abc123-def456-01" http://service-b/api
```

**OTEL SDKs automatically propagate context.** For manual propagation:
```python
from opentelemetry import trace
from opentelemetry.propagate import inject

headers = {}
inject(headers)  # Injects traceparent into headers dict
requests.get('http://downstream', headers=headers)
```

---

## SLIs, SLOs, SLAs

### 🔥 Q: Define SLI, SLO, SLA with examples.

| Term | Definition | Example |
|------|-----------|---------|
| **SLI** (Indicator) | Metric that measures service quality | 99.2% of requests < 200ms |
| **SLO** (Objective) | Target value for an SLI | 99.9% of requests should be < 200ms |
| **SLA** (Agreement) | Contract with consequences | 99.95% uptime or customer gets credits |

**SLI → SLO → SLA:** Measure → Target → Contract

### 🔥 Q: How to calculate error budgets.

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

### ⭐ Q: Common SLIs to track.

| Service Type | SLI | Measurement |
|-------------|-----|-------------|
| **API** | Availability | % of successful requests (non-5xx) |
| **API** | Latency | % of requests under threshold (p99 < 500ms) |
| **Pipeline** | Freshness | % of data processed within time window |
| **Storage** | Durability | % of data retained without loss |
| **Batch** | Throughput | % of jobs completing within deadline |

### ⭐ Q: Multi-window, multi-burn-rate alerting.

**Problem:** Simple threshold alerts (error rate > 5%) are noisy or slow.

**Solution:** Alert on **SLO burn rate** with multiple time windows.

**Example:** 99.9% availability SLO (0.1% error budget per month).

| Burn Rate | Window | Error Budget Consumed | Alert | Action |
|-----------|--------|----------------------|-------|--------|
| 14.4x | 1 hour | 2% (fast burn) | Critical | Page immediately |
| 6x | 6 hours | 2.5% (moderate) | Warning | Investigate in hours |
| 3x | 24 hours | 5% (slow) | Ticket | Investigate next day |

**PromQL for multi-burn:**
```promql
# Fast burn: 14.4x over 1h, 2% budget consumed
(
  sum(rate(http_requests_total{status=~"5.."}[1h])) /
  sum(rate(http_requests_total[1h]))
) > (14.4 * (1 - 0.999))

# Slow burn: 3x over 24h, 5% budget consumed
(
  sum(rate(http_requests_total{status=~"5.."}[24h])) /
  sum(rate(http_requests_total[24h]))
) > (3 * (1 - 0.999))
```

**Why this works:** Fast burns trigger quickly, slow burns avoid noise.

---

## Continuous Profiling

### 💡 Q: What is continuous profiling and why is it important?

**Continuous profiling** = Always-on collection of performance profiles (CPU, memory, I/O) from production.

**Fourth pillar of observability** — complements metrics, logs, traces.

| Signal | Question Answered |
|--------|------------------|
| **Metrics** | What is slow? (p99 latency is high) |
| **Traces** | Where is the slowdown? (database query) |
| **Profiling** | Why is it slow? (inefficient regex in loop) |

**Tools:**
| Tool | Backend | Features |
|------|---------|----------|
| **Pyroscope** | Open-source | Multi-language, flame graphs |
| **Parca** | Open-source (CNCF) | eBPF-based, low overhead |
| **Grafana Faro** | Part of LGTM | Frontend profiling (JavaScript) |
| **Google Cloud Profiler** | Managed | Zero-config for GCP |
| **Datadog Continuous Profiler** | Commercial | Integrated with APM |

### 💡 Q: How to read a flame graph.

**Flame graph** = Visualization of stack traces.

```
┌────────────────────────────────────────────┐ ← Top of stack (leaf function)
│          doExpensiveRegex() 30%            │   (widest = most CPU time)
├────────────────────────────────────────────┤
│          processRequest() 50%              │
├────────────────────────────────────────────┤
│          handleHTTP() 80%                  │
├────────────────────────────────────────────┤
│          main() 100%                       │ ← Bottom (entry point)
└────────────────────────────────────────────┘
```

**How to read:**
- **Width** = % of CPU time
- **Height** = Stack depth (not time)
- **Color** = Usually random (or by library)

**Hot path:** Follow the widest flame from bottom to top.

### 💡 Q: Profiling use cases.

| Use Case | Tool | Example |
|----------|------|---------|
| **CPU hotspots** | pprof, Pyroscope | Which function uses most CPU? |
| **Memory leaks** | Heap profiler | What is allocating memory? |
| **Goroutine leaks** (Go) | Goroutine profiler | Which goroutines never exit? |
| **Mutex contention** | Mutex profiler | Which locks cause blocking? |
| **I/O blocking** | I/O profiler | Which syscalls are slow? |

---

## eBPF-Based Observability

### 💡 Q: What is eBPF and why is it a game-changer for observability?

**eBPF (extended Berkeley Packet Filter)** = Run sandboxed programs in the Linux kernel without kernel modules.

**Observability use cases:**
- **Zero-instrumentation tracing** — Trace any function in the kernel or userspace
- **Network monitoring** — Packet filtering, latency, connection tracking
- **Security** — Syscall auditing, runtime security
- **Performance** — CPU profiling without recompiling apps

**Advantages over traditional instrumentation:**
| Feature | Traditional (SDK/agent) | eBPF |
|---------|------------------------|------|
| Code changes | Required | Zero |
| Language support | Per-language SDK | Any language |
| Overhead | High (5-10%) | Low (<1%) |
| Kernel visibility | No | Yes |

### 💡 Q: eBPF observability tools.

| Tool | Use Case | Backed By |
|------|----------|-----------|
| **Pixie** | Auto-instrumentation, no code changes | CNCF (New Relic) |
| **Cilium** | Network observability, service mesh | CNCF |
| **Parca** | Continuous profiling | CNCF (Polar Signals) |
| **Falco** | Runtime security, anomaly detection | CNCF |
| **bpftrace** | Ad-hoc kernel tracing | Brendan Gregg |
| **Datadog eBPF** | APM without agents | Datadog |

**Example: Pixie auto-traces HTTP, gRPC, MySQL, Postgres, Kafka** — no SDK.

### 💡 Q: Example eBPF use case — trace slow syscalls.

**Problem:** App is slow, but profiling shows no hot functions.

**Solution:** Trace syscalls with eBPF.

```bash
# bpftrace: trace all syscalls taking >10ms
bpftrace -e 'tracepoint:syscalls:sys_exit_* /args->ret > 10000000/ { @[probe] = count(); }'

# Output:
@[tracepoint:syscalls:sys_exit_read]: 1234
@[tracepoint:syscalls:sys_exit_write]: 567
```

**Benefit:** Found that `read()` syscalls are slow → disk I/O is the bottleneck.

---

## Incident Management

### 🔥 Q: Incident response process.

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

### ⭐ Q: What are DORA metrics?

**DORA metrics** = Four key DevOps performance indicators (from Google's DevOps Research & Assessment).

| Metric | Elite | High | Medium | Low |
|--------|-------|------|--------|-----|
| **Deployment Frequency** | On-demand (multiple/day) | Weekly-Monthly | Monthly-Biannual | <1/6months |
| **Lead Time for Changes** | <1 hour | 1 day-1 week | 1-6 months | >6 months |
| **Mean Time to Recovery** | <1 hour | <1 day | 1 day-1 week | >6 months |
| **Change Failure Rate** | 0-15% | 16-30% | 16-30% | >30% |

**How to measure in Prometheus:**
```promql
# Deployment frequency (deploys per day)
sum(increase(deployment_total[1d]))

# Lead time (median time from commit to deploy)
histogram_quantile(0.5, rate(lead_time_seconds_bucket[7d]))

# MTTR (median incident duration)
histogram_quantile(0.5, rate(incident_duration_seconds_bucket[30d]))

# Change failure rate
sum(increase(deployment_total{status="failed"}[30d])) /
sum(increase(deployment_total[30d]))
```

### ⭐ Q: On-call best practices.

| Practice | Why |
|----------|-----|
| **Primary + secondary on-call** | Backup if primary unresponsive |
| **Escalation ladder** | Eng → Senior → Manager → VP |
| **Rotation length** | 1 week (not too short, not burnout) |
| **Follow-the-sun** | Rotate across time zones |
| **Post-incident review** | Blameless, focus on systems |
| **On-call compensation** | Pay or time off for after-hours pages |
| **Alert SLA** | Acknowledge in 5 min, mitigate in 15 min |
| **Runbook for every alert** | No alert without runbook |

---

## Observability Cost Optimization

### ⭐ Q: Observability bill is $50k/month. How to cut costs?

**High-impact levers:**

1. **Metric cardinality**
   - Audit high-cardinality labels (user_id, request_id)
   - Drop via `metric_relabel_configs`
   - Use recording rules to pre-aggregate

2. **Log sampling**
   - Sample DEBUG logs (10%)
   - Drop health-check logs
   - Use tail sampling (keep errors, sample success)

3. **Trace sampling**
   - Head sampling: 1% of traces
   - Tail sampling: keep slow + error traces

4. **Retention**
   - Hot: 7 days (fast query)
   - Warm: 30 days (slower)
   - Cold: 365 days (archive)

5. **Aggregate from logs**
   - Instead of storing all logs, derive metrics
   - Example: Count 5xx errors → Prometheus counter (not log search)

6. **Switch backends**
   - Loki vs Elasticsearch (10x cheaper)
   - VictoriaMetrics vs Thanos (better compression)

**Cost breakdown example:**
```
Metrics: $20k (Prometheus + Thanos)
Logs:    $25k (Elasticsearch)
Traces:  $5k  (Jaeger)
```

**After optimization:**
```
Metrics: $15k (drop high-cardinality labels)
Logs:    $8k  (switch to Loki, sample DEBUG)
Traces:  $2k  (head sampling 1%)
Total savings: $25k/month (50% reduction)
```

### 💡 Q: How to estimate observability data volume?

**Metrics:**
```
Metric cardinality = # unique time series
Volume = cardinality × sample rate × 12 bytes × 30 days

Example:
1M time series × 1 sample/15s × 12 bytes × 2.6M seconds/month
= 1M × 4/min × 12 × 2.6M / 60 = ~2 TB/month
```

**Logs:**
```
Volume = requests/sec × avg log size × 86400 sec/day × 30 days

Example:
10k rps × 500 bytes × 86400 × 30 = ~13 TB/month
```

**Traces:**
```
Volume = requests/sec × sample rate × avg trace size × 86400 × 30

Example (1% sampling):
10k rps × 0.01 × 10 KB × 86400 × 30 = ~2.5 TB/month
```

---

## Scenario-Based Questions

### 🔥 Q: Application is slow but infrastructure looks fine. How to investigate?

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

### 🔥 Q: You're on-call and get paged at 3am. Error rate spiked. What do you do?

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

### 🔥 Q: Design a monitoring stack for a microservices platform.

**Requirements:** 50 services, 10k RPS, Kubernetes, multi-region.

**Stack:**
```
┌─── Data Sources ──────────────────────────────┐
│  50 microservices (OTEL instrumented)         │
│  Kubernetes metrics (cAdvisor, kube-state)   │
│  Infrastructure (node-exporter, blackbox)     │
└──────────────────┬────────────────────────────┘
                   ▼
┌─── Collection Layer ──────────────────────────┐
│  OTEL Collectors (DaemonSet per node)         │
│  - Receive OTLP from apps                     │
│  - Scrape Prometheus metrics                  │
│  - Process, batch, export                     │
└──────────────────┬────────────────────────────┘
                   ▼
┌─── Storage Layer ─────────────────────────────┐
│  Metrics:  Mimir (long-term Prometheus)       │
│  Logs:     Loki (label-indexed logs)          │
│  Traces:   Tempo (no indexing, S3 backend)    │
│  Profiles: Pyroscope (continuous profiling)   │
└──────────────────┬────────────────────────────┘
                   ▼
┌─── Query/Viz Layer ───────────────────────────┐
│  Grafana (unified dashboards for all signals) │
│  - Service dashboards (RED metrics)           │
│  - Infrastructure dashboards (USE)            │
│  - Correlation: metrics → traces → logs       │
└──────────────────┬────────────────────────────┘
                   ▼
┌─── Alerting Layer ────────────────────────────┐
│  Grafana Unified Alerting                     │
│  - SLO burn-rate alerts                       │
│  - Route to PagerDuty/Slack/Email             │
│  - Runbooks linked in annotations             │
└───────────────────────────────────────────────┘
```

**Key decisions:**
- **OTEL Collector** (not Prometheus) → vendor-neutral, multi-signal
- **Mimir** (not Thanos) → better multi-tenancy, simpler ops
- **Tempo** (not Jaeger) → no indexing = lower cost
- **Loki** (not ELK) → 10x cheaper, native Grafana integration
- **Pyroscope** → continuous profiling (4th pillar)

### ⭐ Q: High cardinality is blowing up Prometheus. Fix it.

**Symptoms:**
- Prometheus OOM crashes
- Query timeouts
- Disk full (WAL backlog)

**Diagnosis:**
```bash
# Check cardinality per metric
curl http://prometheus:9090/api/v1/label/__name__/values | jq -r '.data[]' | \
  while read metric; do
    echo -n "$metric: "
    curl -s "http://prometheus:9090/api/v1/query?query=count($metric)" | jq '.data.result[0].value[1]'
  done | sort -t: -k2 -nr | head -20
```

**Culprits found:**
```
http_requests_total{user_id="...", request_id="..."}: 10M time series
```

**Fixes:**

1. **Drop high-cardinality labels:**
```yaml
# prometheus.yml
metric_relabel_configs:
- source_labels: [__name__]
  regex: 'http_requests_total'
  action: keep
- source_labels: [user_id, request_id]
  action: labeldrop
```

2. **Use recording rules to pre-aggregate:**
```yaml
groups:
- name: cardinality_fix
  rules:
  - record: job:http_requests_total:rate5m
    expr: sum(rate(http_requests_total[5m])) by (job, method, status)
```

3. **Use exemplars for trace IDs (not labels):**
```go
// Don't do this:
counter.Add(ctx, 1, attribute.String("trace_id", traceID))

// Do this (exemplar):
counter.Add(ctx, 1)  // trace_id attached as exemplar, not label
```

4. **Horizontal scaling:**
   - Shard Prometheus by namespace
   - Federate to global Prometheus
   - Use Thanos/Mimir for long-term storage

### 🔥 Q: Define SLOs for a payment processing service.

**Service:** API for payment processing (credit card charges).

**SLIs:**
| SLI | Target (SLO) | Why |
|-----|--------------|-----|
| **Availability** | 99.95% (non-5xx) | Downtime = lost revenue |
| **Latency** | 95% of requests < 500ms | User experience |
| **Latency (p99)** | 99% of requests < 2s | Catch tail latency |
| **Correctness** | 99.999% (no double-charge) | Financial accuracy |

**Measurement:**
```promql
# Availability SLI
sum(rate(http_requests_total{status!~"5.."}[30d])) /
sum(rate(http_requests_total[30d]))

# Latency SLI (% of requests < 500ms)
sum(rate(http_request_duration_seconds_bucket{le="0.5"}[30d])) /
sum(rate(http_request_duration_seconds_count[30d]))
```

**Error budget:**
```
99.95% availability = 0.05% error budget per month
30 days × 24 hours × 60 min = 43,200 minutes
43,200 × 0.0005 = 21.6 minutes of downtime allowed
```

**Alert on burn rate:**
```yaml
# Fast burn: 2% budget in 1 hour
- alert: PaymentSLOBurnFast
  expr: |
    (1 - (
      sum(rate(http_requests_total{job="payment",status!~"5.."}[1h])) /
      sum(rate(http_requests_total{job="payment"}[1h]))
    )) > (14.4 * 0.0005)
  for: 2m
  labels:
    severity: critical
```

### 🔥 Q: Debug a latency spike using distributed tracing.

**Scenario:** p99 latency jumped from 200ms → 2s at 10:00 AM.

**Steps:**

1. **Grafana: identify time window**
   - Dashboard shows latency spike at 10:00-10:15 AM

2. **Tempo: find slow traces in that window**
   ```
   Query: {service.name="api-gateway"} && duration > 1s
   Time range: 10:00-10:15 AM
   ```

3. **Analyze trace waterfall:**
   ```
   api-gateway (2000ms total)
     ├─ auth-service (50ms)
     ├─ user-service (100ms)
     └─ payment-service (1800ms) ← Bottleneck!
         ├─ postgres query (1750ms) ← Root cause
         └─ cache lookup (50ms)
   ```

4. **Root cause:** Postgres query slow.

5. **Correlate with logs:**
   - Click "Logs for this span" in Tempo
   - Loki shows: `"query": "SELECT * FROM transactions WHERE user_id = ..."`
   - No index on `user_id`!

6. **Fix:**
   - Add index: `CREATE INDEX idx_user_id ON transactions(user_id);`

7. **Verify:**
   - Deploy index migration
   - Check Tempo: new traces show payment-service @ 50ms
   - Grafana: p99 latency back to 200ms

### 🔥 Q: Reduce alert fatigue — you're getting 100 alerts/day.

**Current state:** 100 alerts/day, team ignores most, real incidents missed.

**Audit:**
```bash
# Count alerts by severity
curl http://alertmanager:9093/api/v2/alerts | \
  jq -r '.[] | .labels.severity' | sort | uniq -c

# Result:
85 warning
10 critical
5 info
```

**Findings:**
- 85% are warnings (not actionable)
- Many alerts fire but auto-resolve in 2 minutes
- No runbooks linked

**Action plan:**

1. **Convert warnings to tickets (not pages):**
   ```yaml
   routes:
   - match:
       severity: warning
     receiver: jira-webhook  # Not PagerDuty
   ```

2. **Increase thresholds for noisy alerts:**
   ```yaml
   # Old (too sensitive)
   - alert: HighCPU
     expr: node_cpu_usage > 70%
   
   # New (actionable)
   - alert: HighCPU
     expr: node_cpu_usage > 85%
     for: 15m  # Sustained, not transient
   ```

3. **Add runbooks to every alert:**
   ```yaml
   annotations:
     runbook: "https://wiki.example.com/runbooks/high-cpu"
   ```

4. **Auto-resolve if self-healed:**
   ```yaml
   # Don't page if error rate drops in 5 min
   - alert: HighErrorRate
     expr: error_rate > 0.05
     for: 5m  # Only page if sustained
   ```

5. **Alert on SLO burn rate (not raw thresholds):**
   ```yaml
   # Instead of "error rate > 5%", use burn rate
   - alert: SLOBurnFast
     expr: (error_rate / slo_target) > 14.4
   ```

**Result after 2 weeks:**
- 100 → 5 alerts/day
- All critical alerts have runbooks
- Team trust restored

### ⭐ Q: Observability for serverless (AWS Lambda).

**Challenges:**
- Short-lived containers (no persistent agents)
- Cold start overhead
- Limited instrumentation options

**Stack:**

| Signal | Solution |
|--------|----------|
| **Metrics** | CloudWatch (built-in) + custom metrics via OTEL |
| **Logs** | CloudWatch Logs → Loki (via Firehose) |
| **Traces** | X-Ray or OTEL → Tempo |
| **Profiling** | CloudWatch Lambda Insights |

**OTEL Lambda Layer:**
```yaml
# serverless.yml
functions:
  myFunction:
    handler: index.handler
    layers:
      - arn:aws:lambda:us-east-1:901920570463:layer:aws-otel-nodejs-amd64-ver-1-18-1:1
    environment:
      AWS_LAMBDA_EXEC_WRAPPER: /opt/otel-handler
      OTEL_EXPORTER_OTLP_ENDPOINT: https://otel-collector:4318
```

**Key metrics:**
```promql
# Lambda cold start rate
sum(rate(aws_lambda_invocations{cold_start="true"}[5m])) /
sum(rate(aws_lambda_invocations[5m]))

# Lambda error rate
sum(rate(aws_lambda_errors[5m])) /
sum(rate(aws_lambda_invocations[5m]))

# Lambda duration p99
histogram_quantile(0.99, rate(aws_lambda_duration_seconds_bucket[5m]))
```

---

## Key Resources

- **Google SRE Book** — https://sre.google/sre-book/table-of-contents/ (free online)
- **Google SRE Workbook** — https://sre.google/workbook/table-of-contents/
- **Prometheus Documentation** — https://prometheus.io/docs/
- **PromQL Cheat Sheet** — https://promlabs.com/promql-cheat-sheet/
- **Grafana Documentation** — https://grafana.com/docs/
- **OpenTelemetry Documentation** — https://opentelemetry.io/docs/
- **OTEL Collector Examples** — https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/examples
- **Awesome Prometheus Alerts** — https://samber.github.io/awesome-prometheus-alerts/
- **Brendan Gregg's Performance** — https://www.brendangregg.com
- **eBPF for Observability** — https://ebpf.io/
- **SLO Workshop (Google)** — https://sre.google/workbook/implementing-slos/
- **Multi-Window Multi-Burn-Rate Alerts** — https://sre.google/workbook/alerting-on-slos/
- **Continuous Profiling Guide** — https://grafana.com/docs/pyroscope/
- **LGTM Stack Intro** — https://grafana.com/blog/2023/04/12/get-started-with-the-lgtm-stack/
