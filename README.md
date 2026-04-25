# 🚀 Local GitOps Platform with Progressive Delivery

> A fully local, production-grade GitOps platform built with Kubernetes, Argo CD, Docker, and GitHub Actions. Deploy applications automatically using Git as the single source of truth — with zero manual `kubectl apply` commands.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tools & Technologies](#tools--technologies)
- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Project Structure](#project-structure)
- [Installation](#installation)
- [CI/CD Pipeline](#cicd-pipeline)
- [GitOps Workflow](#gitops-workflow)
- [Environment Management](#environment-management)
- [Promotion Guide](#promotion-guide)
- [Rollback Guide](#rollback-guide)
- [Access Points](#access-points)
- [Command Reference](#command-reference)
- [Troubleshooting](#troubleshooting)

---

## Overview

This project implements a **GitOps workflow** entirely on a local machine. Every code change flows automatically from Git through a CI/CD pipeline into a Kubernetes cluster — with no manual deployment steps.

### What GitOps Means Here

> **Git is the only source of truth. If it's not in Git, it doesn't exist in the cluster.**

| Traditional Deployment | GitOps Deployment |
|---|---|
| Developer runs `kubectl apply` manually | Git push triggers everything automatically |
| Hard to track who deployed what | Full audit trail in Git history |
| Rollback requires manual commands | Rollback = `git revert` + `git push` |
| Config drift is invisible | Argo CD detects and fixes drift automatically |

### What You Get

- ✅ Automatic deployments on every `git push`
- ✅ Zero-downtime rolling updates
- ✅ Separate `dev` and `staging` environments
- ✅ One-command rollback via Git
- ✅ Visual dashboard (Argo CD UI)
- ✅ Auto-versioning from `package.json`

---

## Architecture

```
👨‍💻 Developer
     │
     │ git push
     ▼
┌─────────────────────────────────────────────────────────┐
│                       GITHUB                            │
│  ┌──────────────────┐      ┌───────────────────────┐   │
│  │   gitops-app     │      │   gitops-manifests    │   │
│  │  (source code)   │      │  (deployment configs) │   │
│  └────────┬─────────┘      └──────────┬────────────┘   │
│           │ triggers                  ▲ updates         │
│           ▼                           │                 │
│  ┌──────────────────────────────┐     │                 │
│  │     GitHub Actions (CI/CD)   │─────┘                 │
│  │  build → push → update       │                       │
│  └──────────────┬───────────────┘                       │
└─────────────────│─────────────────────────────────────--┘
                  │ docker push
                  ▼
         ┌────────────────┐
         │    ghcr.io     │
         │ (image registry│
         └────────┬───────┘
                  │ pull image
┌─────────────────│──────────────────────────────────────┐
│   KUBERNETES CLUSTER (k3d — runs on your laptop)        │
│                 │                                        │
│  ┌──────────────▼──────────────────────────────────┐   │
│  │                  Argo CD                         │   │
│  │  Watches gitops-manifests → syncs cluster        │   │
│  └──────────┬──────────────────────────┬────────────┘   │
│             │                          │                 │
│  ┌──────────▼──────────┐  ┌────────────▼─────────────┐  │
│  │  namespace: dev     │  │  namespace: staging       │  │
│  │  auto-deploy ✅     │  │  manual promotion 🔒      │  │
│  │  localhost:3000     │  │  localhost:3001           │  │
│  └─────────────────────┘  └──────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### End-to-End Flow

```
git push → GitHub Actions → docker build → docker push → ghcr.io
                         → update manifests → Argo CD detects
                                           → Kubernetes rolling update
                                           → localhost:3000 updated ✅
```

---

## Tools & Technologies

| Tool | Version | Purpose |
|---|---|---|
| **k3d** | v5.8.3 | Local Kubernetes cluster inside Docker |
| **kubectl** | v1.35.4 | Kubernetes CLI |
| **Argo CD** | v2.x | GitOps controller |
| **Helm** | v3.20.2 | Kubernetes package manager |
| **Docker** | v29.4.0 | Containerisation |
| **GitHub Actions** | — | CI/CD pipeline |
| **ghcr.io** | — | Container image registry |
| **Node.js** | 18.x | Application runtime |

---

## Prerequisites

### System Requirements

| Resource | Minimum | Recommended |
|---|---|---|
| RAM | 8 GB | 16 GB |
| CPU | 4 cores | 8 cores |
| Disk | 20 GB free | 40 GB free |
| OS | Ubuntu 22.04 / macOS 13 / Windows 10 | Ubuntu 22.04 LTS |

### Required Accounts

- GitHub account (free)
- GitHub Personal Access Token with scopes: `repo`, `write:packages`, `delete:packages`

---

## Quick Start

```bash
# 1. Clone both repositories
git clone https://github.com/YOUR-USERNAME/gitops-app.git
git clone https://github.com/YOUR-USERNAME/gitops-manifests.git

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

# 5. Deploy Argo CD applications
kubectl apply -f argocd-dev-app.yaml
kubectl apply -f argocd-staging-app.yaml

# 6. Verify
curl http://localhost:3000/health
# {"status":"OK","timestamp":"..."}
```

---

## Project Structure

```
gitops-project/
│
├── gitops-app/                      # Application repository
│   ├── src/
│   │   └── app.js                   # Node.js Express API
│   ├── .github/
│   │   └── workflows/
│   │       └── ci.yaml              # GitHub Actions pipeline
│   ├── Dockerfile                   # Container build recipe
│   ├── package.json                 # App version lives here
│   └── .dockerignore
│
├── gitops-manifests/                # Manifest repository
│   ├── dev/
│   │   ├── deployment.yml           # Dev Kubernetes Deployment
│   │   └── service.yml              # Dev Service (NodePort 30000)
│   └── staging/
│       ├── deployment.yml           # Staging Kubernetes Deployment
│       └── service.yml              # Staging Service (NodePort 30001)
│
└── promote.sh                       # Promotion script (dev → staging)
```

---

## Installation

### Step 1 — Install Tools

```bash
# Docker
sudo apt-get update && sudo apt-get install -y docker-ce docker-ce-cli containerd.io
sudo usermod -aG docker $USER && newgrp docker

# k3d
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Argo CD CLI
curl -sSL -o argocd-linux-amd64 \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

Verify all tools:
```bash
docker --version && kubectl version --client && k3d --version && helm version && argocd version --client
```

### Step 2 — Create Kubernetes Cluster

```bash
k3d cluster create gitops-cluster \
  --port "3000:30000@loadbalancer" \
  --port "3001:30001@loadbalancer" \
  --agents 1

kubectl create namespace dev
kubectl create namespace staging

# Verify
kubectl get nodes
# Both nodes must show: STATUS = Ready
```

### Step 3 — Install Argo CD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
kubectl create namespace argocd

helm install argocd argo/argo-cd \
  --namespace argocd \
  --set server.service.type=NodePort

# Wait for pods (2-3 minutes)
kubectl get pods -n argocd -w
# Press Ctrl+C when all show Running

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
echo ""
```

### Step 4 — Access Argo CD Dashboard

```bash
kubectl port-forward svc/argocd-server -n argocd 9090:443 &

argocd login localhost:9090 \
  --username admin \
  --password $(kubectl -n argocd get secret argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d) \
  --insecure
```

Open browser: `https://localhost:9090`

### Step 5 — Create Argo CD Applications

```bash
kubectl apply -f argocd-dev-app.yaml
kubectl apply -f argocd-staging-app.yaml

# Verify
argocd app list
# Both apps should show: Synced + Healthy
```

---

## CI/CD Pipeline

The pipeline (`.github/workflows/ci.yaml`) runs automatically on every push to `main`:

```
Push to main
     │
     ├─ Run tests (npm test)
     ├─ Build Docker image
     ├─ Tag image with commit SHA (sha-xxxxxxx)
     ├─ Push to ghcr.io
     ├─ Update dev/deployment.yml:
     │    image: sha-xxxxxxx
     │    APP_VERSION: vX.X.X  (from package.json)
     └─ Update staging/deployment.yml:
          APP_VERSION: vX.X.X  (version only, image stays manual)
```

### Required GitHub Secrets

| Secret | Scopes Required |
|---|---|
| `MANIFEST_TOKEN` | `repo`, `write:packages`, `delete:packages` |

Add at: `GitHub → gitops-app repo → Settings → Secrets and variables → Actions`

### Image Tagging Strategy

```
ghcr.io/haseef-ahamed/gitops-app:sha-abc1234   ← specific (used in manifests)
ghcr.io/haseef-ahamed/gitops-app:latest        ← always points to newest
```

> ⚠️ Never use `:latest` in Kubernetes manifests — always use the SHA tag for reproducibility.

---

## GitOps Workflow

### Making a Code Change

```bash
# 1. Edit your code
nano gitops-app/src/app.js

# 2. Update version in package.json
nano gitops-app/package.json
# change "version": "X.X.X"

# 3. Push — everything else is automatic
cd gitops-app
git add .
git commit -m "feat: describe your change"
git push origin main

# 4. Monitor pipeline
# GitHub → gitops-app → Actions tab

# 5. Verify after ~2 minutes
curl http://localhost:3000/version
```

### How Argo CD Keeps Cluster in Sync

```
Every 3 minutes, Argo CD checks:
  "Does the cluster match gitops-manifests?"

If NO  → auto-sync (applies the manifest)
If YES → do nothing

If someone runs kubectl manually → Argo CD reverts it (self-healing)
```

---

## Environment Management

| Environment | Namespace | URL | Deploy Trigger |
|---|---|---|---|
| **dev** | `dev` | http://localhost:3000 | Automatic (every push) |
| **staging** | `staging` | http://localhost:3001 | Manual promotion |

### Check Environment Status

```bash
# Dev
kubectl get pods -n dev
curl http://localhost:3000/health
curl http://localhost:3000/version

# Staging
kubectl get pods -n staging
curl http://localhost:3001/health
curl http://localhost:3001/version

# Argo CD
argocd app list
```

---

## Promotion Guide

Promoting means moving a tested dev image to staging.

### Using the Promotion Script (Recommended)

```bash
cd ~/gitops-project
./promote.sh
```

Output:
```
📋 Reading current dev state...
────────────────────────────────────
  Image:    ghcr.io/.../gitops-app:sha-xxxxxxx
  Version:  v4.0.0
────────────────────────────────────
Promote to staging? (y/n): y
🚀 Promoting to staging...
✅ Promotion complete!
```

### Manual Promotion

```bash
cd ~/gitops-project/gitops-manifests
git pull

# Get dev image
DEV_IMAGE=$(grep "image:" dev/deployment.yml | awk '{print $2}')
DEV_VERSION=$(grep -A1 "name: APP_VERSION" dev/deployment.yml \
  | grep value | awk '{print $2}' | tr -d '"')

# Update staging
sed -i "s|image: ghcr.io/.*/gitops-app:.*|image: $DEV_IMAGE|g" staging/deployment.yml
sed -i "/name: APP_VERSION/{n;s|value:.*|value: $DEV_VERSION|}" staging/deployment.yml

git add staging/deployment.yml
git commit -m "promote: $DEV_IMAGE version=$DEV_VERSION to staging"
git push origin main
```

Argo CD auto-syncs staging within 3 minutes. Verify:
```bash
curl http://localhost:3001/version
```

---

## Rollback Guide

> ⚠️ Always rollback via Git — never use `kubectl` directly.

### Rollback Dev

```bash
cd ~/gitops-project/gitops-manifests

# See deployment history
git log --oneline dev/deployment.yml

# Revert last deployment
git revert HEAD --no-edit
git push origin main

# Argo CD auto-syncs within 3 minutes
curl http://localhost:3000/version
```

### Rollback Staging

```bash
cd ~/gitops-project/gitops-manifests

# See staging history
git log --oneline staging/deployment.yml

# Revert to specific commit
git revert <commit-hash> --no-edit
git push origin main

curl http://localhost:3001/version
```

### Emergency Rollback via Argo CD UI

```
1. Open https://localhost:9090
2. Click the affected app
3. Click "History and Rollback"
4. Select the last known good deployment
5. Click "Rollback"
```

> Note: UI rollback is temporary. Follow up with a `git revert` to make it permanent.

---

## Access Points

| Service | URL | Credentials |
|---|---|---|
| Dev Application | http://localhost:3000 | None |
| Staging Application | http://localhost:3001 | None |
| Argo CD Dashboard | https://localhost:9090 | admin / (see installation) |

### API Endpoints

| Endpoint | Description | Example Response |
|---|---|---|
| `GET /` | Home | `{"message":"GitOps Demo App is running!"}` |
| `GET /health` | Health check | `{"status":"OK","timestamp":"..."}` |
| `GET /version` | Version info | `{"version":"v4.0.0","environment":"dev"}` |

---

## Command Reference

### Daily Workflow

```bash
# Start Argo CD session (run after every reboot)
gitops-start

# Check everything is healthy
kubectl get pods -n dev
kubectl get pods -n staging
argocd app list

# Deploy new version
cd ~/gitops-project/gitops-app
git add . && git commit -m "feat: ..." && git push origin main

# Promote dev → staging
cd ~/gitops-project && ./promote.sh

# Rollback
cd ~/gitops-project/gitops-manifests
git revert HEAD --no-edit && git push origin main
```

### Kubernetes Commands

```bash
kubectl get pods -n dev                        # List dev pods
kubectl get pods -n staging                    # List staging pods
kubectl get pods -n argocd                     # List Argo CD pods
kubectl logs -n dev <pod-name>                 # View pod logs
kubectl describe pod -n dev <pod-name>         # Debug pod issues
kubectl rollout restart deployment/gitops-app -n dev  # Restart pods
```

### Argo CD Commands

```bash
argocd app list                                # List all apps
argocd app get gitops-app-dev                  # App details
argocd app sync gitops-app-dev                 # Force sync now
argocd app sync gitops-app-staging             # Force sync staging
argocd app history gitops-app-dev              # Deployment history
```

### Cluster Management

```bash
k3d cluster list                               # List clusters
k3d cluster start gitops-cluster               # Start cluster
k3d cluster stop gitops-cluster                # Stop cluster
k3d cluster delete gitops-cluster              # Delete cluster
```

---

## Troubleshooting

### Pod shows `ImagePullBackOff`

```bash
# Check exact error
kubectl describe pod -n dev $(kubectl get pod -n dev -o name | head -1)

# Fix: Make ghcr.io package public
# GitHub → Packages → gitops-app → Package settings → Change visibility → Public
```

### Argo CD shows `OutOfSync`

```bash
# Force sync immediately
argocd app sync gitops-app-dev

# If login fails, restart port-forward
pkill -f "port-forward svc/argocd"
kubectl port-forward svc/argocd-server -n argocd 9090:443 &
argocd login localhost:9090 --username admin \
  --password $(kubectl -n argocd get secret argocd-initial-admin-secret \
    -o jsonpath="{.data.password}" | base64 -d) --insecure
```

### CI/CD Pipeline fails with `403 Forbidden`

```bash
# Make sure MANIFEST_TOKEN has correct scopes: repo, write:packages
# Check: GitHub → Settings → Developer Settings → Personal Access Tokens
# Also verify the pipeline uses MANIFEST_TOKEN (not GITHUB_TOKEN) for ghcr.io login
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
# Files not pushed to GitHub yet
cd ~/gitops-project/gitops-manifests
git add . && git commit -m "feat: add manifests" && git push origin main
```

### Cluster not reachable after reboot

```bash
# Restart the k3d cluster
k3d cluster start gitops-cluster

# Wait 30 seconds then verify
kubectl get nodes
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Using `:latest` tag in manifests | Always use SHA tag: `sha-xxxxxxx` |
| Running `kubectl apply` manually | All changes must go through Git |
| One repo for code + manifests | Keep two separate repos always |
| Not reading Argo CD sync logs | Check sync status after every push |
| Committing secrets to Git | Use GitHub Secrets, never hardcode |
| Forgetting to `git pull` before editing manifests | Always pull before making changes |

---

## Learning Outcomes

After completing this project you can:

- ✅ Explain GitOps principles and why they matter
- ✅ Set up a local Kubernetes cluster with k3d
- ✅ Deploy and configure Argo CD
- ✅ Build automated CI/CD pipelines with GitHub Actions
- ✅ Manage separate environments (dev/staging)
- ✅ Perform zero-downtime deployments
- ✅ Execute rollbacks using Git history
- ✅ Debug Kubernetes pod issues

---

## Author

**Haseef Ahamed**
GitHub: [@Haseef-Ahamed](https://github.com/Haseef-Ahamed)

---

*Built as a hands-on DevOps learning project. All components run fully locally — no cloud required.*