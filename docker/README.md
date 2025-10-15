# Custom GitHub Actions Runner Image

This directory contains a custom Docker image for GitHub Actions runners with pre-configured HTTPS APT sources.

## Problem

The default `summerwind/actions-runner` image uses HTTP (port 80) for APT package sources. In Kubernetes environments with strict network policies (like k3s with firewall rules), port 80 is often blocked while port 443 (HTTPS) is allowed.

This causes workflows to fail when trying to install packages:
```
Cannot initiate the connection to archive.ubuntu.com:80
Could not connect to security.ubuntu.com:80
E: The repository 'http://archive.ubuntu.com/ubuntu focal-updates Release' does not have a Release file
```

## Solution

This custom image pre-configures all APT sources to use HTTPS instead of HTTP:
- `https://archive.ubuntu.com` (instead of http://)
- `https://security.ubuntu.com` (instead of http://)
- Disables PPAs that don't support HTTPS

## Building the Image

```bash
cd /mnt/Work/1M/arc-github-runners/docker
docker build -t the1studio/actions-runner:https-apt .
```

## Importing to k3s

```bash
docker save the1studio/actions-runner:https-apt | sudo k3s ctr images import -
```

## Usage in Runner Deployments

The image is already configured in `k8s/runner-deployments.yaml`:

```yaml
spec:
  template:
    spec:
      image: the1studio/actions-runner:https-apt
      imagePullPolicy: IfNotPresent
```

## Benefits

✅ **No workflow changes needed** - Workflows can use `apt-get` without modifications
✅ **Works in restricted networks** - Uses HTTPS (port 443) instead of HTTP (port 80)
✅ **Pre-cached APT lists** - Faster startup times
✅ **Production ready** - Tested and verified

## Testing

Verify the fix works:

```bash
# Get a running pod
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
POD=$(kubectl get pods -n arc-runners -l runner-deployment-name=the1studio-org-runners -o name | head -1)

# Check APT sources use HTTPS
kubectl exec -n arc-runners $POD -c runner -- cat /etc/apt/sources.list

# Test apt-get update
kubectl exec -n arc-runners $POD -c runner -- sudo apt-get update
```

Expected: All sources use `https://` and apt-get update succeeds without port 80 errors.

## Maintenance

### Updating the Base Image

When `summerwind/actions-runner` releases a new version:

```bash
# Pull latest base image
docker pull summerwind/actions-runner:latest

# Rebuild custom image
docker build -t the1studio/actions-runner:https-apt .

# Import to k3s
docker save the1studio/actions-runner:https-apt | sudo k3s ctr images import -

# Restart runners
kubectl rollout restart runnerdeployment -n arc-runners
```

## Files

- `Dockerfile` - Image definition with HTTPS APT configuration
- `README.md` - This file

## Related Documentation

- [Main README](../README.md) - ARC setup overview
- [TROUBLESHOOTING](../docs/TROUBLESHOOTING.md) - Port 80 HTTP issue details
- [GitHub Issue](https://github.com/actions/actions-runner-controller/issues/2056) - Upstream issue tracking

---

**Created:** 2025-10-15
**Maintained By:** tuha263
