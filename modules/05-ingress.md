# Module 05 — Ingress

> **Track:** 🟢 Networking | **Difficulty:** Intermediate

**Navigation:** [← ConfigMaps & Secrets](04-configmaps-and-secrets.md) | [Next: DNS & Service Discovery →](06-dns-and-service-discovery.md)

---

## ❓ The Why

In Module 03, we used a `LoadBalancer` Service for external access. This works, but it has a serious problem in the real world:

**Each `LoadBalancer` Service provisions a separate cloud load balancer** — which costs money and requires a separate public IP.

If your To-Do app has a frontend, a backend API, and an admin dashboard, you now need **3 separate load balancers** and **3 separate IPs**. Users have to remember different addresses for each.

| Without Ingress | With Ingress |
|-----------------|--------------|
| `34.1.2.3` → frontend | `todo.com/` → frontend |
| `34.1.2.4` → backend API | `todo.com/api` → backend API |
| `34.1.2.5` → admin dashboard | `admin.todo.com` → admin dashboard |
| 3 load balancers, 3 IPs | **1 load balancer, 1 IP** |

An Ingress also gives you TLS termination (HTTPS), host-based routing, path-based routing, and rate limiting — all in one place.

---

## 💡 The Concept

### Physical Analogy: A Hotel Concierge

An **Ingress** is the hotel concierge at the single front entrance:

```
                          ┌─────────────────────────────────┐
                          │         Ingress Resource         │
                          │         (The Rulebook)           │
                          │                                  │
  Internet ──────────────▶│  todo.com/      → frontend-svc  │
  (one entry point)       │  todo.com/api   → backend-svc   │
                          │  admin.todo.com → admin-svc      │
                          └───────────────┬─────────────────┘
                                          │ reads rules
                                          ▼
                          ┌─────────────────────────────────┐
                          │      Ingress Controller          │
                          │   (Nginx / Traefik / HAProxy)    │
                          │       The actual concierge       │
                          └─────────────────────────────────┘
```

> **Critical distinction:** The **Ingress resource** is just the rulebook (YAML). The **Ingress Controller** is the actual software that reads and enforces those rules. You must install a controller separately — it doesn't come with Kubernetes by default.

### Popular Ingress Controllers

| Controller | Use Case |
|------------|----------|
| **Nginx Ingress** | Most popular, general purpose |
| **Traefik** | Cloud-native, great for dynamic environments |
| **AWS ALB Ingress** | Native AWS Application Load Balancer integration |
| **GKE Ingress** | Native Google Cloud integration |

### Routing Types

**Path-based routing:** Same hostname, different paths

```
todo.com/         → todo-frontend-svc:80
todo.com/api      → todo-backend-svc:80
todo.com/metrics  → prometheus-svc:9090
```

**Host-based routing:** Different hostnames

```
todo.com          → todo-frontend-svc:80
api.todo.com      → todo-backend-svc:80
admin.todo.com    → admin-svc:80
```

---

## 📄 The Blueprint

### `yaml-examples/todo-ingress.yaml`

```yaml
# Ingress for the To-Do application
# Requires an Ingress Controller to be installed first

apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: todo-ingress
  annotations:
    # Annotations configure Nginx-specific behaviour
    # These are read by the Nginx Ingress Controller, not Kubernetes itself

    nginx.ingress.kubernetes.io/rewrite-target: /$2
    # Strips the /api prefix before forwarding to the backend
    # e.g. todo.com/api/todos → backend receives /todos

    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # Redirect all HTTP traffic to HTTPS automatically

    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    # Allow request bodies up to 10MB (for file uploads)

spec:
  ingressClassName: nginx          # Which Ingress Controller handles this resource

  tls:                             # TLS configuration (HTTPS)
    - hosts:
        - todo.example.com
      secretName: todo-tls-secret  # A Secret containing the TLS certificate and private key
                                   # Created by cert-manager or manually with kubectl create secret tls

  rules:
    - host: todo.example.com       # Domain name this rule applies to

      http:
        paths:
          # Path 1: API traffic → backend Service
          - path: /api(/|$)(.*)    # Regex: matches /api, /api/, /api/todos, etc.
            pathType: ImplementationSpecific
            backend:
              service:
                name: todo-backend-svc
                port:
                  number: 80

          # Path 2: All other traffic → frontend Service
          - path: /
            pathType: Prefix       # Prefix: matches / and all sub-paths
            backend:
              service:
                name: todo-frontend-svc
                port:
                  number: 80

---
# Optional: Ingress for the admin panel on a different subdomain

apiVersion: networking.k8s.io/v1
kind: Ingress

metadata:
  name: todo-admin-ingress
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: admin-basic-auth  # HTTP basic auth
    nginx.ingress.kubernetes.io/auth-realm: "Admin Area"

spec:
  ingressClassName: nginx

  rules:
    - host: admin.todo.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: todo-admin-svc
                port:
                  number: 80
```

---

## ⌨️ The Commands

```bash
# ── INSTALL INGRESS CONTROLLER (do this first) ───────────────────────────────

# Option A: Minikube (local development — easiest)
minikube addons enable ingress
minikube addons enable ingress-dns

# Option B: Helm (production clusters)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Verify the Ingress Controller is running
kubectl get pods -n ingress-nginx
kubectl get service -n ingress-nginx


# ── DEPLOY INGRESS ───────────────────────────────────────────────────────────

kubectl apply -f yaml-examples/todo-ingress.yaml


# ── INSPECT ─────────────────────────────────────────────────────────────────

# Shows: NAME | CLASS | HOSTS | ADDRESS | PORTS | AGE
kubectl get ingress

# Full details including rules, backends, and TLS config
kubectl describe ingress todo-ingress

# Watch for the ADDRESS field to populate (may take 1-2 minutes)
kubectl get ingress --watch


# ── TLS / HTTPS SETUP ────────────────────────────────────────────────────────

# Create a self-signed TLS secret for testing
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=todo.example.com/O=todo"

kubectl create secret tls todo-tls-secret \
  --key tls.key \
  --cert tls.crt

# Install cert-manager for automatic Let's Encrypt certificates (production)
helm repo add jetstack https://charts.jetstack.io
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true


# ── LOCAL TESTING (Minikube) ─────────────────────────────────────────────────

# Get Minikube's IP
minikube ip

# Add to /etc/hosts for local DNS resolution
echo "$(minikube ip) todo.example.com" | sudo tee -a /etc/hosts

# Test the routing
curl http://todo.example.com/
curl http://todo.example.com/api/health


# ── DEBUG ────────────────────────────────────────────────────────────────────

# Check Ingress Controller logs for routing decisions and errors
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Check if the Ingress Controller has picked up your Ingress resource
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller | grep todo
```

---

## 🔧 The Drill

### Scenario: 404 on `/api` Routes

Your frontend loads at `todo.example.com` but all API calls return `404`.

**Debugging checklist:**

1. **Is the Ingress Controller running?**
   ```bash
   kubectl get pods -n ingress-nginx
   # All Pods should be Running
   ```

2. **Does the Ingress have an ADDRESS?**
   ```bash
   kubectl get ingress todo-ingress
   # ADDRESS column should not be empty
   ```

3. **Do the path rules match your actual request paths?**
   ```bash
   kubectl describe ingress todo-ingress
   # Check the Rules section carefully
   ```

4. **Does the backend Service name match exactly?**
   ```bash
   kubectl get services
   # Compare with the 'service.name' in your Ingress YAML
   ```

5. **Is the backend Service port correct?**
   ```bash
   kubectl get endpoints todo-backend-svc
   # Should show Pod IPs — not <none>
   ```

6. **Check the Ingress Controller logs:**
   ```bash
   kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --tail=50
   ```

<details>
<summary>💡 Most common root cause</summary>

The `rewrite-target` annotation is misconfigured. When Nginx strips `/api` from the path, it can pass an empty path or the wrong path to the backend. Test without the rewrite annotation first, then add it back.

</details>

---

## ✅ Module Checklist

- [ ] Explain the difference between an Ingress resource and an Ingress Controller
- [ ] Install Nginx Ingress on Minikube
- [ ] Write path-based routing rules for frontend and backend
- [ ] Configure TLS termination using a Secret
- [ ] Debug a 404 using the debugging checklist above

---

**Navigation:** [← ConfigMaps & Secrets](04-configmaps-and-secrets.md) | [Next: DNS & Service Discovery →](06-dns-and-service-discovery.md)
