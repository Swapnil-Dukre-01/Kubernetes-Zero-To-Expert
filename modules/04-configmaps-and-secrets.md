# Module 04 — ConfigMaps & Secrets

> **Track:** 🟣 Foundation | **Difficulty:** Beginner

**Navigation:** [← Services](03-services.md) | [Next: Ingress →](05-ingress.md)

---

## ❓ The Why

Look at your Deployment YAML from Module 02. Your configuration is hardcoded directly into the spec:

```yaml
env:
  - name: MONGO_URI
    value: "mongodb://todo-mongo-svc:27017/todos"
  - name: MONGO_INITDB_ROOT_PASSWORD
    value: "password123"   # ← This is in your Git repo. This is a security incident waiting to happen.
```

This breaks two critical engineering principles:

**1. The Twelve-Factor App — Separate config from code**
Changing `LOG_LEVEL` from `info` to `debug` requires editing the YAML, committing to Git, and redeploying the entire app. The config and the app are unnecessarily coupled.

**2. Security — Secrets don't belong in Git**
Database passwords and API keys hardcoded in version-controlled YAML files are one of the most common causes of data breaches. Any engineer with repo access sees them. Any public repo exposes them to the world.

**ConfigMaps** and **Secrets** solve both problems.

---

## 💡 The Concept

### ConfigMap — Non-Sensitive Configuration

A **ConfigMap** is a Kubernetes object that stores key-value pairs of **non-sensitive** configuration data. Think of it as a shared `.env` file that lives in Kubernetes and gets injected into Pods at runtime.

```
ConfigMap: todo-config
┌─────────────────────────────────┐
│  NODE_ENV    = "production"     │
│  LOG_LEVEL   = "info"           │    ──▶  injected into ──▶  Pod (as env vars)
│  API_PORT    = "3000"           │
│  MONGO_HOST  = "todo-mongo-svc" │
└─────────────────────────────────┘
```

### Secret — Sensitive Configuration

A **Secret** is structurally identical to a ConfigMap but is intended for **sensitive data**. Kubernetes handles Secrets differently:

- Values are **base64-encoded** (not plain text in the API)
- They are kept out of server logs
- Access can be restricted with RBAC (Module 08)
- They can be **encrypted at rest** using KMS in production clusters

> ⚠️ **Important:** Base64 is **encoding**, not **encryption**. Anyone with `kubectl get secret` access can decode them. For true secrets management, integrate with HashiCorp Vault, AWS Secrets Manager, or Sealed Secrets.

### Two Ways to Use Them in Pods

| Method | How | Best For |
|--------|-----|----------|
| `envFrom` | Inject ALL keys as env vars at once | When you want the whole ConfigMap/Secret |
| `env.valueFrom` | Inject individual keys selectively | When you only need specific keys |
| Volume mount | Mount as files inside the container | Configuration files (e.g. `nginx.conf`) |

---

## 📄 The Blueprint

### `yaml-examples/todo-config.yaml`

```yaml
# ConfigMap for non-sensitive To-Do app configuration

apiVersion: v1
kind: ConfigMap

metadata:
  name: todo-config          # Referenced by name in Deployment

data:                        # Plain key-value pairs (no encoding needed)
  NODE_ENV: "production"
  LOG_LEVEL: "info"
  API_PORT: "3000"
  MONGO_HOST: "todo-mongo-svc"   # Service name, not an IP (stable DNS name)
  MONGO_PORT: "27017"
  MONGO_DB: "todos"

---
# Secret for sensitive To-Do app credentials

apiVersion: v1
kind: Secret

metadata:
  name: todo-secrets

type: Opaque               # Generic key-value Secret (most common type)
                           # Other types: kubernetes.io/tls, kubernetes.io/dockerconfigjson

data:                      # Values MUST be base64-encoded
  # To encode: echo -n 'your-value' | base64
  # To decode: echo 'encoded' | base64 --decode

  MONGO_USERNAME: YWRtaW4=              # "admin"
  MONGO_PASSWORD: c3VwZXJzZWNyZXQ=     # "supersecret"
  JWT_SECRET: bXlqd3RzZWNyZXRrZXk=    # "myjwtsecretkey"
```

### Updated Deployment — Consuming ConfigMap & Secret

```yaml
# Updated section of todo-backend-deployment.yaml
# Shows how to consume ConfigMap and Secret

spec:
  template:
    spec:
      containers:
        - name: backend-api
          image: myrepo/todo-backend:v1.0.0

          # ── Method 1: Inject ALL keys from ConfigMap and Secret ──────────
          envFrom:
            - configMapRef:
                name: todo-config      # Every key in the ConfigMap becomes an env var
            - secretRef:
                name: todo-secrets     # Every key in the Secret becomes an env var

          # ── Method 2: Inject individual keys selectively ─────────────────
          env:
            - name: MONGO_URI
              value: "mongodb://$(MONGO_USERNAME):$(MONGO_PASSWORD)@$(MONGO_HOST):$(MONGO_PORT)/$(MONGO_DB)"
              # You can reference other env vars using $(VAR_NAME) syntax

          # ── Method 3: Mount as a file volume ─────────────────────────────
          # (Useful for config files like nginx.conf or app.properties)
          volumeMounts:
            - name: app-config-vol
              mountPath: /etc/config   # Each ConfigMap key becomes a file here

      volumes:
        - name: app-config-vol
          configMap:
            name: todo-config
```

---

## ⌨️ The Commands

```bash
# ── CREATE ──────────────────────────────────────────────────────────────────

# From a YAML file
kubectl apply -f yaml-examples/todo-config.yaml

# Create a ConfigMap directly from literal values (no YAML file needed)
kubectl create configmap todo-config \
  --from-literal=NODE_ENV=production \
  --from-literal=LOG_LEVEL=info

# Create a ConfigMap from a .env file
kubectl create configmap todo-config --from-env-file=.env

# Create a Secret directly from literal values (Kubernetes base64-encodes automatically)
kubectl create secret generic todo-secrets \
  --from-literal=MONGO_PASSWORD=supersecret \
  --from-literal=JWT_SECRET=myjwtsecretkey


# ── INSPECT ─────────────────────────────────────────────────────────────────

kubectl get configmaps
kubectl get cm                          # Short form

kubectl describe configmap todo-config  # Shows all keys and values in plain text
kubectl get configmap todo-config -o yaml

kubectl get secrets
kubectl describe secret todo-secrets    # Keys are shown, but VALUES are hidden

# Decode a specific Secret value
kubectl get secret todo-secrets \
  -o jsonpath='{.data.MONGO_PASSWORD}' | base64 --decode

# View all data in a Secret (decoded)
kubectl get secret todo-secrets -o json | \
  python3 -c "import sys,json,base64; \
  [print(k,'=',base64.b64decode(v).decode()) \
  for k,v in json.load(sys.stdin)['data'].items()]"


# ── UPDATE ──────────────────────────────────────────────────────────────────

# Edit a ConfigMap live in your editor
kubectl edit configmap todo-config

# After editing a ConfigMap, restart the Deployment to pick up new values
# (env vars are only injected at Pod start time)
kubectl rollout restart deployment/todo-backend

# Verify the new value is inside the Pod
kubectl exec -it <pod-name> -- env | grep LOG_LEVEL


# ── DELETE ──────────────────────────────────────────────────────────────────

kubectl delete configmap todo-config
kubectl delete secret todo-secrets
```

---

## 🔧 The Drill

### Exercise: Config Without Rebuild

The goal: change a config value without touching the application image.

**Steps:**

1. Deploy the backend with `LOG_LEVEL=info` in the ConfigMap
2. Verify it's set inside the Pod:
   ```bash
   kubectl exec -it <pod-name> -- env | grep LOG_LEVEL
   # Expected: LOG_LEVEL=info
   ```

3. Edit the ConfigMap to change `LOG_LEVEL` to `debug`:
   ```bash
   kubectl edit configmap todo-config
   # Change: LOG_LEVEL: "info"  →  LOG_LEVEL: "debug"
   ```

4. Restart the Deployment to pick up the change:
   ```bash
   kubectl rollout restart deployment/todo-backend
   ```

5. Verify inside a new Pod:
   ```bash
   kubectl exec -it <new-pod-name> -- env | grep LOG_LEVEL
   # Expected: LOG_LEVEL=debug
   ```

**The image didn't change. Only the config changed.** This is config/code separation in action.

---

### Troubleshooting: Secret Value Not Appearing in Pod

You set a Secret value but the env var inside the Pod is empty or wrong.

**Checklist:**

1. Is the Secret name in `secretRef.name` exactly matching the actual Secret name?
   ```bash
   kubectl get secrets   # Compare names carefully
   ```

2. Is the key name in the Secret matching what your app expects?
   ```bash
   kubectl describe secret todo-secrets   # Check key names
   ```

3. Did you restart the Pod after updating the Secret? (Env vars are **not** hot-reloaded)
   ```bash
   kubectl rollout restart deployment/todo-backend
   ```

4. Was the base64 encoding correct? Test it:
   ```bash
   echo -n 'your-value' | base64        # Encode (note the -n flag — no newline!)
   echo 'encoded-value' | base64 --decode
   ```

<details>
<summary>💡 The -n flag is critical</summary>

`echo 'value' | base64` encodes `value\n` (with a newline).
`echo -n 'value' | base64` encodes `value` (without a newline).

Kubernetes expects `-n`. Encoding with a trailing newline is one of the most common Secret bugs.

</details>

---

## ✅ Module Checklist

- [ ] Explain the difference between ConfigMap and Secret, and when to use each
- [ ] Create a ConfigMap from YAML and from `--from-literal`
- [ ] Consume a ConfigMap using `envFrom` and individual `valueFrom`
- [ ] Decode a Secret value using `kubectl` and `base64 --decode`
- [ ] Demonstrate that changing a ConfigMap + rolling restart updates app config without a rebuild

---

**Navigation:** [← Services](03-services.md) | [Next: Ingress →](05-ingress.md)
