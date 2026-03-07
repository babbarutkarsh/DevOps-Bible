# CI/CD — DevOps Interview Preparation

## Table of Contents
- [CI/CD Fundamentals](#cicd-fundamentals)
- [CI/CD Pipeline Design](#cicd-pipeline-design)
- [GitHub Actions](#github-actions)
- [Jenkins](#jenkins)
- [GitLab CI](#gitlab-ci)
- [ArgoCD & GitOps](#argocd--gitops)
- [Deployment Strategies](#deployment-strategies)
- [Artifact Management](#artifact-management)
- [Testing in CI/CD](#testing-in-cicd)
- [Security in CI/CD (DevSecOps)](#security-in-cicd)
- [Scenario-Based Questions](#scenario-based-questions)

---

## CI/CD Fundamentals

### Q: What is CI/CD?

**Continuous Integration (CI):**
- Developers merge code to main branch frequently
- Each merge triggers automated build + tests
- Catches bugs early, keeps main branch deployable

**Continuous Delivery (CD):**
- Code is always in a deployable state
- Deployment to production requires **manual approval**

**Continuous Deployment:**
- Every change that passes all tests is **automatically deployed** to production
- No human intervention

```
Developer → Push → Build → Unit Test → Integration Test → Security Scan
                                                               │
          Continuous Integration ◄──────────────────────────────┘
                                                               │
          → Deploy to Staging → E2E Tests → Manual Approval → Deploy to Prod
                                                               │
          Continuous Delivery ◄────────────────────────────────┘
                                                               │
          (OR) → Auto Deploy to Prod                           │
          Continuous Deployment ◄──────────────────────────────┘
```

### Q: What are the key principles of a good CI/CD pipeline?

1. **Fast feedback** — Tests should run in < 10 minutes
2. **Reproducible builds** — Same commit → same artifact, every time
3. **Trunk-based development** — Short-lived branches, frequent merges
4. **Automated everything** — Build, test, scan, deploy, rollback
5. **Immutable artifacts** — Build once, deploy to all environments
6. **Environment parity** — Dev ≈ Staging ≈ Production
7. **Fail fast** — Cheap tests first, expensive tests later
8. **Rollback capability** — One-click rollback to previous version

---

## CI/CD Pipeline Design

### Q: Design a production CI/CD pipeline.

```
┌───────────────── CI Pipeline ─────────────────────────┐
│                                                         │
│  1. Code Checkout                                       │
│  2. Dependency Install (cached)                         │
│  3. Lint & Format Check                                 │
│  4. Unit Tests + Coverage                               │
│  5. SAST (Static Application Security Testing)          │
│  6. Build Artifact (Docker image)                       │
│  7. Image Scan (Trivy/Snyk)                             │
│  8. Push to Registry (ECR/GCR/DockerHub)                │
│  9. Integration Tests                                   │
│                                                         │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────── CD Pipeline ─────────────────────────┐
│                                                         │
│  10. Deploy to Dev (auto)                               │
│  11. Smoke Tests                                        │
│  12. Deploy to Staging (auto)                           │
│  13. E2E Tests + Performance Tests                      │
│  14. Deploy to Production                               │
│      ├── Canary (10% → 50% → 100%)                     │
│      └── Monitor metrics between steps                  │
│  15. Post-deploy verification                           │
│  16. Rollback if metrics degrade                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Q: Build once, deploy anywhere pattern.

```
                    Build
                      │
                      ▼
              Docker Image (v1.2.3)
              pushed to Registry
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
       Dev         Staging     Production
    (env vars)   (env vars)   (env vars)

Same artifact, different configuration per environment.
NEVER rebuild for each environment.
```

---

## GitHub Actions

### Q: Write a production-grade GitHub Actions workflow.

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

permissions:
  contents: read
  packages: write
  id-token: write          # For OIDC with AWS

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4

    - uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'

    - run: npm ci
    - run: npm run lint
    - run: npm run test -- --coverage

    - uses: actions/upload-artifact@v4
      with:
        name: coverage
        path: coverage/

  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        severity: 'CRITICAL,HIGH'
        exit-code: '1'

  build-and-push:
    needs: [test, security-scan]
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}

    steps:
    - uses: actions/checkout@v4

    - uses: docker/setup-buildx-action@v3

    - uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=sha,prefix=
          type=semver,pattern={{version}}

    - uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: staging
    steps:
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
        aws-region: us-east-1

    - name: Deploy to EKS
      run: |
        aws eks update-kubeconfig --name staging-cluster
        kubectl set image deployment/myapp \
          myapp=${{ needs.build-and-push.outputs.image-tag }}
        kubectl rollout status deployment/myapp --timeout=300s

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://app.example.com
    steps:
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
        aws-region: us-east-1

    - name: Deploy to Production
      run: |
        aws eks update-kubeconfig --name production-cluster
        kubectl set image deployment/myapp \
          myapp=${{ needs.build-and-push.outputs.image-tag }}
        kubectl rollout status deployment/myapp --timeout=300s
```

### Q: GitHub Actions key concepts.

| Concept | Description |
|---------|-------------|
| **Workflow** | YAML file in `.github/workflows/` |
| **Job** | Set of steps that run on same runner |
| **Step** | Individual command or action |
| **Runner** | Machine that executes jobs (GitHub-hosted or self-hosted) |
| **Action** | Reusable unit (e.g., `actions/checkout@v4`) |
| **Environment** | Named target with protection rules (approvals, wait timers) |
| **Secret** | Encrypted variable (`${{ secrets.API_KEY }}`) |
| **Matrix** | Run job across multiple configurations |
| **Artifact** | Files passed between jobs |
| **Cache** | Speed up dependencies across runs |
| **OIDC** | Keyless auth to cloud providers |

---

## Jenkins

### Q: Jenkins architecture.

```
┌─── Jenkins Controller ───────────────────────┐
│  - Manages jobs/pipelines                      │
│  - Schedules builds                            │
│  - Serves UI                                   │
│  - Stores configuration                        │
└──────┬─────────────────────────────────────────┘
       │
  ┌────┴────────────────────────┐
  ▼                              ▼
┌─── Agent 1 ──┐          ┌─── Agent 2 ──┐
│  (Linux)      │          │  (Docker)     │
│  Executes     │          │  Executes     │
│  builds       │          │  builds       │
└───────────────┘          └───────────────┘
```

### Q: Declarative vs Scripted Pipeline.

```groovy
// Declarative (preferred)
pipeline {
    agent any
    environment {
        REGISTRY = 'myregistry.com'
    }
    stages {
        stage('Build') {
            steps {
                sh 'npm ci'
                sh 'npm run build'
            }
        }
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps { sh 'npm test' }
                }
                stage('Lint') {
                    steps { sh 'npm run lint' }
                }
            }
        }
        stage('Deploy') {
            when { branch 'main' }
            steps {
                sh 'kubectl apply -f k8s/'
            }
        }
    }
    post {
        failure {
            slackSend channel: '#alerts', message: "Build failed: ${env.BUILD_URL}"
        }
        always {
            cleanWs()
        }
    }
}
```

---

## GitLab CI

### Q: GitLab CI pipeline example.

```yaml
# .gitlab-ci.yml
stages:
  - test
  - build
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHORT_SHA

test:
  stage: test
  image: node:20-alpine
  cache:
    key: $CI_COMMIT_REF_SLUG
    paths: [node_modules/]
  script:
    - npm ci
    - npm run lint
    - npm test

build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
    - docker build -t $DOCKER_IMAGE .
    - docker push $DOCKER_IMAGE
  only:
    - main

deploy_production:
  stage: deploy
  image: bitnami/kubectl
  script:
    - kubectl set image deployment/myapp myapp=$DOCKER_IMAGE
  environment:
    name: production
    url: https://app.example.com
  when: manual          # Manual approval gate
  only:
    - main
```

---

## ArgoCD & GitOps

### Q: What is GitOps?

**GitOps** = Git as the single source of truth for declarative infrastructure and applications.

**Principles:**
1. **Declarative** — Entire system described declaratively
2. **Versioned and immutable** — Canonical desired state in Git
3. **Pulled automatically** — Software agents pull desired state from Git
4. **Continuously reconciled** — Agents ensure actual state matches desired

**Push vs Pull model:**

| Model | How | Tools |
|-------|-----|-------|
| **Push** | CI pipeline pushes changes to cluster | Jenkins, GitHub Actions + kubectl |
| **Pull** | Agent in cluster pulls from Git repo | ArgoCD, Flux |

**Pull model (GitOps) advantages:**
- Cluster credentials never leave the cluster
- Self-healing (drift detection and correction)
- Audit trail (Git history)

### Q: ArgoCD architecture and workflow.

```
Developer → Push manifests to Git repo
                    │
ArgoCD (in cluster) │
    │               │
    ▼               ▼
┌────────────────────────────┐
│  Application Controller     │ ← Watches Git repo
│  - Syncs desired state      │
│  - Detects drift             │
│  - Health monitoring         │
└──────────┬─────────────────┘
           │
           ▼
┌────────────────────────────┐
│  Kubernetes Cluster         │
│  Actual state reconciled    │
│  with Git                   │
└────────────────────────────┘
```

```yaml
# ArgoCD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/org/k8s-manifests.git
    targetRevision: main
    path: apps/my-app/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true           # Delete resources removed from Git
      selfHeal: true         # Fix manual changes (drift)
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

---

## Deployment Strategies

### Q: Compare deployment strategies.

| Strategy | Downtime | Rollback Speed | Resource Cost | Risk |
|----------|----------|---------------|---------------|------|
| **Recreate** | Yes | Slow (redeploy) | 1x | High |
| **Rolling Update** | No | Medium | 1x-1.25x | Medium |
| **Blue-Green** | No | Instant (switch) | 2x | Low |
| **Canary** | No | Fast (shift traffic) | 1x + canary | Lowest |
| **A/B Testing** | No | Fast | 1x + variant | Low |

### Q: How do you implement canary deployments?

```
Step 1: Deploy canary (v2) alongside stable (v1)
        v1: 95% traffic    v2: 5% traffic

Step 2: Monitor key metrics:
        - Error rate (4xx, 5xx)
        - Latency (p50, p95, p99)
        - Saturation (CPU, memory)
        - Business metrics (conversion rate)

Step 3: If metrics are healthy, increase traffic:
        v1: 75%    v2: 25%

Step 4: Continue until:
        v1: 0%     v2: 100%

If metrics degrade at any step → automatic rollback to v1
```

**Tools:** Argo Rollouts, Flagger, Istio, AWS AppMesh, Nginx canary annotations

---

## Artifact Management

### Q: Container registry best practices.

- **Tag strategy:** `git-sha`, `semver`, `branch-sha` (avoid `:latest` in prod)
- **Immutable tags** — Prevent overwriting
- **Vulnerability scanning** — Scan on push
- **Lifecycle policies** — Auto-delete old images
- **Replication** — Cross-region for DR
- **Signing** — Cosign / Docker Content Trust

```bash
# ECR lifecycle policy (keep last 30 images)
aws ecr put-lifecycle-policy --repository-name myapp --lifecycle-policy-text '{
  "rules": [{
    "rulePriority": 1,
    "selection": {
      "tagStatus": "any",
      "countType": "imageCountMoreThan",
      "countNumber": 30
    },
    "action": { "type": "expire" }
  }]
}'
```

---

## Testing in CI/CD

### Q: Testing pyramid in CI/CD.

```
         /\
        /  \       E2E Tests (slow, expensive, few)
       /    \      - Selenium, Playwright, Cypress
      /──────\
     /        \    Integration Tests (medium speed)
    /          \   - API tests, DB tests, contract tests
   /────────────\
  /              \  Unit Tests (fast, cheap, many)
 /                \ - Jest, pytest, go test
/──────────────────\
```

**Pipeline test strategy:**
```
PR Pipeline (fast, <5 min):
├── Lint
├── Unit tests
└── SAST scan

Merge Pipeline (thorough, <15 min):
├── All PR checks
├── Integration tests
├── Build artifact
├── Image scan
└── Deploy to dev

Pre-Production (<30 min):
├── Deploy to staging
├── E2E tests
├── Performance tests
├── DAST scan
└── Chaos tests (optional)
```

---

## Security in CI/CD

### Q: DevSecOps — security in the pipeline.

| Stage | Tool Category | Examples |
|-------|--------------|---------|
| **Code** | SAST (Static Analysis) | SonarQube, Semgrep, CodeQL |
| **Dependencies** | SCA (Software Composition) | Snyk, Dependabot, Renovate |
| **Build** | Image Scanning | Trivy, Grype, Snyk Container |
| **Config** | IaC Scanning | tfsec, Checkov, kube-score |
| **Deploy** | DAST (Dynamic Analysis) | OWASP ZAP, Burp Suite |
| **Runtime** | Runtime Security | Falco, Sysdig, Aqua |
| **Secrets** | Secret Detection | TruffleHog, GitLeaks, detect-secrets |

**Key practices:**
- **Shift left** — Find issues early (cheaper to fix)
- **OIDC auth** — No long-lived credentials in CI
- **Signed artifacts** — Cosign for containers
- **SBOM** — Software Bill of Materials (Syft)
- **Branch protection** — Required reviews, status checks
- **Least privilege runners** — No admin access

---

## Scenario-Based Questions

### Q: CI pipeline takes 30 minutes. How to speed it up?

```
1. Parallelize: Run tests, lint, scan in parallel
2. Cache: Dependencies (npm, pip), Docker layers, test databases
3. Optimize tests:
   - Only run affected tests (test impact analysis)
   - Split large test suites across parallel runners
4. Optimize Docker builds:
   - Multi-stage builds
   - BuildKit cache mounts
   - Smaller base images
5. Faster runners: Self-hosted, larger instances, ARM
6. Fail fast: Put quick checks first (lint, type check)
7. Incremental builds: Only rebuild changed components
8. Remote caching: Turborepo, Nx, Bazel for monorepos
```

### Q: A deployment failed in production. Walk through your response.

```
1. DETECT: Automated monitoring alerts (error rate spike, latency increase)

2. TRIAGE: Is it affecting users? What's the blast radius?

3. ROLLBACK: If significant impact, rollback immediately
   - Canary: Shift traffic back to stable
   - Blue-Green: Switch to old version
   - Rolling: kubectl rollout undo deployment/myapp

4. COMMUNICATE: Status page update, Slack notification

5. INVESTIGATE (after rollback):
   - What changed? (Git diff, deployment logs)
   - Why didn't tests catch it?
   - Check staging vs production differences

6. FIX: Root cause fix, not band-aid

7. PREVENT:
   - Add regression test
   - Improve canary analysis
   - Add feature flag for risky changes
   - Blameless post-mortem
```

---

## Key Resources

- **Continuous Delivery (book)** — Jez Humble & David Farley
- **The DevOps Handbook** — Gene Kim et al.
- **Accelerate (book)** — DORA metrics (deployment frequency, lead time, MTTR, change failure rate)
- **GitHub Actions Docs** — https://docs.github.com/en/actions
- **ArgoCD Docs** — https://argo-cd.readthedocs.io
- **Codefresh GitOps Guide** — https://codefresh.io/gitops/
