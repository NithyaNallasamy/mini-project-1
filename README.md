# Multi-Tier Resilient Kubernetes Application & Helm Chart

[![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Helm](https://img.shields.io/badge/Helm-0F1689?style=for-the-badge&logo=Helm&logoColor=white)](https://helm.sh/)
[![Redis](https://img.shields.io/badge/redis-%23DD0031.svg?style=for-the-badge&logo=redis&logoColor=white)](https://redis.io/)
[![NGINX](https://img.shields.io/badge/nginx-%23009639.svg?style=for-the-badge&logo=nginx&logoColor=white)](https://nginx.org/)

An end-to-end cloud-native microservices architecture deployed on Kubernetes. This project transitions a distributed web application from discrete, declarative YAML specifications into a parameterized, production-ready Helm chart—incorporating container self-healing, stateful persistence, Layer 7 ingress routing, and Horizontal Pod Autoscaling (HPA).

---

## 🏛️ Architecture Overview

The application implements a decoupled two-tier architecture:
* **Frontend Tier:** Replicas running `podinfo` (Go microservice), handling HTTP client requests, exposing metrics, and reporting readiness/liveness status.
* **Backend Data Tier:** In-memory, persistent data store running `redis:7-alpine` configured with Append-Only File (AOF) logging.
* **Traffic Ingress:** Managed by NGINX Ingress Controller routing HTTP requests based on host-header rules (`myapp.local`).
* **Elasticity:** Automated horizontal scaling using the Kubernetes Metrics Server based on live CPU thresholds.

[ Client / Browser ]
                         │
                         ▼ ([http://myapp.local](http://myapp.local))
             ┌───────────────────────┐
             │ NGINX Ingress Engine  │
             └───────────┬───────────┘
                         │
                         ▼
             ┌───────────────────────┐
             │  Frontend ClusterIP   │
             └───────────┬───────────┘
                         │
        ┌────────────────┴────────────────┐
        ▼                                 ▼
┌─────────────────┐               ┌─────────────────┐
│  Frontend Pod   │◄─[ HPA Scale ]│  Frontend Pod   │
│  (podinfo:6.5.4)│               │  (podinfo:6.5.4)│
└────────┬────────┘               └────────┬────────┘
│                                 │
└────────────────┬────────────────┘
│ (CoreDNS / TCP 6379)
▼
┌───────────────────────┐
│ Redis Headless Svc    │
│   (clusterIP: None)   │
└───────────┬───────────┘
│
▼
┌───────────────────────┐
│   StatefulSet Pod     │
│      (redis-0)        │
└───────────┬───────────┘
│
▼
┌───────────────────────┐
│ Dynamic PVC (1Gi)     │
│   (Standard/Hostpath) │
└───────────────────────┘


---

## 🚀 Key Engineering Highlights

### 1. Zero-Data-Loss Database Migration (StatefulSet + PVC)
* **Problem:** Ephemeral Deployments risk complete cache/data destruction during crash restarts or node drains.
* **Resolution:** Re-architected Redis using a `StatefulSet` with `volumeClaimTemplates` (1Gi) and a headless service (`clusterIP: None`).
* **Validation:** Verified via intentional pod termination (`kubectl delete pod redis-0`); state and write logs persisted across replacement lifecycles without data degradation.

### 2. High Availability & Automated Self-Healing
* Implemented decoupled **Readiness** and **Liveness** probes:
  * Readiness guards upstream traffic via `/readyz` HTTP endpoints and Redis ping checks.
  * Liveness enforces automatic process reclamation if containers deadlock or drop TCP sockets.
* Verified Kubelet reconciliation handling under manual `SIGKILL` (PID 1 exit 137).

### 3. Layer 7 Routing & Custom Domain Resolution
* Eliminated the need for port-forwarding proxies by deploying an NGINX Ingress Controller.
* Mapped local host resolution (`myapp.local`) into cluster-assigned endpoints with sub-path routing rules.

### 4. Dynamic Metric-Based Autoscaling (HPA)
* Configured Kubernetes Metrics Server to scrape real-time container resource telemetry.
* Deployed an HPA policy automatically scaling the frontend between **2 and 6 replicas** when average CPU utilization crosses **50% of requests**.
* Validated dynamic scale-out and stabilization cooldown via continuous load generation loops using Busybox.

---

## 📂 Project Directory Structure

```text
.
├── k8s-mini-project-1/              # Helm Chart Directory
│   ├── Chart.yaml                  # Chart metadata & application version
│   ├── values.yaml                 # Centralized parameter defaults
│   └── templates/
│       ├── _helpers.tpl            # Template helper macros & label standards
│       ├── configmap.yaml          # Environment and operational configurations
│       ├── secret.yaml             # Encrypted credentials & access secrets
│       ├── redis-service.yaml      # Headless discovery service
│       ├── redis-statefulset.yaml  # Stateful workload with dynamic PVC
│       ├── frontend-service.yaml   # Internal service load balancer
│       ├── frontend-deployment.yaml# Microservice deployment specification
│       ├── ingress.yaml            # NGINX host routing configuration
│       ├── hpa.yaml                # Autoscaling metric specifications
│       └── NOTES.txt               # Post-installation client instructions
└── README.md

Installation & Deployment

Prerequisites

Kubernetes Cluster (v1.26+) via Minikube or Kind

kubectl CLI installed and authenticated

Helm v3 package manager

1. Prepare Environment & Addons
Ensure the required cluster addons are active:

# Enable Ingress & Metrics Server
minikube addons enable ingress
minikube addons enable metrics-server

2. Configure Local DNS
Map the local host domain to your cluster gateway:

# Append Minikube IP to hosts
echo "$(minikube ip) myapp.local" | sudo tee -a /etc/hosts

(If on Windows/WSL2 using Docker Desktop, execute minikube tunnel in an administrative terminal and map 127.0.0.1 myapp.local).

3. Deploy via Helm
Validate template integrity and execute deployment:

# 1. Lint chart syntax
helm lint ./k8s-mini-project-1

# 2. Deploy to a dedicated namespace
helm install mini-app ./k8s-mini-project-1 \
  --namespace mini-project-helm \
  --create-namespace


4. Verify Workload Health

kubectl get all,pvc,ingress,hpa -n mini-project-helm

Operational Runbooks & Testing
Test A: Verify Data Persistence

# Write key-value pair to Redis
kubectl exec -it redis-0 -n mini-project-helm -- redis-cli -a SuperSecureK8sPassword123! set env "production-ready"

# Terminate Pod
kubectl delete pod redis-0 -n mini-project-helm

# Confirm key survived recreation
kubectl wait --for=condition=Ready pod/redis-0 -n mini-project-helm --timeout=60s
kubectl exec -it redis-0 -n mini-project-helm -- redis-cli -a SuperSecureK8sPassword123! get env
# Output: "production-ready"

Test B: Verify Autoscaling Under Synthetic Load

# Run load-generator
kubectl run -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -n mini-project-helm \
  -- /bin/sh -c "while true; do wget -q -O- http://frontend-svc; done"

# Monitor horizontal scale-out
kubectl get hpa -n mini-project-helm -w

Step 1: .gitignore for Kubernetes & Helm

Create a .gitignore file in the root of your project directory (~/projects/my-k8s-project/.gitignore):

# Helm package archives and dependency lock files
*.tgz
*.tar.gz
k8s-mini-project-1/charts/
k8s-mini-project-1/Chart.lock

# Kubernetes & cloud provider secrets/credentials
*.kubeconfig
kubeconfig*
*secret*.yaml
*password*.txt
.env
.env.*

# Local cluster state & runtimes
.minikube/
.kube/

# OS and Editor metadata
.DS_Store
Thumbs.db
.idea/
.vscode/
*.swp
*~

# Temporary outputs and dry-run artifacts
*.log
render-debug.yaml

Why These Entries Matter:
*.tgz / Chart.lock: Prevent pre-packaged binary archives from polluting version control history.

*secret*.yaml / .env: Guardrails against accidentally committing raw secret manifests containing plain-text passwords or base64 tokens.

render-debug.yaml / *.log: Prevents verbose template debugging dumps from being tracked.

Commit the files:

git add .gitignore README.md LICENSE k8s-mini-project-1/
git commit -m "feat: complete mini-project-1 helm chart and project documentation"
