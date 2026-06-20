# Day 28 — Grafana & Alerting

> Dashboards, alert rules, Prometheus data source, SLO/SLI concepts

---

## Theory

### Grafana Architecture

Grafana is a visualization and alerting platform. It does **not store data** — it queries external data sources and renders the results.

![Alt text](grafana.png)
#### Key Concepts

| Concept | Description |
|---|---|
| **Data Source** | A connection to an external system (Prometheus, Loki, etc.) |
| **Dashboard** | A collection of panels arranged on a grid |
| **Panel** | A single visualization (graph, gauge, stat, table, heatmap) |
| **Variable** | A dashboard-level dropdown that filters all panels (e.g., `$namespace`, `$pod`) |
| **Annotation** | A vertical marker on graphs showing deployments, incidents, etc. |
| **Folder** | Organizes dashboards; also used for RBAC |
| **Alert Rule** | A PromQL expression evaluated on a schedule; fires when a condition is met |
| **Contact Point** | Where notifications go (Slack channel, email, PagerDuty key) |
| **Notification Policy** | Routes alerts to the right contact point based on labels |

---

### SLI and SLO Definitions

#### SLI — Service Level Indicator

An SLI is a **quantitative measure** of some aspect of the service's behavior. It is always a ratio or a measured value.

The most common SLIs map to three categories:

| SLI Type | What it measures | Example PromQL |
|---|---|---|
| **Availability** | % of requests that succeed | `sum(rate(http_requests_total{status_code!~"5.."}[5m])) / sum(rate(http_requests_total[5m]))` |
| **Latency** | % of requests faster than a threshold | `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))` |
| **Error Rate** | % of requests that fail | `sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))` |
| **Throughput** | Requests per second | `sum(rate(http_requests_total[5m]))` |
| **Saturation** | How "full" the service is (CPU, memory) | `container_memory_usage_bytes / container_spec_memory_limit_bytes` |

#### SLO — Service Level Objective

An SLO is a **target value** for an SLI over a given time window. It sets the bar for what "good enough" means.

```
SLO = SLI + Target + Time Window

Example:
  SLI:    availability ratio
  Target: ≥ 99.9%
  Window: rolling 30 days

  → "99.9% of requests will succeed over any 30-day window"
```

#### Error Budget

The error budget is the acceptable amount of unreliability within an SLO window:

```
Error Budget = 1 - SLO target

For 99.9% availability over 30 days:
  Error Budget = 0.1% of 30 days
               = 43.8 minutes of downtime allowed per month
```

When the error budget is nearly exhausted, deployments are slowed and reliability work takes priority.

#### SLA — Service Level Agreement

An SLA is a **contractual commitment** between a provider and a customer, with financial consequences if breached. SLOs are internal targets that should be stricter than SLAs to provide a safety buffer.

```
SLA (external, contractual) < SLO (internal target) ← always set SLO stricter
```

---

### Grafana Alerting: Contact Points & Notification Policies

Grafana Alerting (unified alerting, introduced in Grafana 8) works in three layers:

```
Alert Rule (PromQL condition)
        ↓ fires
Alert Instance (with labels: severity=critical, service=node-app)
        ↓ matched by
Notification Policy (route: severity=critical → PagerDuty)
        ↓ sent to
Contact Point (PagerDuty integration key / Slack webhook)
```

**Contact Points** — define *how* to send a notification (the destination):
- Slack: webhook URL + channel
- Email: SMTP config + recipient list
- PagerDuty: integration key
- OpsGenie, Webhook, Telegram, etc.

**Notification Policies** — define *which* alerts go to *which* contact point, using label matchers:

```yaml
# Root policy (default catch-all)
receiver: default-email

# Child routes (evaluated top-to-bottom, first match wins)
routes:
  - matchers:
      - severity = critical
    receiver: pagerduty
    group_wait: 30s
    group_interval: 5m
    repeat_interval: 1h

  - matchers:
      - severity = warning
    receiver: slack-warnings
    repeat_interval: 4h
```

---

### The RED Method

RED is a framework for the three key metrics every service should expose. It was popularized by Tom Wilkie (Grafana Labs) and complements Google's USE method for resources.

| Letter | Metric | Description | PromQL Example |
|---|---|---|---|
| **R** — Rate | Requests per second | How much traffic the service handles | `sum(rate(http_requests_total[5m]))` |
| **E** — Errors | Error rate | Proportion of failed requests | `sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))` |
| **D** — Duration | Latency distribution | How long requests take (p50, p95, p99) | `histogram_quantile(0.99, sum by(le) (rate(http_request_duration_seconds_bucket[5m])))` |

> **RED is for services. USE (Utilization, Saturation, Errors) is for resources** (CPU, memory, disks).

Every Grafana dashboard for a service should have at minimum one panel for each RED signal.

---

### Grafana vs Kibana vs Datadog

| Feature | Grafana | Kibana | Datadog |
|---|---|---|---|
| **Primary use** | Metrics visualization + alerting | Log search + analytics | Full-stack APM + monitoring |
| **Data sources** | Many (Prometheus, Loki, etc.) | Elasticsearch / OpenSearch | Datadog Agent only |
| **Metrics** | ✅ Via Prometheus/Mimir | ⚠️ Limited (Elastic metrics) | ✅ Native |
| **Logs** | ✅ Via Loki | ✅ Native (ELK stack) | ✅ Native |
| **Traces** | ✅ Via Tempo | ✅ Via APM | ✅ Native |
| **Alerting** | ✅ Built-in (Grafana 8+) | ✅ Built-in | ✅ Built-in |
| **Hosting** | Open source / Grafana Cloud | Open source / Elastic Cloud | SaaS only |
| **Cost** | Free OSS, paid cloud | Free OSS, paid cloud | Per-host pricing (expensive at scale) |
| **Best for** | Teams already using Prometheus | Teams using ELK for logs | Orgs wanting a managed all-in-one |

---

## Practical Lab

### 1. Access Grafana in Minikube & Configure Prometheus Data Source

```bash
# Port-forward Grafana (from kube-prometheus-stack installed on Day 27)
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3001:80

# Open http://localhost:3001
# Username: admin
# Password: admin123  (or retrieve with:)
kubectl get secret -n monitoring kube-prometheus-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

**Add Prometheus as a data source (if not already configured):**

1. Go to **Connections → Data Sources → Add data source**
2. Select **Prometheus**
3. Set URL: `http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090`
4. Click **Save & Test** — you should see "Data source is working"

> `kube-prometheus-stack` pre-configures this automatically. Verify at **Connections → Data Sources**.

---

### 2. Build the Dashboard

#### Dashboard Variables (add first for filtering)

Go to **Dashboard Settings → Variables → Add variable**:

```
Name:  namespace
Type:  Query
Query: label_values(kube_pod_info, namespace)
```

```
Name:  pod
Type:  Query
Query: label_values(kube_pod_info{namespace="$namespace"}, pod)
```

These create dropdowns that filter all panels dynamically.

---

#### Panel 1 — HTTP Requests Per Second (Time Series)

```promql
# Query
sum by (route) (
  rate(http_requests_total{namespace="$namespace"}[5m])
)

# Panel settings
Type:       Time series
Title:      HTTP Request Rate (RPS)
Unit:       reqps (requests/sec)
Legend:     {{ route }}
```

---

#### Panel 2 — HTTP Error Rate (Time Series + Threshold)

```promql
# Query A — error rate as a percentage
100 * sum(rate(http_requests_total{status_code=~"5..", namespace="$namespace"}[5m]))
    / sum(rate(http_requests_total{namespace="$namespace"}[5m]))

# Panel settings
Type:       Time series
Title:      HTTP Error Rate (%)
Unit:       percent (0-100)
Thresholds: 0=green, 0.1=yellow, 1=red
```

---

#### Panel 3 — p99 Latency (Time Series)

```promql
# Query
histogram_quantile(0.99,
  sum by (le, route) (
    rate(http_request_duration_seconds_bucket{namespace="$namespace"}[5m])
  )
)

# Panel settings
Type:       Time series
Title:      p99 Latency
Unit:       seconds (s)
Thresholds: 0=green, 0.3=yellow, 0.5=red
Legend:     {{ route }}
```

---

#### Panel 4 — Pod CPU Usage (Time Series)

```promql
# Query
sum by (pod) (
  rate(container_cpu_usage_seconds_total{
    namespace="$namespace",
    container!="",
    pod=~"node-app.*"
  }[5m])
)

# Panel settings
Type:  Time series
Title: Pod CPU Usage (cores)
Unit:  short (cores)
```

---

#### Panel 5 — Pod Memory Usage (Time Series)

```promql
# Query — memory usage vs limit
sum by (pod) (
  container_memory_working_set_bytes{
    namespace="$namespace",
    container!="",
    pod=~"node-app.*"
  }
)

# Panel settings
Type:  Time series
Title: Pod Memory Usage
Unit:  bytes (IEC)
```

---

#### Panel 6 — Availability SLO Gauge (Stat Panel)

```promql
# Query — % of successful requests in last 24h
100 * sum(increase(http_requests_total{status_code!~"5..", namespace="$namespace"}[24h]))
    / sum(increase(http_requests_total{namespace="$namespace"}[24h]))

# Panel settings
Type:       Stat
Title:      Availability (24h)
Unit:       percent (0-100)
Thresholds: 0=red, 99=yellow, 99.9=green
```

---

### 3. Configure Email/Slack Alert for High Error Rate

#### Step 1 — Create a Contact Point

Go to **Alerting → Contact Points → Add contact point**:

**Slack:**
```
Name:        slack-critical
Type:        Slack
Webhook URL: https://hooks.slack.com/services/YOUR/WEBHOOK/URL
Channel:     #alerts
Title:       [{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}
Message:     {{ range .Alerts }}{{ .Annotations.description }}{{ end }}
```

**Email:**
```
Name:        email-oncall
Type:        Email
Addresses:   oncall@yourcompany.com
Subject:     [Grafana Alert] {{ .GroupLabels.alertname }}
```

#### Step 2 — Create the Alert Rule

Go to **Alerting → Alert Rules → New alert rule**:

```
Rule name:   HighHTTPErrorRate
Folder:      NodeApp Alerts
Group:       http-rules
Evaluate:    every 1m  for  5m

Query A (PromQL):
  100 * sum(rate(http_requests_total{status_code=~"5.."}[5m]))
      / sum(rate(http_requests_total[5m]))

Condition:   WHEN last() OF A IS ABOVE 1

Labels:
  severity = critical
  service  = node-app

Annotations:
  summary:     High HTTP error rate on node-app
  description: Error rate is {{ $value | printf "%.2f" }}% — exceeds 1% threshold.
               Investigate recent deployments and application logs.
  runbook_url: https://wiki.internal/runbooks/high-error-rate
```

#### Step 3 — Create a Notification Policy

Go to **Alerting → Notification Policies → Add nested policy**:

```
Matching labels:  severity = critical
Contact point:    slack-critical
Group by:         alertname, service
Group wait:       30s
Group interval:   5m
Repeat interval:  1h
```

---

## Assignment 28

> Define SLOs for the Node.js app: **99.9% availability**, **p99 < 500ms**, **error rate < 0.1%**

### SLO Definitions

```
Service: node-app (Node.js HTTP API)
Window:  Rolling 30 days

SLO 1 — Availability
  SLI: ratio of non-5xx requests to total requests
  Target: ≥ 99.9%
  Error budget: 0.1% = 43.8 minutes/month

SLO 2 — Latency
  SLI: 99th percentile HTTP response time
  Target: p99 < 500ms
  Error budget: up to 0.1% of requests may exceed 500ms

SLO 3 — Error Rate
  SLI: ratio of 5xx responses to total requests
  Target: < 0.1%
  Error budget: at most 1 in 1000 requests may be an error
```

---

### SLI PromQL Expressions

```promql
# SLO 1 — Availability SLI (30-day window)
sum(increase(http_requests_total{status_code!~"5.."}[30d]))
/
sum(increase(http_requests_total[30d]))

# SLO 2 — p99 Latency SLI (current)
histogram_quantile(0.99,
  sum by (le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)

# SLO 3 — Error Rate SLI (current %)
100 * sum(rate(http_requests_total{status_code=~"5.."}[5m]))
    / sum(rate(http_requests_total[5m]))
```

---

### Alert Rules for SLO Breaches

```yaml
# prometheus-slo-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: node-app-slo-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: node-app.slo
      interval: 60s
      rules:

        # ── Recording rules (pre-compute SLIs for efficiency) ──────────────

        - record: job:http_requests_total:rate5m
          expr: sum by (status_code) (rate(http_requests_total[5m]))

        - record: job:http_request_duration_seconds:p99_5m
          expr: |
            histogram_quantile(0.99,
              sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
            )

        # ── SLO 1: Availability < 99.9% ────────────────────────────────────

        - alert: SLOAvailabilityBreach
          expr: |
            (
              sum(increase(http_requests_total{status_code!~"5.."}[1h]))
              /
              sum(increase(http_requests_total[1h]))
            ) < 0.999
          for: 5m
          labels:
            severity: critical
            slo: availability
            service: node-app
          annotations:
            summary: "Availability SLO breach — node-app"
            description: >
              1-hour availability is {{ $value | humanizePercentage }},
              below the 99.9% SLO target.
              Error budget is being consumed rapidly.
            runbook_url: "https://wiki.internal/runbooks/availability-slo"

        # ── SLO 2: p99 latency > 500ms ─────────────────────────────────────

        - alert: SLOLatencyBreach
          expr: job:http_request_duration_seconds:p99_5m > 0.5
          for: 10m
          labels:
            severity: warning
            slo: latency
            service: node-app
          annotations:
            summary: "p99 latency SLO breach — node-app"
            description: >
              p99 latency is {{ $value | humanizeDuration }},
              exceeding the 500ms SLO target.
            runbook_url: "https://wiki.internal/runbooks/latency-slo"

        # ── SLO 3: Error rate > 0.1% ───────────────────────────────────────

        - alert: SLOErrorRateBreach
          expr: |
            100 * sum(rate(http_requests_total{status_code=~"5.."}[5m]))
                / sum(rate(http_requests_total[5m])) > 0.1
          for: 5m
          labels:
            severity: critical
            slo: error_rate
            service: node-app
          annotations:
            summary: "Error rate SLO breach — node-app"
            description: >
              HTTP error rate is {{ $value | printf "%.3f" }}%,
              exceeding the 0.1% SLO target.
            runbook_url: "https://wiki.internal/runbooks/error-rate-slo"

        # ── Error Budget Burn Rate (fast burn = early warning) ─────────────

        - alert: ErrorBudgetFastBurn
          expr: |
            (
              sum(rate(http_requests_total{status_code=~"5.."}[1h]))
              /
              sum(rate(http_requests_total[1h]))
            ) > (14.4 * 0.001)
          # 14.4x burn rate = exhausts monthly budget in 2 days
          for: 5m
          labels:
            severity: critical
            slo: availability
            service: node-app
          annotations:
            summary: "Error budget burning fast — node-app"
            description: >
              Current error rate is consuming the monthly error budget
              at 14.4x the safe rate. Budget will be exhausted in ~2 days.
```

```bash
kubectl apply -f prometheus-slo-alerts.yaml

# Verify rules loaded
kubectl get prometheusrule -n monitoring
```

---

### SLO Dashboard Panels (Grafana)

Add these panels to your dashboard to track SLO compliance:

```promql
# Panel: Availability SLO — 30-day rolling (Stat)
100 * sum(increase(http_requests_total{status_code!~"5.."}[30d]))
    / sum(increase(http_requests_total[30d]))
# Threshold: green ≥ 99.9, yellow ≥ 99, red < 99

# Panel: Error Budget Remaining % (Gauge)
100 * (
  1 - (
    sum(increase(http_requests_total{status_code=~"5.."}[30d]))
    / sum(increase(http_requests_total[30d]))
  ) / 0.001
)
# Threshold: green ≥ 50, yellow ≥ 10, red < 10

# Panel: p99 Latency vs SLO (Stat)
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
) * 1000
# Unit: ms, Threshold: green < 300, yellow < 500, red ≥ 500
```

---

## Quick Reference

```
Grafana alert lifecycle:
  Normal → Pending (for: duration) → Firing → Resolved

Contact Point = destination (Slack webhook, email address)
Notification Policy = routing rules (label matchers → contact point)

SLO formula:
  Error Budget = (1 - SLO%) × window duration
  Burn Rate    = current error rate / SLO error rate threshold
  Budget Left  = 1 - (actual errors / allowed errors)

RED method (services):        USE method (resources):
  Rate    — req/sec             Utilization — % busy
  Errors  — error rate          Saturation  — queue depth
  Duration — latency            Errors      — error count
```

| Tool | Stores data? | Primary strength |
|---|---|---|
| Prometheus | ✅ TSDB | Metrics collection + alerting |
| Grafana | ❌ Query only | Visualization + unified alerting |
| Alertmanager | ❌ Routes only | Alert deduplication + routing |
| Loki | ✅ Log index | Log aggregation (Grafana-native) |
