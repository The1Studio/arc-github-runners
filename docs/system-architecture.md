# System Architecture

**Last Updated**: 2025-11-09
**Repository**: https://github.com/The1Studio/arc-github-runners

---

## Overview

This document describes the system architecture of the self-hosted GitHub Actions runner infrastructure using Actions Runner Controller (ARC) on k3s Kubernetes.

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         GitHub.com                                  │
│  ┌─────────────────┐           ┌──────────────────┐               │
│  │  Organization:  │           │  Repository:     │               │
│  │  the1studio     │           │  tuha263/        │               │
│  │                 │           │  personal-site   │               │
│  └────────┬────────┘           └────────┬─────────┘               │
└───────────┼──────────────────────────────┼──────────────────────────┘
            │                              │
            │ GitHub API (HTTPS/443)       │
            │ Runner Registration          │
            │ Job Assignment               │
            │                              │
┌───────────▼──────────────────────────────▼──────────────────────────┐
│                   Local Infrastructure                              │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                    k3s Cluster (Single Node)                  │ │
│  │                                                               │ │
│  │  ┌──────────────────────────────────────────────────────────┐│ │
│  │  │           Namespace: arc-systems                         ││ │
│  │  │                                                          ││ │
│  │  │  ┌────────────────────────────────────────────────┐    ││ │
│  │  │  │  ARC Controller Pod                            │    ││ │
│  │  │  │  - Manages runner lifecycle                    │    ││ │
│  │  │  │  - Communicates with GitHub API                │    ││ │
│  │  │  │  - Spawns/destroys runner pods                 │    ││ │
│  │  │  │  - Monitors auto-scaling rules                 │    ││ │
│  │  │  └─────────────┬──────────────────────────────────┘    ││ │
│  │  │                │                                        ││ │
│  │  │  ┌─────────────▼──────────────────────────────────┐    ││ │
│  │  │  │  cert-manager                                   │    ││ │
│  │  │  │  - TLS certificate management                   │    ││ │
│  │  │  └────────────────────────────────────────────────┘    ││ │
│  │  │                                                          ││ │
│  │  │  ┌────────────────────────────────────────────────┐    ││ │
│  │  │  │  Secret: controller-manager                     │    ││ │
│  │  │  │  - GitHub PAT token                             │    ││ │
│  │  │  └────────────────────────────────────────────────┘    ││ │
│  │  └──────────────────────────────────────────────────────────┘│ │
│  │                                                               │ │
│  │  ┌──────────────────────────────────────────────────────────┐│ │
│  │  │           Namespace: arc-runners                         ││ │
│  │  │                                                          ││ │
│  │  │  ┌────────────────────────────────────────────────┐    ││ │
│  │  │  │  RunnerDeployment: the1studio-org-runners      │    ││ │
│  │  │  │  - Organization-level runners                  │    ││ │
│  │  │  │  - Min: 3, Max: 10 replicas                    │    ││ │
│  │  │  └─────────────┬──────────────────────────────────┘    ││ │
│  │  │                │                                        ││ │
│  │  │  ┌─────────────▼──────────────────────────────────┐    ││ │
│  │  │  │  HorizontalRunnerAutoscaler                     │    ││ │
│  │  │  │  - Monitors runner utilization                  │    ││ │
│  │  │  │  - Scales up/down based on metrics              │    ││ │
│  │  │  └────────────────────────────────────────────────┘    ││ │
│  │  │                                                          ││ │
│  │  │  ┌────────────────────────────────────────────────┐    ││ │
│  │  │  │  Runner Pods (2-10 replicas)                    │    ││ │
│  │  │  │  ┌──────────────────────────────────────────┐  │    ││ │
│  │  │  │  │  Container: runner                       │  │    ││ │
│  │  │  │  │  - Executes GitHub Actions workflows     │  │    ││ │
│  │  │  │  │  - Mounts Docker socket                  │  │    ││ │
│  │  │  │  │  - Resource: 2 CPU, 4Gi memory (max)     │  │    ││ │
│  │  │  │  └──────────────────────────────────────────┘  │    ││ │
│  │  │  │  ┌──────────────────────────────────────────┐  │    ││ │
│  │  │  │  │  Container: docker (sidecar)             │  │    ││ │
│  │  │  │  │  - Docker-in-Docker daemon               │  │    ││ │
│  │  │  │  │  - Resource: 1 CPU, 2Gi memory (max)     │  │    ││ │
│  │  │  │  └──────────────────────────────────────────┘  │    ││ │
│  │  │  └────────────────────────────────────────────────┘    ││ │
│  │  │                                                          ││ │
│  │  │  ┌────────────────────────────────────────────────┐    ││ │
│  │  │  │  RunnerDeployment: tuha263-personal-runners    │    ││ │
│  │  │  │  - Repository-level runners                    │    ││ │
│  │  │  │  - Min: 1, Max: 5 replicas                     │    ││ │
│  │  │  └────────────────────────────────────────────────┘    ││ │
│  │  │                                                          ││ │
│  │  │  ┌────────────────────────────────────────────────┐    ││ │
│  │  │  │  NetworkPolicy: runner-egress-policy           │    ││ │
│  │  │  │  - Allows: HTTPS (443), SSH (22), DNS (53)     │    ││ │
│  │  │  │  - Blocks: HTTP (80) and all other ports       │    ││ │
│  │  │  └────────────────────────────────────────────────┘    ││ │
│  │  │                                                          ││ │
│  │  │  ┌────────────────────────────────────────────────┐    ││ │
│  │  │  │  PodDisruptionBudget                           │    ││ │
│  │  │  │  - Ensures min 1 runner always available       │    ││ │
│  │  │  └────────────────────────────────────────────────┘    ││ │
│  │  └──────────────────────────────────────────────────────────┘│ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                    │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │                    Docker Host                                │ │
│  │  - Docker socket: /var/run/docker.sock                       │ │
│  │  - Shared with runner pods for Docker-in-Docker              │ │
│  └───────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────┘
```

---

## Component Architecture

### 1. GitHub Integration Layer

#### GitHub API
- **Purpose**: Runner registration, job assignment, workflow orchestration
- **Protocol**: HTTPS (port 443)
- **Authentication**: GitHub Personal Access Token (PAT)
- **Rate Limits**: 5,000 requests/hour (sufficient for current scale)

**Key Interactions**:
1. Runner registration (when pod starts)
2. Job polling (continuous)
3. Job status updates (during execution)
4. Runner de-registration (when pod terminates)

#### Organization Runners
- **Organization**: `the1studio`
- **Scope**: All repositories in organization
- **Runner Group**: Default (ID: 1)
- **Public Repository Access**: Configurable (must be enabled)

#### Repository Runners
- **Repository**: `tuha263/personal-site`
- **Scope**: Single repository only
- **Isolation**: Cannot be used by other repositories

---

### 2. Kubernetes Control Plane (k3s)

#### k3s Overview
- **Type**: Lightweight Kubernetes distribution
- **Components**:
  - API Server (port 6443)
  - Controller Manager
  - Scheduler
  - etcd (embedded)
  - Kubelet
  - Kube-proxy

**Resource Footprint**:
- Memory: ~150MB base
- CPU: Minimal (<0.1 core idle)
- Storage: /var/lib/rancher/k3s (database, snapshots)

#### Configuration
- **Kubeconfig**: `/etc/rancher/k3s/k3s.yaml`
- **Service**: systemd (`k3s.service`)
- **CNI**: Flannel (default)
- **Service Proxy**: kube-proxy (iptables mode)

---

### 3. ARC Controller (`arc-systems` namespace)

#### Actions Runner Controller
**Deployment**: Single pod (can be scaled if needed)

**Responsibilities**:
1. **Runner Lifecycle Management**:
   - Creates runner pods based on RunnerDeployment specs
   - Destroys completed runner pods
   - Handles runner registration with GitHub

2. **Auto-scaling**:
   - Monitors HorizontalRunnerAutoscaler resources
   - Calculates desired replica count
   - Triggers scale-up/scale-down operations

3. **GitHub API Communication**:
   - Authenticates using PAT from secret
   - Polls GitHub for workflow jobs
   - Updates runner status

**Configuration**:
```yaml
Helm Chart: actions-runner-controller/actions-runner-controller
Version: 0.23.0+
Namespace: arc-systems
Replicas: 1
```

**Dependencies**:
- `controller-manager` secret (GitHub PAT)
- cert-manager (TLS certificates)
- Kubernetes API server

#### cert-manager
**Purpose**: Automatic TLS certificate management for webhooks

**Components**:
- cert-manager controller
- webhook component
- cainjector

**Namespace**: `cert-manager`

---

### 4. Runner Infrastructure (`arc-runners` namespace)

#### RunnerDeployment Resources

**Organization Runners**:
```yaml
Name: the1studio-org-runners
Type: Organization-level
Target: organization: the1studio
Image: the1studio/actions-runner:https-apt
Replicas: Managed by autoscaler (3-10)
Labels: [self-hosted, linux, x64, arc, the1studio, org]
```

**Personal Runners**:
```yaml
Name: tuha263-personal-runners
Type: Repository-level
Target: repository: tuha263/personal-site
Image: the1studio/actions-runner:https-apt
Replicas: Managed by autoscaler (1-5)
Labels: [self-hosted, linux, x64, arc, personal]
```

#### Runner Pods

**Pod Structure**:
```
Runner Pod
├── Container: runner
│   ├── Image: the1studio/actions-runner:https-apt
│   ├── Base: summerwind/actions-runner:v2.328.0-ubuntu-22.04
│   ├── User: runner (UID 1000)
│   ├── Volumes: /var/run/docker.sock (mounted from host)
│   └── Resources:
│       ├── CPU: 500m request, 2 core limit
│       └── Memory: 1Gi request, 4Gi limit
│
└── Container: docker (sidecar - optional)
    ├── Image: docker:dind
    ├── Purpose: Docker-in-Docker daemon
    └── Resources:
        ├── CPU: 250m request, 1 core limit
        └── Memory: 512Mi request, 2Gi limit
```

**Note**: Current configuration uses host Docker socket instead of DinD sidecar for better performance.

#### Auto-scaling Logic

**HorizontalRunnerAutoscaler**:
```
Metrics Collection (every 30s):
  ├── Query GitHub API for workflow job queue
  ├── Count running runners
  ├── Count idle runners
  └── Calculate utilization percentage

Decision Logic:
  ├── IF utilization > 75%:
  │   └── Scale up by factor 1.5x (rounded up)
  ├── ELSE IF utilization < 25% AND (time since last scale-up > 5 min):
  │   └── Scale down by factor 0.5x (rounded down)
  └── ELSE:
      └── No change

Constraints:
  ├── Min replicas: 3 (org) / 1 (personal)
  ├── Max replicas: 10 (org) / 5 (personal)
  └── Scale down delay: 300s (org) / 180s (personal)
```

**Scaling Examples**:
```
Scenario 1: Sudden load
  Current: 3 runners (all busy = 100% utilization)
  Trigger: utilization > 75%
  Action: Scale to 3 × 1.5 = 4.5 → 5 runners

Scenario 2: Sustained load
  Current: 5 runners (4 busy = 80% utilization)
  Trigger: utilization > 75%
  Action: Scale to 5 × 1.5 = 7.5 → 8 runners

Scenario 3: Load decrease
  Current: 8 runners (2 busy = 25% utilization)
  Trigger: utilization <= 25% (after 5 min delay)
  Action: Scale to 8 × 0.5 = 4 runners

Scenario 4: Idle state
  Current: 4 runners (0 busy = 0% utilization)
  Trigger: utilization < 25% (after 5 min delay)
  Action: Scale to 4 × 0.5 = 2 → 3 (min replicas)
```

---

### 5. Network Architecture

#### Network Topology
```
┌─────────────────────────────────────────────┐
│           External Network                  │
│  - GitHub.com (api.github.com)             │
│  - GitHub Actions (pipelines.actions.github.com) │
│  - Ubuntu APT (archive.ubuntu.com)         │
│  - PyPI, npm, Docker Hub, etc.             │
└────────────────┬────────────────────────────┘
                 │
                 │ HTTPS (443)
                 │ SSH (22)
                 │
┌────────────────▼────────────────────────────┐
│          Host Network (k3s node)            │
│  - Physical network interface               │
│  - iptables rules (kube-proxy)              │
└────────────────┬────────────────────────────┘
                 │
        ┌────────┴────────┐
        │                 │
┌───────▼──────┐  ┌───────▼──────┐
│ arc-systems  │  │ arc-runners  │
│ namespace    │  │ namespace    │
│              │  │              │
│ ┌──────────┐ │  │ ┌──────────┐ │
│ │Controller│ │  │ │ Runner 1 │ │
│ └────┬─────┘ │  │ └──────────┘ │
│      │       │  │ ┌──────────┐ │
│      │       │  │ │ Runner 2 │ │
│      │       │  │ └──────────┘ │
│      │       │  │ ┌──────────┐ │
│      │       │  │ │ Runner N │ │
│      │       │  │ └──────────┘ │
└──────┼───────┘  └──────────────┘
       │
       │ Kubernetes API (6443)
       │
┌──────▼───────────────────────────────────────┐
│        kube-system namespace                 │
│  - CoreDNS (DNS resolution)                  │
│  - Flannel (CNI)                             │
│  - Metrics Server                            │
└──────────────────────────────────────────────┘
```

#### Network Policies

**Runner Egress Policy**:
```yaml
Allowed Egress:
  ├── DNS (UDP/53) → kube-system namespace
  ├── HTTPS (TCP/443) → All destinations
  └── SSH (TCP/22) → All destinations

Blocked Egress:
  └── HTTP (TCP/80) → Denied by default
```

**Controller Egress Policy**:
```yaml
Allowed Egress:
  ├── DNS (UDP/53) → kube-system namespace
  ├── HTTPS (TCP/443) → All destinations
  └── Kubernetes API (TCP/6443) → Within cluster

Blocked Egress:
  └── All other ports → Denied by default
```

**Ingress Policy**:
```yaml
Runner Ingress:
  └── Only from arc-systems namespace

Controller Ingress:
  └── Cluster-internal only (no external)
```

---

### 6. Storage Architecture

#### Pod Storage
- **Type**: EmptyDir (ephemeral)
- **Lifetime**: Pod lifetime
- **Purpose**: Workspace, build artifacts, cache

**Path Structure**:
```
/runner/
├── _work/           # Workspace for job execution
├── _diag/           # Diagnostic logs
└── _temp/           # Temporary files
```

#### Host Volume Mounts
```yaml
Docker Socket:
  Host Path: /var/run/docker.sock
  Container Path: /var/run/docker.sock
  Type: Socket
  Purpose: Docker-in-Docker support
```

#### Persistent Storage (k3s)
```
/var/lib/rancher/k3s/
├── server/
│   ├── db/
│   │   └── snapshots/     # etcd snapshots (daily)
│   └── manifests/         # Built-in manifests
└── agent/
    └── containerd/        # Image storage
```

---

### 7. Security Architecture

#### Pod Security

**Security Context (Organization Runners)**:
```yaml
Pod Level:
  runAsNonRoot: false      # Required for Docker socket access
  fsGroup: 121            # docker group ID on host

Container Level:
  runAsUser: 1000         # runner user
  allowPrivilegeEscalation: false
```

**Resource Limits**:
- Prevent DoS attacks
- Prevent resource exhaustion
- Ensure fair sharing

#### Network Security

**Defense in Depth**:
```
Layer 1: NetworkPolicy (Kubernetes)
  └── Restrict egress/ingress at pod level

Layer 2: iptables (kube-proxy)
  └── Enforce NetworkPolicy rules

Layer 3: Host Firewall (optional)
  └── Additional protection at host level
```

#### Secret Management

**GitHub PAT Storage**:
```yaml
Type: Kubernetes Secret (Opaque)
Name: controller-manager
Namespace: arc-systems
Key: github_token
Access: ARC controller only
Rotation: Quarterly (manual)
```

**Best Practices**:
- Never commit secrets to Git
- Use Kubernetes secrets for sensitive data
- Limit secret access via RBAC
- Rotate secrets regularly

#### Access Control

**Kubernetes RBAC**:
```
Service Account: actions-runner-controller
Namespace: arc-systems
Permissions:
  ├── runnerdeployments (all verbs)
  ├── horizontalrunnerautoscalers (all verbs)
  ├── pods (all verbs)
  ├── secrets (get, list)
  └── events (create, patch)
```

---

### 8. Monitoring & Observability

#### Metrics Collection
```
Source: ARC Controller
  ├── Runner count (current, desired)
  ├── Runner status (idle, busy, offline)
  └── Auto-scaler state

Source: Kubernetes
  ├── Pod status (running, pending, failed)
  ├── Resource utilization (CPU, memory)
  └── Events (scaling, restarts, failures)

Source: GitHub API
  ├── Workflow job queue
  ├── Job execution time
  └── Success/failure rates
```

#### Logging
```
Component Logs:
  ├── ARC Controller:
  │   └── kubectl logs -n arc-systems -l app.kubernetes.io/name=actions-runner-controller
  ├── Runner Pods:
  │   └── kubectl logs -n arc-runners <pod-name> -c runner
  ├── k3s System:
  │   └── journalctl -u k3s
  └── Workflow Logs:
      └── GitHub Actions UI
```

#### Health Checks
```
ARC Controller:
  ├── Liveness Probe: HTTP /healthz
  ├── Readiness Probe: HTTP /readyz
  └── Check Interval: 10s

Runner Pods:
  ├── Startup Probe: GitHub registration success
  └── Liveness Probe: Process running
```

---

### 9. High Availability & Resilience

#### Pod Disruption Budgets
```yaml
Organization Runners:
  minAvailable: 1
  Purpose: Ensure at least 1 runner during voluntary disruptions

Personal Runners:
  minAvailable: 1
  Purpose: Ensure availability for critical personal projects
```

#### Failure Handling

**Pod Failures**:
```
Failure: Pod crashes
  └── Kubernetes automatically restarts pod

Failure: Node failure
  └── Single-node cluster (no automatic recovery)
  └── Manual intervention required

Failure: Runner registration fails
  └── Pod restarts (exponential backoff)
  └── After 5 failures, pod marked as failed
```

**Auto-scaling Safeguards**:
```
Protection: Min replicas
  └── Never scale below minimum (ensures availability)

Protection: Max replicas
  └── Never scale above maximum (prevents resource exhaustion)

Protection: Scale down delay
  └── Wait 3-5 minutes after scale-up before scaling down
  └── Prevents flapping during intermittent load
```

---

### 10. Data Flow

#### Workflow Execution Flow
```
1. Developer pushes code to GitHub
   │
2. GitHub webhook triggers workflow
   │
3. Workflow waits for runner with matching labels
   │
4. ARC controller polls GitHub API
   │
   ├── Detects queued workflow
   │
   ├── Checks available runners
   │
   └── If no runners available:
       └── Auto-scaler triggers scale-up
           └── New runner pod created
               └── Runner registers with GitHub
   │
5. GitHub assigns job to runner
   │
6. Runner pod executes workflow steps:
   │
   ├── Checkout code (via git over HTTPS/SSH)
   │
   ├── Install dependencies (via apt/npm/pip over HTTPS)
   │
   ├── Build application
   │
   ├── Run tests
   │
   └── Upload artifacts (via GitHub API over HTTPS)
   │
7. Job completes
   │
   ├── Runner reports status to GitHub
   │
   ├── Runner becomes idle (available for next job)
   │
   └── After idle timeout:
       └── Auto-scaler may scale down
           └── Pod terminated gracefully
```

---

## Technology Stack

### Infrastructure Layer
- **Operating System**: Arch Linux (host)
- **Kernel**: Linux 6.x
- **Container Runtime**: Docker 24.x
- **Kubernetes**: k3s v1.28+

### Application Layer
- **Runner Controller**: Actions Runner Controller (Helm chart)
- **Runner Base**: summerwind/actions-runner:v2.328.0
- **Custom Image**: the1studio/actions-runner:https-apt
- **Certificate Management**: cert-manager v1.14+

### Networking Layer
- **CNI**: Flannel (default with k3s)
- **Service Proxy**: kube-proxy (iptables mode)
- **DNS**: CoreDNS
- **Load Balancing**: Built-in (kube-proxy)

### Management Tools
- **Package Manager**: Helm 3
- **CLI**: kubectl, gh (GitHub CLI)
- **Configuration**: YAML manifests (GitOps)

---

## Deployment Model

### Single-Node Cluster
**Pros**:
- Simple setup and maintenance
- Low resource overhead
- Suitable for small-to-medium scale

**Cons**:
- Single point of failure
- Limited scalability
- No automatic failover

**Mitigation**:
- Daily backups (k3s snapshots)
- Comprehensive documentation
- Fast recovery procedures (< 2 hours)

### Resource Allocation
```
Total System Resources:
  CPU: 8 cores
  Memory: 16 GB
  Storage: 256 GB SSD

k3s System:
  CPU: ~0.1 core
  Memory: ~150 MB

ARC Controller:
  CPU: ~0.1 core
  Memory: ~100 MB

Runners (per pod):
  CPU: 0.5-2 cores (request-limit)
  Memory: 1-4 GB (request-limit)

Maximum Concurrent:
  Calculation: 16 GB / 4 GB (max per pod) = 4 pods (conservative)
  Configured: Max 10 pods (allows for burst, may cause memory pressure)
  Typical: 3-5 pods (72% utilization)
```

---

## Future Architecture Enhancements

### Phase 2: High Availability
- Multi-node k3s cluster (3+ nodes)
- Distributed etcd
- Load balancing across nodes
- Automatic pod rescheduling on node failure

### Phase 3: Advanced Scaling
- Cluster autoscaler (add/remove nodes)
- Vertical pod autoscaler (adjust resource requests)
- Scale-to-zero with webhook-based scaling
- GPU runner pools for ML workloads

### Phase 4: Observability
- Prometheus metrics collection
- Grafana dashboards
- AlertManager for notifications
- Distributed tracing (Jaeger)

---

## Security Considerations

### Attack Vectors & Mitigations

**1. Malicious Workflow**:
- **Risk**: Workflow executes malicious code on runner
- **Mitigation**:
  - Resource limits (prevent DoS)
  - Network policies (limit egress)
  - Ephemeral pods (no persistent compromise)
  - Code review before merge (organizational policy)

**2. Network Attack**:
- **Risk**: Runner attempts to access internal resources
- **Mitigation**:
  - NetworkPolicy enforcement
  - Egress allowlist (HTTPS, SSH, DNS only)
  - No ingress from external sources

**3. Privilege Escalation**:
- **Risk**: Container escapes to host
- **Mitigation**:
  - Non-root user (where possible)
  - No privileged containers
  - AppArmor/SELinux profiles (future)

**4. Secret Exposure**:
- **Risk**: GitHub token leaked
- **Mitigation**:
  - Kubernetes secret (not in Git)
  - RBAC limits secret access
  - Quarterly rotation
  - Minimum required permissions

---

## References

- **k3s Architecture**: https://docs.k3s.io/architecture
- **ARC Architecture**: https://github.com/actions/actions-runner-controller/blob/master/docs/about-arc.md
- **Kubernetes Concepts**: https://kubernetes.io/docs/concepts/
- **GitHub Actions**: https://docs.github.com/en/actions/hosting-your-own-runners

---

**Document Version**: 1.0
**Last Reviewed**: 2025-11-09
**Next Review**: Q1 2026
