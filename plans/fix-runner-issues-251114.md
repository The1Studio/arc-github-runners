# [Bug Fix] ARC Runner Issues Implementation Plan

**Date**: 2025-11-14
**Type**: Bug Fix
**Priority**: Critical (PDB), Medium (Labels), Low (Org name)
**Context**: Fix missing PodDisruptionBudgets, inconsistent label casing, and organization name inconsistencies in ARC runner configurations.

## Executive Summary

Three issues identified in ARC runner review:
1. **CRITICAL**: 3 runners missing PodDisruptionBudgets (android-apk-builder, containerized-unity-deploy, containerized-unity-sync)
2. **MEDIUM**: Unity runners use inconsistent label casing ("Linux", "X64" vs standard "linux", "x64")
3. **MINOR**: Organization name inconsistency ("the1studio" vs "The1Studio")

All runners currently operational; fixes will improve reliability, consistency, and workflow predictability.

## Issue Analysis

### Symptoms
- [x] Missing PDBs: 3/5 runners lack disruption protection
- [x] Label casing: Unity runners use "Linux"/"X64" (capitalized)
- [x] Org name: Mixed use of "the1studio" and "The1Studio"

### Root Cause

**PDB Issue**:
- Original deployment focused on main runners (the1studio-org, tuha263-personal)
- Android and Unity runners added later without PDB configuration
- PDB ensures min 1 runner available during voluntary disruptions (k8s upgrades, node drains)

**Label Casing**:
- Unity runners (containerized-unity-deploy, containerized-unity-sync) use capitalized labels
- Inconsistent with GitHub Actions standard practice (lowercase)
- Cause: Manual configuration without standard validation
- Impact: Workflows must use exact case match (`Linux` instead of `linux`)

**Organization Name**:
- GitHub organization: `The1Studio` (capital T and S)
- Current config: Mixed usage
  - `the1studio-org-runners`: organization: "the1studio"
  - `android-apk-builder`: organization: "the1studio"
  - `containerized-unity-deploy`: organization: "The1Studio"
- ARC accepts both (case-insensitive), but standardization improves clarity

### Evidence
- **Current PDB**: `k8s/pod-disruption-budget.yaml` only has 2 PDBs (lines 1-27)
- **Label casing**: `k8s/runner-deployments.yaml` lines 191-193, 245-250
- **Org names**: Verified via `kubectl get runnerdeployment -o json`

## Context Links
- **Repository**: https://github.com/The1Studio/arc-github-runners
- **Live Cluster**: k3s on tuha263's Arch Linux system
- **Related Docs**:
  - `docs/project-overview-pdr.md` - Current deployment overview
  - `docs/MANAGEMENT.md` - Operations guide
  - `k8s/runner-deployments.yaml` - Runner configurations

## Solution Design

### Approach

**PDB Fix**: Add 3 new PodDisruptionBudget resources with minAvailable: 1
- Ensures at least 1 runner survives voluntary disruptions
- Prevents total runner pool outage during k8s maintenance

**Label Fix**: Standardize to lowercase ("linux", "x64")
- Follow GitHub Actions conventions
- Update runner configs + documentation
- Note: Existing workflows using "Linux"/"X64" must be updated

**Org Name Fix**: Standardize to "The1Studio"
- Match GitHub organization casing
- Update 2 runner configs (the1studio-org-runners, android-apk-builder)

### Changes Required

1. **k8s/pod-disruption-budget.yaml**: Add 3 PDBs
2. **k8s/runner-deployments.yaml**: Fix labels (lines 191-193, 245-250), fix org names (lines 17, 129)
3. **README.md**: Update label examples (lines 43-44, 49-50, 233, 244)
4. **docs/project-overview-pdr.md**: Update label documentation

### Testing Changes
- [x] No test suite changes required
- [x] Validation via kubectl apply (dry-run) then live cluster
- [x] Verify runners re-register with correct labels

## Implementation Steps

### Phase 1: Add Missing PodDisruptionBudgets

1. [ ] Edit `k8s/pod-disruption-budget.yaml` - Add android-apk-builder PDB
   ```yaml
   ---
   # PodDisruptionBudget for Android APK builder
   apiVersion: policy/v1
   kind: PodDisruptionBudget
   metadata:
     name: android-apk-builder-pdb
     namespace: arc-runners
   spec:
     minAvailable: 1
     selector:
       matchLabels:
         runner-deployment-name: android-apk-builder
   ```

2. [ ] Edit `k8s/pod-disruption-budget.yaml` - Add containerized-unity-deploy PDB
   ```yaml
   ---
   # PodDisruptionBudget for Unity deployment runners
   apiVersion: policy/v1
   kind: PodDisruptionBudget
   metadata:
     name: containerized-unity-deploy-pdb
     namespace: arc-runners
   spec:
     minAvailable: 1
     selector:
       matchLabels:
         runner-deployment-name: containerized-unity-deploy
   ```

3. [ ] Edit `k8s/pod-disruption-budget.yaml` - Add containerized-unity-sync PDB
   ```yaml
   ---
   # PodDisruptionBudget for Unity Editor image sync runners
   apiVersion: policy/v1
   kind: PodDisruptionBudget
   metadata:
     name: containerized-unity-sync-pdb
     namespace: arc-runners
   spec:
     minAvailable: 1
     selector:
       matchLabels:
         runner-deployment-name: containerized-unity-sync
   ```

4. [ ] Validate PDB syntax
   ```bash
   export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
   kubectl apply -f k8s/pod-disruption-budget.yaml --dry-run=client
   ```

5. [ ] Apply PDB configuration
   ```bash
   kubectl apply -f k8s/pod-disruption-budget.yaml
   ```

6. [ ] Verify PDBs created
   ```bash
   kubectl get pdb -n arc-runners
   # Should show 5 PDBs (2 existing + 3 new)
   ```

### Phase 2: Fix Label Casing

7. [ ] Edit `k8s/runner-deployments.yaml` - Fix containerized-unity-deploy labels (lines 190-193)
   ```yaml
   # OLD:
   labels:
     - self-hosted
     - Linux
     - X64
     - deploy

   # NEW:
   labels:
     - self-hosted
     - linux
     - x64
     - deploy
   ```

8. [ ] Edit `k8s/runner-deployments.yaml` - Fix containerized-unity-sync labels (lines 244-250)
   ```yaml
   # OLD:
   labels:
     - self-hosted
     - Linux
     - X64
     - deploy
     - harbor-access
     - harbor-host

   # NEW:
   labels:
     - self-hosted
     - linux
     - x64
     - deploy
     - harbor-access
     - harbor-host
   ```

9. [ ] Validate runner deployment syntax
   ```bash
   kubectl apply -f k8s/runner-deployments.yaml --dry-run=client
   ```

10. [ ] Apply runner deployment changes
    ```bash
    kubectl apply -f k8s/runner-deployments.yaml
    ```

11. [ ] Wait for runners to re-register (auto-restart with new labels)
    ```bash
    # Watch pods restart
    kubectl get pods -n arc-runners -w

    # After ~30 seconds, verify new labels in GitHub
    gh api orgs/The1Studio/actions/runners --jq '.runners[] | select(.name | contains("containerized-unity")) | {name: .name, labels: [.labels[].name]}'
    ```

### Phase 3: Fix Organization Name Inconsistency

12. [ ] Edit `k8s/runner-deployments.yaml` - Fix the1studio-org-runners org name (line 17)
    ```yaml
    # OLD:
    organization: the1studio

    # NEW:
    organization: The1Studio
    ```

13. [ ] Edit `k8s/runner-deployments.yaml` - Fix android-apk-builder org name (line 129)
    ```yaml
    # OLD:
    organization: the1studio

    # NEW:
    organization: The1Studio
    ```

14. [ ] Apply runner deployment changes
    ```bash
    kubectl apply -f k8s/runner-deployments.yaml
    ```

15. [ ] Verify runners re-register with correct org
    ```bash
    # Wait ~30 seconds
    gh api orgs/The1Studio/actions/runners --jq '.runners[] | {name: .name, status: .status}'
    ```

### Phase 4: Update Documentation

16. [ ] Edit `README.md` - Fix Unity deployment example (line 233)
    ```yaml
    # OLD:
    runs-on: [self-hosted, Linux, X64, deploy]

    # NEW:
    runs-on: [self-hosted, linux, x64, deploy]
    ```

17. [ ] Edit `README.md` - Fix Unity sync example (line 244)
    ```yaml
    # OLD:
    runs-on: [self-hosted, Linux, X64, deploy, harbor-access, harbor-host]

    # NEW:
    runs-on: [self-hosted, linux, x64, deploy, harbor-access, harbor-host]
    ```

18. [ ] Edit `README.md` - Update containerized-unity-deploy description (lines 43-44)
    ```markdown
    # OLD:
    **Labels**: `self-hosted`, `Linux`, `X64`, `deploy`

    # NEW:
    **Labels**: `self-hosted`, `linux`, `x64`, `deploy`
    ```

19. [ ] Edit `README.md` - Update containerized-unity-sync description (lines 49-50)
    ```markdown
    # OLD:
    **Labels**: `self-hosted`, `Linux`, `X64`, `deploy`, `harbor-access`, `harbor-host`

    # NEW:
    **Labels**: `self-hosted`, `linux`, `x64`, `deploy`, `harbor-access`, `harbor-host`
    ```

20. [ ] Edit `README.md` - Update label matching rules section (lines 276-277)
    ```markdown
    # Update bad example:
    # ❌ Case mismatch - won't match Unity runners
    runs-on: [self-hosted, Linux, X64, deploy]  # Should use lowercase "linux" and "x64"
    ```

21. [ ] Edit `docs/project-overview-pdr.md` - Update runner descriptions
    - Update containerized-unity-deploy labels section
    - Update containerized-unity-sync labels section

### Phase 5: Final Verification

22. [ ] Verify all 5 PDBs exist
    ```bash
    kubectl get pdb -n arc-runners -o wide
    ```

23. [ ] Verify all 5 runners are online with correct labels
    ```bash
    gh api orgs/The1Studio/actions/runners --jq '.runners[] | {name: .name, status: .status, labels: [.labels[].name]}'
    ```

24. [ ] Test workflow with new lowercase labels
    ```bash
    # Trigger test workflow in test repo
    gh workflow run test-arc.yml -R The1Studio/arc-github-runners
    ```

25. [ ] Monitor runner pickup of test jobs
    ```bash
    kubectl logs -n arc-runners -l pool=containerized-unity-deploy --tail=50
    ```

## Verification Plan

### Test Cases
- [x] **PDB Test**: Verify PDBs prevent disruption
  ```bash
  # Attempt to drain node (should fail with PDB)
  kubectl drain $(kubectl get nodes -o name | head -1) --dry-run=server
  # Should show: "error: cannot delete Pods with local storage: ..." (expected)
  ```

- [x] **Label Test**: Verify lowercase labels work in workflows
  ```yaml
  # In test workflow:
  jobs:
    test-unity:
      runs-on: [self-hosted, linux, x64, deploy]
      steps:
        - run: echo "Testing new lowercase labels"
  ```

- [x] **Org Name Test**: Verify runners registered under The1Studio
  ```bash
  gh api orgs/The1Studio/actions/runners --jq '.runners[].name'
  ```

- [x] **Regression Test**: Verify existing workflows still work
  - the1studio-org-runners: Already use "linux", "x64" (no change)
  - tuha263-personal-runners: Already use "linux", "x64" (no change)
  - android-apk-builder: Already use "linux", "x64" (no change)

### Rollback Plan

**If PDB issues occur**:
1. Remove problematic PDB: `kubectl delete pdb <pdb-name> -n arc-runners`
2. Revert `k8s/pod-disruption-budget.yaml` to previous version
3. Reapply: `kubectl apply -f k8s/pod-disruption-budget.yaml`

**If label change breaks workflows**:
1. Revert `k8s/runner-deployments.yaml` to previous version (restore "Linux", "X64")
2. Reapply: `kubectl apply -f k8s/runner-deployments.yaml`
3. Wait for runners to re-register with old labels (~30 seconds)

**If org name change breaks registration**:
1. Revert `k8s/runner-deployments.yaml` org names to "the1studio"
2. Reapply: `kubectl apply -f k8s/runner-deployments.yaml`
3. Verify runners re-register: `gh api orgs/The1Studio/actions/runners`

**Full Rollback**:
```bash
git revert <commit-hash>
kubectl apply -f k8s/pod-disruption-budget.yaml
kubectl apply -f k8s/runner-deployments.yaml
```

## Risk Assessment

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| PDB blocks legitimate disruptions | Medium | Low | Test with dry-run, can delete PDB immediately if needed |
| Label change breaks active workflows | High | Medium | Update workflows in parallel, old labels still work until pods restart |
| Runners fail to re-register | High | Low | ARC auto-retries registration, can rollback config if persistent |
| Downtime during runner restart | Low | High | Runners restart gracefully, typically <30s, PDB ensures min 1 available |
| Organization case mismatch | Low | Very Low | GitHub API is case-insensitive for org names |

## Workflow Update Impact

**⚠️ IMPORTANT**: Existing workflows using Unity runners MUST be updated:

### Workflows to Update

**The1Studio/containerized-unity repository**:
- Any workflow using `runs-on: [self-hosted, Linux, X64, ...]`
- Update to: `runs-on: [self-hosted, linux, x64, ...]`

**Example Fix**:
```yaml
# OLD:
jobs:
  deploy:
    runs-on: [self-hosted, Linux, X64, deploy]

# NEW:
jobs:
  deploy:
    runs-on: [self-hosted, linux, x64, deploy]
```

**Migration Strategy**:
1. Update runner configs (this plan)
2. Immediately update workflows in The1Studio/containerized-unity
3. Test workflow execution with new labels
4. Document label change in repository

**Timeline**:
- Runner config update: ~5 minutes
- Runner restart: ~30 seconds
- Workflow updates: ~10 minutes
- Total downtime: ~1 minute per runner pool

## TODO Checklist

### Implementation
- [ ] Add 3 missing PodDisruptionBudgets (android, unity-deploy, unity-sync)
- [ ] Fix label casing in runner configs (Linux→linux, X64→x64)
- [ ] Fix organization name inconsistency (the1studio→The1Studio)
- [ ] Update README.md examples and documentation
- [ ] Update project-overview-pdr.md documentation

### Validation
- [ ] Verify 5 PDBs exist in cluster
- [ ] Verify all 5 runners online with correct labels
- [ ] Test workflow with new lowercase labels
- [ ] Verify no regression in existing workflows
- [ ] Update The1Studio/containerized-unity workflows

### Documentation
- [ ] Update README.md with correct label examples
- [ ] Update project-overview-pdr.md with correct labels
- [ ] Document label change in CHANGELOG (if exists)
- [ ] Notify team about workflow label updates

## Success Criteria

- [x] All 5 runners have PodDisruptionBudgets configured
- [x] All runners use lowercase labels ("linux", "x64")
- [x] All organization runners use "The1Studio" org name
- [x] Documentation matches actual configuration
- [x] No disruption to existing workflows (after updates)
- [x] All runners remain online and operational
- [x] Test workflow successfully runs with new labels

## Post-Implementation Review

**After completion, verify**:
1. All runners healthy: `kubectl get pods -n arc-runners`
2. All PDBs active: `kubectl get pdb -n arc-runners`
3. Runner labels match docs: Compare README.md with `gh api` output
4. No queued workflows: Check GitHub Actions for The1Studio org
5. Unity workflows working: Test deployment in containerized-unity repo

**Estimated Time**: 30 minutes total
- PDB changes: 5 minutes
- Label changes: 10 minutes
- Org name changes: 5 minutes
- Documentation: 10 minutes
- Validation: 10 minutes (includes runner restart time)

---

**Plan created**: 2025-11-14
**Target completion**: 2025-11-14
**Assigned to**: System operator
**Reviewed by**: N/A (self-review)
