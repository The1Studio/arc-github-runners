# GitHub Actions Runner Controller (ARC) Setup

Self-hosted GitHub Actions runners using Actions Runner Controller on k3s.

**Repository:** https://github.com/The1Studio/arc-github-runners

## Overview

This repository contains the configuration for managing self-hosted GitHub Actions runners across multiple repositories and organizations using Kubernetes.

### Features

- ✅ **Auto-scaling runners** - Automatically scale based on workload
- ✅ **Multi-organization support** - Separate runner pools
- ✅ **Kubernetes-based** - Runs on lightweight k3s
- ✅ **Resource efficient** - Minimal overhead (~150MB for k3s)

### Current Deployment

All runners use the custom image `the1studio/actions-runner:https-apt` with HTTPS APT sources pre-configured.

#### 1. the1studio-org-runners (General CI/CD)
- **Purpose**: Default runners for all the1studio repositories
- **Replicas**: Min 3, Max 10 (auto-scaling)
- **Labels**: `self-hosted`, `linux`, `x64`, `arc`, `the1studio`, `org`
- **Resources**: 2 CPU / 4GB RAM per runner

#### 2. tuha263-personal-runners (Personal Site)
- **Purpose**: Build and deploy personal website
- **Replicas**: Min 1, Max 5 (auto-scaling)
- **Labels**: `self-hosted`, `linux`, `x64`, `arc`, `personal`
- **Resources**: 1 CPU / 2GB RAM per runner

#### 3. android-apk-builder (Android Builds)
- **Purpose**: Build Android APK files
- **Replicas**: Min 1, Max 3 (auto-scaling, conservative)
- **Labels**: `self-hosted`, `linux`, `x64`, `arc`, `android`, `apk-builder`
- **Resources**: 8 CPU / 16GB RAM per runner (high resource usage)

#### 4. containerized-unity-deploy (Unity Deployment)
- **Purpose**: Deploy Unity builds to app stores/hosting
- **Replicas**: Min 1, Max 5 (auto-scaling)
- **Labels**: `self-hosted`, `Linux`, `X64`, `deploy`
- **Resources**: 4 CPU / 8GB RAM per runner

#### 5. containerized-unity-sync (Unity Editor Images)
- **Purpose**: Build Unity Editor Docker images and sync to Harbor registry
- **Replicas**: Min 1, Max 2 (auto-scaling, very conservative)
- **Labels**: `self-hosted`, `Linux`, `X64`, `deploy`, `harbor-access`, `harbor-host`
- **Resources**: 8 CPU / 24GB RAM per runner (extreme resource usage)

---

## ⚠️ Important Notes

### ✅ Port 80 HTTP Issue - PERMANENTLY FIXED

**This issue is now fixed** at the runner image level. The custom image `the1studio/actions-runner:https-apt` has pre-configured HTTPS APT sources.

**You no longer need to add HTTP→HTTPS conversion in workflows!**

See [docker/README.md](docker/README.md) for details about the custom image.

### Public Repository Access

**CRITICAL**: Organization-level runners cannot be used by public repositories by default.

If your workflow stays in "Queued" state forever, you need to enable public repository access:

```bash
# Enable public repositories for organization runners
gh api -X PATCH orgs/the1studio/actions/runner-groups/1 \
  -F allows_public_repositories=true

# Verify the change
gh api orgs/the1studio/actions/runner-groups/1 --jq '.allows_public_repositories'
# Should return: true
```

**Alternative**: Use repository-level runners instead of organization-level runners for public repositories. See [examples/additional-runners.yaml](k8s/examples/additional-runners.yaml).

For detailed troubleshooting, see [docs/TROUBLESHOOTING.md](https://github.com/The1Studio/arc-github-runners/blob/master/docs/TROUBLESHOOTING.md#d-public-repository-not-allowed).

---

## Quick Start

### Prerequisites

- Linux system (tested on Arch Linux)
- `curl` and `bash`
- GitHub Personal Access Token with `repo` or `admin:org` scope

### Installation

```bash
# 1. Install k3s
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644

# 2. Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 3. Install cert-manager
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml

# 4. Wait for cert-manager
kubectl wait --for=condition=ready pod -l app=cert-manager -n cert-manager --timeout=120s

# 5. Install ARC controller
kubectl create namespace arc-systems
kubectl create namespace arc-runners

helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller
helm repo update

# Create GitHub token secret
kubectl create secret generic controller-manager \
  --namespace arc-systems \
  --from-literal=github_token="YOUR_GITHUB_PAT"

helm install arc \
  --namespace arc-systems \
  actions-runner-controller/actions-runner-controller

# 6. Deploy runners
kubectl apply -f k8s/runner-deployments.yaml
kubectl apply -f k8s/autoscalers.yaml
```

## Documentation

### Quick Links

**Getting Started**:
- 📖 [Project Overview & PDR](docs/project-overview-pdr.md) - Project goals, requirements, and vision
- 🏗️ [System Architecture](docs/system-architecture.md) - Detailed architecture and component design
- 📝 [Code Standards](docs/code-standards.md) - Coding conventions and best practices
- 📊 [Codebase Summary](docs/codebase-summary.md) - Repository structure and key components

**Operations**:
- 🚀 [Usage Guide](docs/USAGE.md) - How to use runners in workflows
- ⚙️ [Management](docs/MANAGEMENT.md) - Management commands and operations
- 🔧 [Troubleshooting](docs/TROUBLESHOOTING.md) - Common issues and solutions
- 💾 [Backup & Recovery](docs/BACKUP.md) - Disaster recovery procedures

**Technical**:
- 🐳 [Custom Docker Image](docker/README.md) - HTTPS APT fix details
- 📋 [Example Runners](k8s/examples/additional-runners.yaml) - Templates for new runners

### Documentation Structure

```
docs/
├── project-overview-pdr.md    # 📖 Project goals & requirements
├── system-architecture.md     # 🏗️ Architecture & components
├── code-standards.md          # 📝 Standards & conventions
├── codebase-summary.md        # 📊 Repository overview
├── USAGE.md                   # 🚀 Workflow integration
├── MANAGEMENT.md              # ⚙️ Operations & commands
├── TROUBLESHOOTING.md         # 🔧 Common issues
└── BACKUP.md                  # 💾 Disaster recovery
```

## Repository Structure

```
.
├── README.md
├── k8s/
│   ├── runner-deployments.yaml    # Runner deployment configurations
│   ├── autoscalers.yaml            # Auto-scaling rules
│   ├── network-policy.yaml         # Network security policies
│   ├── pod-disruption-budget.yaml # High availability settings
│   └── examples/
│       └── additional-runners.yaml # Template for adding more runners
├── docker/
│   ├── Dockerfile                  # Custom runner image
│   └── README.md                   # Image documentation
├── workflows/
│   └── test-arc.yml                # Sample workflow to test runners
└── docs/
    ├── project-overview-pdr.md     # Project overview & PDR
    ├── system-architecture.md      # System architecture
    ├── code-standards.md           # Code standards
    ├── codebase-summary.md         # Codebase summary
    ├── USAGE.md                    # How to use runners
    ├── MANAGEMENT.md               # Management commands
    ├── TROUBLESHOOTING.md          # Common issues
    └── BACKUP.md                   # Backup & recovery
```

## Usage in Workflows

### For the1studio Organization (General CI/CD)

```yaml
jobs:
  build:
    runs-on: [self-hosted, linux, x64, arc, the1studio, org]
    steps:
      - uses: actions/checkout@v4
      - run: echo "Running on the1studio runner"
```

### For Personal Repositories

```yaml
jobs:
  deploy:
    runs-on: [self-hosted, linux, x64, arc, personal]
    steps:
      - uses: actions/checkout@v4
      - run: echo "Running on personal runner"
```

### For Android APK Builds

```yaml
jobs:
  build-apk:
    runs-on: [self-hosted, linux, x64, arc, android, apk-builder]
    steps:
      - uses: actions/checkout@v4
      - run: echo "Building Android APK"
```

### For Unity Deployment

```yaml
jobs:
  deploy-unity:
    runs-on: [self-hosted, Linux, X64, deploy]
    steps:
      - uses: actions/checkout@v4
      - run: echo "Deploying Unity build"
```

### For Unity Editor Image Builds (Harbor Access)

```yaml
jobs:
  build-unity-image:
    runs-on: [self-hosted, Linux, X64, deploy, harbor-access, harbor-host]
    steps:
      - uses: actions/checkout@v4
      - run: echo "Building Unity Editor image"
```

### Label Matching Rules

⚠️ **Important**: GitHub Actions requires ALL specified labels to match. Always include:
- `self-hosted` - Indicates self-hosted runner
- `linux` or `Linux` - Operating system
- `x64` or `X64` - Architecture
- Additional specific labels for targeting specific runner pools

**Good Examples**:
```yaml
# ✅ Matches the1studio-org-runners
runs-on: [self-hosted, linux, x64, arc, the1studio, org]

# ✅ Matches android-apk-builder
runs-on: [self-hosted, linux, x64, android]

# ✅ Matches any Linux runner
runs-on: [self-hosted, linux, x64]
```

**Bad Examples**:
```yaml
# ❌ Missing linux and x64 - may not match properly
runs-on: [self-hosted, arc, the1studio, org]

# ❌ Case mismatch - won't match Unity runners
runs-on: [self-hosted, linux, x64, deploy]  # Should be "Linux" and "X64"
```

## Management

### Check Runner Status

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# View all runners
kubectl get pods -n arc-runners

# Check auto-scalers
kubectl get hra -n arc-runners

# View logs
kubectl logs -n arc-runners <pod-name> -c runner
```

### Scale Runners Manually

```bash
# Scale organization runners to 5
kubectl scale runnerdeployment the1studio-org-runners \
  -n arc-runners --replicas=5

# Scale personal runners to 3
kubectl scale runnerdeployment tuha263-personal-runners \
  -n arc-runners --replicas=3
```

## Auto-Scaling Behavior

All runners use `PercentageRunnersBusy` metric for auto-scaling decisions.

### 1. the1studio-org-runners (General CI/CD)
- **Replicas:** Min 3, Max 10
- **Scale up:** 75% busy → 1.5x runners
- **Scale down:** 25% busy → 0.5x runners
- **Cooldown:** 5 minutes after scale-up

### 2. tuha263-personal-runners (Personal Site)
- **Replicas:** Min 1, Max 5
- **Scale up:** 75% busy → 1.5x runners
- **Scale down:** 25% busy → 0.5x runners
- **Cooldown:** 3 minutes after scale-up

### 3. android-apk-builder (Android Builds)
- **Replicas:** Min 1, Max 3 (limited due to high resource usage)
- **Scale up:** 80% busy → 1.5x runners (conservative)
- **Scale down:** 20% busy → 0.5x runners
- **Cooldown:** 10 minutes after scale-up (prevent thrashing)

### 4. containerized-unity-deploy (Unity Deployment)
- **Replicas:** Min 1, Max 5
- **Scale up:** 75% busy → 1.5x runners
- **Scale down:** 25% busy → 0.5x runners
- **Cooldown:** 5 minutes after scale-up

### 5. containerized-unity-sync (Unity Editor Images)
- **Replicas:** Min 1, Max 2 (very limited due to extreme resource usage)
- **Scale up:** 90% busy → 2x runners (very conservative)
- **Scale down:** 10% busy → 0.5x runners
- **Cooldown:** 15 minutes after scale-up (prevent resource exhaustion)

## Troubleshooting

See [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) for common issues.

Quick checks:

```bash
# Check if ARC controller is running
kubectl get pods -n arc-systems

# Check if runners are registered in GitHub
gh api orgs/the1studio/actions/runners

# View ARC controller logs
kubectl logs -n arc-systems -l app.kubernetes.io/name=actions-runner-controller
```

## Contributing

This is a personal infrastructure repository. Changes should be tested in a development environment before applying to production.

## License

MIT

## Maintainer

[@tuha263](https://github.com/tuha263)
