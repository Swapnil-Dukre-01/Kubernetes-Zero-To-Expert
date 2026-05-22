# Module 11 — Helm: The Package Manager

> **Track:** 🔴 Ops & Advanced | **Difficulty:** Intermediate

**Navigation:** [← Health Checks](10-health-checks.md) | [Next: StatefulSets & DaemonSets →](12-statefulsets-and-daemonsets.md)

---

## ❓ The Why

Count the YAML files you've created so far for the To-Do app:

```
todo-backend-pod.yaml
todo-mongo-pod.yaml
todo-backend-deployment.yaml
todo-frontend-deployment.yaml
todo-mongo-deployment.yaml
todo-services.yaml
todo-config.yaml
todo-ingress.yaml
todo-storage.yaml
todo-rbac.yaml
todo-hpa.yaml
todo-probes.yaml
```

Now you need to deploy this to **staging** and **production** — two environments with different image tags, replica counts, hostnames, resource limits, and Secrets. Your options without Helm:

- **Copy-paste all 12 files** for each environment and change values manually → error-prone, hard to diff, nightmare to update
- **Use `sed` scripts** to replace values → brittle, unreadable shell scripts
- **Hope nobody forgets** to update the production image tag → they will

Helm gives you a proper package manager: **templates + values = rendered YAML**. One chart, infinite environments.

---

## 💡 The Concept

### Physical Analogy: Flat-Pack Furniture

Helm is IKEA for Kubernetes:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Helm Chart                               │
│                    (the flat-pack kit)                          │
│                                                                 │
│  templates/                    values.yaml                      │
│  ├── deployment.yaml           ├── replicaCount: 3             │
│  ├── service.yaml              ├── image.tag: "v1.0.0"         │
│  ├── ingress.yaml    +         ├── ingress.host: todo.com      │
│  ├── configmap.yaml            └── mongodb.storage: 5Gi        │
│  └── hpa.yaml                                                  │
└─────────────────────────┬───────────────────────────────────────┘
                          │  helm install
                          │  (assembly instructions)
                          ▼
              ┌───────────────────────┐
              │   Release in cluster  │
              │  (assembled furniture)│
              └───────────────────────┘
```

**Key concepts:**

| Term | Meaning |
|------|---------|
| **Chart** | The package — a directory of templates + default values |
| **Values** | Configuration inputs — can be overridden per environment |
| **Release** | One deployed instance of a Chart in the cluster |
| **Repository** | A collection of published Charts (like npm registry) |
| **Revision** | Each `helm upgrade` creates a new revision — enabling rollbacks |

### How Templates Work

A Helm template is a YAML file with Go template directives:

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-backend        # → "todo-production-backend"
spec:
  replicas: {{ .Values.replicaCount }}     # → 3 (from values.yaml)
  template:
    spec:
      containers:
        - name: backend
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          {{- if .Values.hpa.enabled }}     # Conditional block
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- end }}
```

---

## 📄 The Blueprint

### Chart Directory Structure

```
todo-app/                     # Chart root directory
├── Chart.yaml                # Chart metadata (name, version, description)
├── values.yaml               # Default values (overridden per environment)
├── values-staging.yaml       # Staging overrides
├── values-production.yaml    # Production overrides
└── templates/                # All the YAML templates
    ├── _helpers.tpl          # Reusable template snippets
    ├── deployment-backend.yaml
    ├── deployment-frontend.yaml
    ├── deployment-mongo.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    ├── hpa.yaml
    └── NOTES.txt             # Printed after helm install succeeds
```

### `Chart.yaml`

```yaml
apiVersion: v2               # Helm 3 chart format
name: todo-app
description: A three-tier To-Do List application on Kubernetes
type: application            # "application" or "library" (reusable helpers)

version: 1.0.0               # Chart version — bump this when the chart changes
appVersion: "2.1.0"          # Your application version — informational
```

### `values.yaml`

```yaml
# Default values — override per environment with -f values-<env>.yaml

replicaCount:
  frontend: 2
  backend: 3

image:
  frontend:
    repository: myrepo/todo-frontend
    tag: "v2.1.0"
    pullPolicy: IfNotPresent
  backend:
    repository: myrepo/todo-backend
    tag: "v2.1.0"
    pullPolicy: IfNotPresent
  mongo:
    repository: mongo
    tag: "6.0"

service:
  frontend:
    type: ClusterIP
    port: 80
  backend:
    type: ClusterIP
    port: 80

ingress:
  enabled: true
  className: nginx
  host: todo.example.com
  tls: false
  tlsSecret: ""

mongodb:
  storage: 5Gi
  storageClass: standard

hpa:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  cpuTarget: 70

resources:
  backend:
    requests:
      memory: "128Mi"
      cpu: "100m"
    limits:
      memory: "256Mi"
      cpu: "500m"
  frontend:
    requests:
      memory: "64Mi"
      cpu: "50m"
    limits:
      memory: "128Mi"
      cpu: "200m"

config:
  nodeEnv: production
  logLevel: info
```

### `values-staging.yaml` (overrides)

```yaml
# Only override what's different in staging
replicaCount:
  frontend: 1
  backend: 1                 # Save resources in staging

image:
  backend:
    tag: "v2.2.0-rc1"        # Test the release candidate in staging

ingress:
  host: staging.todo.example.com

hpa:
  enabled: false             # No autoscaling needed in staging

resources:
  backend:
    requests:
      memory: "64Mi"         # Lower resource requests in staging
      cpu: "50m"

config:
  logLevel: debug            # More verbose logging in staging
```

### `templates/deployment-backend.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-backend
  labels:
    {{- include "todo-app.labels" . | nindent 4 }}   # Reusable label helper
spec:
  replicas: {{ .Values.replicaCount.backend }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ .Release.Name }}
      app.kubernetes.io/component: backend
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ .Release.Name }}
        app.kubernetes.io/component: backend
    spec:
      containers:
        - name: backend-api
          image: "{{ .Values.image.backend.repository }}:{{ .Values.image.backend.tag }}"
          imagePullPolicy: {{ .Values.image.backend.pullPolicy }}
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              value: {{ .Values.config.nodeEnv | quote }}
            - name: LOG_LEVEL
              value: {{ .Values.config.logLevel | quote }}
          resources:
            {{- toYaml .Values.resources.backend | nindent 12 }}
      {{- if .Values.hpa.enabled }}
      # HPA manages replicas — don't set this explicitly when HPA is enabled
      {{- end }}

---
{{- if .Values.hpa.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ .Release.Name }}-backend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ .Release.Name }}-backend
  minReplicas: {{ .Values.hpa.minReplicas }}
  maxReplicas: {{ .Values.hpa.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.hpa.cpuTarget }}
{{- end }}
```

### `templates/_helpers.tpl`

```
{{/*
Common labels applied to all resources in this chart.
*/}}
{{- define "todo-app.labels" -}}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/name: {{ .Release.Name }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
```

---

## ⌨️ The Commands

```bash
# ── INSTALL HELM ─────────────────────────────────────────────────────────────

# macOS
brew install helm

# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
helm version


# ── REPOSITORIES ────────────────────────────────────────────────────────────

# Add popular repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add jetstack https://charts.jetstack.io        # cert-manager
helm repo update                                          # Refresh repo indices

# Search for a chart
helm search repo mongodb
helm search hub nginx                                     # Search Artifact Hub


# ── DEVELOP & VALIDATE YOUR CHART ───────────────────────────────────────────

# Create a new chart scaffold
helm create todo-app

# Validate template syntax and render locally (no cluster needed)
helm template todo-production ./todo-app -f values-production.yaml

# Validate against the cluster API (dry run)
helm install todo-production ./todo-app \
  -f values-production.yaml \
  --dry-run \
  --debug


# ── INSTALL ──────────────────────────────────────────────────────────────────

# Install a release (local chart)
helm install todo-staging ./todo-app \
  -f values-staging.yaml \
  --namespace staging \
  --create-namespace

# Install a release (from repository)
helm install todo-mongo bitnami/mongodb \
  --set auth.rootPassword=supersecret \
  --namespace production


# ── UPGRADE ──────────────────────────────────────────────────────────────────

# Upgrade a release with new values or chart changes
helm upgrade todo-staging ./todo-app -f values-staging.yaml -n staging

# Upgrade and install if release doesn't exist
helm upgrade --install todo-production ./todo-app \
  -f values-production.yaml \
  -n production \
  --create-namespace

# Just update the image tag
helm upgrade todo-production ./todo-app \
  --reuse-values \
  --set image.backend.tag=v2.2.0 \
  -n production


# ── INSPECT ─────────────────────────────────────────────────────────────────

# List all releases across all namespaces
helm list --all-namespaces
helm ls -A

# Show the status and notes of a release
helm status todo-production -n production

# Show what values a release is using
helm get values todo-production -n production

# Show the rendered YAML for a deployed release
helm get manifest todo-production -n production

# Show revision history
helm history todo-production -n production


# ── ROLLBACK ─────────────────────────────────────────────────────────────────

# Roll back to the previous revision
helm rollback todo-production -n production

# Roll back to a specific revision number
helm rollback todo-production 3 -n production


# ── UNINSTALL ────────────────────────────────────────────────────────────────

# Uninstall a release (deletes all resources it created)
helm uninstall todo-staging -n staging
```

---

## 🔧 The Drill

### Exercise: Multi-Environment Deployment

**Goal:** Deploy the same chart to both `staging` and `production` with different configs.

**Step 1 — Create the chart scaffold:**
```bash
helm create todo-app
```

**Step 2 — Edit `values.yaml`** with the default values shown in The Blueprint above.

**Step 3 — Create `values-staging.yaml`** with staging overrides (1 replica, debug logging, rc image tag).

**Step 4 — Deploy staging:**
```bash
helm install todo-staging ./todo-app \
  -f values-staging.yaml \
  --namespace staging \
  --create-namespace \
  --dry-run   # Preview first
```

```bash
helm install todo-staging ./todo-app \
  -f values-staging.yaml \
  --namespace staging
```

**Step 5 — Deploy production:**
```bash
helm install todo-production ./todo-app \
  --namespace production \
  --create-namespace
```

**Step 6 — Verify both releases:**
```bash
helm list -A
kubectl get pods -n staging
kubectl get pods -n production
```

**Step 7 — Simulate a production upgrade:**
```bash
helm upgrade todo-production ./todo-app \
  --reuse-values \
  --set image.backend.tag=v2.2.0 \
  -n production

helm history todo-production -n production   # See revision 2

# If v2.2.0 is broken:
helm rollback todo-production -n production  # Back to revision 1
```

---

## ✅ Module Checklist

- [ ] Explain the difference between a Chart, a Release, and a Revision
- [ ] Create a chart with `helm create` and understand the scaffold structure
- [ ] Write `values.yaml` with nested config for replicas, images, and ingress
- [ ] Use `helm template` to render and inspect the generated YAML locally
- [ ] Deploy to two environments with different value files
- [ ] Roll back a release to a previous revision

---

**Navigation:** [← Health Checks](10-health-checks.md) | [Next: StatefulSets & DaemonSets →](12-statefulsets-and-daemonsets.md)
