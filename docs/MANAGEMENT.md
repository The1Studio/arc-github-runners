# Managing ARC Runners

Complete guide for managing your GitHub Actions Runner Controller deployment.

## Prerequisites

```bash
# Set kubeconfig (required for all kubectl commands)
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# Or add to your ~/.bashrc or ~/.zshrc:
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
```

---

## Daily Operations

### Check Runner Status

```bash
# View all runner pods
kubectl get pods -n arc-runners

# Watch pods in real-time
kubectl get pods -n arc-runners -w

# Check runner deployments
kubectl get runnerdeployment -n arc-runners

# Check auto-scalers
kubectl get hra -n arc-runners
```

**Expected output:**
```
NAME                                   READY   STATUS    AGE
the1studio-org-runners-xxx-yyy         2/2     Running   1h
the1studio-org-runners-xxx-zzz         2/2     Running   1h
tuha263-personal-runners-xxx-aaa       2/2     Running   1h
```

### View Runner Logs

```bash
# Get pod names
kubectl get pods -n arc-runners

# View logs for specific runner
kubectl logs -n arc-runners the1studio-org-runners-xxx-yyy -c runner

# Follow logs in real-time
kubectl logs -n arc-runners the1studio-org-runners-xxx-yyy -c runner -f

# View init container logs (if pod is starting)
kubectl logs -n arc-runners the1studio-org-runners-xxx-yyy -c runner-init
```

### Check GitHub Registration

```bash
# Check organization runners
gh api orgs/the1studio/actions/runners

# Pretty print with jq
gh api orgs/the1studio/actions/runners --jq '.runners[] | {name, status, busy, labels: [.labels[].name]}'

# Check personal repo runners
gh api repos/tuha263/personal-site/actions/runners --jq '.runners[] | {name, status}'
```

---

## Scaling Operations

### Manual Scaling

```bash
# Scale organization runners to 5
kubectl scale runnerdeployment the1studio-org-runners \
  -n arc-runners --replicas=5

# Scale personal runners to 3
kubectl scale runnerdeployment tuha263-personal-runners \
  -n arc-runners --replicas=3

# Verify scaling
kubectl get runnerdeployment -n arc-runners
```

### Auto-Scaler Management

```bash
# View current auto-scaler settings
kubectl get hra -n arc-runners -o yaml

# Edit auto-scaler
kubectl edit hra the1studio-org-autoscaler -n arc-runners

# Temporarily disable auto-scaling (set min = max)
kubectl patch hra the1studio-org-autoscaler -n arc-runners \
  --type='json' -p='[{"op": "replace", "path": "/spec/minReplicas", "value":5},{"op": "replace", "path": "/spec/maxReplicas", "value":5}]'

# Re-enable auto-scaling
kubectl patch hra the1studio-org-autoscaler -n arc-runners \
  --type='json' -p='[{"op": "replace", "path": "/spec/minReplicas", "value":2},{"op": "replace", "path": "/spec/maxReplicas", "value":10}]'
```

---

## Maintenance Tasks

### Restart Runners

```bash
# Restart all runners in a deployment (rolling restart)
kubectl rollout restart deployment -n arc-runners

# Delete specific runner pod (will auto-recreate)
kubectl delete pod the1studio-org-runners-xxx-yyy -n arc-runners

# Restart ARC controller
kubectl delete pod -n arc-systems -l app.kubernetes.io/name=actions-runner-controller
```

### Update GitHub Token

```bash
# Get new token
GITHUB_TOKEN=$(gh auth token)

# Update secret
kubectl delete secret controller-manager -n arc-systems
kubectl create secret generic controller-manager \
  --namespace arc-systems \
  --from-literal=github_token="$GITHUB_TOKEN"

# Restart controller to pick up new token
kubectl delete pod -n arc-systems -l app.kubernetes.io/name=actions-runner-controller
```

### Clean Up Stale Runners

```bash
# List all GitHub runners (including offline)
gh api orgs/the1studio/actions/runners --jq '.runners[] | select(.status=="offline") | {id, name}'

# Remove offline runner from GitHub (get ID from above)
gh api -X DELETE orgs/the1studio/actions/runners/<runner-id>
```

---

## Adding New Runner Deployments

### For a New Repository

1. Create deployment manifest:

```bash
cat <<EOF > /tmp/new-repo-runner.yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: myrepo-runners
  namespace: arc-runners
spec:
  replicas: 2
  template:
    spec:
      repository: owner/repo-name
      labels:
        - self-hosted
        - linux
        - x64
        - arc
        - myrepo
      env: []
EOF
```

2. Apply:

```bash
kubectl apply -f /tmp/new-repo-runner.yaml
```

3. Optional - Add auto-scaler:

```bash
cat <<EOF > /tmp/new-repo-autoscaler.yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: HorizontalRunnerAutoscaler
metadata:
  name: myrepo-autoscaler
  namespace: arc-runners
spec:
  scaleTargetRef:
    name: myrepo-runners
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: PercentageRunnersBusy
    scaleUpThreshold: '0.75'
    scaleDownThreshold: '0.25'
    scaleUpFactor: '1.5'
    scaleDownFactor: '0.5'
EOF

kubectl apply -f /tmp/new-repo-autoscaler.yaml
```

### For Another Organization

```bash
cat <<EOF > /tmp/org-runner.yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: otherorg-runners
  namespace: arc-runners
spec:
  replicas: 3
  template:
    spec:
      organization: other-org-name
      labels:
        - self-hosted
        - linux
        - x64
        - arc
        - otherorg
EOF

kubectl apply -f /tmp/org-runner.yaml
```

---

## Monitoring & Debugging

### Resource Usage

```bash
# Check pod resource consumption
kubectl top pods -n arc-runners

# Check node resources
kubectl top nodes

# Describe pod for detailed info
kubectl describe pod -n arc-runners <pod-name>
```

### Events

```bash
# View recent events
kubectl get events -n arc-runners --sort-by='.lastTimestamp'

# Watch events
kubectl get events -n arc-runners --watch
```

### ARC Controller Logs

```bash
# View controller logs
kubectl logs -n arc-systems -l app.kubernetes.io/name=actions-runner-controller

# Follow controller logs
kubectl logs -n arc-systems -l app.kubernetes.io/name=actions-runner-controller -f
```

---

## Backup & Recovery

### Backup Configuration

```bash
# Export all runner deployments
kubectl get runnerdeployment -n arc-runners -o yaml > runner-deployments-backup.yaml

# Export autoscalers
kubectl get hra -n arc-runners -o yaml > autoscalers-backup.yaml

# Backup to Git (recommended)
cd /mnt/Work/1M/arc-github-runners
git add k8s/
git commit -m "Update runner configurations"
git push
```

### Restore from Backup

```bash
# Apply saved configurations
kubectl apply -f runner-deployments-backup.yaml
kubectl apply -f autoscalers-backup.yaml
```

---

## System Services

### k3s Management

```bash
# Check k3s status
sudo systemctl status k3s

# Start/stop k3s
sudo systemctl start k3s
sudo systemctl stop k3s

# Restart k3s (restarts all runners)
sudo systemctl restart k3s

# Enable/disable auto-start
sudo systemctl enable k3s
sudo systemctl disable k3s

# View k3s logs
sudo journalctl -u k3s -f

# Check k3s version
kubectl version
```

### Resource Limits

Configure resource limits for runners:

```yaml
apiVersion: actions.summerwind.dev/v1alpha1
kind: RunnerDeployment
metadata:
  name: resource-limited-runners
  namespace: arc-runners
spec:
  replicas: 2
  template:
    spec:
      repository: owner/repo
      labels: [self-hosted, limited]
      resources:
        limits:
          cpu: "2"
          memory: "4Gi"
        requests:
          cpu: "1"
          memory: "2Gi"
```

---

## Troubleshooting Commands

### Pod Issues

```bash
# Pod won't start
kubectl describe pod -n arc-runners <pod-name>
kubectl logs -n arc-runners <pod-name> --all-containers

# Pod crashing
kubectl logs -n arc-runners <pod-name> -c runner --previous

# Force delete stuck pod
kubectl delete pod <pod-name> -n arc-runners --force --grace-period=0
```

### Network Issues

```bash
# Test connectivity from runner
kubectl exec -it -n arc-runners <pod-name> -c runner -- /bin/bash
curl https://api.github.com

# Check DNS
kubectl exec -it -n arc-runners <pod-name> -c runner -- nslookup github.com
```

---

## Performance Tuning

### Increase Runner Count

For high-traffic repositories:

```bash
# Update auto-scaler max
kubectl patch hra the1studio-org-autoscaler -n arc-runners \
  --type='json' -p='[{"op": "replace", "path": "/spec/maxReplicas", "value":20}]'
```

### Adjust Scale Thresholds

```bash
# More aggressive scaling (scale up at 50% instead of 75%)
kubectl patch hra the1studio-org-autoscaler -n arc-runners \
  --type='json' -p='[{"op": "replace", "path": "/spec/metrics/0/scaleUpThreshold", "value":"0.50"}]'
```

---

## Uninstallation

### Remove Runners Only

```bash
# Delete runner deployments
kubectl delete runnerdeployment --all -n arc-runners

# Delete autoscalers
kubectl delete hra --all -n arc-runners
```

### Complete Uninstall

```bash
# Uninstall ARC
helm uninstall arc -n arc-systems

# Uninstall cert-manager
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml

# Uninstall k3s
/usr/local/bin/k3s-uninstall.sh
```

---

**Last Updated:** 2025-10-15
