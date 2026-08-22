# ☸️ End-to-End K8s Manifests — GitOps Deployment Repo

Kubernetes manifests for the **MERN Task App** deployed via **ArgoCD** (GitOps). This repository is the **single source of truth** for the cluster state. Image tags in the deployment files are automatically updated by the Jenkins CI pipeline in [Jenkins-pipeline](https://github.com/Om65234/Jenkins-pipeline).

---

## 📐 Architecture Overview

```
Jenkins CI Pipeline (Jenkins-pipeline repo)
      │  sed replaces image tag, git push
      ▼
 This Repo (k8s manifests)
      │  ArgoCD watches & auto-syncs
      ▼
   AWS EKS Cluster
   ┌──────────────────────────────────────────────────────┐
   │  Namespace: default                                  │
   │  ┌──────────┐  ┌──────────┐  ┌───────────────────┐  │
   │  │ Frontend │  │ Backend  │  │  MongoDB           │  │
   │  │ Deploy   │  │ Deploy   │  │  StatefulSet       │  │
   │  │ 2 pods   │  │ 2 pods   │  │  1 pod + gp3 PVC  │  │
   │  └────┬─────┘  └────┬─────┘  └────────────────────┘  │
   │       └──────────────────────────────────┐           │
   │  ┌───────────────────────────────────────▼──────────┐ │
   │  │            NGINX Ingress Controller               │ │
   │  │   /         → frontend-service:80                │ │
   │  │   /api      → backend-service:3500               │ │
   │  │   /argocd   → argocd-server:80 (insecure)        │ │
   │  │   /grafana  → monitoring-grafana:80               │ │
   │  └────────────────────┬──────────────────────────────┘ │
   └───────────────────────┼──────────────────────────────┘
                           │
   ┌───────────────────────▼──────────────────────────────┐
   │           AWS Network Load Balancer (NLB)            │
   │   Single external endpoint for all services          │
   └──────────────────────────────────────────────────────┘

   ┌──────────────────────────────────────────────────────┐
   │  Namespace: monitoring                               │
   │  Prometheus ──────────► Grafana (at /grafana)        │
   │  Alertmanager          kube-state-metrics            │
   │  node-exporter (DaemonSet on all nodes)              │
   └──────────────────────────────────────────────────────┘

   ┌──────────────────────────────────────────────────────┐
   │  Namespace: argocd                                   │
   │  ArgoCD (at /argocd, --insecure mode)                │
   └──────────────────────────────────────────────────────┘
```

---

## 🗂️ Repository Structure

```
k8s/
├── frontend/
│   ├── frontend-deployment.yaml    # React app — 2 replicas, nodeSelector: workload=app
│   └── frontend-service.yaml       # ClusterIP service on port 80
│
├── backend/
│   ├── backend-deployment.yaml     # Express API — 2 replicas, nodeSelector: workload=app
│   └── backend-service.yaml        # ClusterIP service on port 3500
│
├── mongodb/
│   ├── mongo-deployment.yaml       # MongoDB 5.0 StatefulSet — PVC at /data/db
│   ├── mongo-service.yaml          # ClusterIP service on port 27017
│   ├── mongo-pvc.yaml              # PVC — 1Gi, StorageClass: mongo-gp3
│   ├── mongo-secret.yaml           # K8s Secret for DB credentials
│   └── storageclass.yaml           # mongo-gp3 StorageClass (ebs.csi.aws.com)
│
├── gp3-storageclass.yaml           # gp3 StorageClass — shared by Grafana & Prometheus
├── ingress.yaml                    # NGINX Ingress — path-based routing, no host restriction
├── argocd-values.yaml              # Helm values for ArgoCD (nodeSelector, --rootpath)
└── monitoring-values.yaml          # Helm values for kube-prometheus-stack
```

---

## 🛠️ Tech Stack

| Component        | Technology                                           |
|------------------|------------------------------------------------------|
| Container Orch   | Kubernetes 1.31 (AWS EKS)                            |
| GitOps CD        | ArgoCD (Helm install, path `/argocd`)                |
| Ingress          | NGINX Ingress Controller — path-based routing        |
| Frontend Image   | `omkar1907/mern-frontend:<tag>` (React + Nginx)      |
| Backend Image    | `omkar1907/mern-backend:<tag>` (Node.js + Express)   |
| Database         | MongoDB 5.0 (StatefulSet)                            |
| Storage          | AWS EBS gp3 via EBS CSI Driver (`ebs.csi.aws.com`)   |
| Monitoring       | kube-prometheus-stack (Prometheus + Grafana + Alertmanager) |

---

## 📄 Manifest Details

### Node Groups & Labels

The EKS cluster has **3 node groups**, each labelled for workload separation:

| Node Label | Workload |
|---|---|
| `role=monitoring` | ArgoCD, Grafana, Prometheus, Alertmanager |
| `workload=app` | Frontend, Backend, MongoDB |

> ⚠️ **Important:** Application nodes must have the `workload=app` label. If pods are `Pending`, run:
> ```bash
> kubectl label node <node-name> workload=app
> ```

### StorageClasses

Two StorageClasses are used:

| Name | Used By | File |
|---|---|---|
| `gp3` | Grafana (10Gi), Prometheus (20Gi), Alertmanager (5Gi) | `gp3-storageclass.yaml` |
| `mongo-gp3` | MongoDB (1Gi) | `mongodb/storageclass.yaml` |

Both use `provisioner: ebs.csi.aws.com` with `volumeBindingMode: WaitForFirstConsumer`.

### Ingress Routing

**`ingress.yaml`** — single NGINX Ingress, no host restriction, path-based:

| Path | Service | Namespace |
|---|---|---|
| `/` | `frontend:80` | `default` |
| `/api` | `backend:3500` | `default` |
| `/argocd` | `argocd-server:80` | `argocd` |
| `/grafana` | `monitoring-grafana:80` | `monitoring` |

> ArgoCD runs with `--insecure` + `--rootpath=/argocd`  
> Grafana runs with `serve_from_sub_path: true` + `root_url: .../grafana`

---

## 🚀 Deployment

### Prerequisites

- AWS EKS cluster (K8s 1.23+) with `kubectl` configured
- NGINX Ingress Controller installed
- AWS EBS CSI Driver installed (`aws-ebs-csi-driver` EKS addon)
- Helm 3.12+, ArgoCD installed via Helm (`argo/argo-cd`)
- kube-prometheus-stack installed via Helm (`prometheus-community/kube-prometheus-stack`)

### Step-by-Step Manual Apply

```bash
# 1. Apply StorageClasses first
kubectl apply -f gp3-storageclass.yaml
kubectl apply -f mongodb/storageclass.yaml

# 2. Apply MongoDB resources (in order)
kubectl apply -f mongodb/mongo-secret.yaml
kubectl apply -f mongodb/mongo-pvc.yaml
kubectl apply -f mongodb/mongo-deployment.yaml
kubectl apply -f mongodb/mongo-service.yaml

# 3. Label application nodes (required for nodeSelector)
kubectl label node <app-node-1> <app-node-2> workload=app

# 4. Deploy Backend & Frontend
kubectl apply -f backend/backend-deployment.yaml
kubectl apply -f backend/backend-service.yaml
kubectl apply -f frontend/frontend-deployment.yaml
kubectl apply -f frontend/frontend-service.yaml

# 5. Apply Ingress
kubectl apply -f ingress.yaml

# 6. Install ArgoCD via Helm
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd -f argocd-values.yaml

# 7. Install kube-prometheus-stack via Helm
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring -f monitoring-values.yaml

# 8. Verify everything
kubectl get pods -A
kubectl get pvc -A
kubectl get ingress -A
```

### Via ArgoCD (GitOps — Recommended)

```yaml
# argocd-app.yaml
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
argocd app get mern-app
```

---

## 🌐 Accessing Services

All services share **one Load Balancer** via path-based routing:

| Service | URL |
|---------|-----|
| MERN App | `http://<ELB>/` |
| Backend API | `http://<ELB>/api` |
| ArgoCD UI | `http://<ELB>/argocd` |
| Grafana UI | `http://<ELB>/grafana` |

**Get the ELB address:**
```bash
kubectl get svc -n ingress-nginx ingress-nginx-controller \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

**Get credentials:**
```bash
# ArgoCD admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Grafana admin password
kubectl -n monitoring get secrets monitoring-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d
```

---

## 🔄 GitOps Image Update Flow

Every Jenkins build automatically:

1. Clones this repo
2. Runs `sed` to replace image tags:
   ```bash
   sed -i "s|image: omkar1907/mern-frontend:.*|image: omkar1907/mern-frontend:v<N>|" frontend/frontend-deployment.yaml
   sed -i "s|image: omkar1907/mern-backend:.*|image: omkar1907/mern-backend:v<N>|" backend/backend-deployment.yaml
   ```
3. Commits and pushes → ArgoCD detects diff → rolling update on EKS

---

## 📊 Monitoring

Helm release name: `monitoring` (namespace: `monitoring`)

| Component | Access |
|---|---|
| Grafana | `http://<ELB>/grafana` (admin / see secret above) |
| Prometheus | `kubectl port-forward svc/prometheus-operated 9090:9090 -n monitoring` |
| Alertmanager | `kubectl port-forward svc/alertmanager-operated 9093:9093 -n monitoring` |

**Useful PromQL:**
```promql
# Pod restarts
kube_pod_container_status_restarts_total{namespace="default"}

# CPU usage per pod
rate(container_cpu_usage_seconds_total{namespace="default", container!=""}[5m])

# Memory per pod
container_memory_usage_bytes{namespace="default", container!=""}

# NGINX request rate
rate(nginx_ingress_controller_requests[2m])
```

---

## 🔍 Troubleshooting

```bash
# Pod status across all namespaces
kubectl get pods -A

# Describe a pending/failing pod
kubectl describe pod <pod-name> -n <namespace>

# Check PVC binding
kubectl get pvc -A

# Check StorageClass exists
kubectl get storageclass

# Pod logs
kubectl logs deployment/backend --tail=100
kubectl logs deployment/frontend --tail=100

# Ingress rules
kubectl describe ingress -A

# Node labels (verify workload=app)
kubectl get nodes --show-labels

# ArgoCD sync status
argocd app get mern-app
argocd app sync mern-app
```

**Common issues:**

| Symptom | Cause | Fix |
|---|---|---|
| Pod `Pending` — unbound PVC | StorageClass `gp3` or `mongo-gp3` missing | `kubectl apply -f gp3-storageclass.yaml` |
| Pod `Pending` — no matching node | Missing `workload=app` node label | `kubectl label node <name> workload=app` |
| 404 on `/argocd` or `/grafana` | ArgoCD/Grafana sub-path not configured | Check `argocd-values.yaml` for `--rootpath` and `monitoring-values.yaml` for `serve_from_sub_path` |
| Grafana redirects to `localhost` | `root_url` misconfigured | Ensure `serve_from_sub_path: true` and correct `root_url` in Helm values |

---

## 🏗️ CI/CD Pipeline Reference

| Repo | Role |
|------|------|
| [Jenkins-pipeline](https://github.com/Om65234/Jenkins-pipeline) | Application code, Dockerfiles, Jenkinsfile |
| [End-to-End-k8s-manifests](https://github.com/Om65234/End-to-End-k8s-manifests) *(this repo)* | Kubernetes manifests, ArgoCD GitOps source |

---

## 👤 Author

**Omkar** · [GitHub @Om65234](https://github.com/Om65234)
