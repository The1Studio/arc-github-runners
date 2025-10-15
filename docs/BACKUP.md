# Backup and Disaster Recovery

**Last Updated**: 2025-10-15

## Overview

This document outlines backup procedures and disaster recovery steps for the ARC (Actions Runner Controller) deployment.

---

## What to Backup

### 1. Configuration Files (Critical)

**Location**: `/mnt/Work/1M/arc-github-runners/`

**Files**:
- `k8s/runner-deployments.yaml` - Runner configurations
- `k8s/autoscalers.yaml` - Auto-scaling rules
- `k8s/network-policy.yaml` - Network security policies
- `k8s/pod-disruption-budget.yaml` - Availability guarantees
- `docker/Dockerfile` - Custom runner image definition
- `README.md` - Main documentation
- `docs/` - All documentation files

**Backup Method**:
```bash
# Git repository (already backed up to GitHub)
cd /mnt/Work/1M/arc-github-runners
git add .
git commit -m "Backup: $(date +%Y-%m-%d)"
git push origin master
```

**Frequency**: After every configuration change

### 2. Kubernetes State (Important)

**What to Backup**:
- Runner deployments
- Auto-scalers
- Network policies
- Secrets (encrypted)

**Backup Method**:
```bash
# Export all ARC resources
mkdir -p ~/arc-backup/$(date +%Y-%m-%d)
cd ~/arc-backup/$(date +%Y-%m-%d)

export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# Export runner deployments
kubectl get runnerdeployment -n arc-runners -o yaml > runner-deployments-state.yaml

# Export autoscalers
kubectl get hra -n arc-runners -o yaml > autoscalers-state.yaml

# Export network policies
kubectl get networkpolicy -n arc-runners -o yaml > network-policies.yaml

# Export pod disruption budgets
kubectl get poddisruptionbudget -n arc-runners -o yaml > pdb.yaml

# ⚠️ IMPORTANT: Do NOT backup secrets in plain text
# Instead, document how to recreate them
cat > SECRETS.md <<'EOF'
# Secrets Restoration

GitHub PAT is stored in:
- Secret name: controller-manager
- Namespace: arc-systems
- Key: github_token

To restore:
```bash
GITHUB_TOKEN=$(gh auth token)
kubectl create secret generic controller-manager \
  --namespace arc-systems \
  --from-literal=github_token="$GITHUB_TOKEN"
```
EOF
```

**Frequency**: Weekly or after major changes

### 3. Custom Docker Image (Important)

**Image**: `the1studio/actions-runner:https-apt`

**Backup Method**:
```bash
# Save image to tar file
docker save the1studio/actions-runner:https-apt -o ~/arc-backup/runner-image-$(date +%Y-%m-%d).tar

# Or push to registry (already done)
docker push the1studio/actions-runner:https-apt
docker push the1studio/actions-runner:v1.1-https-apt
```

**Frequency**: After image updates

**Note**: Dockerfile is in Git, so can rebuild from source

### 4. k3s Cluster State (Optional)

**What**: Full cluster backup for complete disaster recovery

**Backup Method**:
```bash
# k3s automatic snapshot (etcd)
sudo systemctl status k3s
# Snapshots stored in: /var/lib/rancher/k3s/server/db/snapshots/

# Manual snapshot
sudo k3s etcd-snapshot save --name arc-backup-$(date +%Y-%m-%d)

# List snapshots
sudo k3s etcd-snapshot list
```

**Frequency**: Daily (automatic)

---

## Disaster Recovery Scenarios

### Scenario 1: Configuration File Loss

**Problem**: Lost k8s YAML files

**Recovery**:
```bash
# 1. Clone from GitHub
cd /mnt/Work/1M
git clone https://github.com/The1Studio/arc-github-runners.git

# 2. Verify files
cd arc-github-runners
ls -la k8s/

# 3. Re-apply if needed
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl apply -f k8s/runner-deployments.yaml
kubectl apply -f k8s/autoscalers.yaml
kubectl apply -f k8s/network-policy.yaml
kubectl apply -f k8s/pod-disruption-budget.yaml
```

**Time to Recover**: 5-10 minutes

### Scenario 2: All Runners Deleted

**Problem**: All runner pods and deployments deleted

**Recovery**:
```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# 1. Verify namespaces exist
kubectl get namespace arc-runners arc-systems

# 2. Re-create runners
kubectl apply -f /mnt/Work/1M/arc-github-runners/k8s/runner-deployments.yaml

# 3. Re-create autoscalers
kubectl apply -f /mnt/Work/1M/arc-github-runners/k8s/autoscalers.yaml

# 4. Verify runners come online
kubectl get pods -n arc-runners
gh api orgs/the1studio/actions/runners --jq '.runners[] | {name, status}'
```

**Time to Recover**: 2-5 minutes

### Scenario 3: Lost GitHub Token Secret

**Problem**: `controller-manager` secret deleted or corrupted

**Symptoms**:
- Runners fail to register with GitHub
- ARC controller shows authentication errors

**Recovery**:
```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# 1. Get new token from gh CLI
GITHUB_TOKEN=$(gh auth token)

# 2. Delete old secret (if exists)
kubectl delete secret controller-manager -n arc-systems 2>/dev/null || true

# 3. Create new secret
kubectl create secret generic controller-manager \
  --namespace arc-systems \
  --from-literal=github_token="$GITHUB_TOKEN"

# 4. Restart ARC controller
kubectl delete pod -n arc-systems -l app.kubernetes.io/name=actions-runner-controller

# 5. Verify runners register
sleep 30
kubectl get pods -n arc-runners
gh api orgs/the1studio/actions/runners
```

**Time to Recover**: 5 minutes

### Scenario 4: Custom Docker Image Lost

**Problem**: `the1studio/actions-runner:https-apt` image deleted from k3s

**Recovery Option 1**: Rebuild from Dockerfile
```bash
cd /mnt/Work/1M/arc-github-runners/docker

# Build image
docker build -t the1studio/actions-runner:https-apt \
             -t the1studio/actions-runner:v1.1-https-apt .

# Import to k3s
docker save the1studio/actions-runner:https-apt | sudo k3s ctr images import -

# Verify
sudo k3s ctr images list | grep the1studio/actions-runner
```

**Recovery Option 2**: Restore from backup tar
```bash
# If you have a saved tar file
docker load -i ~/arc-backup/runner-image-2025-10-15.tar
docker save the1studio/actions-runner:https-apt | sudo k3s ctr images import -
```

**Time to Recover**: 10-15 minutes (rebuild) or 2-3 minutes (restore from tar)

### Scenario 5: k3s Cluster Failure

**Problem**: k3s completely broken or corrupted

**Recovery**:
```bash
# 1. Check k3s status
sudo systemctl status k3s

# 2. Try restarting k3s first
sudo systemctl restart k3s
sleep 30
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get nodes

# 3. If still broken, restore from snapshot
sudo k3s etcd-snapshot list
sudo k3s etcd-snapshot restore arc-backup-2025-10-15
sudo systemctl restart k3s

# 4. If snapshots don't work, reinstall k3s (last resort)
# WARNING: This will destroy all pods and data
sudo /usr/local/bin/k3s-uninstall.sh
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644

# 5. Reinstall cert-manager
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
kubectl wait --for=condition=ready pod -l app=cert-manager -n cert-manager --timeout=120s

# 6. Reinstall ARC
kubectl create namespace arc-systems
kubectl create namespace arc-runners

GITHUB_TOKEN=$(gh auth token)
kubectl create secret generic controller-manager \
  --namespace arc-systems \
  --from-literal=github_token="$GITHUB_TOKEN"

helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller
helm repo update

helm install arc \
  --namespace arc-systems \
  actions-runner-controller/actions-runner-controller

# 7. Restore runners
cd /mnt/Work/1M/arc-github-runners
kubectl apply -f k8s/runner-deployments.yaml
kubectl apply -f k8s/autoscalers.yaml
kubectl apply -f k8s/network-policy.yaml
kubectl apply -f k8s/pod-disruption-budget.yaml

# 8. Import custom image
docker save the1studio/actions-runner:https-apt | sudo k3s ctr images import -

# 9. Verify everything
kubectl get pods -n arc-systems
kubectl get pods -n arc-runners
gh api orgs/the1studio/actions/runners
```

**Time to Recover**: 30-60 minutes

### Scenario 6: Complete System Failure (New Machine)

**Problem**: Need to set up ARC on a completely new machine

**Recovery**:

Follow the installation guide in README.md:
```bash
# 1. Install k3s
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644

# 2. Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 3. Install cert-manager
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.4/cert-manager.yaml
kubectl wait --for=condition=ready pod -l app=cert-manager -n cert-manager --timeout=120s

# 4. Clone configuration repository
cd /mnt/Work/1M
git clone https://github.com/The1Studio/arc-github-runners.git
cd arc-github-runners

# 5. Build custom image
cd docker
docker build -t the1studio/actions-runner:https-apt .
docker save the1studio/actions-runner:https-apt | sudo k3s ctr images import -
cd ..

# 6. Install ARC controller
kubectl create namespace arc-systems
kubectl create namespace arc-runners

GITHUB_TOKEN=$(gh auth token)
kubectl create secret generic controller-manager \
  --namespace arc-systems \
  --from-literal=github_token="$GITHUB_TOKEN"

helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller
helm repo update

helm install arc \
  --namespace arc-systems \
  actions-runner-controller/actions-runner-controller

# 7. Deploy runners
kubectl apply -f k8s/runner-deployments.yaml
kubectl apply -f k8s/autoscalers.yaml
kubectl apply -f k8s/network-policy.yaml
kubectl apply -f k8s/pod-disruption-budget.yaml

# 8. Verify
kubectl get pods -n arc-systems
kubectl get pods -n arc-runners
gh api orgs/the1studio/actions/runners
```

**Time to Recover**: 1-2 hours

---

## Backup Automation

### Automated Daily Backup Script

Create: `/home/tuha/bin/arc-backup.sh`

```bash
#!/bin/bash
set -e

BACKUP_DIR=~/arc-backup/$(date +%Y-%m-%d)
mkdir -p "$BACKUP_DIR"
cd "$BACKUP_DIR"

export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

echo "Backing up ARC configuration..."

# Export Kubernetes resources
kubectl get runnerdeployment -n arc-runners -o yaml > runner-deployments.yaml
kubectl get hra -n arc-runners -o yaml > autoscalers.yaml
kubectl get networkpolicy -n arc-runners -o yaml > network-policies.yaml
kubectl get poddisruptionbudget -n arc-runners -o yaml > pdb.yaml

# Git backup
cd /mnt/Work/1M/arc-github-runners
git add .
git diff-index --quiet HEAD || git commit -m "Auto backup: $(date +%Y-%m-%d)"
git push origin master 2>/dev/null || echo "Git push failed (expected if no changes)"

# k3s snapshot
sudo k3s etcd-snapshot save --name arc-auto-$(date +%Y-%m-%d) 2>/dev/null || echo "k3s snapshot failed"

# Cleanup old backups (keep last 30 days)
find ~/arc-backup/ -type d -mtime +30 -exec rm -rf {} + 2>/dev/null || true
sudo find /var/lib/rancher/k3s/server/db/snapshots/ -mtime +30 -delete 2>/dev/null || true

echo "Backup completed: $BACKUP_DIR"
```

Make executable and add to cron:
```bash
chmod +x /home/tuha/bin/arc-backup.sh

# Add to crontab (runs daily at 2 AM)
crontab -e
# Add line:
0 2 * * * /home/tuha/bin/arc-backup.sh >> /home/tuha/logs/arc-backup.log 2>&1
```

---

## Verification Checklist

After any recovery, verify:

- [ ] k3s is running: `sudo systemctl status k3s`
- [ ] ARC controller is running: `kubectl get pods -n arc-systems`
- [ ] Runner pods are running: `kubectl get pods -n arc-runners`
- [ ] Runners registered in GitHub: `gh api orgs/the1studio/actions/runners`
- [ ] Autoscalers are active: `kubectl get hra -n arc-runners`
- [ ] Network policies applied: `kubectl get networkpolicy -n arc-runners`
- [ ] Custom image present: `sudo k3s ctr images list | grep the1studio/actions-runner`
- [ ] Test workflow runs successfully

---

## Monitoring and Alerts

### Health Checks

**Daily checks**:
```bash
#!/bin/bash
# Check ARC health

export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

echo "=== k3s Status ==="
sudo systemctl is-active k3s || echo "ERROR: k3s not running"

echo "=== ARC Controller ==="
kubectl get pods -n arc-systems | grep -i running || echo "ERROR: Controller not running"

echo "=== Runners ==="
RUNNERS=$(kubectl get pods -n arc-runners | grep -c Running)
echo "Running runners: $RUNNERS"
if [ "$RUNNERS" -lt 1 ]; then
    echo "ERROR: No runners online"
fi

echo "=== GitHub Registration ==="
gh api orgs/the1studio/actions/runners --jq '.runners[] | {name, status}' || echo "ERROR: Cannot fetch runners from GitHub"
```

### Set up notifications (optional)

Configure alerts for:
- k3s service down
- No runners online for >5 minutes
- ARC controller crashes
- Runner pod crashes

---

## Emergency Contacts

- **System Administrator**: [@tuha263](https://github.com/tuha263)
- **GitHub Organization**: [The1Studio](https://github.com/The1Studio)
- **Documentation**: [arc-github-runners repository](https://github.com/The1Studio/arc-github-runners)

---

## Related Documentation

- [README.md](../README.md) - Installation guide
- [MANAGEMENT.md](./MANAGEMENT.md) - Management commands
- [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) - Common issues

---

**Created**: 2025-10-15
**Last Updated**: 2025-10-15
**Maintained By**: The1Studio
