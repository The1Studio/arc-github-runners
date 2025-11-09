# Code Standards & Best Practices

**Last Updated**: 2025-11-09
**Repository**: https://github.com/The1Studio/arc-github-runners

---

## Overview

This document defines the coding standards, naming conventions, and best practices for the ARC (Actions Runner Controller) infrastructure project.

---

## Codebase Structure

### Directory Organization

```
arc-github-runners/
├── docker/                      # Docker images
│   ├── Dockerfile              # Custom runner image definition
│   └── README.md               # Image documentation
├── k8s/                        # Kubernetes manifests
│   ├── runner-deployments.yaml # Runner deployment configurations
│   ├── autoscalers.yaml        # HorizontalRunnerAutoscaler resources
│   ├── network-policy.yaml     # Network security policies
│   ├── pod-disruption-budget.yaml # High availability configurations
│   └── examples/               # Template files for new deployments
│       └── additional-runners.yaml
├── workflows/                  # Sample GitHub Actions workflows
│   └── test-arc.yml           # Test workflow for ARC
├── docs/                       # Documentation
│   ├── USAGE.md               # Usage guide
│   ├── MANAGEMENT.md          # Management commands
│   ├── TROUBLESHOOTING.md     # Troubleshooting guide
│   ├── BACKUP.md              # Backup & recovery
│   ├── project-overview-pdr.md # Project overview & PDR
│   ├── code-standards.md      # This file
│   ├── codebase-summary.md    # Codebase summary
│   └── system-architecture.md # System architecture
├── .claude/                    # AI assistant configuration
├── README.md                   # Main documentation
├── CLAUDE.md                   # AI assistant project instructions
├── .gitignore                  # Git ignore patterns
└── .repomixignore             # Repomix ignore patterns
```

### File Naming Conventions

#### Kubernetes Manifests
- Use **kebab-case**: `runner-deployments.yaml`, `network-policy.yaml`
- Descriptive names indicating resource type: `autoscalers.yaml`, `pod-disruption-budget.yaml`
- Group related resources in single files (separated by `---`)

#### Documentation
- Use **UPPERCASE** for important docs in root: `README.md`, `CLAUDE.md`
- Use **UPPERCASE** for top-level docs: `docs/USAGE.md`, `docs/TROUBLESHOOTING.md`
- Use **kebab-case** for detailed docs: `docs/project-overview-pdr.md`, `docs/code-standards.md`

#### Docker Files
- Use standard names: `Dockerfile` (no extension)
- Version tags: Use semantic versioning (e.g., `v1.1`, `v2.0`)
- Descriptive tags: `https-apt` (indicates the fix applied)

---

## Kubernetes Standards

### YAML Formatting

#### General Rules
```yaml
# Use 2-space indentation (standard Kubernetes)
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: example-runners
  namespace: arc-runners
spec:
  replicas: 2
  template:
    spec:
      # Always document complex configurations
      resources:
        limits:
          cpu: "2"
          memory: "4Gi"
```

#### Comments
```yaml
# Header comment explaining the resource
---
# Section comments for major configuration blocks
apiVersion: v1
kind: ConfigMap
metadata:
  name: example
data:
  # Inline comments for specific settings
  option: value  # Explanation of this value
```

### Resource Naming

#### Kubernetes Resources
**Format**: `{organization|repository}-{type}-{purpose}`

**Examples**:
- `the1studio-org-runners` - Organization runners
- `tuha263-personal-runners` - Personal repository runners
- `myrepo-test-runners` - Repository-specific test runners

#### Namespaces
- `arc-systems` - ARC controller and infrastructure
- `arc-runners` - All runner pods

#### Labels
**Standard labels** (always include):
```yaml
labels:
  app: actions-runner
  pool: the1studio-org
```

**Runner labels** (for GitHub workflow targeting):
```yaml
spec:
  template:
    spec:
      labels:
        - self-hosted   # Required
        - linux         # OS
        - x64           # Architecture
        - arc           # Source (ARC)
        - the1studio    # Organization
        - org           # Pool type
```

### Resource Configuration

#### CPU & Memory
**Use Kubernetes standard formats**:
```yaml
resources:
  limits:
    cpu: "2"          # 2 cores (string with quotes)
    memory: "4Gi"     # 4 gigabytes
  requests:
    cpu: "500m"       # 500 millicores
    memory: "1Gi"     # 1 gigabyte
```

**Guidelines**:
- Always set both limits and requests
- Limits should be 2-4x requests
- Memory limits critical to prevent OOM kills
- CPU limits can be more flexible (throttling vs. killing)

#### Replica Counts
**Auto-scaler configuration**:
```yaml
spec:
  minReplicas: 3      # Minimum for availability
  maxReplicas: 10     # Maximum to prevent resource exhaustion
```

**Guidelines**:
- Min replicas: At least 1 (preferably 2-3 for redundancy)
- Max replicas: Based on total system capacity
- Organization pools: Higher limits (10-20)
- Personal/repo pools: Lower limits (5-10)

---

## Docker Standards

### Dockerfile Best Practices

#### Base Image
```dockerfile
# Pin specific versions for reproducibility
FROM summerwind/actions-runner:v2.328.0-ubuntu-22.04

# NOT: FROM summerwind/actions-runner:latest
```

#### User Management
```dockerfile
# Switch to root only when necessary
USER root

# Perform system modifications
RUN apt-get update && apt-get install -y package

# Switch back to non-root user
USER runner
```

#### Layer Optimization
```dockerfile
# Combine RUN commands to reduce layers
RUN apt-get update && \
    apt-get install -y package1 package2 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# NOT: Multiple RUN commands
```

#### Labels
```dockerfile
LABEL maintainer="username" \
      description="Brief description" \
      version="1.1" \
      base-image="summerwind/actions-runner:v2.328.0-ubuntu-22.04"
```

### Image Naming
**Format**: `{organization}/{image-name}:{tag}`

**Examples**:
- `the1studio/actions-runner:https-apt` - Latest with HTTPS fix
- `the1studio/actions-runner:v1.1-https-apt` - Versioned
- `the1studio/actions-runner:v2.328.0` - Base version pinned

**Guidelines**:
- Use organization prefix for private images
- Descriptive tags (not just version numbers)
- Keep `latest` tag for convenience
- Version-specific tags for reproducibility

---

## GitHub Workflows Standards

### Workflow File Naming
- Use **kebab-case**: `test-arc.yml`, `deploy-prod.yml`
- Place in `.github/workflows/` (standard location)
- Descriptive names: `ci-build-test.yml`, `release-deploy.yml`

### Runner Selection
```yaml
jobs:
  build:
    # Use array format for multiple labels
    runs-on: [self-hosted, arc, the1studio, org]

    # NOT: Single string (less flexible)
    # runs-on: self-hosted
```

### Workflow Structure
```yaml
name: Descriptive Workflow Name

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  workflow_dispatch:  # Manual trigger

jobs:
  job-name:
    runs-on: [self-hosted, arc, the1studio, org]
    steps:
      - name: Descriptive step name
        uses: actions/checkout@v4

      - name: Run build
        run: |
          # Multi-line commands
          npm install
          npm run build
```

---

## Configuration Management

### Secrets
**Never commit secrets to Git**:
```yaml
# ✅ CORRECT: Reference Kubernetes secret
spec:
  env:
    - name: API_KEY
      valueFrom:
        secretKeyRef:
          name: api-credentials
          key: api-key

# ❌ WRONG: Hardcode secret
spec:
  env:
    - name: API_KEY
      value: "sk-abc123..."  # NEVER DO THIS
```

### Environment Variables
```yaml
# Group related variables
env:
  # Build configuration
  - name: NODE_ENV
    value: "production"
  - name: BUILD_TARGET
    value: "linux"

  # Feature flags
  - name: ENABLE_CACHE
    value: "true"
```

### Version Pinning
**Always pin versions for reproducibility**:
```yaml
# Kubernetes images
image: the1studio/actions-runner:v2.328.0-ubuntu-22.04

# GitHub Actions
- uses: actions/checkout@v4  # Pin major version
- uses: actions/cache@v3.3.1  # Pin exact version for critical actions

# Helm charts
helm install arc actions-runner-controller/actions-runner-controller --version 0.23.0
```

---

## Documentation Standards

### Markdown Formatting

#### Headers
```markdown
# Title (H1) - One per document

## Major Section (H2)

### Subsection (H3)

#### Minor Section (H4)
```

#### Code Blocks
```markdown
# Always specify language
```yaml
apiVersion: v1
kind: ConfigMap
```

# NOT just backticks
```
generic code
```
```

#### Links
```markdown
# Relative links for internal docs
See [USAGE.md](./USAGE.md) for details

# Absolute links for external
See [ARC GitHub](https://github.com/actions/actions-runner-controller)
```

### Documentation Structure
Each document should include:
1. **Title** and metadata (last updated, author, etc.)
2. **Overview** section
3. **Table of contents** (for long docs)
4. **Main content** with clear sections
5. **Examples** with code blocks
6. **Troubleshooting** (if applicable)
7. **References** and related links

### Comment Style

#### YAML/Kubernetes
```yaml
# Single-line comments for brief explanations
apiVersion: v1

# Multi-line comments for complex explanations:
# This configuration sets up the runner with specific resource limits
# to prevent resource exhaustion while allowing burst capacity
resources:
  limits:
    cpu: "2"
```

#### Dockerfile
```dockerfile
# Descriptive comment explaining WHY, not just WHAT
# Fix APT sources to use HTTPS instead of HTTP (port 80 blocked in k8s)
RUN sed -i 's|http://archive.ubuntu.com|https://archive.ubuntu.com|g' /etc/apt/sources.list
```

#### Bash Scripts
```bash
# Script header with purpose and usage
#!/bin/bash
# Purpose: Backup ARC configuration
# Usage: ./backup.sh

# Function-level comments
backup_kubernetes() {
  # Export runner deployments
  kubectl get runnerdeployment -n arc-runners -o yaml > backup.yaml
}
```

---

## Git Workflow

### Commit Messages
**Format**: `<type>: <description>`

**Types**:
- `feat`: New feature or capability
- `fix`: Bug fix
- `docs`: Documentation changes
- `chore`: Maintenance tasks
- `refactor`: Code refactoring
- `security`: Security fixes

**Examples**:
```
feat: Add GPU runner pool for ML workflows
fix: Resolve port 80 HTTP blocking issue
docs: Update troubleshooting guide with public repo access
chore: Update base image to v2.328.0
security: Rotate GitHub PAT credentials
```

### Branch Naming
- `main` or `master` - Production branch
- `develop` - Development branch
- `feature/description` - Feature branches
- `fix/issue-number` - Bug fix branches
- `hotfix/critical-fix` - Emergency fixes

### File Organization in Git
```
# Commit related files together
git add k8s/runner-deployments.yaml k8s/autoscalers.yaml
git commit -m "feat: Add new runner pool for ML workloads"

# Update documentation with code changes
git add k8s/ docs/USAGE.md README.md
git commit -m "feat: Add GPU runner pool with documentation"
```

---

## Security Standards

### Network Policies
**Always define egress and ingress rules**:
```yaml
spec:
  policyTypes:
    - Egress
    - Ingress

  # Explicit allow rules only
  egress:
    - ports:
      - protocol: TCP
        port: 443

  # Deny by default (implicit)
```

### Resource Limits
**Prevent DoS and resource exhaustion**:
```yaml
resources:
  limits:
    cpu: "2"          # Prevent CPU hogging
    memory: "4Gi"     # Prevent OOM on host
  requests:
    cpu: "500m"       # Guarantee minimum
    memory: "1Gi"     # Guarantee minimum
```

### Pod Security
```yaml
securityContext:
  # Prefer non-root when possible
  runAsNonRoot: true
  runAsUser: 1000

  # Drop unnecessary capabilities
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE  # Only if needed
```

---

## Testing Standards

### Workflow Testing
**Always test new runner configurations**:
1. Create test workflow with simple job
2. Verify runner picks up job
3. Test Docker access (if required)
4. Test resource limits
5. Test auto-scaling behavior

**Example test workflow**:
```yaml
name: Test New Runner Pool

on: workflow_dispatch

jobs:
  test:
    runs-on: [self-hosted, arc, new-pool]
    steps:
      - name: Basic test
        run: |
          echo "Runner: $RUNNER_NAME"
          echo "OS: $RUNNER_OS"

      - name: Docker test
        run: docker --version

      - name: Resource test
        run: |
          free -h
          nproc
```

### Validation Checklist
Before deploying changes:
- [ ] YAML syntax valid (`kubectl apply --dry-run=client`)
- [ ] Labels match workflow expectations
- [ ] Resource limits set appropriately
- [ ] Network policies allow required traffic
- [ ] Documentation updated
- [ ] Tested in development first
- [ ] Backup created

---

## Maintenance Standards

### Regular Reviews
**Monthly**:
- Review runner utilization
- Check for base image updates
- Review resource limits
- Update documentation

**Quarterly**:
- Rotate GitHub PAT
- Review security policies
- Update Kubernetes components
- Conduct disaster recovery drill

### Update Procedures

#### Base Image Updates
```bash
# 1. Pull new base image
docker pull summerwind/actions-runner:latest

# 2. Update Dockerfile
vim docker/Dockerfile  # Update FROM line

# 3. Build and test
docker build -t the1studio/actions-runner:https-apt .
docker run --rm -it the1studio/actions-runner:https-apt /bin/bash

# 4. Import to k3s
docker save the1studio/actions-runner:https-apt | sudo k3s ctr images import -

# 5. Update documentation
vim docker/README.md  # Update version references

# 6. Commit changes
git add docker/
git commit -m "chore: Update base image to v2.329.0"
```

#### Kubernetes Updates
```bash
# 1. Backup current state
kubectl get runnerdeployment -n arc-runners -o yaml > backup.yaml

# 2. Apply changes
kubectl apply -f k8s/runner-deployments.yaml

# 3. Verify deployment
kubectl get pods -n arc-runners
kubectl describe pod <pod-name> -n arc-runners

# 4. Test workflow
# Trigger test workflow and verify

# 5. Monitor for issues
kubectl logs -n arc-runners <pod-name> -c runner -f
```

---

## Error Handling

### Kubernetes Errors
**Check in this order**:
1. Pod status: `kubectl get pods -n arc-runners`
2. Pod events: `kubectl describe pod <pod-name> -n arc-runners`
3. Pod logs: `kubectl logs <pod-name> -c runner -n arc-runners`
4. Controller logs: `kubectl logs -n arc-systems -l app.kubernetes.io/name=actions-runner-controller`

### Common Issues
**Document solutions for frequent issues**:
- Port 80 HTTP blocking → Custom image with HTTPS APT
- Public repo access denied → Enable in runner group settings
- Pod stuck ContainerCreating → Check secret exists
- Runner not registering → Verify GitHub token permissions

---

## Performance Optimization

### Image Optimization
- Use multi-stage builds (if applicable)
- Minimize layer count
- Clean up package manager cache
- Pin exact versions for cache hits

### Resource Optimization
- Set appropriate request/limit ratios (2-4x)
- Use auto-scaling to match demand
- Monitor and adjust based on actual usage
- Use pod disruption budgets for availability

### Network Optimization
- Use HTTPS instead of HTTP (faster TLS resume)
- Enable DNS caching
- Use local mirrors for packages (future enhancement)

---

## Compliance

### Audit Trail
All changes must be:
- Committed to Git with descriptive messages
- Documented in relevant docs
- Tested before production deployment
- Reviewed by maintainer

### Security Compliance
- No secrets in Git (use Kubernetes secrets)
- Network policies enforced
- Resource limits set
- Regular security reviews
- Audit logs retained (30 days)

---

## Tools & Utilities

### Required Tools
- `kubectl` - Kubernetes CLI
- `helm` - Kubernetes package manager
- `docker` - Container management
- `gh` - GitHub CLI
- `k3s` - Kubernetes distribution

### Recommended Tools
- `k9s` - Kubernetes TUI
- `kubectx` - Context switching
- `stern` - Multi-pod log tailing
- `jq` - JSON processing

---

## References

- **Kubernetes Documentation**: https://kubernetes.io/docs/
- **ARC Documentation**: https://github.com/actions/actions-runner-controller
- **GitHub Actions**: https://docs.github.com/en/actions
- **Docker Best Practices**: https://docs.docker.com/develop/dev-best-practices/
- **YAML Specification**: https://yaml.org/spec/

---

**Document Version**: 1.0
**Last Reviewed**: 2025-11-09
**Next Review**: Q1 2026
