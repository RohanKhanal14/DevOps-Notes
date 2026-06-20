# Day 27 — Prometheus Metrics

> Architecture, scraping, Node.js instrumentation, PromQL

---

## Theory

### Monitoring vs Observability

These terms are related but distinct:

**Monitoring** is knowing *when* something is wrong — it answers "is the system healthy?" using predefined metrics and alerts.

**Observability** is understanding *why* something is wrong — it answers "what is the system doing?" using the three pillars:

| Pillar | What it captures | Tools |
|---|---|---|
| **Metrics** | Numeric measurements over time (counters, gauges) | Prometheus, Datadog |
| **Logs** | Discrete timestamped events with context | Loki, ELK Stack |
| **Traces** | Request journey across distributed services | Jaeger, Tempo, Zipkin |

Prometheus is a **metrics** system. It does not replace logs or traces — all three pillars are needed for full observability.

---

### Prometheus Architecture


![Alt text](/images/Prometheus/prometheus-architecture.gif)

#### Core Components

**Prometheus Server** — the central component. It:
- Scrapes (pulls) metrics from configured targets on a schedule
- Stores metrics in its built-in time-series database (TSDB)
- Evaluates alert rules
- Exposes a PromQL query API

**Exporters** — translate third-party metrics into the Prometheus format:
- `node-exporter` — host CPU, memory, disk, network
- `kube-state-metrics` — Kubernetes object states
- `blackbox-exporter` — HTTP/TCP probing
- `redis-exporter`, `postgres-exporter`, etc.

**Pushgateway** — for **short-lived jobs** (cron jobs, batch scripts) that finish before Prometheus can scrape them. The job *pushes* its metrics to the gateway, and Prometheus scrapes the gateway.

**Alertmanager** — receives firing alerts from Prometheus and handles:
- Routing to the right receiver (Slack, PagerDuty, email)
- Grouping (avoid alert storms)
- Silencing and inhibition

---

### Scrape Config, Jobs, and Targets

Prometheus is configured via `prometheus.yml`:

```yaml
global:
  scrape_interval: 15s        # How often to scrape targets
  evaluation_interval: 15s    # How often to evaluate alert rules

scrape_configs:
  # Job = logical group of targets
  - job_name: 'node-app'
    static_configs:
      - targets: ['localhost:3000']   # target = host:port
        labels:
          env: production

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']

  # Kubernetes service discovery
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: 'true'
```

**Key concepts:**
- A **job** is a collection of instances that serve the same purpose
- A **target** is a single endpoint Prometheus scrapes
- Prometheus adds automatic labels: `job`, `instance`
- Service discovery (Kubernetes, Consul, EC2) replaces static configs in dynamic environments

---

### Node.js: prom-client Metric Types

Install: `npm install prom-client`

There are four metric types. Choosing the right one matters for accurate PromQL queries.

#### Counter

A value that **only increases** (resets to 0 on restart). Use for: requests served, errors, bytes sent.

```javascript
const { Counter } = require('prom-client');

const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
});

// In your request handler:
httpRequestsTotal.inc({ method: 'GET', route: '/users', status_code: '200' });
```

#### Gauge

A value that **goes up and down**. Use for: active connections, memory usage, queue length, temperature.

```javascript
const { Gauge } = require('prom-client');

const activeConnections = new Gauge({
  name: 'active_connections',
  help: 'Number of active WebSocket connections',
});

activeConnections.inc();   // connection opened
activeConnections.dec();   // connection closed
activeConnections.set(42); // set absolute value
```

#### Histogram

Samples observations and counts them in **configurable buckets**. Automatically tracks `_count`, `_sum`, and `_bucket`. Use for: request duration, response size.

```javascript
const { Histogram } = require('prom-client');

const httpDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
});

// Time a request:
const end = httpDuration.startTimer({ method: 'GET', route: '/users' });
// ... handle request ...
end({ status_code: '200' });
```

#### Summary

Like a histogram but calculates **quantiles client-side** over a sliding time window. More accurate than histograms for quantiles but cannot be aggregated across instances. Prefer histograms in distributed systems.

```javascript
const { Summary } = require('prom-client');

const dbQueryDuration = new Summary({
  name: 'db_query_duration_seconds',
  help: 'Database query duration',
  percentiles: [0.5, 0.9, 0.95, 0.99],
});

const end = dbQueryDuration.startTimer();
await db.query('SELECT ...');
end();
```

#### Metric Type Cheat Sheet

| Type | Direction | Aggregatable | Use for |
|---|---|---|---|
| Counter | Up only | ✅ Yes | Requests, errors, events |
| Gauge | Up & down | ✅ Yes | Memory, connections, queue size |
| Histogram | Buckets | ✅ Yes | Latency, request size |
| Summary | Quantiles | ❌ No (across instances) | Single-instance latency |

---

#### Full Express.js Instrumentation Example

```javascript
// metrics.js
const client = require('prom-client');

// Collect default Node.js metrics (GC, event loop, memory)
client.collectDefaultMetrics({ prefix: 'nodejs_' });

const httpRequestsTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
});

const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
});

module.exports = { client, httpRequestsTotal, httpRequestDuration };
```

```javascript
// app.js
const express = require('express');
const { client, httpRequestsTotal, httpRequestDuration } = require('./metrics');

const app = express();

// Metrics middleware
app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer({
    method: req.method,
    route: req.path,
  });

  res.on('finish', () => {
    const labels = {
      method: req.method,
      route: req.path,
      status_code: res.statusCode.toString(),
    };
    httpRequestsTotal.inc(labels);
    end(labels);
  });

  next();
});

// Expose /metrics endpoint for Prometheus to scrape
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});

app.get('/users', (req, res) => {
  res.json([{ id: 1, name: 'Alice' }]);
});

app.listen(3000, () => console.log('App running on :3000'));
```

Prometheus scrapes `http://your-app:3000/metrics` every 15 seconds.

---

### PromQL

PromQL is Prometheus's query language for time-series data. All queries operate on **instant vectors**, **range vectors**, or **scalars**.

#### Key Functions

**`rate()`** — per-second average rate of increase over a time range. Use with counters. Handles counter resets.

```promql
# HTTP requests per second (last 5 minutes)
rate(http_requests_total[5m])
```

**`increase()`** — total increase in a counter over a time range. Equivalent to `rate() * duration`.

```promql
# Total requests in the last hour
increase(http_requests_total[1h])
```

**`histogram_quantile()`** — calculates a quantile (e.g., p99) from histogram buckets.

```promql
# p99 latency across all routes
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# p99 latency per route
histogram_quantile(0.99,
  sum by (route, le) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

**`sum()`, `avg()`, `max()`, `min()`** — aggregation operators.

```promql
# Total requests across all instances, grouped by route
sum by (route) (rate(http_requests_total[5m]))

# Average request rate per pod
avg by (pod) (rate(http_requests_total[5m]))
```

**`by` and `without`** — control which labels to keep in aggregation.

```promql
# Keep only 'route' label after summing
sum by (route) (http_requests_total)

# Drop 'instance' label, keep everything else
sum without (instance) (http_requests_total)
```

---

#### The Three Golden Queries

**1. HTTP Request Rate (requests per second)**

```promql
# All requests
sum(rate(http_requests_total[5m]))

# Per route
sum by (route) (rate(http_requests_total[5m]))
```

**2. p99 Latency**

```promql
histogram_quantile(0.99,
  sum by (le, route) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

**3. Error Rate (HTTP 5xx)**

```promql
# 5xx error rate as a percentage of total requests
sum(rate(http_requests_total{status_code=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100

# Absolute 500 errors per minute
sum(increase(http_requests_total{status_code="500"}[1m]))
```

---

## Practical Lab

### 1. Deploy Prometheus via Helm (kube-prometheus-stack) in Minikube

```bash
# Start Minikube
minikube start --memory=4096 --cpus=2

# Add the prometheus-community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack (Prometheus + Grafana + Alertmanager + node-exporter)
kubectl create namespace monitoring

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=admin123 \
  --set prometheus.prometheusSpec.scrapeInterval=15s

# Verify pods are running
kubectl get pods -n monitoring

# Access Prometheus UI
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090 &

# Access Grafana UI
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3001:80 &
# Login: admin / admin123
```

---

### 2. Instrument Node.js App with prom-client & Expose /metrics

Deploy the instrumented app to Minikube with the correct annotations so Prometheus auto-discovers it:

```yaml
# node-app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: node-app
  template:
    metadata:
      labels:
        app: node-app
      annotations:
        # These annotations enable Prometheus pod discovery
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: node-app
          image: my-nodejs-app:latest
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: node-app-svc
spec:
  selector:
    app: node-app
  ports:
    - port: 3000
      targetPort: 3000
```

```bash
kubectl apply -f node-app-deployment.yaml

# Verify /metrics is reachable
kubectl port-forward svc/node-app-svc 3000:3000 &
curl http://localhost:3000/metrics
```

You should see output like:
```
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",route="/users",status_code="200"} 42
...
```

---

### 3. Write PromQL Queries

Open the Prometheus UI at `http://localhost:9090` and run these queries:

```promql
# HTTP request rate (req/sec, last 5 min)
sum by (route) (rate(http_requests_total[5m]))

# p99 latency per route
histogram_quantile(0.99,
  sum by (le, route) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)

# HTTP 500 error rate (errors per second)
sum(rate(http_requests_total{status_code="500"}[5m]))

# Error rate as % of total traffic
100 * sum(rate(http_requests_total{status_code=~"5.."}[5m]))
     / sum(rate(http_requests_total[5m]))
```

---

## Assignment 27

> Write an alert rule that fires when the **HTTP 500 rate exceeds 5 per minute**.

#### Alert Rule File

```yaml
# alerts/http-alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: http-alert-rules
  namespace: monitoring
  labels:
    # Must match the ruleSelector in kube-prometheus-stack
    release: kube-prometheus-stack
spec:
  groups:
    - name: http.rules
      interval: 30s
      rules:
        # Alert: HTTP 500 rate > 5 per minute
        - alert: HighHTTP500Rate
          expr: |
            sum(increase(http_requests_total{status_code="500"}[1m])) > 5
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "High HTTP 500 error rate detected"
            description: >
              HTTP 500 errors have exceeded 5 per minute for over 2 minutes.
              Current value: {{ $value | humanize }} errors/min.

        # Bonus: Alert when error rate exceeds 5% of total traffic
        - alert: HighErrorRatePercent
          expr: |
            100 * sum(rate(http_requests_total{status_code=~"5.."}[5m]))
                / sum(rate(http_requests_total[5m])) > 5
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "HTTP error rate above 5%"
            description: >
              {{ $value | humanizePercentage }} of requests are returning 5xx errors.
```

#### Apply and Verify

```bash
kubectl apply -f alerts/http-alerts.yaml

# Confirm the rule was picked up by Prometheus
kubectl get prometheusrule -n monitoring

# Check in Prometheus UI: Status → Rules
# The alert should appear under the "http.rules" group
```

#### Trigger the Alert (for testing)

```bash
# Generate 500 errors on your app
for i in $(seq 1 20); do
  curl http://localhost:3000/error-endpoint
  sleep 1
done

# Watch alert state in Prometheus UI: Alerts tab
# States: inactive → pending (for: 2m) → firing
```

#### Route Alert to Slack via Alertmanager

```yaml
# alertmanager-config.yaml (add to Helm values)
alertmanager:
  config:
    global:
      slack_api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'

    route:
      group_by: ['alertname', 'severity']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 1h
      receiver: 'slack-notifications'
      routes:
        - match:
            severity: critical
          receiver: 'slack-notifications'

    receivers:
      - name: 'slack-notifications'
        slack_configs:
          - channel: '#alerts'
            title: '{{ .GroupLabels.alertname }}'
            text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
            send_resolved: true
```

---

## Quick Reference

```
Prometheus pull cycle:
  Every scrape_interval → GET /metrics → parse → store in TSDB

PromQL selector syntax:
  metric_name{label="value"}          exact match
  metric_name{label=~"5.."}          regex match
  metric_name{label!="value"}        not equal
  metric_name{label!~"4.."}         not regex

Time range selectors:
  metric[5m]    last 5 minutes (range vector)
  metric        current value  (instant vector)

Alert states:
  inactive → pending (for: duration) → firing → resolved
```

| PromQL Function | Input | Output | Use for |
|---|---|---|---|
| `rate(c[5m])` | Counter range vector | Per-second rate | Request rate |
| `increase(c[1m])` | Counter range vector | Total increase | Count in window |
| `histogram_quantile(0.99, h)` | Histogram | Quantile value | p99 latency |
| `sum by (label)` | Any vector | Aggregated vector | Group by label |
| `avg_over_time(g[5m])` | Gauge range vector | Average | Smoothed gauge |
