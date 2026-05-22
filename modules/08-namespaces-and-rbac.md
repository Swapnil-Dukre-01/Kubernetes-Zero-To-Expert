# Module 08 — Namespaces & RBAC

> **Track:** 🔴 Ops & Advanced | **Difficulty:** Intermediate

**Navigation:** [← Persistent Volumes](07-persistent-volumes.md) | [Next: Autoscaling →](09-autoscaling.md)

---

## ❓ The Why

Your To-Do app team is growing. You now have:

- **5 developers** who need to deploy to `staging`
- **2 senior engineers** who manage `production`
- **1 SRE** who needs read-only access to everything for monitoring
- **A CI/CD pipeline** that needs to deploy, but should never delete production databases

With no access controls in place:
- A developer running `kubectl delete deployment` in the wrong terminal wipes production
- A junior engineer can view production Secrets (passwords, API keys)
- A compromised CI pipeline token has unlimited cluster access

Namespaces give you **isolation**. RBAC gives you **access control**. Together they let you run multiple teams and environments on one cluster safely.

---

## 💡 The Concept

### Namespaces — Virtual Clusters

A **Namespace** is a virtual partition inside one physical cluster. Think of a large office building: same building, different floors, each floor has its own rooms, its own team, and a keycard that only opens doors on that floor.

```
┌─────────────────────── Kubernetes Cluster ─────────────────────────────┐
│                                                                         │
│  ┌────────────────┐  ┌────────────────┐  ┌───────────────────────────┐ │
│  │  Namespace:    │  │  Namespace:    │  │  Namespace:               │ │
│  │  default       │  │  staging       │  │  production               │ │
│  │                │  │                │  │                           │ │
│  │  (playground)  │  │  dev deploys   │  │  real traffic, real data  │ │
│  │                │  │  test data     │  │  restricted access        │ │
│  └────────────────┘  └────────────────┘  └───────────────────────────┘ │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  kube-system  (CoreDNS, kube-proxy, metrics-server — don't touch) │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

**What is namespaced vs cluster-wide:**

| Namespaced (scoped to a namespace) | Cluster-wide (no namespace) |
|------------------------------------|----------------------------|
| Pods, Deployments, Services | Nodes |
| ConfigMaps, Secrets | PersistentVolumes |
| PersistentVolumeClaims | StorageClasses |
| Roles, RoleBindings | ClusterRoles, ClusterRoleBindings |

### RBAC — Role-Based Access Control

RBAC answers: **"Who can do what to which resources?"**

```
Subject         Verb          Resource          Namespace
(who)           (what)        (which)           (where)
───────────     ──────────    ──────────────    ──────────
alice           create        deployments       staging     ✅
alice           delete        deployments       production  ❌
ci-pipeline     create        deployments       production  ✅
ci-pipeline     delete        secrets           production  ❌
monitoring-sa   get,list      pods,services     *           ✅
```

**The four RBAC objects:**

| Object | Scope | Purpose |
|--------|-------|---------|
| `Role` | One namespace | Defines allowed verbs on resources within a namespace |
| `ClusterRole` | Cluster-wide | Defines allowed verbs on cluster-scoped or all-namespace resources |
| `RoleBinding` | One namespace | Grants a Role to a subject within a namespace |
| `ClusterRoleBinding` | Cluster-wide | Grants a ClusterRole to a subject across the whole cluster |

---

## 📄 The Blueprint

### `yaml-examples/todo-rbac.yaml`

```yaml
# ── Namespaces ───────────────────────────────────────────────────────────────

apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    env: staging
    team: todo-app

---
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    env: production
    team: todo-app

---
# ── Role: Developer access in staging ───────────────────────────────────────
# Can deploy and debug, but cannot touch Secrets or delete PVCs

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: staging           # Only valid in the staging namespace

rules:
  - apiGroups: [""]            # "" means the core API group (Pods, Services, ConfigMaps)
    resources: ["pods", "pods/log", "pods/exec", "services", "configmaps", "endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  - apiGroups: ["apps"]        # The apps API group (Deployments, ReplicaSets)
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]     # Can VIEW secrets but not create or modify them

---
# ── RoleBinding: Grant developer-role to alice ───────────────────────────────

apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: staging           # Grants the role in this namespace only

subjects:                      # Who gets the permissions
  - kind: User
    name: alice                # Must match the username in the cluster's auth system
    apiGroup: rbac.authorization.k8s.io

roleRef:                       # Which Role to grant
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io

---
# ── ClusterRole: Read-only access across all namespaces (for monitoring) ─────

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-read-only

rules:
  - apiGroups: ["", "apps", "extensions"]
    resources: ["*"]           # All resource types
    verbs: ["get", "list", "watch"]   # Read-only verbs only

  - apiGroups: [""]
    resources: ["nodes"]       # Node info (cluster-scoped resource)
    verbs: ["get", "list", "watch"]

---
# ── ClusterRoleBinding: Grant read-only to monitoring service account ─────────

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-read-only-binding

subjects:
  - kind: ServiceAccount
    name: monitoring-sa        # A ServiceAccount (for automated tools, not humans)
    namespace: monitoring      # The namespace where the ServiceAccount lives

roleRef:
  kind: ClusterRole
  name: cluster-read-only
  apiGroup: rbac.authorization.k8s.io

---
# ── ServiceAccount for CI/CD pipeline ────────────────────────────────────────
# Best practice: give automated tools a dedicated ServiceAccount, not a human account

apiVersion: v1
kind: ServiceAccount
metadata:
  name: ci-deployer
  namespace: production

---
# ── Role for CI/CD in production: can deploy, cannot delete databases ─────────

apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-deploy-role
  namespace: production

rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
    # NOTE: "delete" is intentionally omitted

  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch"]

  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "create", "update", "patch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deploy-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: ci-deployer
    namespace: production
roleRef:
  kind: Role
  name: ci-deploy-role
  apiGroup: rbac.authorization.k8s.io
```

---

## ⌨️ The Commands

```bash
# ── NAMESPACES ───────────────────────────────────────────────────────────────

# Create namespaces
kubectl create namespace staging
kubectl create namespace production

# List all namespaces
kubectl get namespaces
kubectl get ns                    # Short form

# Set a default namespace for your session (so you don't type -n every time)
kubectl config set-context --current --namespace=staging

# Verify your current context and namespace
kubectl config view --minify | grep namespace

# Deploy to a specific namespace
kubectl apply -f yaml-examples/todo-backend-deployment.yaml -n production

# List resources across ALL namespaces
kubectl get pods --all-namespaces
kubectl get pods -A              # Short form

# List everything in a namespace
kubectl get all -n production


# ── RBAC ─────────────────────────────────────────────────────────────────────

kubectl apply -f yaml-examples/todo-rbac.yaml

# Inspect Roles
kubectl get roles -n staging
kubectl describe role developer-role -n staging

# Inspect RoleBindings
kubectl get rolebindings -n staging
kubectl describe rolebinding developer-binding -n staging

# Inspect ClusterRoles
kubectl get clusterroles | grep -v system   # Filter out built-in system roles


# ── CHECK PERMISSIONS ────────────────────────────────────────────────────────

# Check what YOU can do
kubectl auth can-i create deployments
kubectl auth can-i create deployments -n production
kubectl auth can-i delete secrets -n production

# Check what ALICE can do (impersonate a user)
kubectl auth can-i create deployments --as=alice -n staging
kubectl auth can-i delete deployments --as=alice -n production   # Should be "no"

# Check what a ServiceAccount can do
kubectl auth can-i delete secrets \
  --as=system:serviceaccount:production:ci-deployer \
  -n production
# Expected: "no"

# List ALL permissions for a subject
kubectl auth can-i --list --as=alice -n staging


# ── SERVICE ACCOUNTS ─────────────────────────────────────────────────────────

# Create a ServiceAccount
kubectl create serviceaccount ci-deployer -n production

# List ServiceAccounts
kubectl get serviceaccounts -n production

# Get the token for a ServiceAccount (for use in CI pipelines)
kubectl create token ci-deployer -n production

# Use a ServiceAccount in a Pod
# (add to Pod spec:)
# spec:
#   serviceAccountName: ci-deployer
```

---

## 🔧 The Drill

### Exercise: Least-Privilege CI/CD Access

**Goal:** Create a ServiceAccount for your CI/CD pipeline that can deploy but cannot delete the MongoDB StatefulSet.

**Steps:**

1. Apply the RBAC config:
   ```bash
   kubectl apply -f yaml-examples/todo-rbac.yaml
   ```

2. Verify the CI deployer **can** update deployments:
   ```bash
   kubectl auth can-i update deployments \
     --as=system:serviceaccount:production:ci-deployer \
     -n production
   # Expected: yes
   ```

3. Verify it **cannot** delete deployments:
   ```bash
   kubectl auth can-i delete deployments \
     --as=system:serviceaccount:production:ci-deployer \
     -n production
   # Expected: no
   ```

4. Verify it **cannot** touch Secrets:
   ```bash
   kubectl auth can-i get secrets \
     --as=system:serviceaccount:production:ci-deployer \
     -n production
   # Expected: no
   ```

---

### Troubleshooting: `Forbidden` Error in a Pod

A Pod trying to call the Kubernetes API (e.g. an operator, monitoring agent) gets:

```
Error: pods is forbidden: User "system:serviceaccount:default:default"
cannot list resource "pods" in API group "" in the namespace "default"
```

**Root cause:** The Pod is using the `default` ServiceAccount, which has no permissions.

**Fix:**
1. Create a dedicated ServiceAccount with the right Role
2. Assign it to the Pod via `spec.serviceAccountName`
3. Restart the Pod

---

## ✅ Module Checklist

- [ ] Create namespaces for `staging` and `production`
- [ ] Write a `Role` that allows developers to deploy but not delete Secrets
- [ ] Bind the Role to a user with a `RoleBinding`
- [ ] Create a ServiceAccount for a CI pipeline
- [ ] Use `kubectl auth can-i` to verify permissions work as expected

---

**Navigation:** [← Persistent Volumes](07-persistent-volumes.md) | [Next: Autoscaling →](09-autoscaling.md)
