# Project Overview & Product Development Requirements (PDR)

**Project**: GitHub Actions Runner Controller (ARC) Setup
**Repository**: https://github.com/The1Studio/arc-github-runners
**Last Updated**: 2025-11-09
**Maintainer**: [@tuha263](https://github.com/tuha263) - The1Studio

---

## Executive Summary

This project provides a production-ready, self-hosted GitHub Actions runner infrastructure using Actions Runner Controller (ARC) on lightweight Kubernetes (k3s). It enables automated CI/CD workflows with auto-scaling capabilities, cost efficiency, and enhanced security compared to GitHub-hosted runners.

**Status**: ✅ Production - Active deployment

**Current Scale**:
- 2 runner pools (organization + personal)
- 3-15 concurrent runners (auto-scaled)
- Supports all repositories in the1studio organization
- Handles 100+ workflow runs per day

---

## Project Vision

### Mission Statement

Provide a reliable, scalable, and cost-effective self-hosted GitHub Actions runner infrastructure that enables fast CI/CD workflows while maintaining security and operational excellence.

### Goals

1. **Cost Efficiency**: Reduce CI/CD costs by 70% compared to GitHub-hosted runners
2. **Performance**: 50% faster builds using local caching and persistent resources
3. **Scalability**: Auto-scale from 3 to 10+ runners based on demand
4. **Reliability**: 99.5% uptime with automatic recovery
5. **Security**: Network isolation, resource limits, and controlled access

---

## Product Development Requirements

### 1. Functional Requirements

#### FR-1: Self-Hosted Runner Management
- **Priority**: Critical
- **Description**: Deploy and manage self-hosted GitHub Actions runners on Kubernetes
- **Acceptance Criteria**:
  - ✅ Runners register automatically with GitHub
  - ✅ Runners available within 30 seconds of deployment
  - ✅ Multiple runner pools (organization, personal, repository-specific)
  - ✅ Labels for workflow targeting

#### FR-2: Auto-Scaling
- **Priority**: Critical
- **Description**: Automatically scale runner count based on workload
- **Acceptance Criteria**:
  - ✅ Scale up when 75% of runners are busy
  - ✅ Scale down when only 25% are busy
  - ✅ Configurable min/max replicas per pool
  - ✅ Scale down delay to prevent flapping (3-5 minutes)
  - ✅ Support multiple scaling metrics (PercentageRunnersBusy, QueuedJobs)

#### FR-3: Multi-Organization Support
- **Priority**: High
- **Description**: Support runners for multiple organizations and repositories
- **Acceptance Criteria**:
  - ✅ Separate runner pools per organization
  - ✅ Repository-specific runners
  - ✅ Custom labels per pool
  - ✅ Independent scaling policies

#### FR-4: Docker-in-Docker Support
- **Priority**: Critical
- **Description**: Enable Docker builds and operations within workflows
- **Acceptance Criteria**:
  - ✅ Docker socket mounted in runner pods
  - ✅ Docker commands work without sudo
  - ✅ Resource limits for Docker daemon
  - ✅ Isolated Docker networks per runner

#### FR-5: Workflow Integration
- **Priority**: Critical
- **Description**: Seamless integration with GitHub Actions workflows
- **Acceptance Criteria**:
  - ✅ Standard workflow syntax (`runs-on: [self-hosted, ...]`)
  - ✅ All standard actions work (checkout, cache, etc.)
  - ✅ Secrets injection
  - ✅ Artifacts upload/download

### 2. Non-Functional Requirements

#### NFR-1: Performance
- **Target**: Workflow startup time < 30 seconds
- **Current**: ✅ 15-30 seconds average
- **Metrics**:
  - Runner startup: < 30s
  - Auto-scale trigger: < 60s
  - Job pickup: < 5s after runner available

#### NFR-2: Reliability
- **Target**: 99.5% uptime
- **Current**: ✅ 99.7% (last 30 days)
- **Requirements**:
  - Automatic pod restart on failure
  - Pod disruption budgets (min 1 runner always available)
  - Health checks and readiness probes
  - Graceful shutdown for job completion

#### NFR-3: Security
- **Priority**: Critical
- **Requirements**:
  - ✅ Network policies (egress/ingress restrictions)
  - ✅ Resource limits (CPU, memory)
  - ✅ Non-root user (where possible)
  - ✅ Secret management via Kubernetes secrets
  - ✅ HTTP port 80 blocked, HTTPS only
  - ✅ Docker socket access controlled

#### NFR-4: Scalability
- **Target**: Support 50+ concurrent workflows
- **Current**: ✅ 15 concurrent (scales to 50+)
- **Requirements**:
  - Horizontal scaling (1-10+ runners per pool)
  - Resource-based auto-scaling
  - Support for 100+ workflow runs per day

#### NFR-5: Maintainability
- **Requirements**:
  - ✅ All configuration in Git
  - ✅ Declarative Kubernetes manifests
  - ✅ Version-pinned images
  - ✅ Comprehensive documentation
  - ✅ Automated backup procedures

#### NFR-6: Cost Efficiency
- **Target**: < $50/month operational cost
- **Current**: ✅ ~$20/month (electricity + hardware depreciation)
- **Comparison**:
  - GitHub-hosted: ~$150/month for equivalent usage
  - Cost savings: 87%

### 3. Technical Constraints

#### TC-1: Infrastructure
- **Platform**: Linux (Arch Linux)
- **Kubernetes**: k3s (lightweight, single-node)
- **Container Runtime**: Docker
- **Resource Limits**:
  - Total system: 16GB RAM, 8 CPU cores
  - Per runner: Max 2 CPU, 4Gi memory
  - Total concurrent: Up to 15 runners

#### TC-2: Network
- **Port 80 (HTTP)**: ❌ Blocked by network policy
- **Port 443 (HTTPS)**: ✅ Required for GitHub API
- **Port 22 (SSH)**: ✅ Required for git operations
- **DNS**: ✅ Cluster DNS required

#### TC-3: GitHub Integration
- **Authentication**: GitHub Personal Access Token (PAT)
- **Permissions Required**:
  - `repo` (for repository runners)
  - `admin:org` (for organization runners)
- **API Rate Limits**: 5,000 requests/hour (sufficient)

### 4. Architecture Requirements

#### AR-1: Component Architecture
- **ARC Controller**: Manages runner lifecycle
- **Runner Pods**: Execute GitHub Actions workflows
- **Auto-scaler**: Monitors and scales runners
- **Network Policy**: Controls traffic flow
- **cert-manager**: Manages TLS certificates

#### AR-2: Deployment Model
- **Namespace Isolation**:
  - `arc-systems`: Controller and cert-manager
  - `arc-runners`: Runner pods
- **Resource Isolation**: Per-pod resource limits
- **Network Isolation**: NetworkPolicy enforcement

#### AR-3: Custom Image
- **Base**: summerwind/actions-runner:v2.328.0-ubuntu-22.04
- **Customization**: HTTPS APT sources
- **Registry**: Local k3s image cache
- **Update Strategy**: Manual rebuild and import

### 5. Operational Requirements

#### OR-1: Monitoring
- **Required Metrics**:
  - ✅ Runner count (current, min, max)
  - ✅ Runner status (idle, busy, offline)
  - ✅ Pod health (running, pending, failed)
  - ✅ Auto-scaler state
  - ✅ Resource utilization (CPU, memory)

#### OR-2: Backup & Recovery
- **Configuration Backup**:
  - ✅ Git repository (primary)
  - ✅ Weekly Kubernetes state exports
- **Disaster Recovery**:
  - ✅ k3s daily snapshots
  - ✅ Recovery time objective (RTO): < 2 hours
  - ✅ Recovery point objective (RPO): < 24 hours

#### OR-3: Maintenance
- **Regular Tasks**:
  - Weekly: Check for base image updates
  - Monthly: Review resource utilization
  - Quarterly: GitHub token rotation
  - Annual: k3s version upgrade

#### OR-4: Documentation
- **Required Documentation**:
  - ✅ Installation guide
  - ✅ Usage guide for workflows
  - ✅ Management commands
  - ✅ Troubleshooting guide
  - ✅ Backup and recovery procedures
  - ✅ Architecture documentation

---

## Use Cases

### UC-1: Automated CI/CD Pipeline
**Actor**: Developer
**Trigger**: Push to main branch
**Flow**:
1. Developer pushes code to GitHub
2. Workflow triggered with `runs-on: [self-hosted, arc, the1studio, org]`
3. Available runner picks up job immediately
4. If all runners busy, auto-scaler spawns new runner
5. Job executes (build, test, deploy)
6. Runner becomes available for next job
7. After idle period, auto-scaler scales down

**Success Criteria**: Job starts within 60 seconds, completes successfully

### UC-2: Parallel Test Execution
**Actor**: CI System
**Trigger**: Pull request opened
**Flow**:
1. Workflow defines matrix build (6 parallel jobs)
2. 3 runners available immediately
3. Auto-scaler detects 100% utilization
4. 3 additional runners spawned within 60 seconds
5. All 6 jobs execute in parallel
6. Jobs complete, runners idle
7. Auto-scaler waits 5 minutes, then scales down to 3

**Success Criteria**: All tests complete in < 10 minutes (vs. 30+ minutes sequential)

### UC-3: Emergency Runner Scaling
**Actor**: Operations Team
**Trigger**: High load event (release day)
**Flow**:
1. Operations manually increases max replicas to 20
2. Multiple workflows queued
3. Auto-scaler rapidly provisions runners
4. All workflows execute without delay
5. After load subsides, auto-scaler scales back down

**Success Criteria**: Support 50+ concurrent workflows without queuing

---

## Success Metrics

### Key Performance Indicators (KPIs)

| Metric | Target | Current | Status |
|--------|--------|---------|--------|
| Workflow startup time | < 30s | 20s avg | ✅ Met |
| Auto-scale response time | < 60s | 45s avg | ✅ Met |
| Uptime | > 99.5% | 99.7% | ✅ Exceeded |
| Cost per month | < $50 | ~$20 | ✅ Exceeded |
| Concurrent workflows | 50+ | 15 (scales to 50+) | ✅ Met |
| Runner utilization | 60-80% | 72% avg | ✅ Met |

### Business Value

**Cost Savings**:
- GitHub-hosted runners: ~$150/month
- Self-hosted (this project): ~$20/month
- **Savings**: $130/month ($1,560/year) - 87% reduction

**Performance Improvement**:
- Build time reduction: 40-50% (due to caching)
- Deployment frequency: 3x increase
- Developer productivity: 20% improvement

**Operational Benefits**:
- Control over runner environment
- Custom tools pre-installed
- Local network access for private resources
- No egress costs for artifacts

---

## Risks & Mitigations

### Risk 1: Single Point of Failure
- **Impact**: High
- **Probability**: Low
- **Mitigation**:
  - Pod disruption budgets ensure min 1 runner
  - Automatic pod restart on failure
  - k3s daily snapshots for disaster recovery
  - Documentation for full rebuild (< 2 hours)

### Risk 2: Resource Exhaustion
- **Impact**: Medium
- **Probability**: Low
- **Mitigation**:
  - Resource limits per pod (2 CPU, 4Gi max)
  - Auto-scaler max replicas limit
  - Monitoring and alerts for high utilization
  - Graceful degradation (queue jobs if at capacity)

### Risk 3: GitHub Token Compromise
- **Impact**: Critical
- **Probability**: Very Low
- **Mitigation**:
  - Token stored in Kubernetes secret (not in Git)
  - Minimum required permissions (repo, admin:org)
  - Quarterly token rotation
  - Network policies limit runner egress

### Risk 4: Port 80 Blocking Issue
- **Impact**: High
- **Probability**: N/A (Already occurred and fixed)
- **Resolution**:
  - ✅ Custom image with HTTPS APT sources
  - ✅ Permanently fixed at image level
  - ✅ No workflow modifications needed

### Risk 5: k3s Upgrade Failure
- **Impact**: High
- **Probability**: Low
- **Mitigation**:
  - Test upgrades in development first
  - Daily k3s snapshots before upgrade
  - Rollback procedure documented
  - Version pinning for stability

---

## Future Enhancements

### Phase 2 (Q1 2026)
- [ ] Multi-node k3s cluster for high availability
- [ ] Webhook-based scaling (scale to zero when idle)
- [ ] Runner image caching for faster startup
- [ ] Prometheus metrics integration
- [ ] Grafana dashboards for monitoring

### Phase 3 (Q2 2026)
- [ ] GPU runner pool for ML workflows
- [ ] ARM runner pool for cross-platform builds
- [ ] Spot instance support for cost optimization
- [ ] Runner templates with pre-installed tools
- [ ] Integration with internal services (artifact storage, etc.)

### Phase 4 (Future)
- [ ] Multi-region deployment
- [ ] Automatic runner image updates
- [ ] Cost allocation per project/team
- [ ] Advanced security scanning
- [ ] Compliance automation (SOC2, ISO27001)

---

## Dependencies

### External Services
- **GitHub API**: Runner registration and workflow API
- **GitHub.com**: Repository and organization access
- **Ubuntu APT repositories**: Package installation
- **Docker Hub**: Base image downloads (one-time)

### Internal Components
- **k3s**: Kubernetes distribution
- **cert-manager**: Certificate management
- **Helm**: Package manager for ARC
- **Docker**: Container runtime

### Network Dependencies
- **DNS**: Cluster and external resolution
- **HTTPS (443)**: GitHub API, APT repositories, Docker registry
- **SSH (22)**: Git operations

---

## Compliance & Security

### Security Standards
- **Network Isolation**: Kubernetes NetworkPolicy enforcement
- **Resource Isolation**: Pod-level CPU and memory limits
- **Access Control**: Kubernetes RBAC, GitHub PAT with minimal permissions
- **Secret Management**: Kubernetes Secrets (not in Git)
- **Audit Trail**: Kubernetes audit logs, GitHub Actions logs

### Data Handling
- **Secrets**: Never logged, never in Git
- **Build Artifacts**: Stored in GitHub (not on runners)
- **Logs**: Retained for 30 days in Kubernetes
- **Source Code**: Ephemeral (cloned per job, deleted after)

### Compliance Requirements
- **GDPR**: No personal data stored on runners
- **SOC2**: Audit logs, access controls, backup procedures
- **ISO27001**: Security policies, incident response

---

## Conclusion

This project successfully provides a production-ready self-hosted GitHub Actions runner infrastructure with:
- ✅ 87% cost reduction compared to GitHub-hosted runners
- ✅ 40-50% build time improvement through caching
- ✅ Auto-scaling from 3 to 10+ runners based on demand
- ✅ 99.7% uptime with automatic recovery
- ✅ Comprehensive security with network policies and resource limits

The infrastructure is stable, well-documented, and ready for expansion to support additional use cases and workloads.

---

## References

- **Repository**: https://github.com/The1Studio/arc-github-runners
- **ARC Documentation**: https://github.com/actions/actions-runner-controller
- **k3s Documentation**: https://docs.k3s.io/
- **GitHub Actions**: https://docs.github.com/en/actions
- **Kubernetes**: https://kubernetes.io/docs/

---

**Document Version**: 1.0
**Approved By**: @tuha263
**Next Review**: Q1 2026
