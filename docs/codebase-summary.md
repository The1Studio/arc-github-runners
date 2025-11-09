# Codebase Summary

**Last Updated**: 2025-11-09

## Overview

This repository contains configuration and deployment files for managing self-hosted GitHub Actions runners using Actions Runner Controller (ARC) on k3s Kubernetes.

## Repository Structure

```
arc-github-runners/
├── docker/                      # Custom runner image
│   ├── Dockerfile              # Custom image with HTTPS APT sources
│   └── README.md               # Image documentation
├── k8s/                        # Kubernetes manifests
│   ├── runner-deployments.yaml # Runner deployment configs
│   ├── autoscalers.yaml        # Auto-scaling rules
│   ├── network-policy.yaml     # Network security policies
│   ├── pod-disruption-budget.yaml # Availability guarantees
│   └── examples/
│       └── additional-runners.yaml # Runner templates
├── workflows/                  # Sample GitHub workflows
│   └── test-arc.yml           # Test workflow for ARC
├── docs/                       # Documentation
│   ├── USAGE.md               # How to use runners
│   ├── MANAGEMENT.md          # Management commands
│   ├── TROUBLESHOOTING.md     # Common issues & fixes
│   ├── BACKUP.md              # Backup & disaster recovery
│   ├── project-overview-pdr.md # Project overview & PDR
│   ├── code-standards.md      # Code standards
│   ├── codebase-summary.md    # This file
│   └── system-architecture.md # System architecture
├── README.md                   # Main documentation
├── CLAUDE.md                   # AI assistant instructions
└── .gitignore                  # Git ignore patterns
```

## Key Components

### 1. Custom Docker Image (`docker/`)

**File**: `Dockerfile`

A custom GitHub Actions runner image based on `summerwind/actions-runner:v2.328.0-ubuntu-22.04`.

**Key modifications**:
- Converts all APT sources from HTTP to HTTPS
- Disables HTTP-only PPAs
- Solves port 80 blocking issue in Kubernetes environments

**Base image**: `summerwind/actions-runner:v2.328.0-ubuntu-22.04`
**Custom image**: `the1studio/actions-runner:https-apt`

### 2. Kubernetes Manifests (`k8s/`)

#### Runner Deployments (`runner-deployments.yaml`)

Defines two runner pools:

**Organization Runners** (`the1studio-org-runners`):
- Organization: `the1studio`
- Labels: `self-hosted`, `linux`, `x64`, `arc`, `the1studio`, `org`
- Resources: 2 CPU / 4Gi memory (limit), 500m CPU / 1Gi memory (request)
- Docker socket mounted for Docker-in-Docker support
- Security: `runAsNonRoot: false` (required for Docker socket access)

**Personal Runners** (`tuha263-personal-runners`):
- Repository: `tuha263/personal-site`
- Labels: `self-hosted`, `linux`, `x64`, `arc`, `personal`
- Resources: 1 CPU / 2Gi memory (limit), 250m CPU / 512Mi memory (request)
- Docker socket mounted

#### Auto-scalers (`autoscalers.yaml`)

**Organization Auto-scaler**:
- Min replicas: 3, Max replicas: 10
- Scale up threshold: 75% busy
- Scale down threshold: 25% busy
- Scale up factor: 1.5x
- Scale down factor: 0.5x
- Scale down delay: 300 seconds (5 minutes)

**Personal Auto-scaler**:
- Min replicas: 1, Max replicas: 5
- Scale up threshold: 75% busy
- Scale down threshold: 25% busy
- Scale up factor: 1.5x
- Scale down factor: 0.5x
- Scale down delay: 180 seconds (3 minutes)

#### Network Policy (`network-policy.yaml`)

**Runner Egress Policy**:
- Allows DNS (UDP/53) to kube-system
- Allows HTTPS (TCP/443) to all destinations
- Allows SSH (TCP/22) for git operations
- Blocks HTTP (TCP/80) by default

**Controller Egress Policy**:
- Allows DNS (UDP/53) to kube-system
- Allows HTTPS (TCP/443) for GitHub API
- Allows Kubernetes API (TCP/6443)

#### Pod Disruption Budget (`pod-disruption-budget.yaml`)

Ensures at least 1 runner is always available during voluntary disruptions for both runner pools.

### 3. Workflow Examples (`workflows/`)

**Test workflow** (`test-arc.yml`):
- 6 parallel jobs to test auto-scaling
- Uses organization runner labels
- Tests basic runner info and Docker access

### 4. Documentation (`docs/`)

Comprehensive documentation covering:
- Usage in GitHub workflows
- Management commands
- Troubleshooting common issues
- Backup and disaster recovery procedures

## Technology Stack

### Infrastructure
- **Kubernetes**: k3s (lightweight Kubernetes distribution)
- **Container Runtime**: Docker (via Docker socket mount)
- **Orchestration**: Kubernetes Custom Resources (CRDs)

### GitHub Actions Integration
- **Controller**: Actions Runner Controller (ARC)
- **Runner Image**: Custom image based on summerwind/actions-runner
- **Package Manager**: Helm 3

### Security
- **Network Policies**: Kubernetes NetworkPolicy
- **Certificate Management**: cert-manager
- **Secret Management**: Kubernetes Secrets

### Monitoring & Management
- **CLI**: kubectl, gh (GitHub CLI)
- **Metrics**: HorizontalRunnerAutoscaler metrics

## Configuration Files

### Kubernetes Secrets
- `controller-manager` (in `arc-systems` namespace): GitHub Personal Access Token

### Environment Variables
- `KUBECONFIG=/etc/rancher/k3s/k3s.yaml` - Kubernetes config location

### Labels
Runner labels determine which workflows can use which runners:

**Organization runners**: `[self-hosted, linux, x64, arc, the1studio, org]`
**Personal runners**: `[self-hosted, linux, x64, arc, personal]`

## Security Considerations

### Network Security
1. **HTTP port 80 blocked** - Only HTTPS (443) and SSH (22) allowed
2. **Ingress restricted** - Runners only accept connections from ARC controller
3. **Egress limited** - Only necessary outbound connections allowed

### Pod Security
1. **Resource limits** - Prevents DoS and resource exhaustion
2. **Non-root user** - Runs as runner user (except Docker socket access)
3. **Pod Disruption Budgets** - Ensures availability

### Access Control
1. **GitHub PAT** - Stored as Kubernetes secret
2. **RBAC** - Kubernetes role-based access control
3. **Namespace isolation** - Separate namespaces for controller and runners

## Critical Issues & Solutions

### Port 80 HTTP Blocking

**Problem**: Default runner image uses HTTP (port 80) for APT sources, which is blocked by network policies.

**Solution**: Custom Docker image with HTTPS APT sources (`the1studio/actions-runner:https-apt`)

**Status**: ✅ Permanently fixed at image level

### Public Repository Access

**Problem**: Organization-level runners cannot be used by public repositories by default.

**Solution**: Enable public repository access via GitHub API:
```bash
gh api -X PATCH orgs/the1studio/actions/runner-groups/1 \
  -F allows_public_repositories=true
```

**Alternative**: Use repository-level runners for public repos

## File Statistics

**Total Files**: 12 files
**Total Tokens**: 7,229 tokens (from repomix analysis)
**Total Characters**: 27,532 characters

**Top 5 Files by Token Count**:
1. README.md - 1,609 tokens (22.3%)
2. k8s/examples/additional-runners.yaml - 1,139 tokens (15.8%)
3. docker/README.md - 801 tokens (11.1%)
4. k8s/runner-deployments.yaml - 684 tokens (9.5%)
5. k8s/network-policy.yaml - 463 tokens (6.4%)

## Maintenance Notes

### Regular Updates
- **Base image updates**: Check for new summerwind/actions-runner releases
- **Kubernetes version**: Monitor k3s updates
- **GitHub token rotation**: Update `controller-manager` secret periodically

### Monitoring
- Check runner status: `kubectl get pods -n arc-runners`
- Check auto-scalers: `kubectl get hra -n arc-runners`
- Check GitHub registration: `gh api orgs/the1studio/actions/runners`

### Backup Strategy
- **Git repository**: Configuration files backed up to GitHub
- **Kubernetes state**: Weekly exports recommended
- **Custom image**: Dockerfile in Git for rebuild capability
- **k3s snapshots**: Daily automatic snapshots

## Related Resources

- **Repository**: https://github.com/The1Studio/arc-github-runners
- **ARC GitHub**: https://github.com/actions/actions-runner-controller
- **k3s Documentation**: https://docs.k3s.io/
- **GitHub Actions**: https://docs.github.com/en/actions

## Maintainer

[@tuha263](https://github.com/tuha263) - The1Studio
