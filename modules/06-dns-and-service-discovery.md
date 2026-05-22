# Module 06 — DNS & Service Discovery

> **Track:** 🟢 Networking | **Difficulty:** Intermediate

**Navigation:** [← Ingress](05-ingress.md) | [Next: Persistent Volumes →](07-persistent-volumes.md)

---

## ❓ The Why

Your backend Deployment connects to MongoDB using:

```yaml
MONGO_URI: "mongodb://todo-mongo-svc:27017/todos"
```

But how does `todo-mongo-svc` resolve to an actual IP address? There is no `/etc/hosts` entry. You didn't configure any DNS server. Yet it just works. How?

Without understanding DNS inside Kubernetes, you'll hit walls like:

- Calling a Service by name from a different namespace and getting `connection refused`
- Wondering why `todo-mongo-svc` works but `todo-mongo-svc.staging` doesn't
- Debugging intermittent DNS failures in production with no idea where to look

---

## 💡 The Concept

### Physical Analogy: An Internal Office Phone Directory

Every company has an internal phone directory. You don't memorise your colleague's desk extension — you look them up by name. Kubernetes runs **CoreDNS**, which is exactly this: an internal phone directory for the cluster.

```
┌──────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                     │
│                                                          │
│  ┌─────────────┐    "todo-mongo-svc?"    ┌────────────┐  │
│  │  Backend    │ ──────────────────────▶ │  CoreDNS   │  │
│  │    Pod      │ ◀────────────────────── │  (kube-dns)│  │
│  └─────────────┘    "10.96.45.12"        └────────────┘  │
│                                                          │
│  ┌──────────────────────────────────────┐               │
│  │  Service: todo-mongo-svc             │               │
│  │  ClusterIP: 10.96.45.12             │               │
│  └──────────────────────────────────────┘               │
└──────────────────────────────────────────────────────────┘
```

### The Full DNS Name Pattern

Every Service in Kubernetes automatically gets a DNS entry following this exact pattern:

```
<service-name>.<namespace>.svc.cluster.local
```

For our To-Do app:

| Short name | Full DNS name |
|------------|---------------|
| `todo-mongo-svc` | `todo-mongo-svc.default.svc.cluster.local` |
| `todo-backend-svc` | `todo-backend-svc.default.svc.cluster.local` |
| `todo-backend-svc` (from staging ns) | `todo-backend-svc.production.svc.cluster.local` |

### Why Short Names Work

Every Pod has a `/etc/resolv.conf` injected by Kubernetes:

```
nameserver 10.96.0.10          # CoreDNS ClusterIP
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

The `search` line means when you query `todo-mongo-svc`, the resolver automatically tries:
1. `todo-mongo-svc.default.svc.cluster.local` ✅ Found → returns IP

This is why short names work within the same namespace.

### Cross-Namespace Resolution

```
                   Namespace: production          Namespace: staging
                  ┌────────────────────┐         ┌────────────────────┐
                  │                    │         │                    │
                  │  backend Pod       │         │  todo-mongo-svc    │
                  │                    │         │  ClusterIP:        │
                  │  Wants MongoDB     │         │  10.96.77.33       │
                  │  in staging ns     │         │                    │
                  └────────────────────┘         └────────────────────┘
                           │
                           │  Must use FULL name:
                           │  todo-mongo-svc.staging.svc.cluster.local
                           ▼
                        CoreDNS resolves ✅
```

Short name `todo-mongo-svc` from the `production` namespace would search:
- `todo-mongo-svc.production.svc.cluster.local` ❌ Not found there

### Headless Services

A regular Service returns a single ClusterIP (load-balanced). A **Headless Service** (ClusterIP: None) returns the individual Pod IPs directly. Used by StatefulSets (Module 12) so clients can address specific replicas.

```yaml
spec:
  clusterIP: None    # Headless — DNS returns Pod IPs directly, not a VIP
```

---

## 📄 The Blueprint

No extra YAML is needed — CoreDNS is built into every cluster. But here are the key DNS records created automatically:

```
# When you create:
# Service name: todo-mongo-svc, namespace: default, ClusterIP: 10.96.45.12

# CoreDNS automatically registers:
todo-mongo-svc.default.svc.cluster.local  →  10.96.45.12

# Each Pod also gets a DNS record (format: pod-ip-dashes.namespace.pod.cluster.local):
10-244-1-5.default.pod.cluster.local     →  10.244.1.5
```

### Headless Service for StatefulSet (preview of Module 12)

```yaml
# A headless Service for stable per-Pod DNS names
apiVersion: v1
kind: Service
metadata:
  name: mongo-headless
spec:
  clusterIP: None          # This makes it headless
  selector:
    app: todo-mongo
  ports:
    - port: 27017
      targetPort: 27017

# With a StatefulSet named 'todo-mongo' and 3 replicas, CoreDNS creates:
# todo-mongo-0.mongo-headless.default.svc.cluster.local → Pod-0 IP
# todo-mongo-1.mongo-headless.default.svc.cluster.local → Pod-1 IP
# todo-mongo-2.mongo-headless.default.svc.cluster.local → Pod-2 IP
```

### CoreDNS ConfigMap (advanced customisation)

```yaml
# View the CoreDNS config
# kubectl get configmap coredns -n kube-system -o yaml

# The Corefile inside controls DNS behaviour:
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {  # Handles cluster DNS
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153           # CoreDNS metrics
        forward . /etc/resolv.conf # Forwards external DNS to node's resolver
        cache 30                   # Cache TTL in seconds
        loop
        reload
        loadbalance
    }
```

---

## ⌨️ The Commands

```bash
# ── INSPECT COREDNS ──────────────────────────────────────────────────────────

# Verify CoreDNS is running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# View CoreDNS logs (useful when DNS resolution fails)
kubectl logs -n kube-system -l k8s-app=kube-dns

# View CoreDNS configuration
kubectl get configmap coredns -n kube-system -o yaml


# ── TEST DNS RESOLUTION ──────────────────────────────────────────────────────

# Spin up a temporary Pod with DNS tools
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- sh

# Inside the Pod, run these:
nslookup todo-mongo-svc                              # Short name (same namespace)
nslookup todo-mongo-svc.default.svc.cluster.local   # Full FQDN
nslookup todo-backend-svc.production.svc.cluster.local  # Cross-namespace

# Check what DNS config is injected into Pods
kubectl exec -it <any-pod> -- cat /etc/resolv.conf

# One-liner DNS test without interactive shell
kubectl run dns-test --image=busybox:1.36 --rm --restart=Never -q -- \
  nslookup todo-mongo-svc


# ── TEST CROSS-NAMESPACE ─────────────────────────────────────────────────────

# Create a staging namespace and deploy a service there
kubectl create namespace staging
kubectl apply -f yaml-examples/todo-services.yaml -n staging

# From default namespace, test cross-namespace resolution
kubectl run dns-test --image=busybox:1.36 --rm -it --restart=Never -- \
  nslookup todo-mongo-svc.staging.svc.cluster.local


# ── DIAGNOSE DNS FAILURES ────────────────────────────────────────────────────

# Check if CoreDNS Service has a ClusterIP
kubectl get svc kube-dns -n kube-system

# Check CoreDNS endpoints (are there healthy CoreDNS Pods?)
kubectl get endpoints kube-dns -n kube-system

# Test DNS from a node directly
kubectl run dns-debug --image=nicolaka/netshoot --rm -it --restart=Never -- \
  dig todo-mongo-svc.default.svc.cluster.local
```

---

## 🔧 The Drill

### Exercise: Cross-Namespace Discovery

**Goal:** Understand exactly when short names work and when you need the full FQDN.

**Steps:**

1. Create two namespaces:
   ```bash
   kubectl create namespace app-a
   kubectl create namespace app-b
   ```

2. Deploy a simple nginx Service in `app-b`:
   ```bash
   kubectl run nginx --image=nginx -n app-b
   kubectl expose pod nginx --port=80 --name=nginx-svc -n app-b
   ```

3. From a Pod in `app-a`, try the short name:
   ```bash
   kubectl run test --image=busybox --rm -it -n app-a --restart=Never -- \
     wget -qO- nginx-svc
   # Expected: FAILS — short name only searches app-a namespace
   ```

4. Try the full FQDN:
   ```bash
   kubectl run test --image=busybox --rm -it -n app-a --restart=Never -- \
     wget -qO- nginx-svc.app-b.svc.cluster.local
   # Expected: SUCCESS — full name resolves across namespaces
   ```

**Key takeaway:** Short names work within a namespace. Cross-namespace always requires the full `<service>.<namespace>.svc.cluster.local` format.

---

### Troubleshooting: Intermittent DNS Failures in Production

Symptoms: Occasional `getaddrinfo ENOTFOUND todo-mongo-svc` errors in your backend logs.

**Common causes and fixes:**

| Cause | Fix |
|-------|-----|
| CoreDNS Pods are OOMKilled (out of memory) | Increase CoreDNS memory limits in its Deployment |
| ndots:5 causing slow resolution (5 search attempts) | Set `dnsConfig.options.ndots: 2` in your Pod spec |
| Too many DNS queries overwhelming CoreDNS | Scale CoreDNS replicas: `kubectl scale deploy/coredns -n kube-system --replicas=3` |
| NodeLocal DNSCache not enabled | Enable NodeLocal DNSCache for low-latency resolution |

---

## ✅ Module Checklist

- [ ] Explain the full DNS name format for a Kubernetes Service
- [ ] Explain why short names work within a namespace (the `search` domain)
- [ ] Use `nslookup` from inside a Pod to resolve Service names
- [ ] Successfully resolve a Service from a different namespace using the full FQDN
- [ ] Explain what a Headless Service is and when you'd use one

---

**Navigation:** [← Ingress](05-ingress.md) | [Next: Persistent Volumes →](07-persistent-volumes.md)
