# Troubleshooting ARC Runners

Common issues and their solutions.

## Quick Diagnostic Commands

```bash
# Set kubeconfig first
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# Check everything
kubectl get pods -n arc-runners
kubectl get pods -n arc-systems
kubectl get hra -n arc-runners
gh api orgs/the1studio/actions/runners
```

---

## Common Issues

### 1. Runners Not Appearing in GitHub

**Symptoms:**
- Pods are running in Kubernetes
- But runners don't show in GitHub Settings → Actions → Runners

**Diagnosis:**
```bash
# Check if pods are actually running
kubectl get pods -n arc-runners

# Check pod logs
kubectl logs -n arc-runners <pod-name> -c runner

# Check ARC controller logs
kubectl logs -n arc-systems -l app.kubernetes.io/name=actions-runner-controller
```

**Common Causes & Solutions:**

#### A. Invalid GitHub Token

```bash
# Verify token exists
kubectl get secret controller-manager -n arc-systems

# Check token value (will show encrypted)
kubectl get secret controller-manager -n arc-systems -o jsonpath='{.data.github_token}' | base64 -d

# Solution: Update token
GITHUB_TOKEN=$(gh auth token)
kubectl delete secret controller-manager -n arc-systems
kubectl create secret generic controller-manager \
  --namespace arc-systems \
  --from-literal=github_token="$GITHUB_TOKEN"

# Restart controller
kubectl delete pod -n arc-systems -l app.kubernetes.io/name=actions-runner-controller
```

#### B. Wrong Repository/Organization Name

```bash
# Check deployment configuration
kubectl get runnerdeployment -n arc-runners the1studio-org-runners -o yaml | grep organization

# Solution: Fix organization name
kubectl edit runnerdeployment the1studio-org-runners -n arc-runners
```

#### C. Network Connectivity Issues

```bash
# Test from runner pod
kubectl exec -it -n arc-runners <pod-name> -c runner -- curl -I https://api.github.com

# Should return HTTP/2 200

# If fails, check DNS
kubectl exec -it -n arc-runners <pod-name> -c runner -- nslookup api.github.com
```

---

### 2. Workflow Stuck "Waiting for Runner"

**Symptoms:**
- Workflow shows: "Waiting for a runner to pick up this job"
- Job never starts

**Diagnosis:**
```bash
# Check if runners are online
gh api orgs/the1studio/actions/runners --jq '.runners[] | {name, status, busy}'

# Check runner labels
gh api orgs/the1studio/actions/runners --jq '.runners[] | {name, labels: [.labels[].name]}'
```

**Common Causes:**

#### A. Label Mismatch

Your workflow uses:
```yaml
runs-on: [self-hosted, arc, the1studio]
```

But runners have labels:
```
[self-hosted, linux, x64, arc, the1studio, org]
```

**Solution:** Update workflow to use exact labels:
```yaml
runs-on: [self-hosted, arc, the1studio, org]
```

#### B. All Runners Busy

```bash
# Check runner status
gh api orgs/the1studio/actions/runners --jq '.runners[] | select(.busy==true)'

# Check auto-scaler
kubectl get hra -n arc-runners

# Wait for auto-scaler to spawn more (30-60 seconds)
# Or manually scale:
kubectl scale runnerdeployment the1studio-org-runners -n arc-runners --replicas=5
```

#### C. Runners Offline

```bash
# Check pod status
kubectl get pods -n arc-runners

# If pods are restarting:
kubectl describe pod -n arc-runners <pod-name>

# Check logs
kubectl logs -n arc-runners <pod-name> -c runner --tail=50
```

#### D. Public Repository Not Allowed

**⚠️ COMMON ISSUE:** Organization runners cannot be used by public repositories by default.

**Symptoms:**
- Runners are online
- Workflow stays in "Queued" state forever
- Repository is public
- Runners are registered at organization level

**Diagnosis:**
```bash
# Check if repo is public
gh repo view owner/repo --json visibility

# Check runner group settings
gh api orgs/the1studio/actions/runner-groups --jq '.runner_groups[] | {name, allows_public_repositories}'
```

**Solution:**
```bash
# Enable public repositories for Default runner group
gh api -X PATCH orgs/the1studio/actions/runner-groups/1 \
  -F allows_public_repositories=true

# Verify the change
gh api orgs/the1studio/actions/runner-groups/1 --jq '.allows_public_repositories'
# Should return: true
```

**Alternative:** Use repository-level runners instead of organization-level runners:
```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: myrepo-runners
  namespace: arc-runners
spec:
  replicas: 2
  template:
    spec:
      repository: owner/public-repo  # Specific repository
      labels: [self-hosted, arc, myrepo]
```

---

### 3. Pod Stuck in ContainerCreating

**Symptoms:**
```
NAME                          READY   STATUS              AGE
runner-xxx-yyy                0/2     ContainerCreating   5m
```

**Diagnosis:**
```bash
kubectl describe pod -n arc-runners <pod-name>
```

**Common Causes:**

#### A. Missing Secret

```
Error: secret "controller-manager" not found
```

**Solution:**
```bash
GITHUB_TOKEN=$(gh auth token)
kubectl create secret generic controller-manager \
  --namespace arc-systems \
  --from-literal=github_token="$GITHUB_TOKEN"
```

#### B. Image Pull Error

```
Failed to pull image "summerwind/actions-runner:latest"
```

**Solutions:**
```bash
# Check internet connectivity
curl -I https://ghcr.io

# Check if k3s can pull images
kubectl run test --image=alpine --rm -it -- /bin/sh

# If fails, restart k3s
sudo systemctl restart k3s
```

#### C. Volume Mount Error

```
MountVolume.SetUp failed
```

**Solution:**
```bash
# Check if volume exists
kubectl get pv
kubectl get pvc -n arc-runners

# Delete and recreate pod
kubectl delete pod -n arc-runners <pod-name>
```

---

### 4. Auto-Scaling Not Working

**Symptoms:**
- Runners stay at min replicas even when jobs are queued
- Or don't scale down when idle

**Diagnosis:**
```bash
# Check auto-scaler status
kubectl get hra -n arc-runners

# Check auto-scaler logs (in ARC controller)
kubectl logs -n arc-systems -l app.kubernetes.io/name=actions-runner-controller | grep -i scale
```

**Common Causes:**

#### A. Auto-Scaler Not Created

```bash
# Check if autoscaler exists
kubectl get hra -n arc-runners

# If empty, create it:
kubectl apply -f k8s/autoscalers.yaml
```

#### B. Thresholds Too High/Low

```bash
# View current settings
kubectl get hra the1studio-org-autoscaler -n arc-runners -o yaml

# Adjust thresholds
kubectl edit hra the1studio-org-autoscaler -n arc-runners
```

#### C. ARC Controller Not Running

```bash
# Check controller
kubectl get pods -n arc-systems

# If not running:
kubectl logs -n arc-systems -l app.kubernetes.io/name=actions-runner-controller

# Restart:
kubectl delete pod -n arc-systems -l app.kubernetes.io/name=actions-runner-controller
```

---

### 5. Runner Pod Crashing

**Symptoms:**
```
NAME                          READY   STATUS             RESTARTS   AGE
runner-xxx-yyy                1/2     CrashLoopBackOff   5          3m
```

**Diagnosis:**
```bash
# Check which container is crashing
kubectl get pod -n arc-runners <pod-name> -o json | jq '.status.containerStatuses'

# View logs of crashed container
kubectl logs -n arc-runners <pod-name> -c runner --previous

# Describe pod for events
kubectl describe pod -n arc-runners <pod-name>
```

**Common Causes:**

#### A. GitHub Registration Failed

```
Error: Failed to register runner with GitHub
```

**Solutions:**
- Check GitHub token permissions
- Verify organization/repo name
- Check if runner limit reached (GitHub has limits per organization)

#### B. Out of Memory

```
OOMKilled
```

**Solution:**
```bash
# Add resource limits
kubectl edit runnerdeployment -n arc-runners

# Add:
spec:
  template:
    spec:
      resources:
        limits:
          memory: "4Gi"
        requests:
          memory: "2Gi"
```

---

### 6. k3s Issues

**Symptoms:**
- kubectl commands fail
- "connection refused" errors

**Diagnosis:**
```bash
# Check k3s service
sudo systemctl status k3s

# Check k3s logs
sudo journalctl -u k3s -n 50
```

**Solutions:**

#### A. k3s Not Running

```bash
# Start k3s
sudo systemctl start k3s

# Enable auto-start
sudo systemctl enable k3s
```

#### B. Kubeconfig Permission Issues

```bash
# Check file exists
ls -la /etc/rancher/k3s/k3s.yaml

# Fix permissions
sudo chmod 644 /etc/rancher/k3s/k3s.yaml

# Set environment variable
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

#### C. k3s Corrupted

```bash
# Last resort: Restart k3s
sudo systemctl restart k3s

# If still broken, check disk space
df -h

# Check system resources
free -h
```

---

### 7. Cert-Manager Issues

**Symptoms:**
- ARC controller won't start
- Certificate errors in logs

**Diagnosis:**
```bash
# Check cert-manager pods
kubectl get pods -n cert-manager

# Check certificates
kubectl get certificates -A
kubectl get certificaterequests -A
```

**Solution:**
```bash
# Restart cert-manager
kubectl delete pod -n cert-manager --all

# If still broken, reinstall
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml

# Wait for ready
kubectl wait --for=condition=ready pod -l app=cert-manager -n cert-manager --timeout=120s
```

---

## Performance Issues

### Slow Runner Startup

**Problem:** Runners take too long to start (>2 minutes)

**Solutions:**
1. **Pre-pull images:**
   ```bash
   kubectl create daemonset runner-image-puller \
     --image=summerwind/actions-runner:latest \
     --namespace=arc-runners
   ```

2. **Use faster storage:**
   - k3s uses local storage by default
   - Consider using SSD if on HDD

3. **Reduce image size:**
   - Use custom runner image with only needed tools

### High Resource Usage

**Check resource consumption:**
```bash
kubectl top pods -n arc-runners
kubectl top nodes
```

**Solutions:**
1. **Set resource limits:**
   ```yaml
   resources:
     limits:
       cpu: "2"
       memory: "4Gi"
     requests:
       cpu: "1"
       memory: "2Gi"
   ```

2. **Scale down idle runners:**
   ```bash
   kubectl patch hra the1studio-org-autoscaler -n arc-runners \
     --type='json' -p='[{"op": "replace", "path": "/spec/minReplicas", "value":1}]'
   ```

---

## Debug Mode

### Enable Verbose Logging

```bash
# Edit runner deployment
kubectl edit runnerdeployment the1studio-org-runners -n arc-runners

# Add environment variable:
spec:
  template:
    spec:
      env:
      - name: RUNNER_DEBUG
        value: "1"
```

### Interactive Debugging

```bash
# Get a shell in runner pod
kubectl exec -it -n arc-runners <pod-name> -c runner -- /bin/bash

# Check runner status
cd /runner
./config.sh --check

# View runner config
cat .runner
cat .credentials
```

---

## Getting Help

### Collect Debug Information

```bash
# Create debug bundle
mkdir arc-debug
kubectl get pods -n arc-runners > arc-debug/pods.txt
kubectl get hra -n arc-runners > arc-debug/autoscalers.txt
kubectl logs -n arc-systems -l app.kubernetes.io/name=actions-runner-controller > arc-debug/controller.log
kubectl describe pod -n arc-runners > arc-debug/pod-describe.txt

# Compress
tar -czf arc-debug.tar.gz arc-debug/
```

### Useful Resources

- **ARC GitHub:** https://github.com/actions/actions-runner-controller
- **k3s Docs:** https://docs.k3s.io/
- **GitHub Actions:** https://docs.github.com/en/actions

---

**Last Updated:** 2025-10-15
