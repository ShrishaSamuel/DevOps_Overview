# Prometheus — Zero to Hero

> A complete, beginner-to-advanced guide to Prometheus for metrics collection, monitoring, and alerting.

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

### 1.1 What is Prometheus?

**Prometheus** is an open-source **systems monitoring and alerting toolkit** originally built at SoundCloud in 2012 and now a graduated **Cloud Native Computing Foundation (CNCF)** project (the second after Kubernetes). It collects **time-series metrics** from configured targets at given intervals, evaluates rule expressions, displays results, and can trigger alerts when conditions are met.

Prometheus has become the de-facto standard for monitoring cloud-native and Kubernetes environments.

### 1.2 What is a time series?

A **time series** is a stream of timestamped values belonging to the same metric and the same set of labels. Prometheus stores all data as time series:

```
metric_name{label1="value1", label2="value2"}  →  (timestamp, value), (timestamp, value), ...

http_requests_total{method="GET", status="200"}  1687000000  1027
http_requests_total{method="GET", status="200"}  1687000015  1041
```

Each unique combination of metric name + labels is a distinct time series.

### 1.3 Why Prometheus?

| Feature | Explanation |
|---------|-------------|
| **Pull-based model** | Prometheus scrapes targets over HTTP (vs. push) — simpler discovery, health detection |
| **Multi-dimensional data** | Metrics identified by name + key/value labels enable powerful querying |
| **PromQL** | A flexible query language for slicing, aggregating, and alerting on metrics |
| **No external dependencies** | Single binary, local storage; reliable even when other systems fail |
| **Service discovery** | Auto-discovers targets (Kubernetes, Consul, EC2, etc.) |
| **Ecosystem** | Hundreds of exporters + Alertmanager + Grafana integration |

### 1.4 Push vs. Pull

```
   PULL (Prometheus)                       PUSH (e.g., StatsD)
   Prometheus ──scrape──► /metrics         App ──push──► Collector
   (server initiates)                      (client initiates)
```

- **Pull:** Prometheus periodically requests metrics from each target's `/metrics` HTTP endpoint. Benefits: centralized control over scrape frequency, easy "is the target up?" detection, no client-side network config.
- For short-lived/batch jobs that can't be scraped, Prometheus offers the **Pushgateway** (jobs push metrics there; Prometheus scrapes the gateway).

### 1.5 Architecture

```
                          +-------------------------+
   Service Discovery ───► |    Prometheus Server    |
   (K8s, Consul, files)   |  ┌───────────────────┐  |
                          |  │ Retrieval (scrape)│  |◄── scrape /metrics
   Targets / Exporters ──►|  ├───────────────────┤  |    from targets
   (node_exporter, apps)  |  │ TSDB (local store)│  |
                          |  ├───────────────────┤  |
                          |  │ Rules engine      │  |
                          |  │ HTTP / PromQL API │  |
                          |  └─────────┬─────────┘  |
                          +────────────┼────────────+
                                       │ alerts            │ queries
                                       ▼                   ▼
                              +----------------+   +----------------+
                              |  Alertmanager  |   |    Grafana     |
                              | dedupe/route/  |   |  dashboards    |
                              | silence → email|   +----------------+
                              | Slack/PagerDuty|
                              +----------------+
```

Components:
- **Prometheus Server** — scrapes and stores time series, runs rules, answers queries.
- **TSDB** — the local time-series database (on-disk).
- **Exporters** — expose metrics from systems that don't natively speak Prometheus (e.g., `node_exporter` for host metrics).
- **Pushgateway** — accepts pushed metrics from short-lived jobs.
- **Alertmanager** — handles alerts: deduplication, grouping, routing, silencing, and notification.
- **Client libraries** — instrument your own apps to expose `/metrics`.

### 1.6 When to use Prometheus

- Monitoring dynamic, containerized, and Kubernetes environments.
- Metrics-based alerting on infrastructure and application health.
- Time-series telemetry with rich dimensional querying.

When **not** ideal: long-term durable storage at scale (use remote storage like Thanos/Cortex/Mimir/VictoriaMetrics), event logging (use ELK/Loki), or 100% accurate billing data (Prometheus is sampled, not event-exact).

---

## 2. Installation & Setup

### 2.1 Download and run the binary

```bash
PROM_VERSION="2.53.0"
wget https://github.com/prometheus/prometheus/releases/download/v${PROM_VERSION}/prometheus-${PROM_VERSION}.linux-amd64.tar.gz
tar xvf prometheus-${PROM_VERSION}.linux-amd64.tar.gz
cd prometheus-${PROM_VERSION}.linux-amd64
./prometheus --config.file=prometheus.yml
```

Open the UI at `http://localhost:9090`.

### 2.2 Run with Docker

```bash
docker run -d --name prometheus -p 9090:9090 \
  -v $(pwd)/prometheus.yml:/etc/prometheus/prometheus.yml \
  prom/prometheus
```

### 2.3 Minimal configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s          # how often to scrape targets
  evaluation_interval: 15s      # how often to evaluate rules

scrape_configs:
  - job_name: 'prometheus'      # Prometheus monitors itself
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
```

### 2.4 Install node_exporter (host metrics)

```bash
NODE_VERSION="1.8.1"
wget https://github.com/prometheus/node_exporter/releases/download/v${NODE_VERSION}/node_exporter-${NODE_VERSION}.linux-amd64.tar.gz
tar xvf node_exporter-${NODE_VERSION}.linux-amd64.tar.gz
./node_exporter-${NODE_VERSION}.linux-amd64/node_exporter
# Metrics now at http://localhost:9100/metrics
```

### 2.5 Verify

```bash
curl -s http://localhost:9090/-/healthy        # Prometheus health
curl -s http://localhost:9100/metrics | head   # node_exporter metrics
```

In the UI: **Status → Targets** should show both jobs as `UP`.

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 Metric types

| Type | Description | Example |
|------|-------------|---------|
| **Counter** | Monotonically increasing value (resets to 0 on restart) | `http_requests_total` |
| **Gauge** | Value that can go up or down | `memory_usage_bytes`, `temperature` |
| **Histogram** | Samples observations into configurable buckets + sum + count | `request_duration_seconds` |
| **Summary** | Like histogram but calculates quantiles client-side | `rpc_duration_seconds` |

- Use **Counter** for things that only increase (requests, errors).
- Use **Gauge** for values that fluctuate (CPU, memory, queue length).
- Use **Histogram** when you need to aggregate percentiles across instances (preferred for latency).

#### 3.1.2 Labels

Labels add dimensions to metrics, enabling filtering and aggregation:

```
http_requests_total{method="POST", handler="/api", status="500"}
```

> Each unique label-value combination creates a new time series. **High-cardinality labels** (user IDs, emails, timestamps) explode the number of series and can crash Prometheus — avoid them.

#### 3.1.3 The /metrics endpoint

Prometheus scrapes a plain-text exposition format:

```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="get",status="200"} 1027
http_requests_total{method="post",status="200"} 87
# HELP process_resident_memory_bytes Resident memory.
# TYPE process_resident_memory_bytes gauge
process_resident_memory_bytes 5.2428e+07
```

#### 3.1.4 Targets and jobs

- **Target:** an endpoint Prometheus scrapes (host:port + path).
- **Job:** a collection of targets with the same purpose (e.g., all `node` exporters).
- Prometheus auto-adds labels: `job`, `instance`.

### 3.2 INTERMEDIATE

#### 3.2.1 PromQL basics

**PromQL** (Prometheus Query Language) is how you retrieve and compute over time series.

```promql
# Instant vector: current value of every matching series
node_cpu_seconds_total

# Filter by label
node_cpu_seconds_total{mode="idle"}

# Regex match
http_requests_total{status=~"5.."}        # 5xx errors
http_requests_total{status!~"2.."}        # non-2xx

# Range vector: values over a time window
node_cpu_seconds_total[5m]
```

#### 3.2.2 Rate and counters

Counters always increase, so you query their **rate** (per-second increase):

```promql
# Per-second request rate over the last 5 minutes
rate(http_requests_total[5m])

# irate uses the last two points (more responsive, spikier)
irate(http_requests_total[5m])

# increase = total increase over the window
increase(http_requests_total[1h])
```

> Always use `rate()`/`increase()` on counters, never raw values. Use a window of at least 4× the scrape interval.

#### 3.2.3 Aggregation operators

```promql
# Sum request rate across all instances
sum(rate(http_requests_total[5m]))

# Group by a label
sum by (status) (rate(http_requests_total[5m]))

# Other aggregations: avg, min, max, count, stddev, topk, bottomk
topk(3, sum by (instance) (rate(http_requests_total[5m])))
avg by (instance) (node_load1)
count(up == 1)                            # how many targets are up
```

#### 3.2.4 Useful computed queries

```promql
# CPU usage percentage per instance
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Memory used percentage
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# Disk space used percentage
100 * (1 - node_filesystem_avail_bytes / node_filesystem_size_bytes)

# Error ratio
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m]))
```

#### 3.2.5 Histograms and quantiles

```promql
# 95th percentile request latency from a histogram
histogram_quantile(0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# Average latency
rate(http_request_duration_seconds_sum[5m])
  / rate(http_request_duration_seconds_count[5m])
```

#### 3.2.6 Recording rules

Precompute expensive/frequent queries and store them as new series.

```yaml
# rules.yml
groups:
  - name: cpu_rules
    interval: 30s
    rules:
      - record: instance:node_cpu_utilization:rate5m
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

Reference the recorded series directly in dashboards/alerts.

#### 3.2.7 Service discovery

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

Supported SD: Kubernetes, Consul, EC2/Azure/GCE, DNS, file-based, and more — targets update automatically as infrastructure changes.

### 3.3 ADVANCED

#### 3.3.1 Alerting rules

```yaml
# alert.rules.yml
groups:
  - name: instance_alerts
    rules:
      - alert: InstanceDown
        expr: up == 0
        for: 5m                       # must be true for 5m before firing
        labels:
          severity: critical
        annotations:
          summary: "Instance {{ $labels.instance }} down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for >5m."

      - alert: HighCPU
        expr: instance:node_cpu_utilization:rate5m > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.instance }}"
```

#### 3.3.2 Alertmanager

Alertmanager receives alerts and handles routing/grouping/silencing/notifications.

```yaml
# alertmanager.yml
route:
  receiver: 'team-default'
  group_by: ['alertname', 'cluster']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - match: { severity: critical }
      receiver: 'pagerduty'

receivers:
  - name: 'team-default'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/XXX'
        channel: '#alerts'
  - name: 'pagerduty'
    pagerduty_configs:
      - routing_key: '<key>'

inhibit_rules:
  - source_match: { severity: critical }
    target_match: { severity: warning }
    equal: ['alertname', 'instance']
```

Wire it to Prometheus:

```yaml
# prometheus.yml
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['localhost:9093']
rule_files:
  - 'alert.rules.yml'
  - 'rules.yml'
```

Key concepts: **grouping** (batch related alerts), **inhibition** (suppress lower-severity when a higher one fires), **silencing** (mute during maintenance).

#### 3.3.3 Exporters and instrumentation

Common exporters:
- `node_exporter` — Linux host metrics.
- `cAdvisor` — container metrics.
- `blackbox_exporter` — probe endpoints (HTTP/TCP/ICMP/DNS).
- `mysqld_exporter`, `postgres_exporter`, `redis_exporter` — databases.
- `kube-state-metrics` — Kubernetes object state.

Instrument an app (Python example):

```python
from prometheus_client import Counter, Histogram, start_http_server
import time, random

REQUESTS = Counter('app_requests_total', 'Total requests', ['endpoint'])
LATENCY  = Histogram('app_request_seconds', 'Request latency', ['endpoint'])

def handle(endpoint):
    REQUESTS.labels(endpoint).inc()
    with LATENCY.labels(endpoint).time():
        time.sleep(random.random())

if __name__ == '__main__':
    start_http_server(8000)        # exposes /metrics on :8000
    while True:
        handle('/api')
```

#### 3.3.4 Storage, retention, and scaling

```bash
./prometheus \
  --storage.tsdb.path=/data \
  --storage.tsdb.retention.time=30d \
  --storage.tsdb.retention.size=50GB
```

- Local TSDB is great for single-node but not long-term/HA.
- **Remote write** to **Thanos / Cortex / Mimir / VictoriaMetrics** for long-term, global view, and HA.
- **Federation** lets a higher-level Prometheus scrape aggregated metrics from others.

```yaml
remote_write:
  - url: "http://mimir:9009/api/v1/push"
```

#### 3.3.5 High availability

Run two identical Prometheus servers scraping the same targets; deduplicate at query time (Thanos Querier) or via Alertmanager clustering for alerts.

#### 3.3.6 The four golden signals (SRE)

Monitor: **Latency**, **Traffic**, **Errors**, **Saturation**. Build your dashboards/alerts around these for every service. Also consider **USE** (Utilization, Saturation, Errors) for resources and **RED** (Rate, Errors, Duration) for request-driven services.

---

## 4. Hands-on Tasks

### Task 1: Run Prometheus and explore

```bash
docker run -d --name prometheus -p 9090:9090 prom/prometheus
```

Open `http://localhost:9090` → **Status → Targets**.

**Expected:** The `prometheus` job is `UP`.

### Task 2: Scrape node_exporter

Add the `node` job (section 2.3), start `node_exporter`, reload Prometheus.

```bash
curl -X POST http://localhost:9090/-/reload    # if --web.enable-lifecycle
```

**Expected:** `node` target shows `UP`; `node_*` metrics queryable.

### Task 3: Run basic PromQL queries

In the **Graph** tab:

```promql
up
rate(prometheus_http_requests_total[5m])
node_memory_MemAvailable_bytes
```

**Expected:** Tables/graphs render for each.

### Task 4: Compute CPU usage

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

**Expected:** A percentage per instance.

### Task 5: Add a recording rule

Create `rules.yml` (section 3.2.6), reference it in `prometheus.yml`, reload, then query `instance:node_cpu_utilization:rate5m`.

**Expected:** The precomputed series appears.

### Task 6: Define an alert rule

Add the `InstanceDown` alert (section 3.3.1), reload, then stop `node_exporter`.

**Expected:** In **Alerts**, the rule moves Pending → Firing after 5m.

### Task 7: Wire up Alertmanager

```bash
docker run -d --name alertmanager -p 9093:9093 \
  -v $(pwd)/alertmanager.yml:/etc/alertmanager/alertmanager.yml prom/alertmanager
```

Configure Prometheus `alerting:` block and reload.

**Expected:** Firing alerts appear in the Alertmanager UI (`:9093`).

### Task 8: Instrument an app

Run the Python app from section 3.3.3, add a scrape job for `localhost:8000`.

**Expected:** `app_requests_total` and latency histogram appear.

### Task 9: Quantile latency

```promql
histogram_quantile(0.95, sum by (le) (rate(app_request_seconds_bucket[5m])))
```

**Expected:** The p95 latency value.

### Task 10: Probe an endpoint with blackbox_exporter

Run `blackbox_exporter`, add a `blackbox` job probing `https://example.com`, then:

```promql
probe_success
probe_duration_seconds
```

**Expected:** `probe_success` is 1 for reachable endpoints.

---

## 5. Projects

### Project 1 (Beginner): Host Monitoring Stack

**Goal:** Monitor a Linux server's CPU, memory, disk, and network.

Steps:
1. Run Prometheus + node_exporter (Docker Compose).
2. Configure scraping and retention.
3. Write PromQL for CPU%, memory%, disk%, and load.
4. Add recording rules for the key metrics.
5. Add an alert for low disk space and high CPU.

**Skills:** scraping, PromQL, node_exporter, recording rules, alerts.

### Project 2 (Intermediate): Application + Infrastructure Observability

**Goal:** Full-stack monitoring of an app and its dependencies.

Steps:
1. Instrument an app with a client library (counters, histograms).
2. Add exporters for the database and container metrics (cAdvisor).
3. Implement the **RED** metrics (Rate, Errors, Duration) per endpoint.
4. Build alerting rules for error ratio and p95 latency SLOs.
5. Route alerts via Alertmanager to Slack with severity-based routing.
6. Add blackbox probes for external uptime.

**Skills:** instrumentation, exporters, SLO alerting, Alertmanager routing.

### Project 3 (Advanced): Scalable, HA Monitoring Platform

**Goal:** Production-grade, long-term, highly available monitoring on Kubernetes.

Steps:
1. Deploy via **kube-prometheus-stack** (Prometheus Operator) with HA pairs.
2. Use **ServiceMonitors/PodMonitors** and Kubernetes service discovery.
3. Add **remote write** to **Thanos/Mimir** for long-term storage and global query.
4. Implement multi-tenant, multi-cluster federation.
5. Define SLOs with multi-window, multi-burn-rate alerts.
6. Configure Alertmanager clustering, inhibition, silences, and on-call routing (PagerDuty).
7. Integrate Grafana dashboards as code.

**Skills:** Prometheus Operator, remote storage, HA, SLO engineering, multi-cluster.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Avoid high-cardinality labels** (user IDs, emails, request IDs) — they explode series count.
- **Use `rate()`/`increase()` on counters**, never raw counter values.
- **Follow metric naming conventions:** `unit` suffixes (`_seconds`, `_bytes`, `_total` for counters), lowercase with underscores.
- **Prefer histograms** for latency to aggregate percentiles across instances.
- **Use recording rules** for expensive/repeated queries.
- **Add `for:` durations** to alerts to avoid flapping.
- **Write meaningful alert annotations** (summary, description, runbook links).
- **Monitor the four golden signals** (latency, traffic, errors, saturation).
- **Set retention** and use remote storage for long-term needs.
- **Label alerts with severity** and route accordingly.
- **Keep `/metrics` cheap** to scrape; don't do heavy work on scrape.

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| High-cardinality labels | Memory blowup, slow/crashing Prometheus | Remove unbounded labels |
| Querying raw counters | Meaningless numbers | Use `rate()`/`increase()` |
| Too-short rate window | Noisy/empty results | Window ≥ 4× scrape interval |
| Alerts without `for:` | Flapping/alert fatigue | Add a stabilization duration |
| Treating Prometheus as long-term DB | Data loss, disk pressure | Remote write to Thanos/Mimir |
| Scraping too frequently | Load + storage cost | Tune `scrape_interval` |
| No HA | Monitoring blind spots during outages | Run redundant Prometheus |
| Summaries for cross-instance percentiles | Can't aggregate quantiles | Use histograms |
| Expensive `/metrics` handlers | Scrape timeouts | Precompute, keep endpoint fast |

---

## 7. Interview Questions

### Beginner

1. **What is Prometheus?**
   *An open-source monitoring/alerting toolkit that collects and stores time-series metrics via a pull model and queries them with PromQL.*

2. **What are the four metric types?**
   *Counter, Gauge, Histogram, Summary.*

3. **What is the difference between a counter and a gauge?**
   *A counter only increases (resets on restart); a gauge can go up or down.*

4. **How does Prometheus collect metrics?**
   *It pulls (scrapes) metrics over HTTP from each target's `/metrics` endpoint at a configured interval.*

5. **What is an exporter?**
   *A component that exposes metrics from a third-party system in Prometheus format (e.g., node_exporter for hosts).*

### Intermediate

6. **Why use `rate()` on counters?**
   *Counters monotonically increase; `rate()` computes the per-second average increase, which is the meaningful, restart-safe value.*

7. **What is label cardinality and why does it matter?**
   *The number of unique label-value combinations. High cardinality creates many time series, consuming memory and degrading performance — avoid unbounded labels.*

8. **What is the difference between a histogram and a summary?**
   *Histograms bucket observations server-side (quantiles computed at query time, aggregatable across instances); summaries compute quantiles client-side (not aggregatable).*

9. **What is a recording rule?**
   *A precomputed PromQL expression stored as a new time series to speed up frequent/expensive queries.*

10. **How does the Pushgateway fit in?**
    *It receives metrics pushed by short-lived/batch jobs that can't be scraped directly; Prometheus then scrapes the gateway.*

### Advanced

11. **How do you make Prometheus highly available and long-term?**
    *Run redundant Prometheus servers and use remote write to Thanos/Cortex/Mimir/VictoriaMetrics for durable, deduplicated, globally queryable, long-term storage; cluster Alertmanager.*

12. **Explain Alertmanager grouping, inhibition, and silencing.**
    *Grouping batches related alerts into one notification; inhibition suppresses lower-severity alerts when a related higher-severity one fires; silencing temporarily mutes alerts (e.g., maintenance).*

13. **What are the four golden signals and how would you alert on them?**
    *Latency, Traffic, Errors, Saturation. Alert on high error ratios, p95/p99 latency SLO breaches (histogram_quantile), saturation (CPU/memory/queue), with `for:` durations and burn-rate logic.*

14. **How does Kubernetes service discovery work in Prometheus?**
    *`kubernetes_sd_configs` queries the K8s API for nodes/pods/services/endpoints, and `relabel_configs` filter/transform them so targets update automatically as pods scale.*

15. **What problems do remote storage systems like Thanos solve?**
    *Long-term retention, global multi-cluster querying, downsampling, deduplication of HA pairs, and unlimited horizontal scaling beyond a single Prometheus node.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** Prometheus collects metrics primarily by:
- A) Pushing  B) Pulling/scraping  C) Polling a database  D) Streaming

**Q2.** Which metric type only increases?
- A) Gauge  B) Counter  C) Histogram  D) Summary

**Q3.** What language queries Prometheus data?
- A) SQL  B) PromQL  C) GraphQL  D) HCL

**Q4.** Which function gives per-second rate of a counter?
- A) `avg()`  B) `sum()`  C) `rate()`  D) `delta()`

**Q5.** What handles alert routing and notifications?
- A) Grafana  B) Pushgateway  C) Alertmanager  D) node_exporter

**Q6.** High-cardinality labels cause:
- A) Faster queries  B) Memory/series explosion  C) Better accuracy  D) Nothing

**Q7.** Which exporter provides Linux host metrics?
- A) blackbox_exporter  B) node_exporter  C) cAdvisor  D) kube-state-metrics

**Q8.** What computes a percentile from a histogram?
- A) `quantile()`  B) `histogram_quantile()`  C) `percentile()`  D) `topk()`

**Q9.** Which is best for long-term, scalable storage?
- A) Local TSDB only  B) Pushgateway  C) Thanos/Mimir via remote write  D) Alertmanager

**Q10.** What does `up == 0` indicate?
- A) Target at 0% CPU  B) Target is down/unreachable  C) No alerts  D) Zero requests

### Short Answer

**S1.** Write a PromQL query for the 5xx error ratio of `http_requests_total`.

**S2.** Why must a rate window be at least several scrape intervals long?

**S3.** What field in an alert rule prevents flapping by requiring a sustained condition?

**S4.** Name one exporter for probing external endpoints.

**S5.** What component lets batch jobs get their metrics into Prometheus?

### Answer Key

**Multiple Choice:** Q1-B, Q2-B, Q3-B, Q4-C, Q5-C, Q6-B, Q7-B, Q8-B, Q9-C, Q10-B

**Short Answer:**
- **S1.** `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))`
- **S2.** To capture enough samples for a meaningful, non-noisy average (and survive missed scrapes); use ≥ 4× scrape interval.
- **S3.** `for:`
- **S4.** `blackbox_exporter`
- **S5.** The **Pushgateway**.

---

## 9. Further Resources

### Official Documentation
- Prometheus Docs — https://prometheus.io/docs/
- PromQL — https://prometheus.io/docs/prometheus/latest/querying/basics/
- Alertmanager — https://prometheus.io/docs/alerting/latest/alertmanager/
- Exporters list — https://prometheus.io/docs/instrumenting/exporters/

### Learning
- "Monitoring with Prometheus" tutorials — https://prometheus.io/docs/tutorials/
- Robust Perception blog — https://www.robustperception.io/blog
- Awesome Prometheus alerts — https://samber.github.io/awesome-prometheus-alerts/

### Tools & Ecosystem
- Grafana (visualization), kube-prometheus-stack (Operator)
- Thanos / Cortex / Mimir / VictoriaMetrics (long-term storage)
- node_exporter, cAdvisor, blackbox_exporter, kube-state-metrics

### Books
- *Prometheus: Up & Running* — Brian Brazil
- *Site Reliability Engineering* (Google, free online) — golden signals, SLOs

### Certifications / Related
- Prometheus Certified Associate (PCA) — CNCF

---

*End of Prometheus — Zero to Hero.*
