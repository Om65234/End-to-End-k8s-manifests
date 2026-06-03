# ☸️ End-to-End K8s Manifests — GitOps Deployment Repo

Kubernetes manifests for the **MERN Task App** deployed via **ArgoCD** (GitOps). This repository is the **single source of truth** for the cluster state. Image tags in the deployment files are automatically updated by the Jenkins CI pipeline in [Jenkins-pipeline](https://github.com/Om65234/Jenkins-pipeline).

---

## 📐 Architecture Overview

```
Jenkins CI Pipeline (Jenkins-pipeline repo)
      │  sed replaces image tag, git push
      ▼
 This Repo (End-to-End-k8s-manifests)
      │  ArgoCD watches & auto-syncs
      ▼
  Kubernetes Cluster
  ┌──────────────────────────────────────────┐
  │  Namespace: default                      │
  │  ┌──────────┐  ┌──────────┐  ┌───────┐  │
  │  │ Frontend │  │ Backend  │  │MongoDB│  │
  │  │ Deploy   │  │ Deploy   │  │Deploy │  │
  │  │ (React)  │  │(Express) │  │       │  │
  │  └────┬─────┘  └────┬─────┘  └───┬───┘  │
  │       │             │             │      │
  │  ┌────▼─────┐  ┌────▼─────┐  ┌───▼───┐  │
  │  │  Svc:80  │  │ Svc:3500 │  │Svc:   │  │
  │  └────┬─────┘  └──────────┘  │27017  │  │
  │       │                      └───────┘  │
  │  ┌────▼─────────────────────┐           │
  │  │     NGINX Ingress         │           │
  │  │  / → frontend:80          │           │
  │  │  /api → backend:3500      │           │
  │  └───────────────────────────┘           │
  │                                          │
  │  PersistentVolumeClaim ─► MongoDB        │
  └──────────────────────────────────────────┘
        │
        ▼
  Prometheus + Grafana (Monitoring)
```

---

## 🗂️ Repository Structure

```
k8s-manifests/
├── frontend/
│   ├── frontend-deployment.yaml   # React app — 2 replicas, pulls omkar1907/mern-frontend:<tag>
│   └── frontend-service.yaml      # ClusterIP service on port 80
│
├── backend/
│   ├── backend-deployment.yaml    # Express API — 2 replicas, pulls omkar1907/mern-backend:<tag>
│   └── backend-service.yaml       # ClusterIP service on port 3500
│
├── mongodb/
│   ├── mongo-deployment.yaml      # MongoDB 5.0 — 1 replica, uses PVC for persistence
│   ├── mongo-service.yaml         # ClusterIP service on port 27017
│   ├── mongo-pvc.yaml             # PersistentVolumeClaim — 1Gi, StorageClass: gp3
│   ├── mongo-secret.yaml          # Kubernetes Secret for DB credentials
│   └── storageclass.yaml          # gp3 StorageClass using EBS CSI driver (ebs.csi.aws.com)
│
└── ingress.yaml                   # NGINX Ingress — routes / and /api (no host restriction)
```

---

## 🛠️ Tech Stack

| Component       | Technology                              |
|-----------------|-----------------------------------------|
| Container Orch  | Kubernetes (K8s)                                    |
| GitOps CD       | ArgoCD                                              |
| Ingress         | NGINX Ingress Controller                            |
| Frontend Image  | `omkar1907/mern-frontend:<tag>` (React + Nginx)     |
| Backend Image   | `omkar1907/mern-backend:<tag>` (Node.js + Express)  |
| Database        | MongoDB 5.0                                         |
| Storage         | gp3 PVC via **EBS CSI Driver** (`ebs.csi.aws.com`)  |
| Monitoring      | Prometheus + Grafana                                |

---

## 📄 Manifest Details

### Frontend

**`frontend/frontend-deployment.yaml`**
- Image: `omkar1907/mern-frontend:<tag>` *(tag auto-updated by Jenkins)*
- Replicas: configurable (default 1)
- Container port: `80`

**`frontend/frontend-service.yaml`**
- Type: `ClusterIP`
- Port: `80`

---

### Backend

**`backend/backend-deployment.yaml`**
- Image: `omkar1907/mern-backend:<tag>` *(tag auto-updated by Jenkins)*
- Replicas: `2`
- Container port: `3500`
- Env vars injected:
  - `PORT=3500`
  - `MONGO_CONN_STR=mongodb://root:<password>@mongodb:27017/tasks?authSource=admin`
  - `USE_DB_AUTH=true`

**`backend/backend-service.yaml`**
- Type: `ClusterIP`
- Port: `3500`

---

### MongoDB

**`mongodb/mongo-deployment.yaml`**
- Image: `mongo:5.0`
- Replicas: `1`
- Credentials loaded from `mongo-secret` Kubernetes Secret
- Data persisted via `mongo-pvc` PersistentVolumeClaim mounted at `/data/db`

**`mongodb/mongo-pvc.yaml`**
- StorageClass: `gp3` (uses EBS CSI driver — required for EKS 1.23+)
- Access mode: `ReadWriteOnce`
- Size: `1Gi`
- Stores MongoDB data across pod restarts

**`mongodb/storageclass.yaml`** *(new)*
- Name: `gp3`
- Provisioner: `ebs.csi.aws.com` (AWS EBS CSI Driver)
- Volume type: `gp3`, fsType: `ext4`
- `volumeBindingMode: WaitForFirstConsumer` — volume created only when pod is scheduled

**`mongodb/mongo-secret.yaml`**
- Holds base64-encoded `mongo-root-username` and `mongo-root-password`
- ⚠️ **Do not commit real credentials in plain text** — use Sealed Secrets or an external secrets manager in production

**`mongodb/mongo-service.yaml`**
- Type: `ClusterIP`
- Port: `27017`

---

### Ingress

**`ingress.yaml`**
- Class: `nginx`
- No host restriction — accepts requests from any hostname / IP (works with any ALB or LoadBalancer)
- Routes:
  - `/` → `frontend:80`
  - `/api` → `backend:3500`

---

## 🚀 Deployment

### Prerequisites

- A running Kubernetes cluster (EKS recommended — K8s 1.23+)
- `kubectl` configured to point to the cluster
- NGINX Ingress Controller installed
- **AWS EBS CSI Driver** installed (`aws-ebs-csi-driver` addon) with `AmazonEBSCSIDriverPolicy` on node IAM role
- ArgoCD installed and configured

### Manual Apply (without ArgoCD)

```bash
# 0. Apply StorageClass first (gp3 for EBS CSI)
kubectl apply -f mongodb/storageclass.yaml

# 1. Apply secrets
kubectl apply -f mongodb/mongo-secret.yaml

# 2. Apply storage
kubectl apply -f mongodb/mongo-pvc.yaml

# 3. Deploy MongoDB
kubectl apply -f mongodb/mongo-deployment.yaml
kubectl apply -f mongodb/mongo-service.yaml

# 4. Deploy Backend
kubectl apply -f backend/backend-deployment.yaml
kubectl apply -f backend/backend-service.yaml

# 5. Deploy Frontend
kubectl apply -f frontend/frontend-deployment.yaml
kubectl apply -f frontend/frontend-service.yaml

# 6. Apply Ingress
kubectl apply -f ingress.yaml

# Verify all pods are running
kubectl get pods
kubectl get svc
kubectl get ingress
kubectl get pvc
```

### Via ArgoCD (GitOps — Recommended)

ArgoCD is configured to watch this repository. Any push to `main` (e.g., Jenkins updating an image tag) automatically triggers a sync:

1. **Create ArgoCD Application** (one-time setup):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mern-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Om65234/End-to-End-k8s-manifests.git
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
kubectl apply -f argocd-app.yaml
```

2. **Watch ArgoCD sync** in the UI or via CLI:
```bash
argocd app get mern-app
argocd app sync mern-app
```

---

## 🔄 GitOps Image Update Flow

Every Jenkins build automatically:

1. Clones this repo
2. Runs `sed` to replace image tags:
   ```bash
   sed -i "s|image: omkar1907/mern-frontend:.*|image: omkar1907/mern-frontend:v<BUILD_NUMBER>|" frontend/frontend-deployment.yaml
   sed -i "s|image: omkar1907/mern-backend:.*|image: omkar1907/mern-backend:v<BUILD_NUMBER>|" backend/backend-deployment.yaml
   ```
3. Commits and pushes → ArgoCD detects the diff and rolls out the new version

---

## 📊 Monitoring — Prometheus & Grafana

The cluster is monitored using **Prometheus** (metrics collection) and **Grafana** (dashboards).

### What's Monitored

| Source | Metrics |
|--------|---------|
| Kubernetes Nodes | CPU, Memory, Disk, Network |
| Pods / Deployments | Restarts, resource usage, scheduling |
| NGINX Ingress | Request rate, error rate, latency |
| MongoDB (optional) | Connections, ops/sec via `mongodb-exporter` |

### Accessing Grafana
```bash
# Port-forward Grafana
kubectl port-forward svc/grafana 3000:80 -n monitoring
# Open: http://localhost:3000
# Default login: admin / prom-operator
```

### Accessing Prometheus
```bash
kubectl port-forward svc/prometheus-operated 9090:9090 -n monitoring
# Open: http://localhost:9090
```

### Useful PromQL Queries

```promql
# Pod restart count
kube_pod_container_status_restarts_total{namespace="default"}

# CPU usage by pod
rate(container_cpu_usage_seconds_total{namespace="default"}[5m])

# Memory usage by pod
container_memory_usage_bytes{namespace="default"}

# NGINX request rate
rate(nginx_ingress_controller_requests[2m])
```

---

## 🔍 Troubleshooting

```bash
# Check pod status
kubectl get pods -o wide

# Check pod logs
kubectl logs deployment/backend
kubectl logs deployment/frontend
kubectl logs deployment/mongodb

# Describe pod for events
kubectl describe pod <pod-name>

# Check ingress
kubectl describe ingress mern-ingress

# Exec into backend pod
kubectl exec -it deployment/backend -- sh
```

---

## 🏗️ CI/CD Pipeline Reference

| Repo | Role |
|------|------|
| [Jenkins-pipeline](https://github.com/Om65234/Jenkins-pipeline) | Application code, Dockerfiles, Jenkinsfile |
| [End-to-End-k8s-manifests](https://github.com/Om65234/End-to-End-k8s-manifests) *(this repo)* | Kubernetes manifests, ArgoCD GitOps source |

---

## 👤 Author

**Omkar** · [GitHub @Om65234](https://github.com/Om65234)
