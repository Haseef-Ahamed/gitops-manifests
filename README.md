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
[CI/CD Pipeline](#-cicd-pipeline) •
[Promotion](#-promotion-guide) •
[Rollback](#-rollback-guide) •
[Commands](#-command-reference) •
[Troubleshooting](#-troubleshooting)

</div>

---

## 📖 Overview

This project implements a **production-grade GitOps workflow** running entirely on a local machine. Every code change flows automatically from Git through a CI/CD pipeline into a Kubernetes cluster — with **zero manual deployment commands**.

### 🤔 What is GitOps?

GitOps is a set of practices where your **Git repository is the single source of truth** for both application code and infrastructure configuration. An automated controller continuously reconciles the cluster state with what is declared in Git.

| Traditional Deployment | GitOps Deployment |
|---|---|
| Developer runs `kubectl apply` manually | `git push` triggers everything automatically |
| Hard to track who deployed what | Full audit trail in Git commit history |
| Rollback requires manual intervention | Rollback = `git revert` + `git push` |
| Config drift is invisible | Argo CD detects and fixes drift automatically |
| Inconsistent environments | Git guarantees identical deployments every time |

### ✨ What This Project Does

- ✅ **Automatic deployments** — every `git push` to main deploys to dev
- ✅ **Zero-downtime** rolling updates via Kubernetes
- ✅ **Separate environments** — `dev` (auto) and `staging` (manual promotion)
- ✅ **One-command rollback** — `git revert` + `git push`
- ✅ **Auto-versioning** — version synced from `package.json` automatically
- ✅ **Health checks** — built into every deployment
- ✅ **Visual dashboard** — Argo CD UI shows live status
- ✅ **Fully local** — no cloud account or credit card needed

---

## 🏗️ Architecture

### System Diagram

```
👨‍💻 Developer
     │
     │  git push  (ONE command — that's all)
     ▼
┌────────────────────────────────────────────────────────────┐
│                        GITHUB                              │
│                                                            │
│  ┌─────────────────────┐      ┌───────────────────────┐   │
│  │    gitops-app       │      │   gitops-manifests    │   │
│  │  (source code)      │      │  (K8s configs)        │   │
│  │                     │      │                       │   │
│  │  src/app.js         │      │  dev/                 │   │
│  │  Dockerfile         │      │    deployment.yml     │   │
│  │  package.json       │      │    service.yml        │   │
│  │  .github/workflows/ │      │  staging/             │   │
│  │    ci.yaml          │      │    deployment.yml     │   │
│  └──────────┬──────────┘      │    service.yml        │   │
│             │ triggers        └──────────┬────────────┘   │
│             ▼                           ▲                  │
│  ┌──────────────────────────────────────┘                  │
│  │           GitHub Actions CI/CD                          │
│  │   Test → Build → Push Image → Update dev manifest       │
│  └─────────────────────────────────────────────────────────┘
│
│  docker push ──────────────────────────────────────────────►  ghcr.io
│                                                               (registry)
└────────────────────────────────────────────────────────────┘
                                                │ pull image
┌───────────────────────────────────────────────▼────────────┐
│              KUBERNETES CLUSTER  (k3d on your laptop)       │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                    ARGO CD                           │   │
│  │   Watches gitops-manifests repo every 3 minutes      │   │
│  │   Detects changes → auto-syncs cluster               │   │
│  └──────────────┬───────────────────────┬───────────────┘   │
│                 │                       │                    │
│    ┌────────────▼──────────┐  ┌─────────▼──────────────┐    │
│    │   namespace: dev      │  │  namespace: staging    │    │
│    │                       │  │                        │    │
│    │  Auto-deploys on      │  │  Updates only when     │    │
│    │  every git push ✅    │  │  ./promote.sh runs 🔒  │    │
│    │                       │  │                        │    │
│    │  localhost:3000        │  │  localhost:3001        │    │
│    └───────────────────────┘  └────────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### Complete End-to-End Flow

```
You type: git push origin main
               │
               │ ~5 seconds
               ▼
    GitHub Actions wakes up
               │
    ┌──────────┴────────────┐
    │  JOB 1: Test          │  npm install + npm test
    └──────────┬────────────┘
               │ ~30 seconds
    ┌──────────┴────────────┐
    │  JOB 2: Build         │  docker build + docker push → ghcr.io
    └──────────┬────────────┘  image tagged: sha-xxxxxxx
               │ ~1 minute
    ┌──────────┴────────────┐
    │  JOB 3: Deploy Dev    │  updates dev/deployment.yml
    └──────────┬────────────┘  image: sha-xxxxxxx
               │               APP_VERSION: vX.X.X
               │ ~3 minutes
               ▼
    Argo CD detects manifest changed
               │
               ▼
    Kubernetes rolling update
    (new pod starts → traffic switches → old pod removed)
               │
               ▼
    curl localhost:3000/version → vX.X.X ✅

    Total time: ~4 minutes
    Your effort: ONE git push command
```

### Two Repository Pattern

```
WHY TWO REPOS?
─────────────────────────────────────────────────────────
gitops-app          → "What the app does" (code)
gitops-manifests    → "How the app is deployed" (config)

Benefits:
✅ Change memory limits → no Docker rebuild needed
✅ Deployment history separate from code history
✅ Clear audit trail for every deployment
✅ Roll back config without touching code
```

---

## 🛠️ Tools & Technologies

| Tool | Version | Purpose |
|---|---|---|
| **k3d** | v5.8.3 | Lightweight Kubernetes cluster inside Docker |
| **kubectl** | v1.35.4 | Kubernetes command-line interface |
| **Argo CD** | v2.x | GitOps continuous delivery controller |
| **Helm** | v3.20.2 | Kubernetes package manager |
| **Docker** | v29.4.0 | Container build and runtime |
| **GitHub Actions** | — | Automated CI/CD pipeline |
| **ghcr.io** | — | GitHub Container Registry (free) |
| **Node.js** | 18 Alpine | Application runtime |
| **Express.js** | 4.18.x | Web framework |

---

## 📋 Prerequisites

### System Requirements

| Resource | Minimum | Recommended |
|---|---|---|
| **RAM** | 8 GB | 16 GB |
| **CPU** | 4 cores | 8 cores |
| **Disk Space** | 20 GB free | 40 GB free |
| **OS** | Ubuntu 22.04 / macOS 13 / Windows 10+ | Ubuntu 22.04 LTS |
| **Internet** | Required for setup | Stable broadband |

### Required Accounts

| Account | Cost | Link |
|---|---|---|
| GitHub account | Free | [github.com](https://github.com) |
| GitHub Personal Access Token | Free | Settings → Developer Settings → PAT |

**Token scopes needed:** `repo` + `write:packages` + `delete:packages`

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
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd
helm install argocd argo/argo-cd --namespace argocd

# 5. Apply Argo CD applications
kubectl apply -f argocd-dev-app.yml
kubectl apply -f argocd-staging-app.yml

# 6. Verify everything works
curl http://localhost:3000/health
# → {"status":"OK","timestamp":"..."}
```

---

## 🔧 Installation

### Step 1 — Install Tools

#### Docker (Ubuntu/Debian)

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

# Run Docker without sudo
sudo usermod -aG docker $USER && newgrp docker
```

✅ Verify: `docker --version`

#### kubectl

```bash
cd ~
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
  https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
```

✅ Verify: `kubectl version --client`

#### k3d

```bash
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

✅ Verify: `k3d --version`

#### Helm

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

✅ Verify: `helm version`

#### Argo CD CLI

```bash
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

✅ Verify: `argocd version --client`

#### Verify All Tools

```bash
docker --version      # Docker version 29.x.x
kubectl version --client  # Client Version: v1.35.x
k3d --version         # k3d version v5.8.x
helm version          # Version:"v3.20.x"
argocd version --client   # argocd: v2.x.x
git --version         # git version 2.x.x
```

---

### Step 2 — Create Kubernetes Cluster

```bash
k3d cluster create gitops-cluster \
  --port "3000:30000@loadbalancer" \
  --port "3001:30001@loadbalancer" \
  --agents 1
```

**Port mapping explained:**
```
3000:30000  →  dev app accessible at localhost:3000
3001:30001  →  staging app at localhost:3001
--agents 1  →  1 worker node (enough for local setup)
```

**Create namespaces:**
```bash
kubectl create namespace dev
kubectl create namespace staging
```

**Verify:**
```bash
kubectl get nodes
# NAME                          STATUS   ROLES
# k3d-gitops-cluster-server-0   Ready    control-plane ✅
# k3d-gitops-cluster-agent-0    Ready    <none>        ✅

kubectl get namespaces | grep -E "dev|staging"
# dev       Active ✅
# staging   Active ✅
```

---

### Step 3 — Install Argo CD

```bash
# Add Helm repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Create namespace
kubectl create namespace argocd

# Install (without --wait to avoid timeout on slower machines)
helm install argocd argo/argo-cd --namespace argocd

# Watch pods start up (takes 2-3 minutes)
kubectl get pods -n argocd -w
# Press Ctrl+C when all pods show: STATUS = Running
```

> ⚠️ If you see `context deadline exceeded` — this is normal on VirtualBox.
> Check `kubectl get pods -n argocd` — if all show Running, installation succeeded.

---

### Step 4 — Access Argo CD Dashboard

```bash
# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo ""
# → Save this password!

# Start port-forward
kubectl port-forward svc/argocd-server -n argocd 9090:443 &
sleep 3

# Login via CLI
argocd login localhost:9090 \
  --username admin \
  --password $(kubectl -n argocd get secret \
    argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d) \
  --insecure
# → 'admin:login' logged in successfully ✅
```

Open in browser: **`https://localhost:9090`**
- Username: `admin`
- Password: *(from command above)*
- Click **Advanced → Proceed** to bypass SSL warning

---

### Step 5 — Configure GitHub Secret

```
1. Go to: github.com/Haseef-Ahamed/gitops-app
2. Settings → Secrets and variables → Actions
3. New repository secret:
   Name:  MANIFEST_TOKEN
   Value: (your GitHub PAT)
4. Add secret ✅
```

---

### Step 6 — Apply Argo CD Applications

```bash
# Apply both app configs
kubectl apply -f argocd-dev-app.yml
kubectl apply -f argocd-staging-app.yml

# Verify
argocd app list
```

Expected:
```
NAME                  STATUS  HEALTH   TARGET
gitops-app-dev        Synced  Healthy  main ✅
gitops-app-staging    Synced  Healthy  main ✅
```

---

### Step 7 — Make Package Public on ghcr.io

After first pipeline run:

```
1. github.com → Your Profile → Packages tab
2. Click "gitops-app"
3. Package settings → Danger Zone
4. Change visibility → Public
5. Type "gitops-app" to confirm ✅
```

> This allows Kubernetes to pull the image without authentication.

---

## 🔁 CI/CD Pipeline

The pipeline lives at `.github/workflows/ci.yaml` and runs on every push to `main`.

### Pipeline Structure

```
git push to main
       │
       ▼
┌──────────────────┐
│   🧪 JOB 1       │
│   Run Tests      │  npm install → npm test
│                  │  Fails here = nothing deploys
└────────┬─────────┘
         │ on success
         ▼
┌──────────────────┐
│   🐳 JOB 2       │
│   Build & Push   │  docker build
│                  │  docker push → ghcr.io:sha-xxxxxxx
└────────┬─────────┘
         │ on success
         ▼
┌──────────────────┐
│   🚀 JOB 3       │
│   Deploy to Dev  │  clones gitops-manifests
│                  │  updates dev/deployment.yml:
│                  │    image: sha-xxxxxxx
│                  │    APP_VERSION: vX.X.X
│                  │  git push → Argo CD syncs
└──────────────────┘
```

### What Gets Auto-Updated

Every `git push` automatically updates `dev/deployment.yml`:

```yaml
# BEFORE push:
image: ghcr.io/haseef-ahamed/gitops-app:sha-old
env:
- name: APP_VERSION
  value: "v11.0.0"

# AFTER push (updated by pipeline):
image: ghcr.io/haseef-ahamed/gitops-app:sha-new  ← new image
env:
- name: APP_VERSION
  value: "v12.0.0"                                ← from package.json
```

### Image Tagging Strategy

```bash
# Every build creates two tags:
ghcr.io/haseef-ahamed/gitops-app:sha-f7fa6bb   ← specific (used in manifests)
ghcr.io/haseef-ahamed/gitops-app:latest         ← always newest

# Manifests ALWAYS use the SHA tag:
image: ghcr.io/haseef-ahamed/gitops-app:sha-f7fa6bb
# ↑ Guarantees reproducible, traceable deployments
```

> ⚠️ Never use `:latest` in Kubernetes manifests — you can't rollback to "latest"

---

## 🎯 Usage

### API Endpoints

| Endpoint | Response |
|---|---|
| `GET /` | `{"message":"GitOps Demo App is running!","version":"v12.0.0","environment":"dev"}` |
| `GET /health` | `{"status":"OK","timestamp":"2026-04-28T..."}` |
| `GET /version` | `{"version":"v12.0.0","environment":"dev"}` |

```bash
# Test dev environment (port 3000)
curl http://localhost:3000/
curl http://localhost:3000/health
curl http://localhost:3000/version

# Test staging environment (port 3001)
curl http://localhost:3001/
curl http://localhost:3001/health
curl http://localhost:3001/version
```

---

### Making a Code Change — Full GitOps Loop

```bash
# 1. Update version in package.json
cd ~/gitops-project/gitops-app
nano package.json
# Change: "version": "12.0.0" → "13.0.0"

# 2. Update fallback version in app.js (optional)
nano src/app.js
# Change: v12.0.0 → v13.0.0

# 3. Push — this is ALL you do
git add .
git commit -m "feat: release v13.0.0"
git push origin main

# 4. Watch pipeline (GitHub → gitops-app → Actions)
# 5. After ~4 minutes verify:
curl http://localhost:3000/version
# → {"version":"v13.0.0","environment":"dev"} ✅
```

---

### Start Your Session (After Reboot)

```bash
# Add this to ~/.bashrc once:
cat >> ~/.bashrc << 'EOF'
gitops-start() {
  k3d cluster start gitops-cluster 2>/dev/null || true
  sleep 2
  pkill -f "port-forward svc/argocd-server" 2>/dev/null || true
  sleep 1
  kubectl port-forward svc/argocd-server -n argocd 9090:443 &
  sleep 3
  argocd login localhost:9090 \
    --username admin \
    --password $(kubectl -n argocd get secret \
      argocd-initial-admin-secret \
      -o jsonpath="{.data.password}" | base64 -d) \
    --insecure 2>/dev/null
  echo "✅ GitOps Platform Ready!"
  echo "   Dev:     http://localhost:3000"
  echo "   Staging: http://localhost:3001"
  echo "   ArgoCD:  https://localhost:9090"
}
EOF
source ~/.bashrc

# Every morning just type:
gitops-start
```

---

## 🚀 Promotion Guide

Promotion moves a **verified version from dev to staging**. This is intentionally manual — a human must approve before staging updates.

### Using promote.sh (Recommended)

```bash
cd ~/gitops-project
./promote.sh
```

**What it does automatically:**
1. Pulls latest manifests from GitHub
2. Shows current state of both environments
3. Checks dev health (`/health` endpoint)
4. Asks for your confirmation
5. Updates `staging/deployment.yml` with dev image + version
6. Pushes to main branch
7. Argo CD auto-syncs staging within ~3 minutes

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
│ DEV          │ sha-f7fa6bb : v12.0.0        │
│ STAGING      │ sha-062ccf5 : v11.0.0        │
└──────────────┴──────────────────────────────┘

🔍 Checking dev health...
   ✅ Dev is healthy

📦 Promotion Plan:
   FROM: sha-062ccf5 (v11.0.0)
   TO:   sha-f7fa6bb (v12.0.0)

✋ Promote to staging? (yes/no): yes

🚀 Promoting to staging...

╔══════════════════════════════════════════════╗
║          PROMOTION COMPLETE! 🎉              ║
║  ✅ staging/deployment.yml updated           ║
║  ⏳ Argo CD syncs in ~3 minutes              ║
║  Verify: curl http://localhost:3001/version  ║
╚══════════════════════════════════════════════╝
```

**Verify after ~3 minutes:**
```bash
curl http://localhost:3001/version
# → {"version":"v12.0.0","environment":"staging"} ✅
```

### Manual Promotion

```bash
cd ~/gitops-project/gitops-manifests
git pull origin main

# Read dev state
DEV_IMAGE=$(grep "image:" dev/deployment.yml | awk '{print $2}')
DEV_VERSION=$(grep -A1 "name: APP_VERSION" dev/deployment.yml \
  | grep value | awk '{print $2}' | tr -d '"')

echo "Promoting: $DEV_IMAGE ($DEV_VERSION)"

# Update staging
sed -i "s|image: ghcr.io/.*/gitops-app:.*|image: $DEV_IMAGE|g" \
  staging/deployment.yml
sed -i "/name: APP_VERSION/{n;s|value:.*|value: $DEV_VERSION|}" \
  staging/deployment.yml

# Push
git add staging/deployment.yml
git commit -m "promote(staging): $DEV_IMAGE version=$DEV_VERSION"
git push origin main
```

---

## ⏪ Rollback Guide

> **Golden Rule: Always rollback via Git. Never touch `kubectl` directly.**

### Why Git Rollback Works

```
Every deployment = one Git commit in gitops-manifests

git log shows:
  f211cb5  promote(staging): sha-f7fa6bb v12.0.0  ← current
  fc10a8a  ci(dev): sha-062ccf5 v11.0.0
  fca61fc  promote(staging): sha-abc123 v10.0.0   ← stable

git revert f211cb5 → creates new commit that undoes f211cb5
git push → Argo CD sees the revert → rolls back cluster
```

### Rollback Dev

```bash
cd ~/gitops-project/gitops-manifests

# See history
git log --oneline dev/deployment.yml

# Rollback last deployment
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

# Rollback
git revert HEAD --no-edit
git push origin main

# Verify after ~3 minutes
curl http://localhost:3001/version
```

### Rollback to Specific Version

```bash
# Find the version you want
git log --oneline

# Revert that specific commit
git revert <commit-hash> --no-edit
git push origin main
```

### Emergency Rollback via Argo CD UI

```
1. Open https://localhost:9090
2. Click the affected application
3. Click "History and Rollback" tab
4. Find the last good deployment
5. Click "Rollback"
```

> ⚠️ UI rollback is temporary. Follow up with `git revert` to make it permanent in Git history.

---

## 📊 Environments

| Environment | Namespace | URL | Deploys When |
|---|---|---|---|
| **Dev** | `dev` | http://localhost:3000 | Every `git push` to main (automatic) |
| **Staging** | `staging` | http://localhost:3001 | After `./promote.sh` (manual approval) |
| **Argo CD** | `argocd` | https://localhost:9090 | — |

---

## 💻 Command Reference

### Daily Workflow

```bash
# Morning startup
gitops-start

# Make and deploy a change
cd ~/gitops-project/gitops-app
# ... edit files ...
git add . && git commit -m "feat: ..." && git push origin main

# Watch pods update
kubectl get pods -n dev -w

# Verify dev
curl http://localhost:3000/version

# Promote to staging
cd ~/gitops-project && ./promote.sh

# Verify staging
curl http://localhost:3001/version

# Rollback if needed
cd ~/gitops-project/gitops-manifests
git revert HEAD --no-edit && git push origin main
```

### Kubernetes

```bash
# Pods
kubectl get pods -n dev                               # List dev pods
kubectl get pods -n staging                           # List staging pods
kubectl get pods -n argocd                            # List Argo CD pods
kubectl get pods -n dev -w                            # Watch live updates

# Debugging
kubectl logs -n dev <pod-name>                        # View logs
kubectl logs -n dev <pod-name> --previous             # Previous pod logs
kubectl describe pod -n dev <pod-name>                # Full pod details

# Deployments
kubectl get deployments -n dev                        # List deployments
kubectl rollout status deployment/gitops-app -n dev   # Watch rollout
kubectl rollout restart deployment/gitops-app -n dev  # Force restart

# All resources
kubectl get all -n dev                                # Everything in dev
kubectl get all -n staging                            # Everything in staging
```

### Argo CD

```bash
argocd app list                           # List all applications
argocd app get gitops-app-dev             # Dev app details + health
argocd app get gitops-app-staging         # Staging app details
argocd app sync gitops-app-dev            # Force sync now (skip 3 min wait)
argocd app sync gitops-app-staging        # Force sync staging
argocd app history gitops-app-dev         # Full deployment history
argocd app rollback gitops-app-dev <id>   # Rollback to specific version
```

### Cluster Management

```bash
k3d cluster list                          # List all clusters
k3d cluster start gitops-cluster          # Start cluster
k3d cluster stop gitops-cluster           # Stop cluster (saves RAM)
k3d cluster delete gitops-cluster         # Delete cluster completely
```

### Git (manifests repo)

```bash
git log --oneline                         # See all deployments
git log --oneline dev/deployment.yml      # Dev deployment history only
git log --oneline staging/deployment.yml  # Staging deployment history only
git revert HEAD --no-edit                 # Rollback last change
git revert <hash> --no-edit              # Rollback specific change
```

---

## 🔑 Key Configuration Files

### `argocd-dev-app.yml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-app-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Haseef-Ahamed/gitops-manifests.git
    targetRevision: main      # watches main branch
    path: dev                 # reads from dev/ folder
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true             # removes deleted resources
      selfHeal: true          # reverts manual kubectl changes
```

### `argocd-staging-app.yml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-app-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Haseef-Ahamed/gitops-manifests.git
    targetRevision: main      # same main branch
    path: staging             # reads from staging/ folder
  destination:
    server: https://kubernetes.default.svc
    namespace: staging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 🔍 Troubleshooting

### Pod shows `ImagePullBackOff`

```bash
# See exact error
kubectl describe pod -n dev \
  $(kubectl get pod -n dev -o name | head -1)
# Read the Events section at the bottom

# Fix: Make ghcr.io package public
# GitHub → Profile → Packages → gitops-app
# → Package settings → Change visibility → Public
```

### `argocd: connection refused`

```bash
# Port-forward stopped — restart it
pkill -f "port-forward svc/argocd" 2>/dev/null || true
kubectl port-forward svc/argocd-server -n argocd 9090:443 &
sleep 3
argocd login localhost:9090 \
  --username admin \
  --password $(kubectl -n argocd get secret \
    argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d) \
  --insecure
```

### CI/CD Pipeline fails `403 Forbidden` on ghcr.io

```bash
# Make sure ci.yaml uses MANIFEST_TOKEN not GITHUB_TOKEN:
# Find the Login step in ci.yaml:
password: ${{ secrets.MANIFEST_TOKEN }}   # ✅ correct
password: ${{ secrets.GITHUB_TOKEN }}     # ❌ wrong (causes 403)

# Also verify provenance: false is in the build step:
uses: docker/build-push-action@v5
with:
  provenance: false   # ← required
```

### Helm install shows `context deadline exceeded`

```bash
# Normal on slower machines — check pods directly:
kubectl get pods -n argocd
# If all Running → install succeeded ✅

helm list -n argocd
# If STATUS = deployed → all good ✅
```

### Cluster not responding after reboot

```bash
k3d cluster start gitops-cluster
sleep 30
kubectl get nodes
# Both nodes must show: STATUS = Ready
```

### `promote.sh` fails — `no tracking information`

```bash
cd ~/gitops-project/gitops-manifests
git branch --set-upstream-to=origin/main main
git pull origin main
# Now re-run ./promote.sh
```

### Port already in use

```bash
sudo lsof -i :9090          # find what's using port
sudo kill -9 <PID>          # kill it
# Or just use a different port:
kubectl port-forward svc/argocd-server -n argocd 9191:443 &
argocd login localhost:9191 ...
```

---

## ⚠️ Common Mistakes

| Mistake | What Happens | Fix |
|---|---|---|
| Using `:latest` tag in manifests | Can't rollback to specific version | Always use SHA: `sha-xxxxxxx` |
| Running `kubectl apply` manually | Argo CD reverts it (self-healing) | All changes go through Git only |
| One repo for code + manifests | Mixes concerns, hard to manage | Always keep two separate repos |
| Not checking Argo CD logs | Silent failures go unnoticed | Check `argocd app get` after every push |
| Committing tokens to Git | Security breach | Use GitHub Secrets only |
| Not pulling before editing | Merge conflicts | Always `git pull origin main` first |
| Using `GITHUB_TOKEN` for ghcr.io | 403 Forbidden | Use `MANIFEST_TOKEN` (PAT) instead |

---

## 📐 GitOps Principles — Applied Here

```
1️⃣  DECLARATIVE
    All infrastructure described in YAML files
    "What should exist" not "how to create it"
    ✅ Applied: dev/ and staging/ manifests

2️⃣  VERSIONED AND IMMUTABLE
    Every change tracked in Git history
    Docker images tagged with commit SHA
    Complete audit trail — forever
    ✅ Applied: sha-xxxxxxx image tags + git log

3️⃣  PULLED AUTOMATICALLY
    Argo CD pulls from Git every 3 minutes
    Nobody pushes directly to the cluster
    More secure than push-based systems
    ✅ Applied: Argo CD auto-sync enabled

4️⃣  CONTINUOUSLY RECONCILED
    Argo CD checks cluster vs Git constantly
    Manual kubectl changes are reverted automatically
    Cluster ALWAYS matches Git — guaranteed
    ✅ Applied: selfHeal: true in Argo CD apps
```

---

## 🎓 Skills Demonstrated

By building this project:

- ✅ GitOps principles and production workflow design
- ✅ Local Kubernetes cluster setup and management (k3d)
- ✅ Argo CD installation, configuration and application management
- ✅ Multi-job GitHub Actions CI/CD pipeline
- ✅ Two-repository GitOps pattern
- ✅ Docker containerisation and image registry management
- ✅ Zero-downtime rolling deployments
- ✅ Environment separation (dev/staging)
- ✅ Git-based rollback strategy
- ✅ Kubernetes pod debugging and troubleshooting
- ✅ Shell scripting for DevOps automation

---

## 📁 Related Repositories

| Repository | Description | Link |
|---|---|---|
| **gitops-app** | Application source code + CI/CD pipeline | [github.com/Haseef-Ahamed/gitops-app](https://github.com/Haseef-Ahamed/gitops-app) |
| **gitops-manifests** | Kubernetes deployment configurations | [github.com/Haseef-Ahamed/gitops-manifests](https://github.com/Haseef-Ahamed/gitops-manifests) |

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

*Fully local setup — no cloud account required*

⭐ **If this project helped you, please give it a star!** ⭐

</div>