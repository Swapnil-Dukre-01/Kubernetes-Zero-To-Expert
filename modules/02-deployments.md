# Module 02 — Deployments

> **Track:** 🟣 Foundation | **Difficulty:** Beginner

**Navigation:** [← Pods & Nodes](01-pods-and-nodes.md) | [Next: Services →](03-services.md)

---

## ❓ The Why

In Module 01 we created raw Pods. Now delete your backend Pod:

```bash
kubectl delete pod todo-backend
```

It's gone. It **stays gone**. There is no automatic restart. Now imagine:

- Your backend crashes at 3am → app is down until someone manually recreates the Pod
- Traffic spikes → you have to write and apply new YAML files by hand to scale up
- You deploy v2 of your backend → you delete v1 and start v2, creating a gap of downtime

Raw Pods have three fatal production flaws:

| Problem | Impact |
|---------|--------|
| No self-healing | Crashed Pod = permanent outage |
| No scaling | More capacity = manual YAML work |
| No rolling updates | New version = downtime |

**Deployments solve all three.**

---

## 💡 The Concept

### Physical Analogy: A Staffing Agency Contract

A **Deployment** is a contract with a staffing agency:

> *"I always want exactly 3 backend workers on the floor. If one quits or gets sick, hire a replacement immediately. When I need to upgrade their skills, replace them one at a time so the service is never interrupted."*

The Deployment continuously compares your **desired state** against **actual state** and reconciles the difference. This is the core Kubernetes loop.

```
You declare:  "I want 3 backend Pods running image v1"
                            │
                            ▼
              ┌─────────────────────────┐
              │      Deployment         │
              │  desired replicas: 3    │
              └────────────┬────────────┘
                           │ manages
                           ▼
              ┌─────────────────────────┐
              │       ReplicaSet        │
              │  current replicas: 3    │
              └────────────┬────────────┘
                           │ owns
                    ┌──────┴──────┐
                    ▼      ▼      ▼
                  Pod-1  Pod-2  Pod-3
```

> **Note:** You never interact with ReplicaSets directly. Deployments manage them for you.

### Rolling Updates (Zero-Downtime Deploys)

When you change the image version, Kubernetes doesn't kill everything at once:

```
Before update:   [v1] [v1] [v1]

During update:   [v1] [v1] [v2]   ← one new Pod brought up
                 [v1] [v2] [v2]   ← another swapped
After update:    [v2] [v2] [v2]   ← done, zero downtime
```

This is controlled by two settings: `maxSurge` and `maxUnavailable`.

---

## 📄 The Blueprint

### `yaml-examples/todo-backend-deployment.yaml`

```yaml
# Deployment manifest for the To-Do backend API.
# This replaces the raw Pod from Module 01.

apiVersion: apps/v1        # Deployments live in the 'apps' API group
kind: Deployment

metadata:
  name: todo-backend
  labels:
    app: todo

spec:
  replicas: 3              # Run exactly 3 copies of the backend Pod at all times
                           # If one dies, Kubernetes immediately creates a replacement

  selector:                # How the Deployment FINDS the Pods it manages
    matchLabels:           # Must match the Pod template labels below — exactly
      app: todo
      tier: backend

  strategy:
    type: RollingUpdate    # Replace Pods gradually (not all at once)
    rollingUpdate:
      maxSurge: 1          # Allow 1 extra Pod during update (4 total briefly)
      maxUnavailable: 0    # Never let the ready count drop below 'replicas' value

  template:                # The Pod blueprint — same as a Pod spec, but nested here
    metadata:
      labels:
        app: todo
        tier: backend      # Must match selector.matchLabels above

    spec:
      containers:
        - name: backend-api
          image: myrepo/todo-backend:v1.0.0   # Replace with your actual image
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              value: "production"
            - name: MONGO_URI
              value: "mongodb://todo-mongo-svc:27017/todos"
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "500m"
```

### `yaml-examples/todo-frontend-deployment.yaml`

```yaml
# Deployment for the React frontend, served by Nginx

apiVersion: apps/v1
kind: Deployment

metadata:
  name: todo-frontend
  labels:
    app: todo

spec:
  replicas: 2              # Two frontend replicas for redundancy

  selector:
    matchLabels:
      app: todo
      tier: frontend

  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0

  template:
    metadata:
      labels:
        app: todo
        tier: frontend
    spec:
      containers:
        - name: frontend
          image: myrepo/todo-frontend:v1.0.0
          ports:
            - containerPort: 80    # Nginx serves on port 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "200m"
```

---

## ⌨️ The Commands

```bash
# ── DEPLOY ──────────────────────────────────────────────────────────────────

kubectl apply -f yaml-examples/todo-backend-deployment.yaml
kubectl apply -f yaml-examples/todo-frontend-deployment.yaml


# ── INSPECT ─────────────────────────────────────────────────────────────────

# Shows: NAME | READY | UP-TO-DATE | AVAILABLE | AGE
kubectl get deployments

# Deep-dive into the Deployment (events, rollout status, selector info)
kubectl describe deployment todo-backend

# See all Pods the Deployment created (filtered by label)
kubectl get pods -l app=todo,tier=backend

# Check the auto-created ReplicaSet underneath
kubectl get replicasets


# ── SCALING ─────────────────────────────────────────────────────────────────

# Scale manually (overrides the replicas field)
kubectl scale deployment todo-backend --replicas=5

# Scale back down
kubectl scale deployment todo-backend --replicas=3


# ── ROLLING UPDATES ─────────────────────────────────────────────────────────

# Update the container image (triggers a rolling update)
kubectl set image deployment/todo-backend backend-api=myrepo/todo-backend:v2.0.0

# Watch the rolling update happen live
kubectl rollout status deployment/todo-backend

# View the rollout history (each deploy creates a new revision)
kubectl rollout history deployment/todo-backend

# Rollback to the previous version
kubectl rollout undo deployment/todo-backend

# Rollback to a specific revision number
kubectl rollout undo deployment/todo-backend --to-revision=2

# Pause a rollout (useful to canary test mid-update)
kubectl rollout pause deployment/todo-backend

# Resume the paused rollout
kubectl rollout resume deployment/todo-backend


# ── EDIT LIVE ───────────────────────────────────────────────────────────────

# Open the deployment in your editor and save to apply changes
kubectl edit deployment todo-backend
```

---

## 🔧 The Drill

### Exercise: Prove Self-Healing

Do this live to see the Deployment's self-healing in action.

**Step 1 — Open two terminals side by side.**

Terminal 1 — watch Pods continuously:
```bash
kubectl get pods --watch
```

Terminal 2 — delete one of the running Pods:
```bash
# First, get a Pod name
kubectl get pods -l tier=backend

# Delete it by name
kubectl delete pod todo-backend-<hash>
```

**What you should see in Terminal 1:** Within 10–15 seconds, a new Pod appears and reaches `Running` state. The Deployment detected the desired count dropped from 3 to 2 and immediately created a replacement.

---

### Troubleshooting Scenario: Deployment Stuck in `Pending`

You apply the Deployment but see:

```
NAME           READY   UP-TO-DATE   AVAILABLE
todo-backend   0/3     3            0
```

All Pods show `Pending`. Work through the checklist:

1. `kubectl describe pod <pod-name>` → check Events for the reason
2. Is it `Insufficient memory` or `Insufficient cpu`? → Your Node doesn't have enough capacity. Lower the resource `requests` or add more Nodes.
3. Is it `no nodes are available that match all of the following predicates`? → Your `nodeSelector` or `tolerations` may be misconfigured.

<details>
<summary>💡 Hint</summary>

`Pending` almost always means the **scheduler can't find a suitable Node**. The Events section of `kubectl describe pod` will tell you exactly why.

</details>

---

## ✅ Module Checklist

- [ ] Understand the Deployment → ReplicaSet → Pod hierarchy
- [ ] Write a Deployment manifest with a `selector` and `template`
- [ ] Perform a rolling update and watch it with `kubectl rollout status`
- [ ] Roll back a bad deployment with `kubectl rollout undo`
- [ ] Demonstrate self-healing by deleting a Pod and watching it get recreated

---

**Navigation:** [← Pods & Nodes](01-pods-and-nodes.md) | [Next: Services →](03-services.md)
