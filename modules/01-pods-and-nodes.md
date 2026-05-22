# Module 01 — Pods & Nodes

> **Track:** 🟣 Foundation | **Difficulty:** Beginner

**Navigation:** [← Back to README](../README.md) | [Next: Deployments →](02-deployments.md)

---

## ❓ The Why

Imagine your To-Do app is running — a React frontend, a Node.js backend, and MongoDB — all started manually with `node server.js` and `mongod`. It works on your laptop. But consider these problems:

- **What happens when your backend crashes at 3am?** It stays dead. Nobody restarts it.
- **What if you need 3 copies of the backend for load?** You'd manage 3 terminal windows manually.
- **Deploying a new version** means stopping the old one and starting the new one — downtime every time.
- **Moving to a real server** means SSH-ing in and setting everything up by hand.

Kubernetes solves all of this. It is a **container orchestrator** — a system that takes your Docker containers and intelligently runs, restarts, scales, and manages them across a fleet of machines. To understand how, we need its two most fundamental concepts: **Nodes** and **Pods**.

---

## 💡 The Concept

### Physical Analogy: A Shipping Port

Think of your entire Kubernetes setup as a **shipping port**:

```
┌─────────────────────────────── Kubernetes Cluster (The Port) ──────────────────────────────┐
│                                                                                              │
│  ┌────────────────────┐    ┌──────────────────────────┐    ┌──────────────────────────┐    │
│  │  Control Plane     │    │  Worker Node 1           │    │  Worker Node 2           │    │
│  │  (Management       │    │  (Warehouse A)           │    │  (Warehouse B)           │    │
│  │   Office)          │    │                          │    │                          │    │
│  │                    │    │  ┌────────────────────┐  │    │  ┌────────────────────┐  │    │
│  │  • API Server      │───▶│  │ Pod: frontend      │  │    │  │ Pod: mongodb       │  │    │
│  │  • Scheduler       │    │  │ (React container)  │  │    │  │ (Mongo container)  │  │    │
│  │  • etcd            │    │  └────────────────────┘  │    │  └────────────────────┘  │    │
│  │  • Controller Mgr  │    │  ┌────────────────────┐  │    │                          │    │
│  └────────────────────┘    │  │ Pod: backend       │  │    │  kubelet, kube-proxy     │    │
│                             │  │ (Node.js container)│  │    │  container runtime       │    │
│                             │  └────────────────────┘  │    └──────────────────────────┘    │
│                             └──────────────────────────┘                                    │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

- The **port** = your Kubernetes **Cluster**
- Each **warehouse** = a **Node** — a physical or virtual machine with CPU and RAM
- Each **shipping container** inside a warehouse = a **Pod** — the smallest deployable unit in Kubernetes

### What is a Node?

A Node is simply a server (physical or cloud VM). Every Kubernetes cluster has two types:

| Type | Role |
|------|------|
| **Control Plane** | The brain — schedules Pods, watches state, exposes the API |
| **Worker Node** | The muscle — actually runs your application Pods |

### What is a Pod?

A Pod is a **wrapper around one (or more) Docker containers**. It is the atom of Kubernetes — everything else manages groups of Pods.

> **Critical rule:** You never run a Docker container directly in Kubernetes. You always wrap it in a Pod.

Key Pod properties:

- Has its own **cluster-internal IP address** (e.g. `10.244.1.5`)
- Containers within the same Pod share that IP and talk via `localhost`
- Pods are **ephemeral** — when a Pod dies, it's gone. Its IP is recycled.
- Defined by a **YAML manifest** you hand to Kubernetes

---

## 📄 The Blueprint

### `yaml-examples/todo-backend-pod.yaml`

```yaml
# A Pod manifest for our To-Do List backend API.
# Save as: yaml-examples/todo-backend-pod.yaml

apiVersion: v1          # The Kubernetes API version for this object type
kind: Pod               # We are defining a Pod (not a Deployment, Service, etc.)

metadata:               # Data ABOUT the Pod (not what runs inside it)
  name: todo-backend    # The unique name of this Pod in the cluster
  labels:               # Key-value tags — like luggage labels on a shipping container
    app: todo           # Tags this as part of the 'todo' application
    tier: backend       # And specifically the 'backend' tier
                        # Labels are critical — Services and Deployments use them
                        # to find and target this Pod

spec:                   # The SPECIFICATION — what actually runs inside this Pod
  containers:           # A list of containers (usually just one per Pod)
    - name: backend-api           # A name for this container (used in logs)
      image: node:18-alpine       # The Docker image to run
                                  # In production: "myrepo/todo-backend:v1.2.0"
      ports:
        - containerPort: 3000     # The port your Node.js app listens on INSIDE the container
                                  # This is documentation — it doesn't open/close firewall ports
      env:                        # Environment variables injected into the container
        - name: NODE_ENV
          value: "production"
        - name: MONGO_URI         # How our backend finds MongoDB
          value: "mongodb://todo-mongo:27017/todos"
      resources:                  # Resource guardrails for this container
        requests:                 # MINIMUM resources the scheduler guarantees
          memory: "128Mi"         # 128 Mebibytes of RAM
          cpu: "100m"             # 100 millicores = 0.1 of one CPU core
        limits:                   # MAXIMUM allowed before action is taken
          memory: "256Mi"         # Exceed this → container is killed and restarted
          cpu: "500m"             # Exceed this → CPU is throttled (not killed)
```

### `yaml-examples/todo-mongo-pod.yaml`

```yaml
# Pod manifest for our MongoDB database.
# Save as: yaml-examples/todo-mongo-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: todo-mongo
  labels:
    app: todo
    tier: database
spec:
  containers:
    - name: mongodb
      image: mongo:6.0              # Official MongoDB 6.0 image from Docker Hub
      ports:
        - containerPort: 27017      # MongoDB's default port
      env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: "admin"
        - name: MONGO_INITDB_ROOT_PASSWORD
          value: "password123"      # ⚠️ Never hardcode secrets! Module 04 fixes this.
      resources:
        requests:
          memory: "256Mi"
          cpu: "200m"
        limits:
          memory: "512Mi"
          cpu: "1000m"              # MongoDB can use up to 1 full CPU core
```

---

## ⌨️ The Commands

```bash
# ── DEPLOY ──────────────────────────────────────────────────────────────────

# Apply a manifest file to your cluster.
# kubectl apply is the standard "make it so" command.
kubectl apply -f yaml-examples/todo-backend-pod.yaml
kubectl apply -f yaml-examples/todo-mongo-pod.yaml


# ── INSPECT ─────────────────────────────────────────────────────────────────

# List all Pods in the current namespace.
# STATUS: Pending → ContainerCreating → Running  (the happy path)
kubectl get pods

# More detail: which Node the Pod landed on, and its cluster IP
kubectl get pods -o wide

# Full description including Events (your #1 debugging tool)
# Always check the "Events:" section at the bottom for errors
kubectl describe pod todo-backend

# Watch Pods in real time (refreshes continuously)
kubectl get pods --watch


# ── LOGS ────────────────────────────────────────────────────────────────────

# View logs from the container inside a Pod
kubectl logs todo-backend

# Follow (tail) logs in real time
kubectl logs -f todo-backend

# View logs from a previous (crashed) container instance
kubectl logs todo-backend --previous

# If a Pod has multiple containers, specify which one
kubectl logs todo-backend -c backend-api


# ── EXEC (shell into a running Pod) ─────────────────────────────────────────

# Open an interactive shell inside the Pod's container
kubectl exec -it todo-backend -- /bin/sh

# Run a one-off command without an interactive shell
kubectl exec todo-backend -- env | grep NODE


# ── DELETE ──────────────────────────────────────────────────────────────────

# Delete a Pod by name (it won't restart — see Module 02 for self-healing)
kubectl delete pod todo-backend

# Delete using the original manifest file
kubectl delete -f yaml-examples/todo-backend-pod.yaml
```

> **Important observation:** After `kubectl delete pod todo-backend` — the Pod is gone forever. It does **not** come back. This is the core limitation of raw Pods, and exactly why **Module 02 (Deployments)** exists.

---

## 🔧 The Drill

### Troubleshooting Scenario: `ImagePullBackOff`

You apply your `todo-backend-pod.yaml`. You run `kubectl get pods` and see:

```
NAME           READY   STATUS             RESTARTS   AGE
todo-backend   0/1     ImagePullBackOff   0          2m
todo-mongo     1/1     Running            0          2m
```

`ImagePullBackOff` means Kubernetes tried to pull the Docker image and **failed**. It's backing off and retrying.

**Your challenge — diagnose and fix it:**

1. Run `kubectl describe pod todo-backend`. Scroll to the **Events** section at the bottom. What does it say?
2. What are the **three most likely causes** of `ImagePullBackOff`?
3. How would you fix each cause?

<details>
<summary>💡 Hint (click to reveal)</summary>

The three most common causes:
- **Typo in the image name** → Fix the `image:` field in your YAML
- **Tag doesn't exist** → Check the image exists on Docker Hub with that exact tag
- **Private registry** → Add an `imagePullSecrets` entry pointing to a registry credential Secret

</details>

**Bonus challenge:** Once the backend Pod is `Running`, exec into it and run:
```bash
wget -qO- localhost:3000/health
```
What does this tell you about the app's internal state vs. its Pod state?

---

## ✅ Module Checklist

Before moving on, make sure you can:

- [ ] Explain the difference between a Node and a Pod
- [ ] Write a Pod manifest from memory (with `metadata`, `spec`, `containers`, `resources`)
- [ ] Run `kubectl apply`, `get pods`, `describe`, `logs`, and `exec`
- [ ] Explain why raw Pods are not suitable for production

---

**Navigation:** [← Back to README](../README.md) | [Next: Deployments →](02-deployments.md)
