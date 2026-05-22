# Module 13 — Observability: Logs, Metrics & Tracing

> **Track:** 🔴 Ops & Advanced | **Difficulty:** Advanced

**Navigation:** [← StatefulSets & DaemonSets](12-statefulsets-and-daemonsets.md) | [← Back to README](../README.md)

---

## ❓ The Why

Your To-Do app is running in production. At 2am, your on-call phone rings. Response times are slow, some requests are failing, but all Pods show `Running` and the HPA hasn't triggered. You have no idea where to look.

Without observability, you're debugging blind:
- SSHing into individual Pods and `grep`-ing logs manually
- Guessing which of your 10 backend replicas is misbehaving
- No historical data to see when the problem started
- No way to tell if it's the backend, MongoDB, or an external dependency

**Observability** is your production superpower. It consists of three pillars:

| Pillar | Answers | Tool |
|--------|---------|------|
| **Logs** | What happened and when? | Loki + Grafana |
| **Metrics** | How is the system performing over time? | Prometheus + Grafana |
| **Traces** | Where did this specific request spend its time? | Tempo / Jaeger |

---

## 💡 The Concept

### The Three Pillars

```
User request: POST /api/todos  → 2.3 seconds → 500 error

        LOGS                    METRICS                  TRACES
┌─────────────────────┐  ┌──────────────────────┐  ┌─────────────────────┐
│ 02:14:33 ERROR      │  │ Error rate: 12%      │  │ → Nginx: 1ms        │
│ MongoServerError:   │  │ P99 latency: 2.3s    │  │ → Backend: 2ms      │
│ connection timeout  │  │ Throughput: 450 rps  │  │ → MongoDB: 2290ms ← │
│                     │  │ CPU: 45%             │  │   SLOW              │
└─────────────────────┘  └──────────────────────┘  └─────────────────────┘

What: MongoDB timeout    How bad: 12% errors       Where: MongoDB query
```

### The Recommended Stack

```
                        ┌──────────────────────┐
                        │       Grafana        │  ← Single pane of glass
                        │  (all three pillars) │
                        └──────┬───────┬───────┘
                               │       │
                    ┌──────────┘       └──────────┐
                    ▼                              ▼
         ┌─────────────────┐            ┌─────────────────┐
         │   Prometheus    │            │      Loki        │
         │   (metrics)     │            │     (logs)       │
         └────────┬────────┘            └────────┬─────────┘
                  │                              │
         ┌────────┘                    ┌─────────┘
         ▼                             ▼
┌─────────────────┐          ┌─────────────────────┐
│ Metrics Server  │          │  Fluent Bit          │
│ node-exporter   │          │  (DaemonSet from     │
│ kube-state-     │          │   Module 12)         │
│ metrics         │          └─────────────────────┘
└─────────────────┘
```

**Golden Signals** — the four metrics that matter most for any service:

| Signal | Metric | Alert threshold |
|--------|--------|-----------------|
| **Latency** | P99 request duration | > 500ms |
| **Traffic** | Requests per second | Sudden drop or spike |
| **Errors** | HTTP 5xx rate | > 1% |
| **Saturation** | CPU/memory utilisation | > 80% |

---

## 📄 The Blueprint

### Install the Full Observability Stack

```yaml
# values-monitoring.yaml — passed to kube-prometheus-stack Helm chart

grafana:
  enabled: true
  adminPassword: "changeme123"    # Change this!
  persistence:
    enabled: true
    size: 5Gi

prometheus:
  prometheusSpec:
    retention: 15d                # Keep 15 days of metrics
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: standard
          resources:
            requests:
              storage: 20Gi

alertmanager:
  enabled: true
  config:
    global:
      slack_api_url: "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
    receivers:
      - name: slack-alerts
        slack_configs:
          - channel: "#alerts"
            title: "{{ .GroupLabels.alertname }}"
            text: "{{ range .Alerts }}{{ .Annotations.description }}{{ end }}"
    route:
      receiver: slack-alerts

kubeStateMetrics:
  enabled: true

nodeExporter:
  enabled: true
```

### `yaml-examples/todo-monitoring.yaml`

```yaml
# ── ServiceMonitor: Tell Prometheus to scrape the backend ───────────────────
# Requires kube-prometheus-stack (it installs the ServiceMonitor CRD)

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: todo-backend-monitor
  namespace: monitoring            # Must be in the same namespace as Prometheus
  labels:
    release: monitoring            # Must match Prometheus's serviceMonitorSelector label

spec:
  selector:
    matchLabels:
      app: todo
      tier: backend                # Selects our todo-backend-svc

  namespaceSelector:
    matchNames:
      - production                 # Which namespace to look in

  endpoints:
    - port: http                   # The named port on the Service
      path: /metrics               # Your app must expose Prometheus metrics here
      interval: 15s                # Scrape every 15 seconds

---
# ── PrometheusRule: Alert when error rate is too high ───────────────────────

apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: todo-alerts
  namespace: monitoring
  labels:
    release: monitoring

spec:
  groups:
    - name: todo-app-alerts
      interval: 30s                # Evaluate every 30 seconds

      rules:
        # Alert 1: High error rate
        - alert: HighErrorRate
          expr: |
            rate(http_requests_total{job="todo-backend", status=~"5.."}[5m])
            /
            rate(http_requests_total{job="todo-backend"}[5m])
            > 0.05
          for: 2m                  # Must be true for 2 continuous minutes
          labels:
            severity: critical
            team: todo-team
          annotations:
            summary: "High error rate on todo backend"
            description: "Error rate is {{ humanizePercentage $value }} over the last 5 minutes"
            runbook: "https://wiki.example.com/runbooks/todo-high-error-rate"

        # Alert 2: High P99 latency
        - alert: HighLatency
          expr: |
            histogram_quantile(0.99,
              rate(http_request_duration_seconds_bucket{job="todo-backend"}[5m])
            ) > 0.5
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "P99 latency above 500ms"
            description: "P99 latency is {{ humanizeDuration $value }}"

        # Alert 3: Pod keeps restarting
        - alert: PodCrashLooping
          expr: |
            increase(kube_pod_container_status_restarts_total{
              namespace="production"
            }[15m]) > 3
          for: 0m
          labels:
            severity: critical
          annotations:
            summary: "Pod {{ $labels.pod }} is crash looping"
            description: "Container {{ $labels.container }} has restarted more than 3 times in 15 minutes"

        # Alert 4: PVC running out of space
        - alert: PVCAlmostFull
          expr: |
            kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.85
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "PVC {{ $labels.persistentvolumeclaim }} is 85% full"
```

### Exposing Prometheus Metrics from Node.js

```javascript
// Install: npm install prom-client

const promClient = require('prom-client');

// Enable default metrics (CPU, memory, event loop, GC)
promClient.collectDefaultMetrics({ prefix: 'todo_backend_' });

// Custom metrics for your app
const httpRequestCounter = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status'],
});

const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
});

// Middleware to record every request
app.use((req, res, next) => {
  const end = httpRequestDuration.startTimer();
  res.on('finish', () => {
    httpRequestCounter.inc({
      method: req.method,
      route: req.route?.path || req.path,
      status: res.statusCode,
    });
    end({ method: req.method, route: req.route?.path || req.path, status: res.statusCode });
  });
  next();
});

// Expose the /metrics endpoint for Prometheus to scrape
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', promClient.register.contentType);
  res.send(await promClient.register.metrics());
});
```

### Loki + Fluent Bit Configuration

```yaml
# ConfigMap for Fluent Bit — ships Pod logs to Loki

apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: kube-system
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush        5
        Log_Level    info
        Parsers_File parsers.conf

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            docker
        Tag               kube.*
        Refresh_Interval  5
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Merge_Log           On
        Keep_Log            Off
        K8S-Logging.Parser  On

    [OUTPUT]
        Name            loki
        Match           kube.*
        Host            loki.monitoring.svc.cluster.local
        Port            3100
        Labels          job=fluent-bit, namespace=$kubernetes['namespace_name'], pod=$kubernetes['pod_name']
        Auto_Kubernetes_Labels On
```

---

## ⌨️ The Commands

```bash
# ── INSTALL THE FULL STACK ───────────────────────────────────────────────────

# kube-prometheus-stack: Prometheus + Grafana + Alertmanager + node-exporter
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f values-monitoring.yaml

# Install Loki (log aggregation)
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set fluent-bit.enabled=true \
  --set grafana.enabled=false    # Already installed above


# ── ACCESS DASHBOARDS ────────────────────────────────────────────────────────

# Grafana (admin / changeme123)
kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring

# Prometheus UI
kubectl port-forward svc/monitoring-kube-prometheus-prometheus 9090:9090 -n monitoring

# Alertmanager UI
kubectl port-forward svc/monitoring-kube-prometheus-alertmanager 9093:9093 -n monitoring


# ── VERIFY SCRAPING ──────────────────────────────────────────────────────────

# In Prometheus UI: Status → Targets
# Your todo-backend should appear as "UP"

# Apply the ServiceMonitor
kubectl apply -f yaml-examples/todo-monitoring.yaml

# Check ServiceMonitor was picked up
kubectl get servicemonitor -n monitoring


# ── QUERY METRICS (PromQL) ────────────────────────────────────────────────────

# Request rate (requests per second over last 5 min)
# rate(http_requests_total{job="todo-backend"}[5m])

# Error rate (%)
# rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) * 100

# P99 latency
# histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Memory usage per Pod
# container_memory_working_set_bytes{namespace="production", container="backend-api"}

# CPU usage per Pod
# rate(container_cpu_usage_seconds_total{namespace="production", container="backend-api"}[5m])


# ── QUERY LOGS (LogQL in Grafana/Loki) ──────────────────────────────────────

# All logs from production namespace
# {namespace="production"}

# Only error logs from the backend
# {namespace="production", container="backend-api"} |= "ERROR"

# Log rate of errors over time
# rate({namespace="production"} |= "ERROR" [5m])

# Parse JSON logs and filter
# {namespace="production"} | json | level="error" | duration > 1s


# ── KUBECTL LOG COMMANDS ──────────────────────────────────────────────────────

# Stream logs from ALL backend Pods at once
kubectl logs -l app=todo,tier=backend --all-containers -f

# Stream logs from all Pods in production namespace
kubectl logs -n production -l app=todo -f --max-log-requests=10

# Previous container logs (after a crash)
kubectl logs <pod-name> --previous


# ── APPLY ALERTS ────────────────────────────────────────────────────────────

kubectl apply -f yaml-examples/todo-monitoring.yaml

# Check alert rules loaded correctly
kubectl get prometheusrule -n monitoring

# See firing alerts
kubectl port-forward svc/monitoring-kube-prometheus-alertmanager 9093 -n monitoring
# Visit: http://localhost:9093
```

---

## 🔧 The Drill

### Exercise: Full Incident Simulation

**Goal:** Experience the complete observability loop — from alert to resolution.

**Setup:**
```bash
# Ensure the monitoring stack is running
kubectl get pods -n monitoring

# Deploy a "bad" version of the backend that returns 500 for 20% of requests
# (in your backend code: if (Math.random() < 0.2) throw new Error('simulated error'))
kubectl set image deployment/todo-backend backend-api=myrepo/todo-backend:chaos-v1 -n production
```

**Step 1 — Watch the golden signals light up:**
- Open Grafana: `kubectl port-forward svc/monitoring-grafana 3000:80 -n monitoring`
- Import dashboard ID **12740** (Kubernetes App Metrics)
- Watch the error rate climb past 5%

**Step 2 — Alert fires:**
- After 2 minutes, the `HighErrorRate` alert should fire
- Check Alertmanager: it routes to your Slack channel

**Step 3 — Diagnose with logs:**
```
In Grafana → Explore → Loki:
{namespace="production", container="backend-api"} |= "ERROR" | json
```

**Step 4 — Diagnose with traces** (if you have Tempo installed):
- Find a slow request trace
- See which span (backend vs MongoDB) took the most time

**Step 5 — Fix and verify:**
```bash
# Roll back to the good version
kubectl rollout undo deployment/todo-backend -n production

# Watch the error rate drop in Grafana
# Confirm the alert resolves in Alertmanager
```

---

### Key Grafana Dashboards to Import

| Dashboard ID | Name | What it shows |
|-------------|------|---------------|
| 315 | Kubernetes cluster monitoring | Node CPU, memory, network |
| 12740 | Kubernetes App Metrics | Per-Deployment golden signals |
| 13639 | Logs / Loki | Log volume and error rate |
| 11074 | Node Exporter Full | Detailed Node-level metrics |

---

## ✅ Module Checklist

- [ ] Install kube-prometheus-stack and access Grafana
- [ ] Write a `ServiceMonitor` to add your backend as a Prometheus scrape target
- [ ] Expose `/metrics` from your Node.js backend using `prom-client`
- [ ] Write a `PrometheusRule` that alerts on >5% error rate
- [ ] Query logs in Loki using LogQL
- [ ] Complete the incident simulation exercise from start to resolution

---

## 🎓 Congratulations — You've Completed the Curriculum!

You now understand the complete lifecycle of a production Kubernetes application:

```
Foundation          Networking          Storage             Ops & Advanced
───────────         ──────────          ───────             ──────────────
✅ Pods             ✅ Ingress          ✅ PVC              ✅ RBAC
✅ Deployments      ✅ DNS              ✅ StorageClass      ✅ Autoscaling
✅ Services                                                 ✅ Health Checks
✅ ConfigMaps                                               ✅ Helm
                                                            ✅ StatefulSets
                                                            ✅ Observability
```

### What's Next?

- **CKA (Certified Kubernetes Administrator)** — [kubernetes.io/training](https://kubernetes.io/training/)
- **CKAD (Certified Kubernetes Application Developer)** — focuses on Modules 1–10
- **GitOps with ArgoCD** — declarative, Git-driven deployments
- **Service Mesh with Istio** — mTLS, traffic management, advanced observability
- **Cluster API** — manage Kubernetes clusters with Kubernetes
- **eBPF with Cilium** — next-generation networking and security

---

**Navigation:** [← StatefulSets & DaemonSets](12-statefulsets-and-daemonsets.md) | [← Back to README](../README.md)
