<div align="center">

# 🚀 Local GitOps Platform with Progressive Delivery

### A fully local, production-grade GitOps platform built with Kubernetes, Argo CD, Docker, and GitHub Actions

[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![ArgoCD](https://img.shields.io/badge/Argo%20CD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)](https://argoproj.github.io/cd/)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![GitHub Actions](https://img.shields.io/badge/GitHub%20Actions-2088FF?style=for-the-badge&logo=github-actions&logoColor=white)](https://github.com/features/actions)
[![Node.js](https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=node.js&logoColor=white)](https://nodejs.org/)

<br/>

> **"Git is the single source of truth. If it's not in Git, it doesn't exist."**

<br/>

[Overview](#-overview) •
[Architecture](#-architecture) •
[Prerequisites](#-prerequisites) •
[Quick Start](#-quick-start) •
[Installation](#-installation) •
[Usage](#-usage) •
[Promotion](#-promotion-guide) •
[Rollback](#-rollback-guide) •
[Troubleshooting](#-troubleshooting)

</div>

---

## 📖 Overview

This project implements a **production-grade GitOps workflow** running entirely on a local machine. Every code change flows automatically from Git through a CI/CD pipeline into a Kubernetes cluster — with **zero manual deployment commands**.

### What is GitOps?

GitOps is a set of practices where your **Git repository is the single source of truth** for both application code and infrastructure configuration. Automated tools continuously reconcile the cluster state with what is declared in Git.

| Traditional Deployment | GitOps Deployment |
|---|---|
| Developer runs `kubectl apply` manually | `git push` triggers everything automatically |
| Hard to track who deployed what | Full audit trail in Git commit history |
| Rollback requires manual commands | Rollback = `git revert` + `git push` |
| Config drift is invisible | Argo CD detects and fixes drift automatically |
| Inconsistent environments | Git guarantees identical deployments |

### ✨ Key Features

- ✅ **Automatic deployments** on every `git push` — no manual steps
- ✅ **Zero-downtime** rolling updates via Kubernetes
- ✅ **Separate environments** — `dev` (auto-deploy) and `staging` (manual promotion)
- ✅ **One-command rollback** via `git revert`
- ✅ **Visual dashboard** via Argo CD UI
- ✅ **Auto-versioning** from `package.json`
- ✅ **Health checks** built into every deployment
- ✅ **Fully local** — no cloud account needed

---

## 🏗️ Architecture

### System Overview

```
👨‍💻 Developer
     │
     │  git push (one command)
     ▼
┌─────────────────────────────────────────────────────────────┐
│                         GITHUB                              │
│                                                             │
│  ┌───────────────────────┐    ┌──────────────────────────┐  │
│  │     gitops-app        │    │    gitops-manifests      │  │
│  │   (source code)       │    │   (deployment configs)   │  │
│  │                       │    │                          │  │
│  │  src/app.js           │    │  dev/                    │  │
│  │  Dockerfile           │    │    deployment.yml        │  │
│  │  package.json         │    │    service.yml           │  │
│  │  .github/workflows/   │    │  staging/                │  │
│  │    ci.yaml            │    │    deployment.yml        │  │
│  └──────────┬────────────┘    └──────────┬───────────────┘  │
│             │ triggers                   ▲ auto-updates      │
│             ▼                           │                    │
│  ┌──────────────────────────────────────┘                    │
│  │         GitHub Actions (CI/CD Pipeline)                   │
│  │  ┌─────────────────────────────────────────────────────┐  │
│  │  │ Job 1: Test → Job 2: Build → Job 3: Deploy to Dev   │  │
│  │  └─────────────────────────────────────────────────────┘  │
└──┼─────────────────────────────────────────────────────────--┘
   │
   │ docker push
   ▼
┌──────────────────────┐
│  ghcr.io (Registry)  │
│  gitops-app:sha-xxx  │
│  gitops-app:sha-yyy  │
└──────────┬───────────┘
           │ pull image
┌──────────▼───────────────────────────────────────────────────┐
│              KUBERNETES CLUSTER (k3d — your laptop)           │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐   │
│  │                     ARGO CD                            │   │
│  │  Watches gitops-manifests → syncs cluster every 3 min  │   │
│  └──────────────────┬─────────────────────┬───────────────┘   │
│                     │                     │                    │
│         ┌───────────▼──────┐   ┌──────────▼────────┐          │
│         │  namespace: dev  │   │ namespace: staging │          │
│         │                  │   │                    │          │
│         │  auto-deploy ✅  │   │  manual promo 🔒   │          │
│         │  localhost:3000  │   │  localhost:3001    │          │
│         └──────────────────┘   └────────────────────┘          │
└───────────────────────────────────────────────────────────────┘
```

### End-to-End Flow

```
git push → GitHub Actions triggers
         → Tests run (npm test)
         → Docker image built (sha-xxxxxxx)
         → Image pushed to ghcr.io
         → dev/deployment.yml updated (image + version)
         → Argo CD detects change within 3 minutes
         → Kubernetes rolling update (zero downtime)
         → localhost:3000 shows new version ✅
```

### Repository Structure

```
gitops-project/
│
├── gitops-app/                        # 📦 Application Repository
│   ├── src/
│   │   └── app.js                     # Node.js Express API
│   ├── .github/
│   │   └── workflows/
│   │       ├── ci.yaml                # Main CI/CD pipeline
│   │       └── promote-staging.yaml   # Staging promotion pipeline
│   ├── Dockerfile                     # Container recipe
│   ├── package.json                   # App version source of truth
│   ├── .dockerignore
│   └── README.md
│
├── gitops-manifests/                  # 📋 Manifest Repository
│   ├── dev/
│   │   ├── deployment.yml             # Dev Kubernetes Deployment
│   │   └── service.yml                # Dev Service (NodePort 30000)
│   └── staging/
│       ├── deployment.yml             # Staging Kubernetes Deployment
│       └── service.yml                # Staging Service (NodePort 30001)
│
└── promote.sh                         # 🚀 Promotion script
```

---

## 🛠️ Tools & Technologies

| Tool | Version | Purpose |
|---|---|---|
| **k3d** | v5.8.3 | Lightweight Kubernetes cluster inside Docker |
| **kubectl** | v1.35.4 | Kubernetes command-line tool |
| **Argo CD** | v2.x | GitOps continuous delivery controller |
| **Helm** | v3.20.2 | Kubernetes package manager |
| **Docker** | v29.4.0 | Container build and runtime |
| **GitHub Actions** | — | Automated CI/CD pipeline |
| **ghcr.io** | — | GitHub Container Registry (free) |
| **Node.js** | 18.x Alpine | Application runtime |

---

## 📋 Prerequisites

### System Requirements

| Resource | Minimum | Recommended |
|---|---|---|
| **RAM** | 8 GB | 16 GB |
| **CPU** | 4 cores | 8 cores |
| **Disk** | 20 GB free | 40 GB free |
| **OS** | Ubuntu 22.04 / macOS 13 | Ubuntu 22.04 LTS |
| **Internet** | Required (initial setup) | Stable broadband |

### Required Accounts

- ✅ **GitHub account** (free) — [github.com](https://github.com)
- ✅ **GitHub Personal Access Token** with scopes: `repo`, `write:packages`, `delete:packages`

---

## ⚡ Quick Start

```bash
# 1. Clone both repositories
git clone https://github.com/Haseef-Ahamed/gitops-app.git
git clone https://github.com/Haseef-Ahamed/gitops-manifests.git

# 2. Create Kubernetes cluster
k3d cluster create gitops-cluster \
  --port "3000:30000@loadbalancer" \
  --port "3001:30001@loadbalancer" \
  --agents 1

# 3. Create namespaces
kubectl create namespace dev
kubectl create namespace staging

# 4. Install Argo CD
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
kubectl create namespace argocd
helm install argocd argo/argo-cd --namespace argocd

# 5. Apply Argo CD applications
kubectl apply -f argocd-dev-app.yaml
kubectl apply -f argocd-staging-app.yaml

# 6. Verify
curl http://localhost:3000/health
# → {"status":"OK","timestamp":"..."}
```

---

## 🔧 Installation

### Step 1 — Install All Tools

#### Docker

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release

sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER && newgrp docker
```

#### kubectl

```bash
cd ~
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
  https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
```

#### k3d

```bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

#### Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

#### Argo CD CLI

```bash
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

#### Verify all tools

```bash
docker --version        # Docker version 29.x.x
kubectl version --client  # Client Version: v1.35.x
k3d --version           # k3d version v5.8.x
helm version            # version.BuildInfo{Version:"v3.x.x"...}
argocd version --client # argocd: v2.x.x
git --version           # git version 2.x.x
```

---

### Step 2 — Create Kubernetes Cluster

```bash
# Create cluster with port mappings
k3d cluster create gitops-cluster \
  --port "3000:30000@loadbalancer" \
  --port "3001:30001@loadbalancer" \
  --agents 1

# Create namespaces
kubectl create namespace dev
kubectl create namespace staging
```

**Verify:**
```bash
kubectl get nodes
# NAME                          STATUS   ROLES
# k3d-gitops-cluster-server-0   Ready    control-plane ✅
# k3d-gitops-cluster-agent-0    Ready    <none>        ✅
```

---

### Step 3 — Install Argo CD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd

# Install without --wait (avoids timeout on slower machines)
helm install argocd argo/argo-cd --namespace argocd

# Watch pods start (2-3 minutes)
kubectl get pods -n argocd -w
# Press Ctrl+C when all show: STATUS = Running
```

---

### Step 4 — Access Argo CD Dashboard

```bash
# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo ""

# Start port-forward
kubectl port-forward svc/argocd-server -n argocd 9090:443 &
sleep 2

# Login via CLI
argocd login localhost:9090 \
  --username admin \
  --password $(kubectl -n argocd get secret \
    argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d) \
  --insecure
# → 'admin:login' logged in successfully ✅
```

Open browser: **`https://localhost:9090`**
> Accept the SSL warning → click Advanced → Proceed to localhost

---

### Step 5 — Configure GitHub

#### Create Personal Access Token

```
GitHub → Settings → Developer Settings
→ Personal Access Tokens → Tokens (classic)
→ Generate new token

Name: MANIFEST_TOKEN
Scopes:
  ✅ repo (full control)
  ✅ write:packages
  ✅ delete:packages

→ Generate token → Copy and save it!
```

#### Add Secret to gitops-app Repo

```
GitHub → gitops-app → Settings
→ Secrets and variables → Actions
→ New repository secret
  Name:  MANIFEST_TOKEN
  Value: (paste your token)
→ Add secret ✅
```

---

### Step 6 — Apply Argo CD Applications

```bash
cd ~/gitops-project

# Apply both applications
kubectl apply -f argocd-dev-app.yaml
kubectl apply -f argocd-staging-app.yaml

# Verify
argocd app list
```

Expected:
```
NAME                 CLUSTER     NAMESPACE  STATUS  HEALTH
gitops-app-dev       in-cluster  dev        Synced  Healthy ✅
gitops-app-staging   in-cluster  staging    Synced  Healthy ✅
```

---

### Step 7 — Make Package Public on ghcr.io

After your first pipeline run:

```
GitHub → Your Profile → Packages tab
→ Click "gitops-app"
→ Package settings (right side)
→ Scroll to "Danger Zone"
→ Change visibility → Public
→ Type "gitops-app" to confirm
→ Click confirm ✅
```

---

## 🎯 Usage

### API Endpoints

| Endpoint | Method | Response |
|---|---|---|
| `/` | GET | `{"message":"GitOps Demo App is running!","version":"vX.X.X"}` |
| `/health` | GET | `{"status":"OK","timestamp":"2026-..."}` |
| `/version` | GET | `{"version":"vX.X.X","environment":"dev"}` |

```bash
# Dev environment
curl http://localhost:3000/health
curl http://localhost:3000/version

# Staging environment
curl http://localhost:3001/health
curl http://localhost:3001/version
```

---

### Making a Code Change (Full GitOps Loop)

```bash
# Step 1: Update version in package.json
cd ~/gitops-project/gitops-app
nano package.json
# Change: "version": "1.0.0" → "2.0.0"

# Step 2: Update fallback in app.js
nano src/app.js
# Change: v1.0.0 → v2.0.0

# Step 3: Push — this is ALL you do
git add .
git commit -m "feat: release v2.0.0"
git push origin main
```

**What happens automatically (no more input from you):**

```
0:00  git push detected
0:05  GitHub Actions starts
0:30  npm test passes ✅
1:30  Docker image built → sha-xxxxxxx
1:45  Image pushed to ghcr.io ✅
2:00  dev/deployment.yml updated ✅
2:30  Argo CD detects manifest change
3:00  Kubernetes rolling update starts
3:30  New pod Running, old pod Terminating
4:00  curl localhost:3000/version → v2.0.0 ✅
```

---

### Every Day — Start Session Commands

```bash
# Add this function to ~/.bashrc for convenience:
cat >> ~/.bashrc << 'EOF'
gitops-start() {
  k3d cluster start gitops-cluster 2>/dev/null || true
  pkill -f "port-forward svc/argocd-server" 2>/dev/null || true
  sleep 1
  kubectl port-forward svc/argocd-server -n argocd 9090:443 &
  sleep 2
  argocd login localhost:9090 \
    --username admin \
    --password $(kubectl -n argocd get secret \
      argocd-initial-admin-secret \
      -o jsonpath="{.data.password}" | base64 -d) \
    --insecure 2>/dev/null
  echo "✅ GitOps platform ready!"
  echo "   Dev:     http://localhost:3000"
  echo "   Staging: http://localhost:3001"
  echo "   ArgoCD:  https://localhost:9090"
}
EOF
source ~/.bashrc

# Then just run:
gitops-start
```

---

## 🚀 Promotion Guide

Promotion moves a **tested version from dev to staging**. This is intentionally manual — a human must approve before staging updates.

### Using the Promotion Script

```bash
cd ~/gitops-project
./promote.sh
```

**Example session:**
```
╔══════════════════════════════════════════╗
║    GitOps Promotion — Dev → Staging      ║
╚══════════════════════════════════════════╝

📥 Pulling latest manifests...

┌─────────────────────────────────────────────┐
│              CURRENT STATE                  │
├──────────────┬──────────────────────────────┤
│ Environment  │ Image : Version              │
├──────────────┼──────────────────────────────┤
│ DEV          │ sha-abc123 : v2.0.0          │
│ STAGING      │ sha-def456 : v1.0.0          │
└──────────────┴──────────────────────────────┘

🔍 Checking dev health...
   ✅ Dev is healthy

📦 Promotion Plan:
   FROM: sha-def456 (v1.0.0)
   TO:   sha-abc123 (v2.0.0)

✋ Promote to staging? (yes/no): yes

✅ PROMOTION COMPLETE!
   Argo CD syncs in ~3 minutes
   Verify: curl http://localhost:3001/version
```

### Manual Promotion

```bash
cd ~/gitops-project/gitops-manifests
git pull origin main

# Read dev state
DEV_IMAGE=$(grep "image:" dev/deployment.yml | awk '{print $2}')
DEV_VERSION=$(grep -A1 "name: APP_VERSION" dev/deployment.yml \
  | grep value | awk '{print $2}' | tr -d '"')

# Update staging
sed -i "s|image: ghcr.io/.*/gitops-app:.*|image: $DEV_IMAGE|g" staging/deployment.yml
sed -i "/name: APP_VERSION/{n;s|value:.*|value: $DEV_VERSION|}" staging/deployment.yml

# Commit and push
git add staging/deployment.yml
git commit -m "promote(staging): $DEV_IMAGE version=$DEV_VERSION"
git push origin main

# Verify after ~3 minutes
curl http://localhost:3001/version
```

---

## ⏪ Rollback Guide

> ⚠️ **Always rollback via Git — never use `kubectl` directly.**

### Rollback Dev

```bash
cd ~/gitops-project/gitops-manifests

# See deployment history
git log --oneline dev/deployment.yml
# abc1234 ci(dev): image=sha-new version=v2.0.0  ← CURRENT (broken)
# def5678 ci(dev): image=sha-old version=v1.0.0  ← LAST GOOD

# Revert last commit
git revert HEAD --no-edit
git push origin main

# Verify after ~3 minutes
curl http://localhost:3000/version
```

### Rollback Staging

```bash
cd ~/gitops-project/gitops-manifests

# See staging history
git log --oneline staging/deployment.yml

# Revert
git revert HEAD --no-edit
git push origin main

# Verify
curl http://localhost:3001/version
```

### Rollback to Specific Version

```bash
# Find the commit you want to go back to
git log --oneline

# Revert that specific commit
git revert <commit-hash> --no-edit
git push origin main
```

### Emergency Rollback via Argo CD UI

```
1. Open https://localhost:9090
2. Click the affected application tile
3. Click "History and Rollback" tab
4. Select the last known good deployment
5. Click "Rollback"
```

> 💡 UI rollback is temporary. Always follow up with `git revert` to make it permanent.

---

## 📊 Environment Reference

| Environment | Namespace | URL | Deploy Trigger |
|---|---|---|---|
| **Dev** | `dev` | http://localhost:3000 | Automatic on `git push` |
| **Staging** | `staging` | http://localhost:3001 | Manual `./promote.sh` |
| **Argo CD** | `argocd` | https://localhost:9090 | — |

---

## 💻 Full Command Reference

### Deployment

```bash
# Deploy to dev (automatic)
git add . && git commit -m "feat: ..." && git push origin main

# Promote dev → staging
cd ~/gitops-project && ./promote.sh

# Force Argo CD sync (skip 3 min wait)
argocd app sync gitops-app-dev
argocd app sync gitops-app-staging

# Rollback last deployment
cd ~/gitops-project/gitops-manifests
git revert HEAD --no-edit && git push origin main
```

### Kubernetes

```bash
kubectl get pods -n dev                               # List dev pods
kubectl get pods -n staging                           # List staging pods
kubectl get pods -n argocd                            # List Argo CD pods
kubectl logs -n dev <pod-name>                        # View pod logs
kubectl logs -n dev <pod-name> --previous             # Crashed pod logs
kubectl describe pod -n dev <pod-name>                # Debug pod issues
kubectl rollout status deployment/gitops-app -n dev   # Watch rollout
kubectl rollout restart deployment/gitops-app -n dev  # Restart pods
kubectl get all -n dev                                # All dev resources
```

### Argo CD

```bash
argocd app list                          # List all apps
argocd app get gitops-app-dev            # Detailed app info
argocd app sync gitops-app-dev           # Force sync now
argocd app history gitops-app-dev        # Deployment history
argocd app rollback gitops-app-dev <id>  # Rollback to version
```

### Cluster

```bash
k3d cluster list                         # List clusters
k3d cluster start gitops-cluster         # Start cluster
k3d cluster stop gitops-cluster          # Stop cluster
k3d cluster delete gitops-cluster        # Delete cluster
```

---

## 🔑 CI/CD Pipeline Details

### Pipeline Structure

```
Push to main branch
       │
       ▼
┌──────────────┐   ┌──────────────────┐   ┌─────────────────────┐
│   🧪 Test    │ → │  🐳 Build+Push   │ → │  🚀 Deploy to Dev   │
│              │   │                  │   │                     │
│ npm install  │   │ docker build     │   │ Update image tag    │
│ npm test     │   │ docker push      │   │ Update APP_VERSION  │
│              │   │ → ghcr.io        │   │ git push manifests  │
└──────────────┘   └──────────────────┘   └─────────────────────┘
```

### Image Tagging

```bash
# Every build creates two tags:
ghcr.io/haseef-ahamed/gitops-app:sha-abc1234   # specific SHA (used in manifests)
ghcr.io/haseef-ahamed/gitops-app:latest         # always newest

# Manifests always use SHA tag for reproducibility:
image: ghcr.io/haseef-ahamed/gitops-app:sha-abc1234
```

### What Gets Auto-Updated in `dev/deployment.yml`

```yaml
# Both of these are updated automatically by the pipeline:
image: ghcr.io/haseef-ahamed/gitops-app:sha-abc1234   # ← new image
env:
- name: APP_VERSION
  value: "v2.0.0"                                       # ← from package.json
```

---

## 🔍 Troubleshooting

### Pod shows `ImagePullBackOff`

```bash
# See exact error
kubectl describe pod -n dev $(kubectl get pod -n dev -o name | head -1)
# Look at Events section

# Fix: Make ghcr.io package public
# GitHub → Profile → Packages → gitops-app
# → Package settings → Change visibility → Public
```

### Argo CD shows `OutOfSync`

```bash
# Force sync
argocd app sync gitops-app-dev

# If login fails, restart port-forward:
pkill -f "port-forward svc/argocd"
kubectl port-forward svc/argocd-server -n argocd 9090:443 &
sleep 2
argocd login localhost:9090 --username admin \
  --password $(kubectl -n argocd get secret \
    argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d) --insecure
```

### CI/CD fails with `403 Forbidden` on ghcr.io

```bash
# Use MANIFEST_TOKEN instead of GITHUB_TOKEN in ci.yaml:
# Find the login step and change:
password: ${{ secrets.GITHUB_TOKEN }}    # ← wrong
password: ${{ secrets.MANIFEST_TOKEN }}  # ← correct

# Also add provenance: false to build step:
uses: docker/build-push-action@v5
with:
  provenance: false   # ← add this
```

### Port already in use

```bash
# Find what's using the port
sudo lsof -i :9090

# Kill it
sudo kill -9 <PID>

# Or use a different port
kubectl port-forward svc/argocd-server -n argocd 9191:443 &
```

### `sed: can't read dev/deployment.yml: No such file or directory`

```bash
# Files not pushed yet
cd ~/gitops-project/gitops-manifests
ls dev/    # check they exist locally

git add .
git commit -m "feat: add manifests"
git push origin main
```

### Helm install shows `context deadline exceeded`

```bash
# Normal on slower machines — check if pods are running:
kubectl get pods -n argocd
# If all show Running → installation succeeded ✅

# Verify:
helm list -n argocd
# Shows: STATUS = deployed ✅
```

### Cluster not reachable after reboot

```bash
k3d cluster start gitops-cluster
sleep 30
kubectl get nodes
# Both nodes must show: STATUS = Ready
```

### promote.sh fails with `no tracking information`

```bash
cd ~/gitops-project/gitops-manifests
git branch --set-upstream-to=origin/main main
git pull origin main
```

---

## ⚠️ Common Mistakes to Avoid

| Mistake | Impact | Fix |
|---|---|---|
| Using `:latest` tag in manifests | Can't rollback reliably | Always use SHA tag: `sha-xxxxxxx` |
| Running `kubectl apply` manually | Argo CD will revert it (self-healing) | All changes through Git only |
| Using one repo for code + manifests | Mixes concerns | Keep two separate repos always |
| Not checking Argo CD sync logs | Silent failures | Check sync status after every push |
| Committing secrets to Git | Security risk | Use GitHub Secrets only |
| Not pulling before editing manifests | Merge conflicts | Always `git pull origin main` first |
| Using `GITHUB_TOKEN` for ghcr.io | 403 Forbidden error | Use `MANIFEST_TOKEN` (PAT) instead |

---

## 📐 GitOps Principles Applied

```
1. DECLARATIVE
   Everything described in YAML files
   "What should exist" not "how to create it"

2. VERSIONED AND IMMUTABLE
   All changes tracked in Git history
   Docker images tagged with commit SHA
   Complete audit trail — forever

3. PULLED AUTOMATICALLY
   Argo CD pulls from Git (cluster is never pushed to)
   More secure than push-based deployments

4. CONTINUOUSLY RECONCILED
   Argo CD checks every 3 minutes
   Manual cluster changes are automatically reverted
   Cluster ALWAYS matches what Git says
```

---

## 🎓 What You Learn From This Project

- ✅ GitOps principles and production workflow
- ✅ Kubernetes cluster setup with k3d
- ✅ Argo CD installation and configuration
- ✅ GitHub Actions CI/CD pipeline design
- ✅ Two-repository GitOps pattern
- ✅ Zero-downtime rolling deployments
- ✅ Environment separation (dev/staging)
- ✅ Instant rollback using Git history
- ✅ Docker image tagging strategies
- ✅ Kubernetes pod debugging

---

## 👤 Author

**Haseef Ahamed**

[![GitHub](https://img.shields.io/badge/GitHub-Haseef--Ahamed-181717?style=for-the-badge&logo=github)](https://github.com/Haseef-Ahamed)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0077B5?style=for-the-badge&logo=linkedin)](https://linkedin.com/in/haseef-ahamed)

---

## 📄 License

This project is open source and available under the [MIT License](LICENSE).

---

<div align="center">

**Built with ❤️ as a hands-on DevOps learning project**

*All components run fully locally — no cloud account required*

⭐ **If this project helped you, please give it a star!** ⭐

</div>