# Module 07 — Persistent Volumes & Claims

> **Track:** 🟡 Storage | **Difficulty:** Intermediate

**Navigation:** [← DNS & Service Discovery](06-dns-and-service-discovery.md) | [Next: Namespaces & RBAC →](08-namespaces-and-rbac.md)

---

## ❓ The Why

Right now your MongoDB Pod stores all data inside the container filesystem. Watch what happens:

```bash
# Insert a To-Do item
kubectl exec -it todo-mongo -- mongosh --eval 'db.todos.insert({title: "Buy milk"})'

# Delete the Pod (simulating a crash or reschedule)
kubectl delete pod todo-mongo

# A new Pod starts (via the Deployment)... connect to it
kubectl exec -it todo-mongo-<new-hash> -- mongosh --eval 'db.todos.find()'
# Result: []   ← ALL DATA IS GONE
```

Container storage is **ephemeral by design**. When a container dies, its filesystem dies with it. For stateless apps (backend, frontend) this is fine. For databases, this is catastrophic.

You need storage that:
- **Survives Pod deletion and rescheduling**
- **Lives independently** of any specific Pod
- **Can be claimed** by whichever Pod needs it

---

## 💡 The Concept

### Physical Analogy: Renting a Storage Unit

```
Storage Provider                 Your Application
(AWS, GCP, NFS...)
                                 ┌──────────────────┐
┌─────────────────┐              │ PersistentVolume  │
│ PersistentVolume│◀─── binds ───│     Claim (PVC)   │
│     (PV)        │              │                   │
│                 │              │ "I need 5Gi,      │
│ "10Gi SSD disk  │              │  ReadWriteOnce"   │
│  available for  │              └────────┬─────────┘
│  rent"          │                       │ mounted into
└─────────────────┘                       ▼
                                 ┌──────────────────┐
                                 │    MongoDB Pod    │
                                 │  /data/db ──────▶ │ (data lives here)
                                 └──────────────────┘
```

- **PersistentVolume (PV):** A piece of storage provisioned in the cluster — an actual disk that exists independently of any Pod. Like a storage unit available for rent.
- **PersistentVolumeClaim (PVC):** Your rental agreement — "I need a 5GB unit with read/write access." Kubernetes finds a matching PV and binds them together.
- **StorageClass:** The catalogue of available storage types (SSD, HDD, NFS). In cloud environments, StorageClasses enable **dynamic provisioning** — PVs are created automatically when you make a PVC.

### PVC Lifecycle

```
PVC Created → Kubernetes finds matching PV → Bound → Mounted to Pod
                                                          │
                                                    Pod deleted?
                                                          │
                                                   PVC still exists
                                                   PV still exists
                                                   Data still intact ✅
```

### Access Modes

| Mode | Short | Meaning | Use Case |
|------|-------|---------|----------|
| ReadWriteOnce | RWO | One Node mounts read/write | Databases (MongoDB, Postgres) |
| ReadOnlyMany | ROX | Many Nodes mount read-only | Shared config/assets |
| ReadWriteMany | RWX | Many Nodes mount read/write | Shared file systems (NFS, EFS) |

> MongoDB requires **RWO**. Only one Node writes to the data directory at a time.

### Reclaim Policies

What happens to the PV when the PVC is deleted?

| Policy | Behaviour |
|--------|-----------|
| `Retain` | PV stays, data preserved. Manual cleanup required. |
| `Delete` | PV and underlying storage are deleted automatically. |
| `Recycle` | (Deprecated) Scrubs the volume for reuse. |

---

## 📄 The Blueprint

### `yaml-examples/todo-storage.yaml`

```yaml
# ── Option A: Manual PersistentVolume (for local/on-prem) ───────────────────
# In cloud environments, skip this — StorageClass handles PV creation dynamically

apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongo-pv
spec:
  capacity:
    storage: 5Gi                # Total size of this storage unit
  accessModes:
    - ReadWriteOnce             # Only one Node mounts at a time
  persistentVolumeReclaimPolicy: Retain  # Keep the data when PVC is deleted
  storageClassName: standard    # Must match the PVC's storageClassName
  hostPath:                     # Use a directory on the Node (dev/testing only)
    path: /data/mongodb         # Do NOT use hostPath in production — use cloud volumes

---
# ── PersistentVolumeClaim — what your app actually requests ──────────────────

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc               # Referenced by name in the Pod/Deployment spec

spec:
  accessModes:
    - ReadWriteOnce             # Must match the PV's access mode

  storageClassName: standard    # Which StorageClass to use
                                # In cloud: "gp3" (AWS), "standard" (GKE), "managed-premium" (AKS)
                                # Leaving this empty uses the cluster's default StorageClass

  resources:
    requests:
      storage: 5Gi              # Request 5 Gibibytes
                                # Kubernetes finds a PV with at least this much capacity

---
# ── MongoDB Deployment using the PVC ────────────────────────────────────────

apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-mongo
spec:
  replicas: 1                   # Always 1 for a non-clustered MongoDB (use StatefulSet for HA)
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
          envFrom:
            - secretRef:
                name: todo-secrets    # MONGO_USERNAME, MONGO_PASSWORD from Module 04

          volumeMounts:
            - name: mongo-storage     # Must match the volume name below
              mountPath: /data/db     # MongoDB stores all data in this directory

      volumes:
        - name: mongo-storage
          persistentVolumeClaim:
            claimName: mongo-pvc     # Attach the PVC we created above
```

### StorageClass Examples (Cloud)

```yaml
# AWS — gp3 SSD StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # Make this the default
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
reclaimPolicy: Delete
allowVolumeExpansion: true       # Allow PVCs to grow without recreation

---
# GKE — Standard StorageClass (already exists by default)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-standard
reclaimPolicy: Delete
allowVolumeExpansion: true
```

---

## ⌨️ The Commands

```bash
# ── DEPLOY ───────────────────────────────────────────────────────────────────

kubectl apply -f yaml-examples/todo-storage.yaml


# ── INSPECT ─────────────────────────────────────────────────────────────────

# List PersistentVolumeClaims — STATUS must be "Bound" before the Pod can start
kubectl get pvc
# NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS
# mongo-pvc   Bound    mongo-pv   5Gi        RWO            standard

# List PersistentVolumes (cluster-wide resource — no namespace)
kubectl get pv

# Full detail on a PVC (check Events if STATUS is Pending)
kubectl describe pvc mongo-pvc

# List available StorageClasses
kubectl get storageclass
kubectl get sc                  # Short form


# ── DIAGNOSE PVC STUCK IN PENDING ───────────────────────────────────────────

# If PVC STATUS stays "Pending", run:
kubectl describe pvc mongo-pvc
# Look at the Events section:
# "no persistent volumes available for this claim" → No PV matches your request
# "waiting for a volume to be created"            → Dynamic provisioner is working


# ── PROVE DATA PERSISTENCE ──────────────────────────────────────────────────

# Step 1: Insert test data
kubectl exec -it $(kubectl get pod -l tier=database -o name | head -1) -- \
  mongosh --eval 'db.todos.insertOne({title: "Persistent test", done: false})'

# Step 2: Delete the Pod (NOT the Deployment — let it recreate)
kubectl delete pod $(kubectl get pod -l tier=database -o name | head -1 | cut -d/ -f2)

# Step 3: Wait for the new Pod to come up
kubectl get pods -l tier=database --watch

# Step 4: Query the new Pod — data should still be there
kubectl exec -it $(kubectl get pod -l tier=database -o name | head -1) -- \
  mongosh --eval 'db.todos.find().pretty()'


# ── EXPAND A PVC (if StorageClass allows) ───────────────────────────────────

# Edit the PVC to request more storage
kubectl patch pvc mongo-pvc -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'

# Verify the expansion
kubectl get pvc mongo-pvc


# ── BACKUP (basic) ──────────────────────────────────────────────────────────

# Take a mongodump from inside the Pod
kubectl exec -it <mongo-pod> -- mongodump --out /tmp/backup

# Copy it to your local machine
kubectl cp <mongo-pod>:/tmp/backup ./mongo-backup
```

---

## 🔧 The Drill

### Exercise: Prove Data Survives a Pod Restart

This is the most important exercise in this module. Do it live.

**Steps:**

1. Deploy MongoDB with the PVC:
   ```bash
   kubectl apply -f yaml-examples/todo-storage.yaml
   kubectl get pvc   # Wait for STATUS = Bound
   ```

2. Insert a document:
   ```bash
   kubectl exec -it $(kubectl get pod -l tier=database -o jsonpath='{.items[0].metadata.name}') -- \
     mongosh todos --eval 'db.items.insertOne({task: "survive the restart", done: false})'
   ```

3. Forcefully delete the Pod:
   ```bash
   kubectl delete pod -l tier=database
   ```

4. Watch the replacement start (the Deployment recreates it):
   ```bash
   kubectl get pods -l tier=database --watch
   ```

5. Query the new Pod:
   ```bash
   kubectl exec -it $(kubectl get pod -l tier=database -o jsonpath='{.items[0].metadata.name}') -- \
     mongosh todos --eval 'db.items.find().pretty()'
   ```

Expected: Your document is still there. The PVC kept the data alive independently of the Pod.

---

### Troubleshooting: PVC Stuck in `Pending`

```
NAME        STATUS    VOLUME   CAPACITY   ACCESS MODES
mongo-pvc   Pending                                    ← stuck here
```

**Checklist:**

| Check | Command |
|-------|---------|
| Is there a matching PV? | `kubectl get pv` — look for one with matching size and access mode |
| Is the StorageClass correct? | `kubectl get sc` — does the name match exactly? |
| Is the dynamic provisioner running? | `kubectl get pods -n kube-system` — look for csi-provisioner pods |
| Detailed error | `kubectl describe pvc mongo-pvc` — read the Events |

---

## ✅ Module Checklist

- [ ] Explain the PV → PVC → Pod chain in your own words
- [ ] Write a PVC manifest requesting 5Gi with ReadWriteOnce access
- [ ] Mount a PVC into a MongoDB Pod at `/data/db`
- [ ] Prove that data survives a Pod deletion by running the exercise above
- [ ] Explain the difference between `Retain` and `Delete` reclaim policies

---

**Navigation:** [← DNS & Service Discovery](06-dns-and-service-discovery.md) | [Next: Namespaces & RBAC →](08-namespaces-and-rbac.md)
