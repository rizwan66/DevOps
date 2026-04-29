# Module 6 — Monitoring with Prometheus, Grafana, Loki

> *"Implemented monitoring and alerting services using Prometheus, Grafana and Loki, significantly reducing issue detection and response time for high-usage applications."*

This module covers the three pillars of observability — metrics, logs, and (briefly) traces — and the specific stack mentioned in the job description.

## The three pillars and what they're for

Modern observability has three pillars. Each has a specialty.

| Pillar | What it captures | Best for | Cost characteristics |
|--------|------------------|----------|----------------------|
| **Metrics** | Numeric measurements over time | Trends, alerts, dashboards | Cheap to store, structured |
| **Logs** | Text events with timestamps | Debugging individual events | Expensive at scale, high cardinality |
| **Traces** | Request paths across services | Finding latency bottlenecks in distributed systems | Specialised storage |

A common mistake: trying to use one pillar for what another is good at. Examples:

- Using logs for alerting → expensive and slow.
- Using metrics for debugging individual requests → impossible (metrics aggregate).
- Using traces for high-frequency monitoring → too expensive to capture every request.

### What you'll typically use each for in production

- **Metrics**: "Is the system healthy right now?" "Is the error rate up?" "Is response time degraded?" "Has CPU spiked?" → Prometheus
- **Logs**: "What did request `xyz123` actually do?" "Why did this specific transaction fail?" "What was the stack trace at 3:42am?" → Loki
- **Traces**: "Why is this endpoint slow? Is the database the bottleneck, or is it the auth service?" → Tempo / Jaeger

## Prometheus

### How it works in one paragraph

Prometheus is a **pull-based** time-series database for metrics. Your applications expose metrics on an HTTP endpoint (typically `/metrics`). Prometheus periodically scrapes those endpoints and stores the data. You query it with PromQL, you alert on PromQL expressions, and you visualise it in Grafana.

The pull model has implications. Compared to push (StatsD, Datadog agents):

- **Service discovery is required**: Prometheus needs to know which endpoints to scrape. In Kubernetes, this is automatic via annotations or `ServiceMonitor` CRDs.
- **Short-lived jobs are awkward**: a batch job that runs for 30 seconds might not be scraped at all. Solution: a separate "Push Gateway" component, but use it sparingly.
- **You can't lose metrics in transit**: if Prometheus is down for 5 minutes, you have a 5-minute hole. Plan for HA.

### The four metric types

```
Counter   — only goes up (or resets to 0). Total requests, total errors.
Gauge     — goes up and down. Memory usage, active connections.
Histogram — distribution of values in buckets. Request duration p50/p95/p99.
Summary   — like histogram but quantiles computed client-side. Less common.
```

Choosing the right type matters. Common mistake: using a gauge for total requests, then trying to compute rate. Doesn't work — use a counter.

### Useful naming conventions

```
http_requests_total                      # counter, "_total" suffix
http_request_duration_seconds            # histogram, base SI units
node_memory_MemAvailable_bytes           # bytes, not megabytes
process_cpu_seconds_total                # seconds for time
```

Always use base SI units (seconds, bytes). Prometheus is opinionated about this; functions like `rate()` assume per-second.

### PromQL basics that you actually need

```promql
# Rate of requests over the last 5 minutes
rate(http_requests_total[5m])

# 95th percentile latency over 5 minutes
histogram_quantile(0.95, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# Error rate (errors / total)
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# CPU usage per pod
sum by (pod) (rate(container_cpu_usage_seconds_total[5m]))

# Memory usage as % of limit
sum by (pod) (container_memory_working_set_bytes) / sum by (pod) (kube_pod_container_resource_limits{resource="memory"})
```

### Recording rules vs alerting rules

PromQL queries can be expensive. **Recording rules** pre-compute frequently-used queries on a schedule:

```yaml
groups:
  - name: http_recording_rules
    interval: 30s
    rules:
      - record: job:http_requests:rate5m
        expr: sum by (job) (rate(http_requests_total[5m]))
```

Then your dashboards and alerts query `job:http_requests:rate5m` instead of recomputing the underlying expression. Big performance win for popular queries.

**Alerting rules** fire when a condition holds for a duration:

```yaml
groups:
  - name: http_alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          / sum(rate(http_requests_total[5m])) by (service)
          > 0.05
        for: 10m
        labels:
          severity: page
        annotations:
          summary: "{{ $labels.service }} has a high error rate"
          description: "Error rate is {{ $value | humanizePercentage }} for >10m"
```

The `for: 10m` is critical. It says: only fire if the condition is true for 10 continuous minutes. Without it, every transient blip pages your team. With too long a `for`, real incidents go undetected for too long.

## Grafana

Grafana is the visualisation layer. It connects to many data sources (Prometheus, Loki, Tempo, MySQL, CloudWatch, etc.) and produces dashboards and alerts.

### Dashboards: what good ones look like

A useful dashboard answers a *question*. "How is the payments service doing?" "Are users seeing login failures?" "Is the cluster running out of capacity?"

A useless dashboard shows everything. Most graphs go unread. The signal-to-noise ratio is the dashboard's quality metric.

Structure a dashboard with the **USE** or **RED** method:

- **USE** (for resources): Utilisation, Saturation, Errors. Good for hardware/infra.
- **RED** (for services): Rate (requests/sec), Errors (errors/sec), Duration (latency distribution). Good for application services.

Every service should have a RED dashboard. Every cluster should have a USE dashboard for its nodes.

### Variables and templating

Hard-coded service names in dashboards is a maintenance disaster. Use Grafana's **template variables**:

- A `$service` dropdown that reads from `label_values(http_requests_total, service)`.
- A `$environment` dropdown.
- Pass them into queries: `rate(http_requests_total{service="$service", env="$environment"}[5m])`.

One templated dashboard replaces 30 hard-coded ones.

### Dashboard-as-code

Don't click in Grafana to build dashboards if you're at scale. Either:

- Store dashboards as JSON in Git, and load them via Grafana's provisioning configuration.
- Generate them with [Grafonnet](https://github.com/grafana/grafonnet) (Jsonnet) or [grizzly](https://github.com/grafana/grizzly).

This makes dashboards versioned, reviewable, and reproducible across environments.

## Loki

Loki is a log aggregation system from Grafana Labs, designed to look like Prometheus for logs. The key insight: **don't index the log content; only index labels**. This makes Loki much cheaper than Elasticsearch-based systems for storage, at the cost of slower full-text search.

### Loki's mental model

In Loki, a log stream is identified by a set of **labels** (just like Prometheus metrics). Within a stream, log lines are unindexed — finding text inside them requires scanning, but Loki is fast at it because the label set is small.

Example query (LogQL):

```logql
{namespace="payments", container="api"} |= "ERROR" | json | duration > 500ms
```

This says: find all streams with these labels, filter to lines containing "ERROR", parse them as JSON, and keep only those where the parsed `duration` field is > 500ms.

### Critical: label cardinality

The biggest Loki performance footgun is too many labels.

- **Good labels**: namespace, app, container, level, env. Bounded set.
- **Bad labels**: user_id, request_id, ip_address. Unbounded. Each unique value creates a new stream. With 1 million users, you have 1 million streams.

Rule of thumb: if a label has > ~10,000 unique values, it shouldn't be a label. Put it in the log line itself and use LogQL parsing to filter on it.

### Promtail / Alloy / Vector

Loki doesn't ship logs to itself. You need an agent on each node:

- **Promtail** — Loki's original sidecar. Being deprecated.
- **Grafana Alloy** — the new unified agent for Grafana stack. Use for new deployments.
- **Vector** — third-party, very fast, supports many backends.
- **Fluent Bit** — common Kubernetes choice.

For new Kubernetes deployments, install the kube-prometheus-stack + Loki + Alloy via Helm and you have a working setup in 30 minutes.

## Alerting that doesn't get muted

The fastest way to make your alerting useless is to alert on too many things. The team mutes the channel and stops looking. Then real alerts get missed. This is the most common observability failure mode.

Rules for alerts that survive:

1. **Alert on symptoms, not causes.** "User can't log in" is a symptom. "Auth pod restarted" is a cause. The user doesn't care about the pod; they care about logging in.
2. **Every alert must be actionable.** When the alert fires, what does the on-call engineer do? If the answer is "wait and see", delete the alert.
3. **Use SLO-based alerting.** Define a Service Level Objective (e.g. "99.9% of requests return in <500ms") and alert when you're burning your error budget too fast. Tools: [Sloth](https://sloth.dev/), [Pyrra](https://github.com/pyrra-dev/pyrra).
4. **Severity tiers**: page (wake me up), warning (look at it tomorrow), info (just for context). Most alerts should be warning, not page.
5. **Link to runbooks.** Every alert annotation should include a link to "what to do when this fires".

### A quick taste of SLO-based alerting

```promql
# Burn rate alert: fire when we're consuming the 30-day error budget at a rate
# that would exhaust it in less than 1 hour.
(
  sum(rate(http_requests_total{status=~"5.."}[1h])) by (service)
  / sum(rate(http_requests_total[1h])) by (service)
) > 14.4 * (1 - 0.999)
```

The `14.4` is a magic number that comes from Google's SRE workbook. It corresponds to "burning at this rate for 1 hour exhausts the budget for 30 days at 99.9% SLO".

## Putting it together: kube-prometheus-stack

The standard way to install all of this on Kubernetes is the [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) Helm chart. It deploys:

- Prometheus
- Alertmanager
- Grafana with default dashboards
- node-exporter (per-node metrics)
- kube-state-metrics (cluster-state metrics)
- All the CRDs (`ServiceMonitor`, `PodMonitor`, `PrometheusRule`)
- All the standard alerts (etcd down, node not ready, etc.)

For Loki, install the `loki` and `loki-stack` charts.

## See `lab.md` to install the stack and write a useful alert.
