# 🚀 Kubernetes: Zero to Expert

> A complete, module-based Kubernetes learning guide — applied to a real **To-Do List Application** (React frontend · Node.js API · MongoDB).

[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.29+-326CE5?style=flat&logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![Modules](https://img.shields.io/badge/Modules-13-brightgreen)]()
[![Level](https://img.shields.io/badge/Level-Zero%20to%20Expert-blue)]()

---

## 🗺️ What You'll Build

Throughout this guide you'll take a simple three-tier To-Do app and learn to deploy, scale, secure, and operate it on Kubernetes — the same way real production teams do.

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│  React UI   │────▶│  Node.js API │────▶│   MongoDB    │
│  (Frontend) │     │  (Backend)   │     │  (Database)  │
└─────────────┘     └──────────────┘     └──────────────┘
```

Every module covers one concept using this same app, so you always have real, runnable context for every YAML file and command.

---

## 📚 Curriculum

### 🟣 Foundation Track

| Module | Topic | What You'll Learn |
|--------|-------|-------------------|
| [01](modules/01-pods-and-nodes) | **Pods & Nodes** | The atoms and factories of Kubernetes |
| [02](modules/02-deployments.md) | **Deployments** | Self-healing, scalable workloads |
| [03](modules/03-services.md) | **Services** | Stable networking and load balancing |
| [04](modules/04-configmaps-and-secrets.md) | **ConfigMaps & Secrets** | Separating config from code |

### 🟢 Networking Track

| Module | Topic | What You'll Learn |
|--------|-------|-------------------|
| [05](modules/05-ingress.md) | **Ingress** | Routing external HTTP traffic |
| [06](modules/06-dns-and-service-discovery.md) | **DNS & Service Discovery** | How services find each other internally |

### 🟡 Storage Track

| Module | Topic | What You'll Learn |
|--------|-------|-------------------|
| [07](modules/07-persistent-volumes.md) | **Persistent Volumes & Claims** | Durable disk storage for MongoDB |

### 🔴 Ops & Advanced Track

| Module | Topic | What You'll Learn |
|--------|-------|-------------------|
| [08](modules/08-namespaces-and-rbac.md) | **Namespaces & RBAC** | Multi-tenancy and access control |
| [09](modules/09-autoscaling.md) | **Autoscaling & Resources** | HPA, VPA, and performance tuning |
| [10](modules/10-health-checks.md) | **Health Checks & Probes** | Self-healing with liveness/readiness |
| [11](modules/11-helm.md) | **Helm** | Package manager for Kubernetes |
| [12](modules/12-statefulsets-and-daemonsets.md) | **StatefulSets & DaemonSets** | Advanced workload types |
| [13](modules/13-observability.md) | **Observability** | Logs, metrics, and tracing |

---

## 🏁 Prerequisites

- **Docker** installed and basic container knowledge
- **kubectl** — the Kubernetes CLI ([Install guide](https://kubernetes.io/docs/tasks/tools/))
- A local cluster — [Minikube](https://minikube.sigs.k8s.io/docs/start/) recommended

```bash
kubectl version --client
minikube start
kubectl get nodes
```

---

## 📁 Repository Structure

```
Kubernetes-Zero-To-Expert/
├── README.md
├── modules/
│   ├── 01-pods-and-nodes.md
│   ├── 02-deployments.md
│   ├── 03-services.md
│   ├── 04-configmaps-and-secrets.md
│   ├── 05-ingress.md
│   ├── 06-dns-and-service-discovery.md
│   ├── 07-persistent-volumes.md
│   ├── 08-namespaces-and-rbac.md
│   ├── 09-autoscaling.md
│   ├── 10-health-checks.md
│   ├── 11-helm.md
│   ├── 12-statefulsets-and-daemonsets.md
│   └── 13-observability.md
└── yaml-examples/
    ├── todo-backend-pod.yaml
    ├── todo-mongo-pod.yaml
    ├── todo-backend-deployment.yaml
    ├── todo-services.yaml
    ├── todo-config.yaml
    ├── todo-ingress.yaml
    ├── todo-storage.yaml
    ├── todo-rbac.yaml
    ├── todo-hpa.yaml
    └── todo-mongo-statefulset.yaml
```

---

## 🧭 Module Format

Every module follows this structure:

| Section | What it covers |
|---------|---------------|
| **❓ The Why** | The real problem you face before this feature |
| **💡 The Concept** | The idea explained with a physical analogy |
| **📄 The Blueprint** | Exact YAML with every line commented |
| **⌨️ The Commands** | `kubectl` commands to deploy, inspect, debug |
| **🔧 The Drill** | Hands-on exercise or troubleshooting scenario |

---

## 📖 Recommended Learning Path

```
Week 1  →  Modules 01–04  (Foundation)
Week 2  →  Modules 05–07  (Networking + Storage)
Week 3  →  Modules 08–10  (Operations)
Week 4  →  Modules 11–13  (Advanced)
```

---

<p align="center">Start with <a href="modules/01-pods-and-nodes.md"><strong>Module 01: Pods & Nodes →</strong></a></p>
