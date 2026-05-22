# Module 10 — Health Checks & Probes

> **Track:** 🔴 Ops & Advanced | **Difficulty:** Intermediate

**Navigation:** [← Autoscaling](09-autoscaling.md) | [Next: Helm →](11-helm.md)

---

## ❓ The Why

A Pod can show `Running` and be completely broken at the same time.

**Scenario 1 — The Zombie Pod:**
Your Node.js backend starts. The process is alive. Kubernetes reports `Running`. But the backend failed to connect to MongoDB, so every single request returns `500 Internal Server Error`. Kubernetes happily routes 33% of user traffic to this broken Pod.

**Scenario 2 — The Premature Pod:**
You deploy a new version. The container starts and the process launches. Kubernetes immediately marks it ready and sends traffic. But Node.js takes 8 seconds to load the app, compile routes, and connect to the database. Every request during those 8 seconds fails.

**Scenario 3 — The Deadlocked Pod:**
Your backend has been running for 3 days. A memory leak causes it to deadlock — the process is alive but no longer processing requests. Kubernetes never knows. The Pod stays in rotation forever.

Probes are your solution to all three scenarios.

---

## 💡 The Concept

### The Three Probes

Kubernetes gives you three independent health check hooks:

```
Container lifecycle:
                                           ┌──────────────────────┐
  Container starts                         │    Startup Probe     │
       │                                   │                      │
       │   Is the app done starting?       │  Runs until success  │
       │──────────────────────────────────▶│  or failureThreshold │
       │                                   │  is hit              │
       │                                   └──────────┬───────────┘
       │                                              │ passes
       ▼                                              ▼
  ┌────────────────────────────┐    ┌─────────────────────────────────┐
  │     Liveness Probe         │    │       Readiness Probe           │
  │                            │    │                                 │
  │  Is the app alive?         │    │  Is the app ready for traffic?  │
  │  Fail → RESTART container  │    │  Fail → REMOVE from Service     │
  │  Runs forever              │    │  (but don't restart)            │
  └────────────────────────────┘    └─────────────────────────────────┘
```

| Probe | Failure Action | Use it to check |
|-------|---------------|-----------------|
| **Startup** | Kill container (like liveness) | App has finished initialising |
| **Liveness** | Kill and restart container | App is not deadlocked |
| **Readiness** | Remove from Service endpoints | All dependencies are healthy |

### Probe Mechanisms

| Mechanism | How it works | Best for |
|-----------|-------------|----------|
| `httpGet` | HTTP GET to a path; 2xx/3xx = healthy | HTTP services |
| `tcpSocket` | Opens TCP connection; success = healthy | TCP services (MongoDB, Redis) |
| `exec` | Runs a command inside container; exit 0 = healthy | Custom checks |
| `grpc` | gRPC health check protocol | gRPC services |

### What Your App Should Expose

```
GET /health   → 200 OK (always, as long as process is alive)
               Used by: liveness probe
               Checks: nothing external — just "am I running?"

GET /ready    → 200 OK (only when all dependencies are healthy)
               Used by: readiness probe
               Checks: MongoDB connected? Redis connected? Downstream APIs reachable?
```

---

## 📄 The Blueprint

### `yaml-examples/todo-probes.yaml`

```yaml
# Full Deployment for the backend with all three probes configured

apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-backend
spec:
  replicas: 3
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

          # ── Startup Probe ──────────────────────────────────────────────────
          # Gives slow-starting apps time to initialise before liveness takes over.
          # If this fails failureThreshold times, the container is killed and restarted.
          # While this probe is running, liveness and readiness are DISABLED.
          startupProbe:
            httpGet:
              path: /health          # Same endpoint as liveness — just "is the process up?"
              port: 3000
            initialDelaySeconds: 0   # Start checking immediately after container starts
            periodSeconds: 3         # Check every 3 seconds
            failureThreshold: 20     # Allow up to 20 failures = 60 seconds to start
            successThreshold: 1      # One success is enough to pass startup

          # ── Liveness Probe ────────────────────────────────────────────────
          # Checks whether the app is alive and not deadlocked.
          # Failure → Kubernetes kills the container and restarts it.
          # Keep this CHEAP — it runs every periodSeconds forever.
          livenessProbe:
            httpGet:
              path: /health          # Must return 2xx; should be instant
              port: 3000
            initialDelaySeconds: 0   # Startup probe handles the delay; set this to 0
            periodSeconds: 10        # Check every 10 seconds
            failureThreshold: 3      # Restart after 3 consecutive failures (30s of failure)
            successThreshold: 1      # 1 success clears the failure counter
            timeoutSeconds: 5        # Probe times out after 5 seconds (counts as failure)

          # ── Readiness Probe ───────────────────────────────────────────────
          # Checks whether the app is ready to serve traffic.
          # Failure → Pod is removed from Service endpoints (no traffic sent to it).
          # The container is NOT restarted — it stays Running but gets no traffic.
          # This is the right place to check external dependencies.
          readinessProbe:
            httpGet:
              path: /ready           # Checks: DB connected? Cache reachable? APIs up?
              port: 3000
            initialDelaySeconds: 0
            periodSeconds: 5         # Check more frequently than liveness (faster recovery)
            failureThreshold: 3      # Remove from rotation after 3 failures
            successThreshold: 2      # Require 2 consecutive successes to re-add to rotation
            timeoutSeconds: 3

          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"

---
# ── MongoDB TCP probe ────────────────────────────────────────────────────────
# MongoDB doesn't have an HTTP endpoint, so we use tcpSocket

apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-mongo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: todo
      tier: database
  template:
    metadata:
      labels:
        app: todo
        tier: database
    spec:
      containers:
        - name: mongodb
          image: mongo:6.0
          ports:
            - containerPort: 27017

          startupProbe:
            tcpSocket:
              port: 27017
            initialDelaySeconds: 5    # Give MongoDB a head start
            periodSeconds: 5
            failureThreshold: 12      # Allow 60 seconds for MongoDB to start

          livenessProbe:
            tcpSocket:
              port: 27017            # Just checks TCP connection is accepted
            periodSeconds: 10
            failureThreshold: 3

          readinessProbe:
            exec:
              command:               # Run a real MongoDB ping to verify it's serving queries
                - mongosh
                - --eval
                - "db.adminCommand('ping')"
            periodSeconds: 10
            failureThreshold: 3
            successThreshold: 1
```

### Health endpoint implementations (Node.js example)

```javascript
// In your Express backend — what /health and /ready should do

// Liveness: just confirm the process is alive. Keep it trivial.
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'alive', timestamp: Date.now() });
});

// Readiness: check every dependency your app needs to serve requests.
app.get('/ready', async (req, res) => {
  const checks = {};

  // Check MongoDB connection
  try {
    await mongoose.connection.db.admin().ping();
    checks.mongodb = 'ok';
  } catch (err) {
    checks.mongodb = 'error: ' + err.message;
  }

  // Check Redis (if you use it)
  try {
    await redisClient.ping();
    checks.redis = 'ok';
  } catch (err) {
    checks.redis = 'error: ' + err.message;
  }

  const allOk = Object.values(checks).every(v => v === 'ok');
  res.status(allOk ? 200 : 503).json({ status: allOk ? 'ready' : 'not ready', checks });
});
```

---

## ⌨️ The Commands

```bash
# ── DEPLOY ───────────────────────────────────────────────────────────────────

kubectl apply -f yaml-examples/todo-probes.yaml


# ── OBSERVE PROBE STATUS ────────────────────────────────────────────────────

# READY column: 1/1 = readiness probe passing, 0/1 = readiness probe failing
kubectl get pods

# RESTARTS column: rising count = liveness probe failing repeatedly
kubectl get pods -w

# Detailed probe failure events — always check "Events:" section
kubectl describe pod <pod-name>

# Example event output for a failing liveness probe:
# Warning  Unhealthy  Liveness probe failed: HTTP probe failed with statuscode: 500
# Warning  Killing    Container backend-api failed liveness probe, will be restarted


# ── LOGS FROM CRASHED CONTAINERS ─────────────────────────────────────────────

# If a Pod is in CrashLoopBackOff, get logs from the PREVIOUS container run
kubectl logs <pod-name> --previous
kubectl logs <pod-name> -c backend-api --previous


# ── SIMULATE PROBE FAILURE (for testing) ────────────────────────────────────

# Simulate a readiness failure: block the process inside the container
kubectl exec -it <pod-name> -- kill -STOP 1

# Watch the Pod get removed from Service endpoints
kubectl get endpoints todo-backend-svc --watch

# The Pod stays Running (liveness still passes) but disappears from endpoints
# Resume the process to restore readiness:
kubectl exec -it <pod-name> -- kill -CONT 1


# ── CHECK EXIT CODES ─────────────────────────────────────────────────────────

# Exit code 1    = application error (check app logs)
# Exit code 137  = OOMKilled (out of memory — increase memory limits)
# Exit code 143  = SIGTERM (graceful shutdown requested — normal during rolling update)

kubectl describe pod <pod-name> | grep "Exit Code"
```

---

## 🔧 The Drill

### Scenario: CrashLoopBackOff Investigation

You deploy a new version and see this after 3 minutes:

```
NAME                          READY   STATUS             RESTARTS   AGE
todo-backend-7d9f8b-xkq2p    0/1     CrashLoopBackOff   5          4m
```

**`CrashLoopBackOff` means:** The container starts, crashes, Kubernetes restarts it, it crashes again. The backoff time between restarts grows exponentially (10s, 20s, 40s, 80s, 160s, 300s…).

**Your investigation steps:**

1. **Get the crash logs:**
   ```bash
   kubectl logs todo-backend-7d9f8b-xkq2p --previous
   ```

2. **Check the exit code and last probe status:**
   ```bash
   kubectl describe pod todo-backend-7d9f8b-xkq2p
   # Look for: "Exit Code", "Last State", and "Events"
   ```

3. **Interpret what you find:**

   | Exit Code | Meaning | Next step |
   |-----------|---------|-----------|
   | `1` | App threw an unhandled error | Read the application logs |
   | `137` | OOMKilled — ran out of memory | Increase `limits.memory` |
   | `143` | SIGTERM — normal graceful shutdown | Check if something is killing it |
   | `2` | Misuse of shell built-in | Check container CMD / entrypoint |

4. **Fix and redeploy:**
   ```bash
   # If it's an OOMKill, patch the memory limit
   kubectl set resources deployment/todo-backend \
     -c backend-api --limits=memory=512Mi
   ```

---

### Design Challenge: What should `/ready` check?

Your backend connects to MongoDB and also calls an external payment API. Design your `/ready` endpoint:

- **Should it check MongoDB?** Yes — without the DB, the app cannot function.
- **Should it check the payment API?** Maybe — if all payment calls fail but your app still works for non-payment features, returning `503` removes you from rotation unnecessarily.

**Rule of thumb:** `/ready` should fail only when you genuinely cannot serve **any** useful traffic. Partial degradation is better handled with feature flags inside the app.

---

## ✅ Module Checklist

- [ ] Explain the difference between liveness, readiness, and startup probes
- [ ] Write all three probes for an HTTP Node.js service
- [ ] Write a TCP liveness probe for MongoDB
- [ ] Implement `/health` and `/ready` endpoints in a backend app
- [ ] Diagnose `CrashLoopBackOff` using `--previous` logs and exit codes

---

**Navigation:** [← Autoscaling](09-autoscaling.md) | [Next: Helm →](11-helm.md)
