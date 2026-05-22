# Module 09 — Autoscaling & Resources

> **Track:** 🔴 Ops & Advanced | **Difficulty:** Intermediate

**Navigation:** [← Namespaces & RBAC](08-namespaces-and-rbac.md) | [Next: Health Checks →](10-health-checks.md)

---

## ❓ The Why

Your To-Do app has variable traffic patterns:

```
Users online:
Mon–Fri 9am–6pm:   ████████████  ~500 concurrent users
Weeknights:         ███           ~50 concurrent users
Weekends:           █             ~10 concurrent users
```

**Option A — Always run enough for peak:** 10 backend replicas 24/7.
Cost: you pay for 10 replicas at 3am Saturday when 2 users are online.

**Option B — Always run minimal:** 2 replicas.
Risk: 500 users hit 2 overloaded replicas, requests time out, users leave.

**Option C — Autoscale:** Start at 2 replicas, automatically grow to 10 at peak, shrink back to 2 overnight. Pay for what you use. Never get crushed by traffic spikes.

---

## 💡 The Concept

### Resource Requests and Limits (prerequisite)

Before autoscaling can work, every Pod must declare how many resources it needs. This is both a scheduling hint and a safety guard.

```
Node: 4 CPU, 8Gi RAM
├── Pod A: requests 1 CPU, 2Gi  ← scheduler guarantees this
│         limits   2 CPU, 3Gi  ← hard ceiling
├── Pod B: requests 1 CPU, 2Gi
└── Remaining: 2 CPU, 4Gi (available for scheduling)
```

| Setting | Meaning | What happens if exceeded |
|---------|---------|--------------------------|
| `requests.cpu` | Minimum CPU guaranteed | Nothing — it's a scheduling input |
| `limits.cpu` | Maximum CPU allowed | CPU is **throttled** (slowed, not killed) |
| `requests.memory` | Minimum memory guaranteed | Nothing — it's a scheduling input |
| `limits.memory` | Maximum memory allowed | Container is **OOMKilled** and restarted |

### Horizontal Pod Autoscaler (HPA)

The HPA watches a metric (CPU, memory, or custom) and adjusts the `replicas` count of a Deployment automatically.

```
Every 15 seconds:

Metrics Server
  │  "Backend avg CPU = 85%"
  │
  ▼
HPA Controller
  │  Target: 70% CPU
  │  Current: 85% → above target → scale UP
  │
  ▼
Deployment
  │  replicas: 3 → 5
  │
  ▼
Kubernetes Scheduler places 2 new Pods on available Nodes
```

Scale-down is conservative: the HPA waits **5 minutes** of sustained low usage before scaling down, to avoid flapping.

### HPA vs VPA vs KEDA

| Tool | What it scales | Best for |
|------|---------------|----------|
| **HPA** | Number of Pod replicas | Stateless services with variable load |
| **VPA** | CPU/memory of existing Pods | Single-instance workloads, right-sizing |
| **KEDA** | Replicas based on custom metrics | Queue depth, HTTP req/s, cron schedules |

---

## 📄 The Blueprint

### `yaml-examples/todo-hpa.yaml`

```yaml
# ── Updated Deployment with resource requests (required for HPA) ──────────────

apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-backend
spec:
  replicas: 2                  # HPA will override this dynamically
  selector:
    matchLabels:
      app: todo
      tier: backend
  template:
    metadata:
      labels:
        app: todo
        tier: backend
    spec:
      containers:
        - name: backend-api
          image: myrepo/todo-backend:v1.0.0
          ports:
            - containerPort: 3000
          resources:
            requests:          # REQUIRED for HPA to calculate utilisation
              memory: "128Mi"
              cpu: "100m"      # 0.1 CPU core
            limits:
              memory: "256Mi"
              cpu: "500m"      # 0.5 CPU core

---
# ── Horizontal Pod Autoscaler ────────────────────────────────────────────────

apiVersion: autoscaling/v2              # v2 supports multiple metrics
kind: HorizontalPodAutoscaler

metadata:
  name: todo-backend-hpa

spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: todo-backend              # The Deployment to autoscale

  minReplicas: 2                    # Never go below 2 (high availability — survive one Pod failure)
  maxReplicas: 10                   # Never exceed 10 (cost guard)

  metrics:
    # ── Metric 1: CPU utilisation ─────────────────────────────────────────
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # Scale up when average CPU across all Pods > 70%
                                    # Scale down when consistently < 70%

    # ── Metric 2: Memory utilisation ─────────────────────────────────────
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80    # Scale up when average memory > 80%

  behavior:                         # Fine-tune scale-up and scale-down behaviour
    scaleUp:
      stabilizationWindowSeconds: 30    # Wait 30s of high load before scaling up (avoid spikes)
      policies:
        - type: Pods
          value: 2                      # Add at most 2 Pods per scale-up event
          periodSeconds: 60

    scaleDown:
      stabilizationWindowSeconds: 300   # Wait 5 minutes of low load before scaling down
      policies:
        - type: Pods
          value: 1                      # Remove at most 1 Pod per scale-down event
          periodSeconds: 120            # Wait 2 minutes between each scale-down step

---
# ── KEDA ScaledObject example (custom metric: queue depth) ──────────────────
# Requires KEDA installed: helm install keda kedacore/keda -n kube-system

apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: todo-backend-keda
spec:
  scaleTargetRef:
    name: todo-backend
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
    - type: redis                       # Scale based on Redis queue length
      metadata:
        address: redis-svc:6379
        listName: todo-job-queue
        listLength: "10"               # 1 replica per 10 items in queue
```

### Resource Quotas Per Namespace (optional but recommended)

```yaml
# Prevent any one team from consuming all cluster resources

apiVersion: v1
kind: ResourceQuota
metadata:
  name: staging-quota
  namespace: staging
spec:
  hard:
    pods: "20"                         # Max 20 Pods in staging
    requests.cpu: "4"                  # Total CPU requests across all Pods
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    persistentvolumeclaims: "5"        # Max 5 PVCs
    services.loadbalancers: "1"        # Max 1 LoadBalancer Service
```

---

## ⌨️ The Commands

```bash
# ── PREREQUISITES ────────────────────────────────────────────────────────────

# Install Metrics Server (required for HPA to read CPU/memory)
# Minikube:
minikube addons enable metrics-server

# Production clusters (via Helm):
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm install metrics-server metrics-server/metrics-server -n kube-system

# Verify Metrics Server is running and collecting data
kubectl top nodes
kubectl top pods


# ── DEPLOY HPA ───────────────────────────────────────────────────────────────

kubectl apply -f yaml-examples/todo-hpa.yaml


# ── INSPECT HPA ─────────────────────────────────────────────────────────────

# Shows: TARGETS (current/target), MINPODS, MAXPODS, REPLICAS
kubectl get hpa
kubectl get hpa todo-backend-hpa

# Full detail including recent scaling events
kubectl describe hpa todo-backend-hpa

# Watch the HPA continuously (great during a load test)
kubectl get hpa --watch


# ── RESOURCE USAGE ──────────────────────────────────────────────────────────

# Current CPU/memory usage per Pod
kubectl top pods

# Current CPU/memory usage per Node
kubectl top nodes

# Usage for a specific namespace
kubectl top pods -n production


# ── LOAD TESTING ─────────────────────────────────────────────────────────────

# Generate load to trigger the HPA (in a separate terminal)
kubectl run load-gen --image=busybox --rm -it --restart=Never -- \
  sh -c 'while true; do wget -q -O- http://todo-backend-svc/health; done'

# Apache Bench (more controlled load)
kubectl run ab --image=httpd --rm -it --restart=Never -- \
  ab -n 50000 -c 100 http://todo-backend-svc/api/todos

# Watch replicas scale in real time (in the main terminal)
kubectl get hpa todo-backend-hpa --watch


# ── RESOURCE QUOTAS ──────────────────────────────────────────────────────────

kubectl get resourcequota -n staging
kubectl describe resourcequota staging-quota -n staging
```

---

## 🔧 The Drill

### Exercise: Observe Live Autoscaling

**Setup:**
```bash
# Ensure Metrics Server is running
kubectl top pods

# Deploy the HPA with minReplicas: 2, maxReplicas: 8, CPU target: 50%
kubectl apply -f yaml-examples/todo-hpa.yaml
```

**Step 1 — Baseline:**
```bash
kubectl get hpa todo-backend-hpa
# TARGETS should show something like 5%/50% and REPLICAS: 2
```

**Step 2 — Generate load (Terminal 2):**
```bash
kubectl run load-gen --image=busybox --rm -it --restart=Never -- \
  sh -c 'while true; do wget -q -O- http://todo-backend-svc/health > /dev/null; done'
```

**Step 3 — Watch autoscaling (Terminal 1):**
```bash
watch -n 5 kubectl get hpa todo-backend-hpa
```

**Step 4 — Observe scale-up** (should happen within 1–2 minutes of sustained high CPU).

**Step 5 — Stop the load generator** (Ctrl+C in Terminal 2).

**Step 6 — Observe scale-down** (happens after ~5 minutes due to the stabilization window).

**Questions to answer:**
- How many replicas did the HPA create at peak load?
- How long did it take to scale down after load stopped?
- What was the CPU target vs actual at peak?

---

### Troubleshooting: HPA Shows `<unknown>/70%`

```
NAME               TARGETS         MINPODS   MAXPODS   REPLICAS
todo-backend-hpa   <unknown>/70%   2         10        2
```

`<unknown>` means the HPA cannot read the current metric.

**Checklist:**
1. Is Metrics Server installed and running?
   ```bash
   kubectl get pods -n kube-system | grep metrics-server
   ```
2. Do your Pods have `resources.requests.cpu` defined?
   ```bash
   kubectl describe pod <todo-backend-pod> | grep -A5 Requests
   ```
3. Wait 2–3 minutes — Metrics Server needs time to collect initial data.

---

## ✅ Module Checklist

- [ ] Explain the difference between `requests` and `limits` for CPU and memory
- [ ] Install the Metrics Server and confirm `kubectl top pods` works
- [ ] Write an HPA targeting 70% CPU with min 2 and max 10 replicas
- [ ] Generate load and observe the HPA scale the Deployment up
- [ ] Explain why scale-down is slower than scale-up (stabilization window)

---

**Navigation:** [← Namespaces & RBAC](08-namespaces-and-rbac.md) | [Next: Health Checks →](10-health-checks.md)
