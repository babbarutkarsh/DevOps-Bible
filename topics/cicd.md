---
title: CI/CD
nav_order: 50
description: "CI/CD — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# CI/CD — DevOps Interview Preparation

> **Question frequency:** 🔥 very frequently asked · ⭐ important · 💡 good to know

## Table of Contents
- [CI/CD Fundamentals](#cicd-fundamentals)
- [CI/CD Pipeline Design](#cicd-pipeline-design)
- [GitHub Actions](#github-actions)
- [Jenkins](#jenkins)
- [GitLab CI](#gitlab-ci)
- [ArgoCD & GitOps](#argocd--gitops)
- [Deployment Strategies](#deployment-strategies)
- [Progressive Delivery](#progressive-delivery)
- [Artifact Management](#artifact-management)
- [Supply Chain Security (2025-26 HOT)](#supply-chain-security-2025-26-hot)
- [Testing in CI/CD](#testing-in-cicd)
- [Security in CI/CD (DevSecOps)](#security-in-cicd-devsecops)
- [DORA Metrics & Observability](#dora-metrics--observability)
- [Monorepo vs Polyrepo CI](#monorepo-vs-polyrepo-ci)
- [Scenario-Based Questions](#scenario-based-questions)

---

## CI/CD Fundamentals

### 🔥 Q: What is CI/CD?

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

### 🔥 Q: What are the key principles of a good CI/CD pipeline?

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

### 🔥 Q: Design a production CI/CD pipeline.

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

### ⭐ Q: Build once, deploy anywhere pattern.

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

### ⭐ Q: Write a production-grade GitHub Actions workflow.

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

### 🔥 Q: GitHub Actions key concepts.

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

### 🔥 Q: OIDC keyless authentication to AWS/GCP/Azure (HOT in 2025-26).

**Problem:** Long-lived credentials (access keys, service account keys) in CI are risky (rotation, leakage).

**Solution:** OIDC federation — GitHub issues short-lived JWT tokens, cloud provider exchanges them for temporary credentials.

```yaml
# GitHub → AWS (no secrets!)
permissions:
  id-token: write
  contents: read

steps:
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
    aws-region: us-east-1
# AWS IAM trust policy allows sts:AssumeRoleWithWebIdentity
# from GitHub's OIDC provider token
```

**GCP:**
```yaml
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: 'projects/123/locations/global/workloadIdentityPools/github/providers/github'
    service_account: 'gh-actions@project.iam.gserviceaccount.com'
```

**Benefits:** Zero secrets, auto-expiry (1hr), scoped to repo/branch, audit trail.

### ⭐ Q: Reusable workflows vs composite actions — when to use each?

| Feature | Reusable Workflow | Composite Action |
|---------|------------------|------------------|
| **Level** | Job-level | Step-level |
| **Location** | `.github/workflows/` | `.github/actions/` or separate repo |
| **Can call** | Multiple jobs | Multiple steps |
| **Secrets** | Inherited or passed explicitly | No direct secret access (pass as input) |
| **Environment** | Can define environment | Cannot |
| **Use case** | Full CI/CD template | Reusable step bundle (install, build) |

**Example reusable workflow:**
```yaml
# .github/workflows/deploy.yml
on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4
      - run: ./deploy.sh
```

**Caller:**
```yaml
jobs:
  deploy-staging:
    uses: ./.github/workflows/deploy.yml
    with:
      environment: staging
```

### ⭐ Q: Self-hosted vs ephemeral runners (2025-26 trend).

| Type | Pros | Cons |
|------|------|------|
| **Persistent self-hosted** | Cached deps, fast startup | Dirty state, security risk (secrets leak across runs) |
| **Ephemeral (auto-scaled)** | Clean slate, secure, elastic | Slower (no cache), setup overhead |

**2025-26 best practice:** Ephemeral runners with remote caching (BuildKit, Turborepo).

**Tools:** GitHub Actions Runner Controller (ARC) on K8s, AWS CodeBuild ephemeral, Buildkite elastic.

```yaml
# ARC runner scale set
jobs:
  build:
    runs-on: [self-hosted, arc-runner-set, ephemeral]
```

### ⭐ Q: Matrix strategy for multi-platform/multi-version builds.

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [18, 20, 22]
      fail-fast: false    # Continue even if one combo fails
    runs-on: ${{ matrix.os }}
    steps:
    - uses: actions/setup-node@v4
      with:
        node-version: ${{ matrix.node }}
    - run: npm test
```

**Parallel execution:** 3 OS × 3 Node versions = 9 jobs run concurrently (GitHub limits apply).

---

## Jenkins

### 🔥 Q: Jenkins architecture.

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

**Modern pattern:** Controller for orchestration only, all builds on ephemeral agents (Kubernetes plugin).

### ⭐ Q: Declarative vs Scripted Pipeline.

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

### ⭐ Q: Jenkins shared libraries (reusable pipeline code).

**Problem:** Copy-paste pipelines across repos → drift and maintenance nightmare.

**Solution:** Shared library in Git, imported by all pipelines.

```groovy
// vars/standardPipeline.groovy (in shared library repo)
def call(Map config) {
    pipeline {
        agent any
        stages {
            stage('Build') {
                steps { sh "${config.buildCmd}" }
            }
            stage('Test') {
                steps { sh "${config.testCmd}" }
            }
            stage('Deploy') {
                when { branch 'main' }
                steps { sh "${config.deployCmd}" }
            }
        }
    }
}
```

```groovy
// Jenkinsfile (in app repo)
@Library('my-shared-library@v1.2.0') _

standardPipeline(
    buildCmd: 'npm run build',
    testCmd: 'npm test',
    deployCmd: 'kubectl apply -f k8s/'
)
```

**Versioning:** Pin library versions (`@v1.2.0`), not `@main`.

---

## GitLab CI

### ⭐ Q: GitLab CI pipeline example.

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

### 🔥 Q: What is GitOps?

**GitOps** = Git as the single source of truth for declarative infrastructure and applications.

**Principles:**
1. **Declarative** — Entire system described declaratively
2. **Versioned and immutable** — Canonical desired state in Git
3. **Pulled automatically** — Software agents pull desired state from Git
4. **Continuously reconciled** — Agents ensure actual state matches desired

**Push vs Pull model:**

| Model | How | Tools | Use case |
|-------|-----|-------|----------|
| **Push** | CI pipeline pushes changes to cluster | Jenkins, GitHub Actions + kubectl | Simple, direct deploy |
| **Pull** | Agent in cluster pulls from Git repo | ArgoCD, Flux | Multi-cluster, self-healing |

**Pull model (GitOps) advantages:**
- Cluster credentials never leave the cluster
- Self-healing (drift detection and correction)
- Audit trail (Git history)
- Disaster recovery (cluster state in Git)

**When NOT to use GitOps:**
- Stateful systems (databases) — operationally driven
- Secrets (use sealed secrets / external secrets operator)
- High-frequency deploys (100s/day) — too much Git churn

### ⭐ Q: ArgoCD architecture and workflow.

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

### ⭐ Q: ArgoCD vs Flux — which to choose?

| Feature | ArgoCD | Flux |
|---------|--------|------|
| **UI** | Rich web UI | No UI (CLI/K9s) |
| **Multi-tenancy** | AppProjects, RBAC | Strong (Kustomize overlays) |
| **Helm support** | Native | Via Helm Controller |
| **Image automation** | Via Argo CD Image Updater | Built-in (ImageUpdateAutomation) |
| **Multi-cluster** | Easy (cluster add) | Manual (kubeconfig per cluster) |
| **Community** | Larger, CNCF graduated | CNCF graduated, GitOps Toolkit |
| **Complexity** | Heavier (multiple components) | Lighter (controller per feature) |

**2025-26 trend:** ArgoCD for platform teams (multi-cluster, UI), Flux for GitOps-purist teams (lightweight, GitOps Toolkit composability).

### 💡 Q: GitOps at scale — app-of-apps pattern.

**Problem:** 100s of microservices = 100s of Application manifests.

**Solution:** App-of-apps — one parent Application generates child Applications.

```yaml
# apps/root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/org/k8s-manifests.git
    path: apps/            # Directory with all app manifests
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```
apps/
├── root-app.yaml           (parent)
├── service-a.yaml          (child)
├── service-b.yaml          (child)
└── infra/
    ├── ingress.yaml        (child)
    └── cert-manager.yaml   (child)
```

**Advanced:** ApplicationSet for dynamic app generation (cluster generator, git generator).

---

## Deployment Strategies

### 🔥 Q: Compare deployment strategies.

| Strategy | Downtime | Rollback Speed | Resource Cost | Risk |
|----------|----------|---------------|---------------|------|
| **Recreate** | Yes | Slow (redeploy) | 1x | High |
| **Rolling Update** | No | Medium | 1x-1.25x | Medium |
| **Blue-Green** | No | Instant (switch) | 2x | Low |
| **Canary** | No | Fast (shift traffic) | 1x + canary | Lowest |
| **A/B Testing** | No | Fast | 1x + variant | Low |

### 🔥 Q: How do you implement canary deployments?

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

### ⭐ Q: Feature flags vs canary deployments — when to use each?

| Dimension | Feature Flags | Canary Deployment |
|-----------|--------------|-------------------|
| **Scope** | Code-level (if/else) | Infrastructure-level (traffic split) |
| **Rollback** | Instant (toggle off) | Fast (shift traffic) |
| **Targeting** | User attributes (ID, role, region) | Traffic % (random) |
| **Risk** | Low (code path exists, tested) | Medium (new infra) |
| **Use case** | A/B test, gradual rollout, kill switch | New service version validation |
| **Overhead** | Code changes, flag management | Deployment orchestration |

**Best practice:** Combine both — canary validates infra health, feature flag controls user exposure.

```
Deploy v2 (canary 10%)
  ↓
Metrics healthy?
  ↓
Feature flag: enable for 10% of users
  ↓
Business metrics healthy?
  ↓
Increase canary to 100%, feature flag to 100%
```

---

## Progressive Delivery

### ⭐ Q: What is progressive delivery?

**Progressive Delivery** = Continuous Delivery + fine-grained control over blast radius.

**Techniques:**
- **Canary analysis** — Automated metrics-based rollout
- **A/B testing** — Compare variants for business metrics
- **Feature flags** — Runtime control over features
- **Staged rollouts** — Gradual rollout by region/cohort

**Tools:** Argo Rollouts, Flagger, LaunchDarkly, Split, Unleash

### 💡 Q: Argo Rollouts — canary with automated analysis.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: myapp
spec:
  replicas: 5
  strategy:
    canary:
      steps:
      - setWeight: 20           # 20% traffic to canary
      - pause: {duration: 5m}    # Observe for 5 minutes
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 50
      - pause: {duration: 5m}
      - analysis:
          templates:
          - templateName: success-rate
      - setWeight: 100
      canaryService: myapp-canary
      stableService: myapp-stable
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:v2
---
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
  - name: success-rate
    interval: 1m
    successCondition: result >= 0.99
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{status!~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
```

**If analysis fails → automatic rollback.**

### 🔥 Q: Deployment strategies — tradeoffs.

| Strategy | Zero-downtime | Fast rollback | Extra resources | Complexity | Best for |
|----------|--------------|---------------|-----------------|------------|----------|
| **Recreate** | ❌ | ❌ | ✅ (1x) | Low | Dev/test |
| **Rolling** | ✅ | Medium | ✅ (1x) | Low | Stateless apps |
| **Blue-Green** | ✅ | ✅ (instant) | ❌ (2x) | Medium | Critical apps |
| **Canary** | ✅ | ✅ | ✅ (1x + small) | High | High-traffic apps |

---

## Artifact Management

### ⭐ Q: Container registry best practices.

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

## Supply Chain Security (2025-26 HOT)

### 🔥 Q: What is supply chain security in CI/CD?

**Supply chain attack** = Compromise dependencies, build tools, or artifacts to inject malicious code.

**Examples:**
- SolarWinds (2020) — Build system compromised
- codecov bash uploader (2021) — Credential stealer in CI script
- npm/PyPI typosquatting — Malicious packages with similar names

**Defense layers:**

```
┌─────────────────────────────────────┐
│  1. Dependency scanning              │ ← Snyk, Dependabot, Renovate
├─────────────────────────────────────┤
│  2. SBOM generation                  │ ← Syft, CycloneDX
├─────────────────────────────────────┤
│  3. Build provenance (SLSA)          │ ← Witness, in-toto
├─────────────────────────────────────┤
│  4. Artifact signing (cosign)        │ ← Sigstore, cosign
├─────────────────────────────────────┤
│  5. Policy gates (OPA)               │ ← Gatekeeper, Kyverno
├─────────────────────────────────────┤
│  6. Runtime verification             │ ← Admission controllers
└─────────────────────────────────────┘
```

### 🔥 Q: SLSA framework — what is it? (2025-26 interview hot topic)

**SLSA** (Supply-chain Levels for Software Artifacts) — Google-originated framework for supply chain integrity.

**Levels:**

| Level | Requirements |
|-------|-------------|
| **SLSA 1** | Build process documented |
| **SLSA 2** | Signed provenance (who built, when, from what source) |
| **SLSA 3** | Provenance verified, hardened build (no tampering) |
| **SLSA 4** | Two-party review + hermetic build (bit-for-bit reproducible) |

**Provenance** = metadata about how an artifact was built:
- Source commit SHA
- Build command
- Build environment (runner image)
- Builder identity

```bash
# GitHub Actions generates SLSA provenance automatically
# with reusable workflow:
uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v1.9.0
```

### 🔥 Q: SBOM (Software Bill of Materials) — why is it required now?

**SBOM** = List of all components (libraries, versions) in your software.

**Why (2025-26):**
- US Executive Order 14028 (2021) requires SBOM for govt software
- Log4Shell (2021) showed orgs didn't know what they had deployed
- Compliance (NIST, EU Cyber Resilience Act)

**Generate SBOM:**
```bash
# Syft (Anchore)
syft packages myapp:latest -o cyclonedx-json > sbom.json

# Docker Scout (Docker Desktop)
docker scout sbom myapp:latest

# Trivy
trivy image --format cyclonedx myapp:latest
```

**Verify on deploy:**
```yaml
# Kubernetes admission controller checks SBOM for CVEs
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: sbom-validator
webhooks:
- name: validate-sbom.example.com
  rules:
  - operations: ["CREATE"]
    apiGroups: ["apps"]
    resources: ["deployments"]
  clientConfig:
    service:
      name: sbom-validator
      namespace: default
```

### ⭐ Q: Cosign / Sigstore — signing container images.

**Problem:** How do you know the image in production was built by CI, not tampered with?

**Solution:** Sign images with cosign (Sigstore), verify signature on deploy.

```bash
# CI signs image
cosign sign --key cosign.key myregistry.io/myapp:v1.2.3

# Kubernetes verifies signature (policy controller)
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: require-signatures
spec:
  images:
  - glob: "myregistry.io/**"
  authorities:
  - key:
      data: |
        -----BEGIN PUBLIC KEY-----
        ...
        -----END PUBLIC KEY-----
```

**Keyless signing (2025-26):**
```bash
# OIDC-based, no key management
cosign sign myregistry.io/myapp:v1.2.3
# Signs with ephemeral key, cert stored in Rekor transparency log
```

**Verify:**
```bash
cosign verify --certificate-identity CI@github.com \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  myregistry.io/myapp:v1.2.3
```

### ⭐ Q: Policy as code in pipelines (OPA / Kyverno).

**Use case:** Enforce standards in CI/CD — "all images must be signed", "no latest tag", "must have SBOM".

**OPA (Open Policy Agent) example:**
```rego
# policy.rego
package kubernetes.admission

deny[msg] {
  input.request.kind.kind == "Deployment"
  image := input.request.object.spec.template.spec.containers[_].image
  endswith(image, ":latest")
  msg := sprintf("Image %v uses ':latest' tag, which is not allowed", [image])
}

deny[msg] {
  input.request.kind.kind == "Deployment"
  not has_signature(input.request.object)
  msg := "Deployment must use signed images"
}
```

**Kyverno example:**
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-image-signature
spec:
  validationFailureAction: enforce
  rules:
  - name: verify-signature
    match:
      resources:
        kinds:
        - Deployment
    verifyImages:
    - imageReferences:
      - "myregistry.io/*"
      attestors:
      - count: 1
        entries:
        - keys:
            publicKeys: |-
              -----BEGIN PUBLIC KEY-----
              ...
              -----END PUBLIC KEY-----
```

---

## Testing in CI/CD

### 🔥 Q: Testing pyramid in CI/CD.

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

## Security in CI/CD (DevSecOps)

### 🔥 Q: DevSecOps — security in the pipeline.

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

### 🔥 Q: Secret management in CI/CD.

**Options:**

| Tool | Pros | Cons |
|------|------|------|
| **GitHub Secrets** | Native, easy | Per-repo duplication, no rotation |
| **HashiCorp Vault** | Centralized, dynamic secrets | Complex setup |
| **AWS Secrets Manager** | Managed, auto-rotation | AWS-locked |
| **SOPS (encrypted in Git)** | GitOps-friendly | Key management overhead |
| **External Secrets Operator** | K8s-native, multi-backend | K8s only |

**Best practice (2025-26):** OIDC to cloud provider → fetch secrets from cloud secret manager (no secrets in CI env vars).

```yaml
# GitHub Actions → AWS Secrets Manager
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123:role/GHActions
    aws-region: us-east-1

- name: Get secrets
  run: |
    DB_PASSWORD=$(aws secretsmanager get-secret-value \
      --secret-id prod/db/password --query SecretString --output text)
    echo "::add-mask::$DB_PASSWORD"
```

### ⭐ Q: How to rotate leaked secrets in CI/CD?

**Incident response:**
```
1. REVOKE: Immediately revoke the leaked secret
2. AUDIT: Check access logs — was it used?
3. ROTATE: Generate new secret, update systems
4. REMEDIATE:
   - Remove from Git history (git-filter-repo, BFG)
   - Update CI/CD secrets
   - Scan for usage in other repos (GitHub secret scanning)
5. PREVENT:
   - Enable pre-commit hooks (detect-secrets, TruffleHog)
   - Enable GitHub secret scanning push protection
   - Use OIDC instead of long-lived credentials
```

---

## DORA Metrics & Observability

### 🔥 Q: What are DORA metrics?

**DORA** (DevOps Research & Assessment) — 4 key metrics for software delivery performance.

| Metric | Elite | High | Medium | Low |
|--------|-------|------|--------|-----|
| **Deployment Frequency** | Multiple/day | Weekly–Monthly | Monthly–Yearly | Yearly |
| **Lead Time for Changes** | < 1 hour | 1 day–1 week | 1 week–1 month | 1–6 months |
| **Time to Restore Service** | < 1 hour | < 1 day | 1 day–1 week | 1 week–1 month |
| **Change Failure Rate** | 0-15% | 16-30% | 31-45% | 46-60% |

**How to measure:**
- **Deploy frequency:** Count prod deploys (from Git tags, CI logs, ArgoCD)
- **Lead time:** Time from commit to deploy (Git commit timestamp → deploy timestamp)
- **MTTR:** Time from incident alert to resolution (PagerDuty, incident.io)
- **Change failure rate:** Failed deploys / total deploys (rollbacks, hotfixes)

**Tools:** Sleuth, LinearB, Haystack, Waydev, custom Datadog/Grafana dashboards.

### ⭐ Q: CI/CD observability — what to monitor?

**Pipeline metrics:**
- Build success rate, duration (p50, p95)
- Flakiness rate (same commit, different result)
- Queue time (time waiting for runner)

**Deployment metrics:**
- Deploy frequency (per service, per environment)
- Rollback rate
- Canary health score

**Business metrics:**
- Feature flag adoption (% users on new feature)
- A/B test results (conversion rate delta)

**Alerts:**
```
Alert: High build failure rate
Condition: build_success_rate < 0.8 for 1 hour
Action: Notify #platform-team

Alert: Slow pipeline
Condition: p95_build_duration > 15 minutes for 3 builds
Action: Auto-create ticket in backlog
```

---

## Monorepo vs Polyrepo CI

### ⭐ Q: Monorepo CI challenges and solutions.

**Challenges:**
1. **Long builds** — Changed 1 file, rebuild everything?
2. **Flaky tests** — One team breaks the build for everyone
3. **Dependency graph** — Which services are affected by a change?

**Solutions:**

| Challenge | Solution | Tools |
|-----------|----------|-------|
| Long builds | Incremental builds, test impact analysis | Bazel, Turborepo, Nx, Pants |
| Flaky tests | Test ownership, quarantine flaky tests | Buildkite test analytics, Gradle Enterprise |
| Dependency graph | Build only affected projects | Nx affected, Bazel query |

**Monorepo CI pattern:**
```yaml
# GitHub Actions with Nx
jobs:
  affected:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
      with:
        fetch-depth: 0    # Full history for affected analysis

    - name: Derive affected projects
      id: affected
      run: |
        npx nx affected:apps --base=origin/main --head=HEAD --plain > affected.txt
        echo "projects=$(cat affected.txt | tr '\n' ',')" >> $GITHUB_OUTPUT

    - name: Build affected
      run: npx nx affected --target=build --base=origin/main

    - name: Test affected
      run: npx nx affected --target=test --base=origin/main
```

**Remote caching:**
```bash
# Turborepo remote cache (Vercel)
npx turbo build --token=$TURBO_TOKEN

# Bazel remote cache (Google Cloud Storage)
bazel build //... --remote_cache=https://storage.googleapis.com/my-bazel-cache
```

### 💡 Q: Polyrepo CI — how to maintain consistency across 100s of repos?

**Challenges:**
- Pipeline drift (each repo has different CI)
- Security updates (patch 100 repos)
- Tooling upgrades (node 18 → 20)

**Solutions:**

| Approach | Description | Tools |
|----------|-------------|-------|
| **Shared workflows** | Reusable GitHub Actions workflows | `.github/workflows/shared.yml` |
| **Shared libraries** | Jenkins shared library, CircleCI orbs | Groovy vars, orbs |
| **Pipeline templates** | Enforce standard pipeline | Renovate, GitHub repo templates |
| **Automated PRs** | Bot opens PRs to update all repos | Renovate, Dependabot, custom bot |

**Example: Renovate updates all repos:**
```json
// renovate.json
{
  "extends": ["config:base"],
  "automerge": true,
  "automergeType": "pr",
  "packageRules": [
    {
      "matchUpdateTypes": ["patch", "minor"],
      "automerge": true
    }
  ]
}
```

---

## Scenario-Based Questions

### 🔥 Q: CI pipeline takes 30 minutes. How to speed it up?

**Diagnosis first:**
```bash
# Profile pipeline — which stage is slow?
# GitHub Actions: workflow run time breakdown
# Jenkins: Blue Ocean timeline
```

**Optimizations:**
```
1. Parallelize: Run tests, lint, scan in parallel
2. Cache: Dependencies (npm, pip), Docker layers, test databases
3. Optimize tests:
   - Only run affected tests (test impact analysis)
   - Split large test suites across parallel runners (matrix strategy)
   - Run fast tests first (fail fast)
4. Optimize Docker builds:
   - Multi-stage builds
   - BuildKit cache mounts (--mount=type=cache)
   - Smaller base images (alpine, distroless)
   - Layer ordering (COPY package.json → npm install → COPY src)
5. Faster runners: Self-hosted, larger instances, ARM (2x cheaper, faster)
6. Fail fast: Put quick checks first (lint < 30s, type check < 1 min)
7. Incremental builds: Only rebuild changed components (Nx, Bazel)
8. Remote caching: Turborepo, Nx Cloud, Bazel remote cache
9. Reduce test coverage in CI: 80% in PR, 100% in nightly
```

**Example improvement:**
```
Before: 30 minutes serial
├── Install deps (5 min)
├── Lint (2 min)
├── Unit tests (10 min)
├── Build (5 min)
├── Integration tests (8 min)

After: 8 minutes parallel + cache
├── Install deps (1 min, cached)
└── Parallel:
    ├── Lint (2 min)
    ├── Unit tests (5 min, split 3-way)
    ├── Build (3 min, cached layers)
    └── Integration tests (8 min)
Max = 8 min (longest parallel track)
```

### 🔥 Q: CI pipeline is flaky (passes/fails randomly). How do you fix it?

**Diagnosis:**
```bash
# Run same commit 10 times, collect failures
for i in {1..10}; do
  gh workflow run ci.yml --ref main
done

# Analyze failure patterns
# - Same test fails? → Flaky test (timing, random data)
# - Different tests fail? → Resource contention, external dependency
# - All fail at same stage? → Infra issue (runner OOM, network)
```

**Common causes + fixes:**

| Cause | Fix |
|-------|-----|
| Race condition | Add explicit waits, use test fixtures |
| External API (flaky 3rd party) | Mock/stub, or use VCR/pact for recorded responses |
| Shared state between tests | Isolated test databases, cleanup hooks |
| Time-dependent tests | Mock time (`jest.useFakeTimers()`) |
| Resource exhaustion (OOM) | Larger runner, reduce parallelism |
| Network flake | Retry with backoff, local mock server |

**Quarantine pattern:**
```yaml
# Mark flaky tests, run separately
test:
  - run: npm test -- --testPathIgnorePatterns=flaky
  - run: npm test -- --testPathPattern=flaky --retries=3
    continue-on-error: true    # Don't block merge
```

**Track flakiness:**
```bash
# Buildkite Test Analytics, GitHub test reports
# Metric: flakiness rate = (flaky runs / total runs)
```

### 🔥 Q: A deployment failed in production. Walk through your response.

```
1. DETECT: Automated monitoring alerts (error rate spike, latency increase)

2. TRIAGE:
   - Blast radius? (% users affected, which region)
   - Severity? (can't login vs. cosmetic bug)
   - What changed? (Git diff, deploy logs, feature flags)

3. ROLLBACK: If significant impact, rollback immediately
   - Canary: Shift traffic back to stable (kubectl argo rollouts abort)
   - Blue-Green: Switch DNS/LB to old version
   - Rolling: kubectl rollout undo deployment/myapp
   - Feature flag: Disable new feature (LaunchDarkly, Unleash)

4. COMMUNICATE: Status page update, Slack notification, customer support

5. INVESTIGATE (after rollback):
   - What changed? (Git diff, config changes)
   - Why didn't tests catch it? (staging vs prod difference?)
   - Check logs, traces (APM), metrics

6. FIX: Root cause fix, not band-aid

7. PREVENT:
   - Add regression test
   - Improve canary analysis (add metric that would have caught it)
   - Add feature flag for risky changes
   - Blameless post-mortem (document in runbook)
   - Improve staging parity with prod
```

### 🔥 Q: How do you promote a build from dev → staging → production?

**Anti-pattern:** Rebuild for each environment (drift risk).

**Best practice:** Build once, promote same artifact.

```
┌─── CI Pipeline (triggered by commit) ──────┐
│  1. Build Docker image: myapp:sha-abc123    │
│  2. Push to registry                        │
│  3. Tag: myapp:dev-abc123                   │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│  CD Pipeline: Deploy to dev                 │
│    kubectl set image ... myapp:sha-abc123   │
│    Smoke tests pass                         │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│  Promote to staging (manual approval)       │
│    Tag: myapp:staging-abc123                │
│    kubectl set image ... myapp:sha-abc123   │
│    E2E tests pass                           │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│  Promote to prod (manual approval + window) │
│    Tag: myapp:prod-abc123                   │
│    Canary deploy: 10% → 50% → 100%          │
│    Automated analysis (error rate, latency) │
└─────────────────────────────────────────────┘
```

**GitOps pattern:**
```bash
# dev/staging/prod folders in Git
k8s-manifests/
├── dev/myapp.yaml       (image: myapp:sha-abc123)
├── staging/myapp.yaml   (image: myapp:sha-xyz456)
└── prod/myapp.yaml      (image: myapp:sha-xyz456)

# Promotion = PR to change image tag in next env
```

### ⭐ Q: A secret was leaked in CI logs. What do you do?

**Immediate actions:**
```
1. REVOKE: Rotate the leaked secret immediately
   - API keys: Revoke in provider console
   - Database password: Change password
   - SSH keys: Remove from authorized_keys

2. AUDIT: Check if it was used
   - Cloud provider access logs (CloudTrail, GCP audit logs)
   - GitHub Actions logs (who viewed the run?)
   - Service logs (unauthorized API calls?)

3. REMOVE from history:
   - GitHub: Delete workflow run (Settings → Actions → delete run)
   - Git history: git-filter-repo to remove from commits
   - Notify GitHub support to purge from caches

4. NOTIFY:
   - Security team
   - Affected service owners
   - Compliance (if regulated data)
```

**Prevention:**
```
1. Enable GitHub secret scanning push protection
   - Blocks pushes with secrets

2. Pre-commit hooks:
   - detect-secrets, TruffleHog, git-secrets

3. Use ::add-mask:: in GitHub Actions:
   - echo "::add-mask::$SECRET"
   - Redacts from logs automatically

4. OIDC instead of long-lived credentials:
   - No secrets stored in CI

5. Rotate secrets regularly (30-90 days)
```

### ⭐ Q: Design a zero-downtime deployment strategy for a stateful app.

**Challenge:** Stateful apps (databases, caches) can't just kill pods.

**Strategy:**

```
1. PRE-DEPLOY: Schema migration (if needed)
   - Backward-compatible migrations ONLY
   - Add new column (nullable)
   - Deploy app that writes to both old + new
   - Backfill data
   - Deploy app that reads from new (next release)
   - Drop old column (next release)

2. ROLLING UPDATE with readiness gates:
   - New pod starts
   - Readiness probe: checks DB connection, cache warm
   - Only then: traffic shifts to new pod
   - Old pod: graceful shutdown (SIGTERM → drain connections)

3. BLUE-GREEN for stateful:
   - Run blue + green in parallel (both write to same DB)
   - Switch reads to green gradually (canary DNS)
   - Monitor for data consistency issues
   - Full cutover
   - Drain blue (backup period)

4. PDB (PodDisruptionBudget):
   - Ensure minimum replicas always available
   - Prevents cascading failure
```

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2        # Always keep 2 pods running
  selector:
    matchLabels:
      app: myapp
```

### ⭐ Q: How do you handle rollback in a multi-service deployment?

**Scenario:** Deploy service-a (v2) + service-b (v2) together. service-a fails in prod.

**Options:**

| Approach | When to use | Tradeoff |
|----------|-------------|----------|
| **Rollback both** | Services tightly coupled | Safe, but service-b v2 might have been fine |
| **Rollback only failed** | Services loosely coupled, backward-compatible | Risky if protocol changed |
| **Forward fix** | Rollback is expensive (migration) | Requires fast fix, high confidence |
| **Feature flag off** | New feature, old code still exists | Best option if available |

**Best practice:**
```
1. Backward compatibility: New version must work with old version
   - API versioning (/v1, /v2)
   - Feature flags for breaking changes

2. Deployment order:
   - Deploy consumer first (can handle old + new producer)
   - Then deploy producer

3. Rollback strategy in runbook:
   - Document dependencies
   - Test rollback in staging
   - Automated rollback triggers (error rate > 5%)
```

### ⭐ Q: CI/CD for a monorepo with 50 microservices — how do you optimize?

**Challenges:**
- Changed 1 service → rebuild all 50?
- Flaky test in service-a blocks deploy of service-b
- Pipeline takes 2 hours

**Solutions:**

```
1. AFFECTED ANALYSIS: Only build/test changed services
   - Nx: npx nx affected --target=build
   - Bazel: bazel test //services/service-a/...

2. PARALLEL BUILDS: Build each service in parallel
   - GitHub Actions matrix strategy
   - Jenkins parallel stages

3. REMOTE CACHING: Share build artifacts across CI runs
   - Nx Cloud, Bazel remote cache
   - Speed: 30 min → 3 min (90% cache hit)

4. INDEPENDENT DEPLOY: Each service deploys independently
   - ArgoCD Application per service
   - Trigger deploy only for changed services

5. SHARED PIPELINE TEMPLATES: DRY principle
   - GitHub reusable workflows
   - Jenkins shared library

6. DEPENDENCY GRAPH: Know what's affected
   - Nx graph
   - Bazel query 'rdeps(//..., //services/service-a:*)'
```

**Example workflow:**
```yaml
jobs:
  affected:
    runs-on: ubuntu-latest
    outputs:
      services: ${{ steps.affected.outputs.services }}
    steps:
    - run: npx nx affected:apps --base=origin/main --head=HEAD --json > affected.json
    - id: affected
      run: echo "services=$(cat affected.json | jq -c '.projects')" >> $GITHUB_OUTPUT

  build:
    needs: affected
    strategy:
      matrix:
        service: ${{ fromJson(needs.affected.outputs.services) }}
    runs-on: ubuntu-latest
    steps:
    - run: npx nx build ${{ matrix.service }}
    - run: npx nx test ${{ matrix.service }}
    - run: npx nx deploy ${{ matrix.service }}
```

### ⭐ Q: Supply chain attack via compromised dependency — how do you defend?

**Defense in depth:**

```
1. DEPENDENCY SCANNING: Continuous monitoring
   - Snyk, Dependabot, Renovate
   - Block high/critical CVEs in CI

2. LOCK FILES: Pin exact versions
   - package-lock.json (npm), Gemfile.lock (Ruby)
   - Prevents surprise updates

3. SBOM: Know what's in production
   - Generate SBOM in CI (Syft)
   - Store with artifact
   - Query: "Which services use log4j 2.14?"

4. SIGNATURE VERIFICATION: Only allow signed packages
   - npm provenance (2023+)
   - sigstore for containers

5. PRIVATE REGISTRY PROXY: Control what enters
   - Artifactory, Nexus
   - Mirror public registries, scan on ingress
   - Block unapproved packages

6. POLICY GATES: Enforce at deploy time
   - OPA: deny if SBOM contains CVE > CVSS 7
   - Admission controller in K8s

7. RUNTIME DETECTION: Monitor for suspicious behavior
   - Falco, Sysdig
   - Alert: process spawned unexpected child
```

**Incident response (if compromised dependency deployed):**
```
1. IDENTIFY: Which services use the compromised version?
   - SBOM query across all artifacts

2. QUARANTINE: Block deploys from affected images
   - Image admission policy: deny image with CVE-XYZ

3. REBUILD: Rebuild all services with patched version
   - Trigger mass rebuild pipeline

4. REDEPLOY: Canary deploy fixed version
   - Monitor for regressions

5. AUDIT: Check logs for exploitation attempts
   - SIEM query for IOCs

6. REPORT: Incident report, lessons learned
```

---

## Key Resources

### Books
- **Continuous Delivery** — Jez Humble & David Farley (2010, still canonical)
- **The DevOps Handbook** — Gene Kim et al.
- **Accelerate** — DORA metrics (deployment frequency, lead time, MTTR, change failure rate)
- **Release It! (2nd ed)** — Michael Nygard (stability patterns)

### Docs & Specs
- **GitHub Actions** — https://docs.github.com/en/actions
- **ArgoCD** — https://argo-cd.readthedocs.io
- **Flux** — https://fluxcd.io/docs/
- **SLSA** — https://slsa.dev (supply chain levels)
- **Sigstore** — https://www.sigstore.dev (keyless signing)
- **SBOM Guide** — https://www.cisa.gov/sbom (NTIA formats)

### Tools (2025-26)
- **Supply chain:** Sigstore, Syft, Trivy, Grype, in-toto
- **GitOps:** ArgoCD, Flux, Argo Rollouts, Flagger
- **Policy:** OPA, Kyverno, Gatekeeper
- **Observability:** Sleuth, LinearB, Datadog CI Visibility
- **Monorepo:** Nx, Turborepo, Bazel, Pants

### Trends to Watch
- **Ephemeral runners** (security + scale)
- **OIDC keyless auth** (zero long-lived secrets)
- **SLSA L3 adoption** (Google, GitHub push)
- **GitOps at scale** (ApplicationSets, multi-cluster)
- **Progressive delivery** (Argo Rollouts mainstream)
- **DORA metrics** (platform team KPIs)
