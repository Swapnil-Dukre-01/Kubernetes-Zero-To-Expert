# Module 03 — Services

> **Track:** 🟣 Foundation | **Difficulty:** Beginner

**Navigation:** [← Deployments](02-deployments.md) | [Next: ConfigMaps & Secrets →](04-configmaps-and-secrets.md)

---

## ❓ The Why

Your Deployment now runs 3 backend Pods. Each Pod has its own cluster IP address — but these IPs are **ephemeral**. Every time a Pod is replaced (crash, rolling update, scaling), it gets a **brand new IP**. The old IP is gone.

Problems this creates for our To-Do app:

- The **frontend** can't hardcode a backend IP — it changes constantly
- With 3 backend replicas, **which one** does the frontend call?
- The **backend** can't hardcode MongoDB's IP — it changes too
- External users can't reach **any** of your Pods (they have no public IP)

You need a **stable address** that automatically load-balances across all healthy Pods and never changes, regardless of which specific Pods are running underneath.

---

## 💡 The Concept

### Physical Analogy: A Reception Desk

A **Service** is a company's reception desk:

```
External caller                    Receptionist              Available staff
(frontend Pod)                     (Service)                 (backend Pods)
      │                                │
      │  "I need the backend"          │         ┌─── Pod-1 (10.244.1.5)
      │──────────────────────────────▶ │ ───────▶├─── Pod-2 (10.244.2.3)
      │                                │         └─── Pod-3 (10.244.3.7)
      │  Response comes back           │
      │◀────────────────────────────── │
```

- Callers use the **main number** (Service's stable IP or DNS name) — it never changes
- The receptionist routes calls to **whichever staff member is available**
- Staff (Pods) can quit and be replaced — the main number stays the same

### The Three Service Types

| Type | Who Can Access | Use Case |
|------|---------------|----------|
| `ClusterIP` | Only inside the cluster | Backend → Database, Frontend → Backend |
| `NodePort` | Anyone who can reach a Node IP | Simple external access (dev/testing) |
| `LoadBalancer` | Public internet (via cloud LB) | Production internet-facing services |

### How Services Find Pods — Label Selectors

A Service doesn't know Pod IPs. Instead, it uses **label selectors** to find Pods:

```
Service selector:          Pod labels:
  app: todo          ────▶   app: todo      ✅ Match → Pod gets traffic
  tier: backend      ────▶   tier: backend  ✅ Match → Pod gets traffic

                             app: todo      ✅
                             tier: database ❌ No match → Pod is excluded
```

This is why labels in Module 01 were so important.

---

## 📄 The Blueprint

### `yaml-examples/todo-services.yaml`

```yaml
# All three Services for the To-Do app in one file, separated by ---

# ── Service 1: Frontend (LoadBalancer — external access) ────────────────────
apiVersion: v1
kind: Service

metadata:
  name: todo-frontend-svc

spec:
  type: LoadBalancer         # Provisions a cloud load balancer (or use NodePort locally)
                             # On Minikube: run 'minikube tunnel' to get an external IP

  selector:                  # Route traffic to Pods with these labels
    app: todo
    tier: frontend

  ports:
    - name: http
      protocol: TCP
      port: 80               # Port THIS Service listens on (what external callers use)
      targetPort: 80         # Port on the selected Pods (Nginx listens on 80)

---
# ── Service 2: Backend (ClusterIP — internal only) ──────────────────────────
apiVersion: v1
kind: Service

metadata:
  name: todo-backend-svc     # DNS name inside cluster: todo-backend-svc.default.svc.cluster.local

spec:
  type: ClusterIP            # Default — only reachable from inside the cluster

  selector:
    app: todo
    tier: backend

  ports:
    - name: http
      protocol: TCP
      port: 80               # Frontend calls: http://todo-backend-svc:80
      targetPort: 3000       # Forwards to port 3000 on the backend Pods

---
# ── Service 3: MongoDB (ClusterIP — internal only) ──────────────────────────
apiVersion: v1
kind: Service

metadata:
  name: todo-mongo-svc       # Backend connects to: mongodb://todo-mongo-svc:27017/todos

spec:
  type: ClusterIP

  selector:
    app: todo
    tier: database

  ports:
    - name: mongo
      protocol: TCP
      port: 27017             # MongoDB's default port (same on Service and Pod)
      targetPort: 27017
```

---

## ⌨️ The Commands

```bash
# ── DEPLOY ──────────────────────────────────────────────────────────────────

kubectl apply -f yaml-examples/todo-services.yaml


# ── INSPECT ─────────────────────────────────────────────────────────────────

# List all Services (notice ClusterIP vs LoadBalancer and the EXTERNAL-IP column)
kubectl get services

# Short form
kubectl get svc

# Full detail: selector, endpoints, ports, events
kubectl describe service todo-backend-svc

# See which Pod IPs are behind the Service right now
# This is the KEY debugging command — if this is empty, traffic can't flow
kubectl get endpoints todo-backend-svc


# ── TEST CONNECTIVITY ────────────────────────────────────────────────────────

# Spin up a temporary Pod to test the Service from inside the cluster
kubectl run curl-test --image=curlimages/curl --rm -it -- \
  curl http://todo-backend-svc:80/health

# Or use busybox
kubectl run net-test --image=busybox --rm -it -- \
  wget -qO- http://todo-backend-svc:80/health


# ── LOCAL ACCESS (Minikube) ──────────────────────────────────────────────────

# Get the URL for a NodePort or LoadBalancer Service
minikube service todo-frontend-svc --url

# Open it directly in your browser
minikube service todo-frontend-svc

# For LoadBalancer Services, run the tunnel in a separate terminal
minikube tunnel


# ── PORT FORWARD (quick local access without a Service type change) ──────────

# Forward your local port 8080 to the Service's port 80
kubectl port-forward service/todo-backend-svc 8080:80

# Now test: curl http://localhost:8080/health
```

---

## 🔧 The Drill

### Scenario: The Selector Trap — `<none>` Endpoints

You apply the backend Service. But when you test it, all requests hang or get `Connection refused`. You run:

```bash
kubectl get endpoints todo-backend-svc
```

Output:
```
NAME               ENDPOINTS   AGE
todo-backend-svc   <none>      5m
```

`<none>` means the Service has **no Pods selected**. Zero traffic will flow.

**Your debugging steps:**

1. Check what the Service's selector is:
   ```bash
   kubectl describe service todo-backend-svc | grep Selector
   ```

2. Check what labels the actual Pods have:
   ```bash
   kubectl get pods --show-labels
   ```

3. Compare them. Find the mismatch (a typo, a missing label, a different value).

4. Fix either the Service selector or the Pod labels, then re-apply.

5. Verify: `kubectl get endpoints todo-backend-svc` — you should now see Pod IPs listed.

<details>
<summary>💡 Common causes</summary>

- `tier: Backend` in the Service vs `tier: backend` in the Pod (case mismatch)
- Pods are in a different **namespace** than the Service
- Pods haven't started yet (still `Pending`) so they aren't registered as endpoints

</details>

---

### Bonus: Understand Load Balancing

```bash
# Check your backend has 3 replicas running
kubectl get pods -l tier=backend

# Hit the Service 6 times and observe which Pod responds
# (Your app should return the Pod hostname in the response)
for i in {1..6}; do
  kubectl run curl-$i --image=curlimages/curl --rm --restart=Never -q -- \
    curl -s http://todo-backend-svc/whoami
done
```

You should see requests distributed across different Pods. This is **kube-proxy's** round-robin load balancing.

---

## ✅ Module Checklist

- [ ] Explain the difference between ClusterIP, NodePort, and LoadBalancer
- [ ] Write Service YAML with the correct `selector`, `port`, and `targetPort`
- [ ] Understand why `port` and `targetPort` are different (and when they match)
- [ ] Diagnose empty endpoints with `kubectl get endpoints`
- [ ] Test a ClusterIP Service from inside the cluster using a temporary Pod

---

**Navigation:** [← Deployments](02-deployments.md) | [Next: ConfigMaps & Secrets →](04-configmaps-and-secrets.md)
