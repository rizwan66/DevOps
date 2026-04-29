# Module 6 — Lab: a complete monitoring stack

You'll install Prometheus, Grafana, and Loki on the kind cluster you used in module 4, instrument an application, write a meaningful alert, and prove that the alert fires when (and only when) it should.

## Prerequisites

- The kind cluster from module 4 (or a fresh one)
- `helm` installed
- The payments-svc app from module 2

## Step 1 — Install kube-prometheus-stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

helm install kps prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=admin \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
```

Verify:
```bash
kubectl -n monitoring get pods
kubectl -n monitoring get servicemonitor
```

Port-forward Grafana:
```bash
kubectl -n monitoring port-forward svc/kps-grafana 3000:80
```

Open http://localhost:3000, log in with admin/admin. You'll see pre-built dashboards under "Dashboards → Default" — the Kubernetes / cluster ones are excellent.

## Step 2 — Instrument the payments-svc

Update `app/main.py` to expose Prometheus metrics:

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST
from fastapi.responses import Response
import time

app = FastAPI()

requests_total = Counter(
    "http_requests_total",
    "Total HTTP requests",
    ["method", "endpoint", "status"]
)

request_duration = Histogram(
    "http_request_duration_seconds",
    "HTTP request latency in seconds",
    ["method", "endpoint"]
)

class PaymentRequest(BaseModel):
    amount: float
    currency: str

@app.middleware("http")
async def metrics_middleware(request, call_next):
    start = time.time()
    response = await call_next(request)
    duration = time.time() - start
    requests_total.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()
    request_duration.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(duration)
    return response

@app.get("/metrics")
def metrics():
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)

@app.get("/healthz")
def healthz():
    return {"status": "ok"}

@app.post("/charge")
def charge(req: PaymentRequest):
    if req.amount <= 0:
        raise HTTPException(status_code=400, detail="amount must be positive")
    if req.currency not in {"USD", "EUR", "GBP"}:
        raise HTTPException(status_code=400, detail="unsupported currency")
    return {"status": "charged", "id": "txn_abc123"}
```

Add `prometheus-client==0.21.0` to `requirements.txt`. Rebuild the image, push it, deploy it.

## Step 3 — Tell Prometheus to scrape it

Create a `ServiceMonitor`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payments-svc
  namespace: default
  labels:
    app: payments-svc
spec:
  selector:
    app: payments-svc
  ports:
    - name: http
      port: 8000
      targetPort: 8000
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: payments-svc
  namespace: default
  labels:
    release: kps  # must match the kube-prometheus-stack Helm release label
spec:
  selector:
    matchLabels:
      app: payments-svc
  endpoints:
    - port: http
      interval: 15s
      path: /metrics
```

Apply both. After ~30 seconds, in Prometheus (port-forward `kps-kube-prometheus-stack-prometheus:9090`) check **Status → Targets**. You should see `payments-svc` listed and "UP".

Query in Prometheus:
```
http_requests_total
```

Generate some traffic:
```bash
kubectl run loadgen --rm -it --image=curlimages/curl -- sh
# inside the pod:
while true; do
  curl -s http://payments-svc.default:8000/healthz > /dev/null
  curl -s -X POST http://payments-svc.default:8000/charge \
    -H 'content-type: application/json' \
    -d '{"amount":10,"currency":"USD"}' > /dev/null
  sleep 0.1
done
```

After a minute, query:
```
rate(http_requests_total[1m])
```

You should see request rate per second.

## Step 4 — Build a RED dashboard

In Grafana, create a new dashboard. Add three panels:

**Rate panel:**
```promql
sum by (endpoint) (rate(http_requests_total{job="payments-svc"}[5m]))
```

**Errors panel:**
```promql
sum by (endpoint) (rate(http_requests_total{job="payments-svc", status=~"5.."}[5m]))
```

**Duration panel (p95):**
```promql
histogram_quantile(0.95,
  sum by (endpoint, le) (rate(http_request_duration_seconds_bucket{job="payments-svc"}[5m]))
)
```

Set the dashboard refresh to 10s. Watch it update. This is the most useful dashboard you'll build for any service.

## Step 5 — Write an alerting rule

Now write an alert that fires only on real problems.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: payments-svc-alerts
  namespace: default
  labels:
    release: kps
spec:
  groups:
    - name: payments-svc
      rules:
        - alert: PaymentsHighErrorRate
          expr: |
            (
              sum(rate(http_requests_total{job="payments-svc", status=~"5.."}[5m]))
              /
              sum(rate(http_requests_total{job="payments-svc"}[5m]))
            ) > 0.05
          for: 10m
          labels:
            severity: page
            service: payments
          annotations:
            summary: "Payments service error rate is above 5%"
            description: "Error rate is {{ $value | humanizePercentage }} over the last 5m"
            runbook: "https://wiki.example.com/runbooks/payments-high-error-rate"

        - alert: PaymentsHighLatency
          expr: |
            histogram_quantile(0.95,
              sum by (le) (rate(http_request_duration_seconds_bucket{job="payments-svc"}[5m]))
            ) > 1
          for: 15m
          labels:
            severity: warning
            service: payments
          annotations:
            summary: "Payments p95 latency above 1s"
            description: "p95 latency is {{ $value }}s over the last 5m"
```

Apply. Check in Prometheus's UI under **Alerts** that both rules are listed.

## Step 6 — Test the alert intentionally

Modify the app to deliberately fail 10% of `/charge` requests:

```python
import random
@app.post("/charge")
def charge(req: PaymentRequest):
    if random.random() < 0.1:
        raise HTTPException(status_code=500, detail="injected failure")
    if req.amount <= 0:
        ...
```

Redeploy. Run your traffic generator. Watch the error rate climb in Grafana. After ~10 minutes (because of the `for:` clause), the alert should fire.

You can speed this up by editing the `for:` to `for: 1m` for testing, then putting it back to `10m` afterwards.

## Step 7 — Add Loki

```bash
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=false \
  --set loki.persistence.enabled=false
```

In Grafana, **Configuration → Data Sources → Add → Loki**, URL `http://loki:3100`. Test connection.

Open the **Explore** view, switch to Loki, and run:
```logql
{namespace="default", app="payments-svc"}
```

You should see logs from your pod streaming.

Now run the failure-injection scenario again, but query for errors:
```logql
{namespace="default", app="payments-svc"} |= "500"
```

See the actual error log lines appear. **This is the moment metrics + logs become much more powerful together than either alone.** You can see the metric (error rate going up) AND the specific log lines causing it.

## Step 8 — Correlate metrics and logs in a dashboard

Add a logs panel to your dashboard alongside the RED panels. Set the time range to "last 15 minutes". Now when you see an error spike on the metrics graph, the log lines next to it tell you why.

This single dashboard answers most "what's happening right now?" questions.

## Tasks

1. **Add an SLO**. Define an SLO of 99.9% for `/charge` success rate. Compute the error budget. Write a multi-window, multi-burn-rate alert (look up Google's SRE workbook chapter on SLO alerting for the exact formulas).
2. **Write a deliberately bad alert.** Make one that pages on every transient error. Watch the channel get spammed. Then tune `for:` and the threshold until it only pages on real issues. This is calibration practice — no amount of theory teaches it as well as feeling the pain once.
3. **Investigate label cardinality.** Add a label to your metrics that has high cardinality (e.g. `request_id`). Watch Prometheus's memory usage. Now remove it. Note the difference.
4. **Set up Alertmanager routing.** Configure alerts to go to different Slack channels based on `severity`. Page-severity to `#oncall`; warnings to `#payments-monitoring`.

## Reflection

1. Your developer says: "Just add a metric for everything. Disk is cheap." What's wrong with that argument?
2. Why does Prometheus pull instead of push? Why might you sometimes wish it pushed? (Hint: think about workloads behind firewalls.)
3. The `for: 10m` on your alert means real incidents are detected ~10 minutes late. Is that acceptable? When isn't it? What do you do for the cases where it isn't?
4. Loki is "much cheaper than Elasticsearch". What did you give up to get the cost savings? When would you choose ES anyway?
