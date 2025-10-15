# Using ARC Runners in GitHub Workflows

This guide explains how to use the self-hosted ARC runners in your GitHub Actions workflows.

## Available Runner Pools

### 1. the1studio Organization Runners

**Labels:** `self-hosted`, `linux`, `x64`, `arc`, `the1studio`, `org`

**Capacity:**
- Minimum: 2 runners
- Maximum: 10 runners (auto-scales)

**Use for:** Any repository in the `the1studio` organization

### 2. Personal Repository Runners

**Labels:** `self-hosted`, `linux`, `x64`, `arc`, `personal`

**Capacity:**
- Minimum: 1 runner
- Maximum: 5 runners (auto-scales)

**Use for:** `tuha263/personal-site` repository

---

## Basic Workflow Examples

### Example 1: Simple Build (the1studio)

```yaml
name: Build and Test

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: [self-hosted, arc, the1studio, org]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Node.js
        run: |
          node --version
          npm --version

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

      - name: Build
        run: npm run build
```

### Example 2: Docker Build

```yaml
name: Docker Build

on:
  push:
    branches: [ main ]

jobs:
  docker:
    runs-on: [self-hosted, arc, the1studio, org]

    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: |
          docker build -t myapp:${{ github.sha }} .

      - name: Run container tests
        run: |
          docker run --rm myapp:${{ github.sha }} npm test

      - name: Push to registry
        run: |
          echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
          docker tag myapp:${{ github.sha }} myapp:latest
          docker push myapp:latest
```

### Example 3: Parallel Jobs

```yaml
name: Parallel Testing

on: [push]

jobs:
  unit-tests:
    runs-on: [self-hosted, arc, the1studio, org]
    steps:
      - uses: actions/checkout@v4
      - run: npm run test:unit

  integration-tests:
    runs-on: [self-hosted, arc, the1studio, org]
    steps:
      - uses: actions/checkout@v4
      - run: npm run test:integration

  e2e-tests:
    runs-on: [self-hosted, arc, the1studio, org]
    steps:
      - uses: actions/checkout@v4
      - run: npm run test:e2e
```

**Note:** With auto-scaling, if all 3 jobs start simultaneously and only 2 runners are available, the autoscaler will spawn a 3rd runner automatically.

---

## Matrix Builds

```yaml
name: Matrix Build

on: [push]

jobs:
  test:
    runs-on: [self-hosted, arc, the1studio, org]
    strategy:
      matrix:
        node-version: [16, 18, 20]

    steps:
      - uses: actions/checkout@v4

      - name: Use Node.js ${{ matrix.node-version }}
        run: |
          nvm install ${{ matrix.node-version }}
          nvm use ${{ matrix.node-version }}

      - run: npm ci
      - run: npm test
```

---

## Using Different Runner Pools

### Personal Repository Workflow

```yaml
name: Personal Site Deploy

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: [self-hosted, arc, personal]

    steps:
      - uses: actions/checkout@v4
      - run: npm run build
      - run: npm run deploy
```

---

## Best Practices

### 1. Always Use Specific Labels

❌ **Bad:**
```yaml
runs-on: self-hosted  # Too generic
```

✅ **Good:**
```yaml
runs-on: [self-hosted, arc, the1studio, org]  # Specific pool
```

### 2. Clean Up After Yourself

```yaml
steps:
  - uses: actions/checkout@v4

  - name: Build
    run: docker build -t myapp .

  - name: Test
    run: docker run --rm myapp npm test

  - name: Cleanup
    if: always()
    run: |
      docker system prune -f
      rm -rf node_modules
```

### 3. Use Caching Wisely

Runners are ephemeral by default, but you can use shared volumes:

```yaml
steps:
  - uses: actions/checkout@v4

  - name: Cache node modules
    uses: actions/cache@v3
    with:
      path: ~/.npm
      key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
```

### 4. Handle Secrets Securely

```yaml
env:
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
  API_KEY: ${{ secrets.API_KEY }}

steps:
  - name: Deploy
    run: |
      # Secrets are available as env vars
      ./deploy.sh
```

---

## Monitoring Runner Usage

### Check Active Jobs

```bash
# Via kubectl
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get pods -n arc-runners

# Via GitHub API
gh api orgs/the1studio/actions/runners --jq '.runners[] | {name, status, busy}'
```

### View Runner Logs

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl logs -n arc-runners <pod-name> -c runner -f
```

---

## Troubleshooting

### Job Stuck in Queue

**Symptom:** Workflow shows "Waiting for a runner to pick up this job"

**Causes:**
1. All runners are busy → Auto-scaler will spawn more (wait ~30s)
2. Label mismatch → Check your `runs-on` labels
3. Runners offline → Check: `kubectl get pods -n arc-runners`

### Runner Capacity

**Check current capacity:**
```bash
kubectl get hra -n arc-runners
```

**Manually scale if needed:**
```bash
kubectl scale runnerdeployment the1studio-org-runners -n arc-runners --replicas=5
```

---

## Advanced Features

### Conditional Runner Selection

```yaml
jobs:
  build:
    runs-on: ${{ github.event_name == 'pull_request' && '[self-hosted, arc, the1studio, org]' || 'ubuntu-latest' }}
```

### Workflow-Specific Runners

You can create dedicated runner deployments for specific workflows by adding custom labels.

---

## FAQ

**Q: How long does it take for a runner to start?**
A: ~15-30 seconds when auto-scaling triggers

**Q: Can I use GitHub-hosted runners and self-hosted runners in the same workflow?**
A: Yes! Just specify different `runs-on` for different jobs

**Q: What happens if all runners are at max capacity?**
A: Jobs will queue until a runner becomes available or scales down

**Q: Can I access Docker inside the runner?**
A: Yes! Runners have Docker socket mounted (`/var/run/docker.sock`)

---

**Last Updated:** 2025-10-15
