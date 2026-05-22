# Module 12 — StatefulSets & DaemonSets

> **Track:** 🔴 Ops & Advanced | **Difficulty:** Advanced

**Navigation:** [← Helm](11-helm.md) | [Next: Observability →](13-observability.md)

---

## ❓ The Why

### The Problem with Deployments for Databases

In Module 02 we learned that Deployments are ideal for stateless apps. But MongoDB is stateful. Let's see what happens when you try to run a MongoDB replica set as a Deployment with 3 replicas:

**Problem 1 — Unstable identity:**
Pods get random names like `todo-mongo-7d9f8b-xkq2p`. If this Pod dies and gets replaced, the new Pod has a completely different name. MongoDB replica set members need **stable, permanent identities** to find each other.

**Problem 2 — Shared or missing storage:**
With a Deployment, all 3 replicas share one PVC, or each gets no PVC at all. MongoDB replica set members each need their **own independent storage** — member 0's data directory is never member 1's.

**Problem 3 — Unordered startup/shutdown:**
A Deployment starts all replicas simultaneously and shuts them down in any order. A MongoDB replica set needs ordered, sequential startup: primary first, then secondaries.

**StatefulSets solve all three.**

---

## 💡 The Concept

### StatefulSet — Ordered, Stable, Stateful Workloads

```
Deployment (stateless):           StatefulSet (stateful):
┌─────────────────────┐           ┌─────────────────────────────────┐
│ todo-backend-abc123 │           │ todo-mongo-0  (always "0")      │
│ todo-backend-def456 │           │ todo-mongo-1  (always "1")      │
│ todo-backend-ghi789 │           │ todo-mongo-2  (always "2")      │
│                     │           │                                  │
│ Any Pod can die     │           │ todo-mongo-1 dies → replaced as  │
│ and be replaced by  │           │ todo-mongo-1 (same name, same   │
│ any other Pod       │           │ PVC, same DNS name)              │
└─────────────────────┘           └─────────────────────────────────┘
```

StatefulSet guarantees:

| Guarantee | Meaning |
|-----------|---------|
| **Stable network identity** | Pod names are `<name>-0`, `<name>-1`, `<name>-2` — permanent |
| **Stable storage** | Each Pod gets its own PVC (from `volumeClaimTemplates`) — persists across restarts |
| **Ordered deployment** | Pods start in order (0, 1, 2) and stop in reverse (2, 1, 0) |
| **Ordered updates** | Rolling updates go in order — waits for each Pod to be ready before updating the next |

### StatefulSet DNS Names

With a Headless Service (Module 06), each Pod gets its own stable DNS record:

```
# StatefulSet name: todo-mongo, Headless Service: mongo-headless
# Each Pod is addressable at:

todo-mongo-0.mongo-headless.default.svc.cluster.local
todo-mongo-1.mongo-headless.default.svc.cluster.local
todo-mongo-2.mongo-headless.default.svc.cluster.local

# MongoDB replica set config uses these stable names — not IPs:
rs.initiate({
  members: [
    { _id: 0, host: "todo-mongo-0.mongo-headless:27017" },
    { _id: 1, host: "todo-mongo-1.mongo-headless:27017" },
    { _id: 2, host: "todo-mongo-2.mongo-headless:27017" }
  ]
})
```

### DaemonSet — One Pod Per Node

A **DaemonSet** ensures exactly one copy of a Pod runs on **every Node** in the cluster (or a subset, using `nodeSelector`).

```
Cluster (3 Nodes):

Node 1           Node 2           Node 3
┌────────────┐   ┌────────────┐   ┌────────────┐
│ App Pod(s) │   │ App Pod(s) │   │ App Pod(s) │
│            │   │            │   │            │
│ fluentd    │   │ fluentd    │   │ fluentd    │  ← DaemonSet
│ (log agent)│   │ (log agent)│   │ (log agent)│
└────────────┘   └────────────┘   └────────────┘

Add Node 4 → DaemonSet automatically places a fluentd Pod on it.
Remove Node 2 → DaemonSet automatically removes its fluentd Pod.
```

**Common DaemonSet use cases:**

| Use case | Example |
|----------|---------|
| Log collection | Fluentd, Fluent Bit |
| Metrics collection | Prometheus node-exporter |
| Network plugins | Calico, Cilium, WeaveNet |
| Storage drivers | Longhorn, Rook |
| Security scanning | Falco, Sysdig |

---

## 📄 The Blueprint

### `yaml-examples/todo-mongo-statefulset.yaml`

```yaml
# ── Headless Service (required for stable Pod DNS names) ────────────────────

apiVersion: v1
kind: Service
metadata:
  name: mongo-headless         # This name is used in volumeClaimTemplates and DNS
spec:
  clusterIP: None              # Makes it headless — DNS returns Pod IPs, not a VIP
  selector:
    app: todo-mongo
  ports:
    - name: mongo
      port: 27017
      targetPort: 27017

---
# ── Regular Service (for app connections to the primary) ────────────────────

apiVersion: v1
kind: Service
metadata:
  name: todo-mongo-svc
spec:
  type: ClusterIP
  selector:
    app: todo-mongo
  ports:
    - port: 27017
      targetPort: 27017

---
# ── StatefulSet ───────────────────────────────────────────────────────────────

apiVersion: apps/v1
kind: StatefulSet

metadata:
  name: todo-mongo

spec:
  serviceName: "mongo-headless"   # Must reference the Headless Service above
  replicas: 3                     # Creates: todo-mongo-0, todo-mongo-1, todo-mongo-2

  selector:
    matchLabels:
      app: todo-mongo

  # Ordered, graceful rolling update — waits for each Pod to be Ready
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0              # Update all Pods (set to N to only update Pods >= N)

  template:
    metadata:
      labels:
        app: todo-mongo

    spec:
      # Run initContainers to configure the replica set before MongoDB starts
      initContainers:
        - name: init-mongo
          image: busybox:1.36
          command: ['sh', '-c', 'until nslookup mongo-headless; do echo waiting; sleep 2; done']
          # Waits for headless Service DNS to be available before proceeding

      containers:
        - name: mongodb
          image: mongo:6.0
          command:
            - mongod
            - "--replSet"
            - rs0                  # Replica set name
            - "--bind_ip_all"      # Listen on all interfaces
          ports:
            - containerPort: 27017
              name: mongo

          env:
            - name: MONGO_INITDB_ROOT_USERNAME
              valueFrom:
                secretKeyRef:
                  name: todo-secrets
                  key: MONGO_USERNAME
            - name: MONGO_INITDB_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: todo-secrets
                  key: MONGO_PASSWORD

          livenessProbe:
            tcpSocket:
              port: 27017
            initialDelaySeconds: 30
            periodSeconds: 10

          readinessProbe:
            exec:
              command:
                - mongosh
                - "--eval"
                - "db.adminCommand('ping')"
            initialDelaySeconds: 5
            periodSeconds: 10

          resources:
            requests:
              memory: "512Mi"
              cpu: "200m"
            limits:
              memory: "1Gi"
              cpu: "1000m"

          volumeMounts:
            - name: mongo-data         # References the volumeClaimTemplate name below
              mountPath: /data/db
            - name: mongo-config
              mountPath: /data/configdb

  # ── volumeClaimTemplates: each Pod gets its OWN PVC ─────────────────────────
  # Kubernetes auto-creates: mongo-data-todo-mongo-0, mongo-data-todo-mongo-1, etc.
  volumeClaimTemplates:
    - metadata:
        name: mongo-data             # Referenced in volumeMounts above
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 5Gi

    - metadata:
        name: mongo-config
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 1Gi

---
# ── DaemonSet: Fluent Bit log collector on every Node ────────────────────────

apiVersion: apps/v1
kind: DaemonSet

metadata:
  name: fluent-bit
  namespace: kube-system
  labels:
    app: fluent-bit

spec:
  selector:
    matchLabels:
      app: fluent-bit

  updateStrategy:
    type: RollingUpdate          # Update one Node at a time

  template:
    metadata:
      labels:
        app: fluent-bit

    spec:
      # DaemonSet Pods usually need access to host-level resources
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule     # Also run on control plane Nodes (optional)

      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:2.1
          resources:
            requests:
              memory: "50Mi"
              cpu: "50m"
            limits:
              memory: "100Mi"
              cpu: "100m"
          volumeMounts:
            - name: varlog
              mountPath: /var/log        # Read container logs from the Node
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: fluent-bit-config
              mountPath: /fluent-bit/etc/

      volumes:
        - name: varlog
          hostPath:
            path: /var/log              # Mount host's /var/log into the container
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: fluent-bit-config
          configMap:
            name: fluent-bit-config
```

---

## ⌨️ The Commands

```bash
# ── STATEFULSET ──────────────────────────────────────────────────────────────

kubectl apply -f yaml-examples/todo-mongo-statefulset.yaml

# Observe ordered Pod creation: 0 starts first, 1 waits until 0 is Running
kubectl get pods -l app=todo-mongo --watch
# todo-mongo-0   0/1   ContainerCreating
# todo-mongo-0   1/1   Running            ← 0 is ready
# todo-mongo-1   0/1   ContainerCreating  ← now 1 starts
# todo-mongo-1   1/1   Running
# todo-mongo-2   0/1   ContainerCreating

# Each Pod gets its own PVC automatically
kubectl get pvc
# NAME                      STATUS   VOLUME   CAPACITY
# mongo-data-todo-mongo-0   Bound    pv-a     5Gi
# mongo-data-todo-mongo-1   Bound    pv-b     5Gi
# mongo-data-todo-mongo-2   Bound    pv-c     5Gi

# Connect to a specific replica by its stable name
kubectl exec -it todo-mongo-0 -- mongosh

# Connect to the primary specifically
kubectl exec -it todo-mongo-0 -- mongosh --eval 'rs.status()'

# Initialise the replica set (run once after first deploy)
kubectl exec -it todo-mongo-0 -- mongosh --eval '
  rs.initiate({
    _id: "rs0",
    members: [
      { _id: 0, host: "todo-mongo-0.mongo-headless:27017" },
      { _id: 1, host: "todo-mongo-1.mongo-headless:27017" },
      { _id: 2, host: "todo-mongo-2.mongo-headless:27017" }
    ]
  })
'

# Scale the StatefulSet (new replicas always appended at the end)
kubectl scale statefulset todo-mongo --replicas=5

# Rolling update: updates in order 2→1→0 (reverse order)
kubectl set image statefulset/todo-mongo mongodb=mongo:7.0


# ── DAEMONSET ────────────────────────────────────────────────────────────────

kubectl apply -f yaml-examples/todo-mongo-statefulset.yaml  # includes the DaemonSet

# One Pod per Node — count should match your Node count
kubectl get daemonset -n kube-system fluent-bit

# List the individual Pods (each on a different Node)
kubectl get pods -n kube-system -l app=fluent-bit -o wide

# DaemonSet update: rolls out one Node at a time
kubectl set image daemonset/fluent-bit fluent-bit=fluent/fluent-bit:2.2 -n kube-system
kubectl rollout status daemonset/fluent-bit -n kube-system
```

---

## 🔧 The Drill

### Exercise: StatefulSet vs Deployment for Databases

**Part 1 — Deployment failure mode:**
```bash
# Deploy MongoDB as a regular Deployment (no PVC)
kubectl create deployment mongo-test --image=mongo:6.0 --replicas=2

# Insert data into one of the Pods
POD=$(kubectl get pods -l app=mongo-test -o name | head -1 | cut -d/ -f2)
kubectl exec -it $POD -- mongosh --eval 'db.test.insertOne({msg: "will this survive?"})'

# Delete that Pod
kubectl delete pod $POD

# Query the replacement
NEW_POD=$(kubectl get pods -l app=mongo-test -o name | head -1 | cut -d/ -f2)
kubectl exec -it $NEW_POD -- mongosh --eval 'db.test.find()'
# Result: []  ← data is gone
```

**Part 2 — StatefulSet with PVC:**
```bash
# Deploy as StatefulSet
kubectl apply -f yaml-examples/todo-mongo-statefulset.yaml
kubectl get pvc   # Wait for Bound

# Insert data into todo-mongo-0
kubectl exec -it todo-mongo-0 -- mongosh --eval 'db.test.insertOne({msg: "will this survive?"})'

# Delete todo-mongo-0
kubectl delete pod todo-mongo-0

# Watch it come back as todo-mongo-0 (same name!)
kubectl get pods -l app=todo-mongo --watch

# Query todo-mongo-0 again
kubectl exec -it todo-mongo-0 -- mongosh --eval 'db.test.find()'
# Result: [{msg: "will this survive?"}]  ← data survived! ✅
```

---

## ✅ Module Checklist

- [ ] Explain the three guarantees of a StatefulSet (identity, storage, ordering)
- [ ] Write a StatefulSet with `volumeClaimTemplates` creating per-Pod PVCs
- [ ] Explain why a Headless Service is required for a StatefulSet
- [ ] Initialise a MongoDB replica set using stable Pod DNS names
- [ ] Write a DaemonSet and explain when you'd use one over a Deployment

---

**Navigation:** [← Helm](11-helm.md) | [Next: Observability →](13-observability.md)
