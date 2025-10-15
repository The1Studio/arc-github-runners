# GitHub Actions Runner Controller (ARC) Setup

Self-hosted GitHub Actions runners using Actions Runner Controller on k3s.

## Overview

This repository contains the configuration for managing self-hosted GitHub Actions runners across multiple repositories and organizations using Kubernetes.

### Features

- ✅ **Auto-scaling runners** - Automatically scale based on workload
- ✅ **Multi-organization support** - Separate runner pools
- ✅ **Kubernetes-based** - Runs on lightweight k3s
- ✅ **Resource efficient** - Minimal overhead (~150MB for k3s)

### Current Deployment

**Organization Runners (the1studio)**
- Min: 2 runners, Max: 10 runners
- Labels: `self-hosted`, `arc`, `the1studio`, `org`
- Auto-scales based on job queue

**Personal Runners (tuha263)**
- Min: 1 runner, Max: 5 runners
- Labels: `self-hosted`, `arc`, `personal`
- Auto-scales based on job queue

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

For detailed troubleshooting, see [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md#d-public-repository-not-allowed).

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

## Repository Structure

```
.
├── README.md
├── k8s/
│   ├── runner-deployments.yaml    # Runner deployment configurations
│   ├── autoscalers.yaml            # Auto-scaling rules
│   └── examples/
│       └── additional-runners.yaml # Template for adding more runners
├── workflows/
│   └── test-arc.yml                # Sample workflow to test runners
└── docs/
    ├── USAGE.md                    # How to use runners in workflows
    ├── MANAGEMENT.md               # Management commands
    └── TROUBLESHOOTING.md          # Common issues and fixes
```

## Usage in Workflows

### For the1studio Organization

```yaml
jobs:
  build:
    runs-on: [self-hosted, arc, the1studio, org]
    steps:
      - uses: actions/checkout@v4
      - run: echo "Running on the1studio runner"
```

### For Personal Repositories

```yaml
jobs:
  deploy:
    runs-on: [self-hosted, arc, personal]
    steps:
      - uses: actions/checkout@v4
      - run: echo "Running on personal runner"
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

### Organization Runners (the1studio)
- **Min replicas:** 2
- **Max replicas:** 10
- **Scale up:** When 75% of runners are busy
- **Scale down:** When only 25% are busy
- **Scale up factor:** 2x (doubles the runners)
- **Scale down factor:** 0.5x (halves the runners)

### Personal Runners (tuha263)
- **Min replicas:** 1
- **Max replicas:** 5
- **Scale up:** When 75% of runners are busy
- **Scale down:** When only 25% are busy
- **Scale up factor:** 1.5x
- **Scale down factor:** 0.5x

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
