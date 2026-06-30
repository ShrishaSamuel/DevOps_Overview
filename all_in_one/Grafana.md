# Grafana — Zero to Hero

> A complete, beginner-to-advanced guide to Grafana for observability dashboards, visualization, and alerting.

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

### 1.1 What is Grafana?

**Grafana** is an open-source **analytics and interactive visualization platform**. Created by Torkel Ödegaard in 2014, it connects to dozens of data sources (Prometheus, Loki, Elasticsearch, InfluxDB, SQL databases, cloud monitoring, and more) and lets you build rich, dynamic **dashboards** of charts, gauges, tables, and alerts. It is the de-facto front-end for the observability stack.

Grafana **does not store data** itself (in its core role) — it **queries** existing data sources and visualizes the results. This separation is key: your metrics live in Prometheus, logs in Loki/Elasticsearch, traces in Tempo/Jaeger, and Grafana unifies them into a single pane of glass.

### 1.2 Why Grafana?

| Feature | Explanation |
|---------|-------------|
| **Multi-source** | Combine metrics, logs, and traces from many backends in one dashboard |
| **Rich visualizations** | Time series, gauges, heatmaps, tables, geomaps, stat panels |
| **Templating** | Variables make dashboards dynamic and reusable across hosts/services |
| **Alerting** | Unified alerting across data sources with flexible routing |
| **Sharing** | Export/import dashboards as JSON; thousands of community dashboards |
| **Observability suite** | Pairs with Loki (logs), Tempo (traces), Mimir (metrics) |

### 1.3 The three pillars of observability

Grafana is the visualization layer across the three pillars:

```
   METRICS              LOGS                 TRACES
   (Prometheus,         (Loki,               (Tempo,
    Mimir,              Elasticsearch)       Jaeger)
    Graphite)
        \                  |                   /
         \                 |                  /
          ▼                ▼                 ▼
        +-------------------------------------+
        |             GRAFANA                 |
        |  Dashboards · Explore · Alerting    |
        +-------------------------------------+
```

- **Metrics** — numeric time series (what is happening; trends).
- **Logs** — discrete event records (details/context).
- **Traces** — request paths across services (where time is spent).

### 1.4 Architecture

```
   +---------+   +-----------+   +-------------+   +-----------+
   | Browser |◄─►|  Grafana  |◄─►| Data Source |   | Grafana   |
   |  (UI)   |   |  Server   |   | (Prometheus,|   | Database  |
   +---------+   | - panels  |   |  Loki, SQL) |   | (SQLite/  |
                 | - alerting|   +-------------+   |  MySQL/PG)|
                 | - plugins |                     | dashboards|
                 +-----------+                     | users,etc |
```

- **Grafana Server** — serves the UI/API, runs queries against data sources, evaluates alerts.
- **Data sources** — external systems Grafana queries (it doesn't store the telemetry).
- **Grafana database** — stores Grafana's own state (dashboards, users, settings) in SQLite (default), MySQL, or PostgreSQL.

### 1.5 When to use Grafana

- Visualizing Prometheus/other metrics in dashboards.
- Correlating metrics, logs, and traces in one place.
- Building NOC/operations dashboards and SLO views.
- Unified alerting across heterogeneous data sources.

When **not** primary: Grafana is a visualization/alerting layer — it's not a database or a data collector. You still need Prometheus/Loki/etc. to gather and store data.

---

## 2. Installation & Setup

### 2.1 Run with Docker (quickest)

```bash
docker run -d --name grafana -p 3000:3000 grafana/grafana
# UI: http://localhost:3000  (default login admin / admin)
```

### 2.2 Install on Ubuntu

```bash
sudo apt-get install -y apt-transport-https software-properties-common
wget -q -O - https://apt.grafana.com/gpg.key | sudo gpg --dearmor -o /usr/share/keyrings/grafana.gpg
echo "deb [signed-by=/usr/share/keyrings/grafana.gpg] https://apt.grafana.com stable main" | \
  sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install -y grafana
sudo systemctl enable --now grafana-server
```

### 2.3 Docker Compose with Prometheus

```yaml
# docker-compose.yml
services:
  prometheus:
    image: prom/prometheus
    ports: ["9090:9090"]
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
  grafana:
    image: grafana/grafana
    ports: ["3000:3000"]
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
volumes:
  grafana-data:
```

```bash
docker compose up -d
```

### 2.4 First login

1. Open `http://localhost:3000`.
2. Log in with `admin` / `admin` and set a new password.
3. Go to **Connections → Data sources → Add data source → Prometheus**.
4. URL: `http://prometheus:9090` (Compose) or `http://localhost:9090`.
5. **Save & test** — expect "Data source is working".

---

## 3. Core Concepts

### 3.1 BEGINNER

#### 3.1.1 Data sources

A **data source** is a backend Grafana queries. Add via **Connections → Data sources**. Common ones:

| Data source | Type |
|-------------|------|
| Prometheus / Mimir | Metrics |
| Loki | Logs |
| Tempo / Jaeger | Traces |
| Elasticsearch | Logs/metrics |
| InfluxDB / Graphite | Metrics |
| MySQL / PostgreSQL | SQL |
| CloudWatch / Azure Monitor / Google Cloud Monitoring | Cloud |

#### 3.1.2 Dashboards and panels

- **Dashboard** — a set of panels arranged on a grid, sharing a time range and variables.
- **Panel** — a single visualization (a chart, table, gauge, etc.) backed by one or more **queries**.

Create: **Dashboards → New → New dashboard → Add visualization → choose data source → write query.**

#### 3.1.3 Panel/visualization types

| Type | Use |
|------|-----|
| **Time series** | Trends over time (most common) |
| **Stat** | Single big number (current value) |
| **Gauge** | Value within a range with thresholds |
| **Bar gauge** | Multiple values as bars |
| **Table** | Tabular data |
| **Heatmap** | Distribution over time (histograms) |
| **Logs** | Log lines (with Loki/Elasticsearch) |
| **Pie chart / Bar chart** | Proportions/comparisons |
| **Geomap** | Geographic data |

#### 3.1.4 Writing a query (Prometheus example)

In a Time series panel with a Prometheus data source:

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

Set the panel title ("CPU Usage %"), unit (percent 0-100), and legend (`{{instance}}`).

#### 3.1.5 Time range and refresh

- **Time picker** (top right): last 5m, 1h, 24h, 7d, or custom/absolute.
- **Auto-refresh:** 5s, 10s, 30s, 1m, etc.
- Panels respect the dashboard's global time range unless overridden.

### 3.2 INTERMEDIATE

#### 3.2.1 Template variables

**Variables** make dashboards dynamic and reusable. Define under **Dashboard settings → Variables**.

```
Name: instance
Type: Query
Data source: Prometheus
Query: label_values(node_cpu_seconds_total, instance)
```

Use it in panel queries:

```promql
node_load1{instance="$instance"}
```

Variable types: **Query** (from data source), **Custom** (static list), **Interval** (time windows), **Datasource**, **Constant**, **Text box**, **Ad hoc filters**. Enable **Multi-value** and **Include All** for flexible selection.

#### 3.2.2 Legends, units, and field options

- **Units:** set per axis (bytes, percent, seconds, requests/sec) for readable values.
- **Legend:** customize with templates like `{{job}} - {{instance}}`; show min/max/avg/last in a table legend.
- **Thresholds:** color values by range (e.g., green < 70, yellow < 85, red ≥ 85).
- **Value mappings:** map numbers to text (e.g., `1 → UP`, `0 → DOWN`).
- **Overrides:** apply different settings to specific series.

#### 3.2.3 Transformations

Transformations post-process query results before visualization (no extra queries):
- **Reduce** — collapse series to a single value (last/max/mean).
- **Merge / Join by field** — combine series/tables.
- **Organize fields** — rename/reorder/hide.
- **Group by**, **Filter by value**, **Add field from calculation**.

Useful for building tables from multiple metrics or computing ratios in the UI.

#### 3.2.4 Annotations

**Annotations** mark events on graphs (deployments, incidents). Sources: manual, a query (e.g., from a data source), or built-in (alerts). They appear as vertical lines, giving context ("the latency spike started right after this deploy").

#### 3.2.5 Explore mode

**Explore** is an ad-hoc query workspace (not tied to a dashboard) — ideal for troubleshooting. With logs+metrics data sources you can pivot from a metric spike to correlated logs/traces (with derived fields and exemplars).

#### 3.2.6 Provisioning (dashboards/data sources as code)

Avoid click-ops by provisioning via YAML/JSON files:

```yaml
# /etc/grafana/provisioning/datasources/ds.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

```yaml
# /etc/grafana/provisioning/dashboards/dash.yml
apiVersion: 1
providers:
  - name: 'default'
    folder: 'Provisioned'
    type: file
    options:
      path: /var/lib/grafana/dashboards
```

Place exported dashboard JSON files in that path. This makes dashboards reproducible and version-controlled.

#### 3.2.7 Importing community dashboards

Grafana.com hosts thousands of prebuilt dashboards (each has an ID). **Dashboards → New → Import → enter ID** (e.g., **1860** = Node Exporter Full) → select your data source.

### 3.3 ADVANCED

#### 3.3.1 Grafana unified alerting

Grafana's built-in alerting evaluates rules across any data source.

Components:
- **Alert rule:** a query + condition + evaluation interval + `for` duration.
- **Contact point:** where notifications go (email, Slack, PagerDuty, webhook, Teams, etc.).
- **Notification policy:** routing tree matching labels to contact points (grouping, timing).
- **Silences & mute timings:** suppress notifications (maintenance windows).

Example rule logic:

```
WHEN  avg(rate(http_requests_total{status=~"5.."}[5m])) / avg(rate(http_requests_total[5m]))
IS ABOVE 0.05
FOR 5m  →  severity=critical → route to PagerDuty
```

#### 3.3.2 Loki integration (logs)

Add **Loki** as a data source and use **LogQL**:

```logql
{job="myapp"} |= "error"                       # logs containing "error"
{job="myapp"} | json | status >= 500           # parse JSON, filter
sum(rate({job="myapp"} |= "error" [5m]))        # metric from logs
```

Add a **Logs panel** and correlate with metrics on the same dashboard. Use **derived fields** to link a log line to a trace.

#### 3.3.3 Traces (Tempo/Jaeger)

Add **Tempo**, visualize distributed traces, and configure **trace-to-logs** and **metrics-to-traces** correlations (exemplars) for end-to-end debugging within one UI.

#### 3.3.4 Authentication and access control

- **Auth:** built-in users, LDAP, OAuth (Google/GitHub/Azure AD/Okta), SAML (Enterprise).
- **Organizations & Teams:** multi-tenant separation.
- **Roles:** Viewer, Editor, Admin; **RBAC** (Enterprise) for fine-grained control.
- **Folders:** group dashboards with per-folder permissions.

```ini
# grafana.ini (OAuth example)
[auth.github]
enabled = true
client_id = ...
client_secret = ...
allowed_organizations = my-org
```

#### 3.3.5 Dashboards as code

- Export dashboards as JSON; store in Git; deploy via provisioning or the API.
- Tools: **Grafonnet** (Jsonnet libraries), **Terraform Grafana provider**, **grizzly**.
- This enables review, versioning, and consistent environments.

```hcl
# Terraform
resource "grafana_dashboard" "main" {
  config_json = file("dashboards/overview.json")
}
```

#### 3.3.6 Performance & scaling

- Use **recording rules** in Prometheus for heavy queries; keep dashboard queries light.
- Limit panel count and time ranges; avoid very high-resolution over long windows.
- Set sensible **min interval** and use `$__rate_interval`.
- For large orgs, run Grafana with an external DB (Postgres/MySQL), behind a load balancer, stateless replicas.

#### 3.3.7 The Grafana observability stack (LGTM)

- **L**oki — logs
- **G**rafana — visualization
- **T**empo — traces
- **M**imir — scalable Prometheus-compatible metrics

Together they form a cohesive, horizontally scalable observability platform.

---

## 4. Hands-on Tasks

### Task 1: Run Grafana and add Prometheus

```bash
docker compose up -d        # from section 2.3
```

Add the Prometheus data source (`http://prometheus:9090`), Save & test.

**Expected:** "Data source is working".

### Task 2: Create your first panel

New dashboard → Add visualization → Prometheus → query `up` → save as "Targets".

**Expected:** A time series showing target up/down state.

### Task 3: Build a CPU usage panel

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

Set unit = Percent (0-100), legend = `{{instance}}`.

**Expected:** A clean CPU% chart per instance.

### Task 4: Add a Stat and a Gauge panel

- Stat panel: `count(up == 1)` → "Targets Up".
- Gauge panel: memory used % with thresholds (green/yellow/red).

**Expected:** A big number and a colored gauge.

### Task 5: Add a template variable

Create variable `instance` = `label_values(node_cpu_seconds_total, instance)`, then filter panels with `{instance="$instance"}`.

**Expected:** A dropdown switches the dashboard between instances.

### Task 6: Import a community dashboard

Import ID **1860** (Node Exporter Full), select your Prometheus source.

**Expected:** A full host dashboard renders immediately.

### Task 7: Add thresholds and value mappings

On a panel, color values by threshold; map `1 → UP`, `0 → DOWN`.

**Expected:** Cells/values show colors and friendly text.

### Task 8: Use a transformation

Add two metric queries, then a **Reduce** or **Organize fields** transformation to build a summary table.

**Expected:** A combined table from multiple series.

### Task 9: Create an alert rule

Alerting → New alert rule → query error ratio → condition `> 0.05` for 5m → contact point (email/Slack/webhook).

**Expected:** Rule shows Normal/Pending/Firing; notification on firing.

### Task 10: Provision a dashboard as code

Export the dashboard JSON, place it under the provisioning path, restart Grafana.

**Expected:** The dashboard appears automatically (reproducible).

---

## 5. Projects

### Project 1 (Beginner): Infrastructure Dashboard

**Goal:** A single-host monitoring dashboard.

Steps:
1. Stand up Prometheus + node_exporter + Grafana (Compose).
2. Add Prometheus data source.
3. Build panels: CPU%, memory%, disk%, network I/O, load average, uptime.
4. Add a host template variable.
5. Set units, thresholds, and a tidy layout.

**Skills:** data sources, panels, PromQL, variables, units/thresholds.

### Project 2 (Intermediate): Application SLO Dashboard

**Goal:** Visualize service health using RED metrics and SLOs.

Steps:
1. Instrument an app exposing request rate, errors, latency histograms.
2. Build panels: request rate, error ratio, p50/p95/p99 latency (heatmap + lines).
3. Add SLO panels (e.g., availability %, error budget burn).
4. Use variables for service/endpoint selection.
5. Add deployment **annotations** to correlate releases with changes.
6. Configure alerts on SLO breaches with Slack routing.

**Skills:** RED/SLO visualization, histograms, annotations, alerting.

### Project 3 (Advanced): Unified Observability Platform (LGTM)

**Goal:** Single-pane-of-glass metrics + logs + traces, as code.

Steps:
1. Deploy Grafana with **Prometheus/Mimir** (metrics), **Loki** (logs), **Tempo** (traces).
2. Correlate: click a latency spike → jump to related logs (derived fields) → open the trace.
3. Implement **dashboards as code** (Terraform/Grafonnet) in Git with CI deployment.
4. Configure **unified alerting** with notification policies, grouping, and on-call routing.
5. Set up **OAuth/SAML auth**, teams, folders, and RBAC.
6. Run HA Grafana with an external Postgres backend.

**Skills:** LGTM stack, correlation, dashboards-as-code, auth/RBAC, HA, advanced alerting.

---

## 6. Best Practices & Common Pitfalls

### Best Practices

- **Use template variables** for reusable, dynamic dashboards instead of hard-coded hosts.
- **Set proper units and thresholds** so values are instantly readable.
- **Keep dashboards focused** — one purpose per dashboard; link related ones.
- **Provision dashboards/data sources as code** (Git) for reproducibility.
- **Leverage community dashboards** (e.g., 1860) as starting points.
- **Use recording rules** in Prometheus for heavy queries; keep panels light.
- **Add annotations** for deployments/incidents to add context.
- **Use folders + permissions** to organize and secure dashboards.
- **Use `$__rate_interval`** and sensible min intervals for correct rates across zoom levels.
- **Correlate metrics, logs, and traces** for faster root-cause analysis.
- **Limit auto-refresh** frequency to reduce load.

### Common Pitfalls

| Pitfall | Consequence | Fix |
|---------|-------------|-----|
| Hard-coded instances in queries | Non-reusable dashboards | Use template variables |
| No units set | Misleading values (bytes vs MB) | Configure field units |
| Too many panels/long ranges | Slow dashboards, backend load | Split dashboards, recording rules |
| Click-ops only | Lost work, drift | Provision as code + Git |
| Treating Grafana as a datastore | Misunderstanding architecture | It queries; data lives in sources |
| Overly aggressive refresh | Backend overload | Increase refresh interval |
| Ignoring `$__rate_interval` | Wrong rates when zooming | Use it for counter rates |
| Alerts without grouping/routing | Notification storms | Configure notification policies |
| No auth/RBAC | Unauthorized access | Enable OAuth/SAML + roles |
| Duplicated dashboards everywhere | Maintenance nightmare | Variables + shared, versioned dashboards |

---

## 7. Interview Questions

### Beginner

1. **What is Grafana?**
   *An open-source visualization and analytics platform that queries data sources and builds dashboards and alerts.*

2. **Does Grafana store metrics?**
   *No — it queries external data sources (like Prometheus). It only stores its own config (dashboards, users) in its database.*

3. **What is a data source?**
   *A backend Grafana connects to and queries (Prometheus, Loki, SQL, CloudWatch, etc.).*

4. **What is the difference between a dashboard and a panel?**
   *A dashboard is a collection of panels; a panel is a single visualization backed by queries.*

5. **Name a few panel types.**
   *Time series, stat, gauge, table, heatmap, logs.*

### Intermediate

6. **What are template variables and why use them?**
   *Dynamic placeholders (e.g., `$instance`) that make dashboards reusable and interactive via dropdowns.*

7. **What is the purpose of transformations?**
   *To post-process query results (reduce, join, rename, compute fields) before visualization, without extra queries.*

8. **How do annotations help?**
   *They mark events (deploys, incidents) on graphs to correlate them with metric changes.*

9. **What is Explore mode?**
   *An ad-hoc, dashboard-free query workspace for troubleshooting, supporting pivoting between metrics, logs, and traces.*

10. **How do you make dashboards reproducible?**
    *Provision data sources/dashboards from code (YAML/JSON in Git), or manage them via Terraform/Grafonnet/API.*

### Advanced

11. **Explain Grafana unified alerting components.**
    *Alert rules (query+condition+`for`), contact points (where to notify), notification policies (label-based routing/grouping), and silences/mute timings.*

12. **How do you correlate metrics, logs, and traces in Grafana?**
    *Use the LGTM stack (Prometheus/Mimir, Loki, Tempo) with derived fields/exemplars so you can jump from a metric spike to logs and from logs to traces in one UI.*

13. **How would you scale Grafana for a large organization?**
    *Run stateless replicas behind a load balancer with an external Postgres/MySQL backend, use folders/RBAC/teams, provision as code, and offload heavy queries to recording rules/Mimir.*

14. **What is the difference between `$__interval` and `$__rate_interval`?**
    *`$__interval` is the auto step based on time range/width; `$__rate_interval` is tuned for counter `rate()` queries to remain correct across zoom levels (≥ 4× scrape interval).*

15. **How do you secure Grafana?**
    *Integrate OAuth/SAML/LDAP, enforce roles/RBAC, use organizations/teams and folder permissions, disable anonymous access, and run behind TLS.*

---

## 8. Quizzes

### Multiple Choice

**Q1.** Grafana primarily provides:
- A) Metric storage  B) Visualization & alerting  C) Log collection  D) Tracing backend

**Q2.** Which is NOT a typical Grafana data source?
- A) Prometheus  B) Loki  C) Docker  D) PostgreSQL

**Q3.** A single visualization in Grafana is a:
- A) Dashboard  B) Panel  C) Folder  D) Variable

**Q4.** What makes dashboards dynamic/reusable?
- A) Annotations  B) Transformations  C) Template variables  D) Thresholds

**Q5.** Which post-processes results without new queries?
- A) Variables  B) Transformations  C) Alerts  D) Panels

**Q6.** Where does Grafana store its own dashboards/users?
- A) Prometheus  B) Its database (SQLite/MySQL/PG)  C) Loki  D) Browser only

**Q7.** Which query language is used with Loki?
- A) PromQL  B) LogQL  C) SQL  D) TraceQL

**Q8.** A famous Node Exporter community dashboard ID is:
- A) 1860  B) 9090  C) 3000  D) 100

**Q9.** Which marks events like deployments on a graph?
- A) Threshold  B) Annotation  C) Legend  D) Override

**Q10.** The LGTM stack stands for:
- A) Logs, Graphs, Time, Metrics  B) Loki, Grafana, Tempo, Mimir  C) Linux, Git, Tools, Monitoring  D) Logging, Graphing, Tracing, Metrics

### Short Answer

**S1.** What is the default Grafana port and default login?

**S2.** Write a template-variable query to list `instance` label values from `node_cpu_seconds_total`.

**S3.** Which Grafana feature routes alert notifications based on labels?

**S4.** Name the four components of the LGTM observability stack.

**S5.** Why should heavy dashboard queries be moved to Prometheus recording rules?

### Answer Key

**Multiple Choice:** Q1-B, Q2-C, Q3-B, Q4-C, Q5-B, Q6-B, Q7-B, Q8-A, Q9-B, Q10-B

**Short Answer:**
- **S1.** Port **3000**, login `admin` / `admin`.
- **S2.** `label_values(node_cpu_seconds_total, instance)`
- **S3.** **Notification policies** (in unified alerting).
- **S4.** Loki (logs), Grafana (visualization), Tempo (traces), Mimir (metrics).
- **S5.** To precompute expensive queries so dashboards load fast and reduce backend load.

---

## 9. Further Resources

### Official Documentation
- Grafana Docs — https://grafana.com/docs/grafana/latest/
- Dashboards best practices — https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/best-practices/
- Grafana Alerting — https://grafana.com/docs/grafana/latest/alerting/
- Provisioning — https://grafana.com/docs/grafana/latest/administration/provisioning/

### Learning
- Grafana Play (live demo) — https://play.grafana.org
- Grafana Fundamentals tutorials — https://grafana.com/tutorials/
- Community dashboards — https://grafana.com/grafana/dashboards/

### Tools & Ecosystem
- Loki, Tempo, Mimir (the LGTM stack)
- Grafana Terraform provider, Grafonnet, grizzly (dashboards as code)
- Grafana OnCall (on-call management), Grafana Agent / Alloy (collection)

### Books & Guides
- *Grafana documentation* (comprehensive, free)
- *Observability Engineering* — Charity Majors et al.
- *Site Reliability Engineering* (Google) — SLOs & dashboards

---

*End of Grafana — Zero to Hero.*
