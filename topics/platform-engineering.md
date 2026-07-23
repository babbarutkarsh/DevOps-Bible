---
title: Platform Engineering & GitOps
nav_order: 52
description: "Platform engineering, internal developer platforms, GitOps at scale, and progressive delivery — the modern-platform questions top companies ask in 2025-26."
---

# Platform Engineering & GitOps — DevOps Interview Preparation

> **Question frequency:** 🔥 very frequently asked · ⭐ important · 💡 good to know

## Table of Contents
- [What is Platform Engineering?](#what-is-platform-engineering)
- [Internal Developer Platforms (IDPs)](#internal-developer-platforms-idps)
- [GitOps Deep Dive](#gitops-deep-dive)
- [Progressive Delivery & Rollouts](#progressive-delivery--rollouts)
- [Infrastructure as Software & Control Planes](#infrastructure-as-software--control-planes)
- [Service Mesh & Networking](#service-mesh--networking)
- [Multi-Tenancy & Cluster Fleet Management](#multi-tenancy--cluster-fleet-management)
- [FinOps & Cost Management](#finops--cost-management)
- [DevEx & Golden Paths](#devex--golden-paths)
- [Scenario-Based Questions](#scenario-based-questions)
- [Key Resources](#key-resources)

---

## What is Platform Engineering?

### 🔥 Q: What is platform engineering and why did it emerge?

**Platform Engineering** = Building and operating an internal platform that enables app teams to self-serve infrastructure, environments, and tooling with minimal cognitive load.

**Why it emerged (2023-2026):**
- **DevOps promise vs reality** — "You build it, you run it" worked at two-pizza teams but not at scale (500+ engineers)
- **Cognitive load crisis** — App teams drowning in Terraform, Kubernetes YAML, observability config, security scanning
- **Toil & ticket ops** — Infrastructure teams became bottlenecks (ticket-driven provisioning, manual reviews)
- **Team Topologies book** — Platform team as a distinct team type, reducing cognitive load for stream-aligned teams

```
Old Model (2015-2020):                  Platform Team Model (2023-2026):
┌─────────────────────┐                 ┌─────────────────────┐
│   App Team          │                 │   Platform Team      │
│  - Writes code       │                 │  - Builds IDP        │
│  - Writes Terraform  │                 │  - Golden paths      │
│  - Writes K8s YAML   │                 │  - Self-service UI   │
│  - Manages observ.   │                 │  - Backstage portal  │
│  - Manages security  │                 │  - Policy guardrails │
│  - On-call           │                 └──────────┬───────────┘
└─────────────────────┘                            │
                                                    ▼
                                        ┌─────────────────────┐
                                        │   App Team          │
                                        │  - Writes code       │
                                        │  - Click "New Svc"   │
                                        │  - Gets: pre-wired   │
                                        │    CI/CD, observ.,   │
                                        │    secrets, infra    │
                                        │  - On-call (app)     │
                                        └─────────────────────┘
```

**Key shift:** Platform as a product, app teams as customers.

### ⭐ Q: What makes a good internal platform?

**Golden Path (Paved Road) principles:**
```
1. Self-service — No tickets, no manual steps
2. Opinionated defaults — 80% use case works out-of-the-box
3. Escape hatches — 20% edge cases can customize
4. Low cognitive load — App teams don't need to know Kubernetes internals
5. Fast feedback — Provision a new service in < 10 minutes
6. Observable — Platform team monitors adoption, lead time, friction
7. Documentation & onboarding — 5-minute quickstart, clear runbooks
```

**Anti-patterns:**
- ❌ **"Kubernetes as a Service"** — Handing app teams raw K8s clusters with no abstractions
- ❌ **Ticket-driven provisioning** — Platform team as bottleneck, slow cycle time
- ❌ **Too flexible** — 50 config knobs, app teams paralyzed by choice
- ❌ **No documentation** — "Just read the code"
- ❌ **No adoption metrics** — Platform team doesn't know if anyone uses their platform

---

## Internal Developer Platforms (IDPs)

### 🔥 Q: What is an Internal Developer Platform (IDP)?

**IDP** = A curated layer of tooling and workflows that codifies best practices and provides self-service capabilities for app teams.

**Core components:**
```
┌─────────────────────────────────────────────────────┐
│  Developer Portal (Backstage, Port, Cortex)          │
│  - Service catalog (who owns what, APIs, docs)        │
│  - Software templates (scaffold new services)        │
│  - TechDocs (auto-generated from Markdown)           │
└──────────┬──────────────────────────────────────────┘
           │
┌──────────▼──────────────────────────────────────────┐
│  Control Plane (Orchestration)                       │
│  - Provision infra (Crossplane, Terraform)           │
│  - Wire up CI/CD (GitHub Actions, ArgoCD)            │
│  - Configure observability (Grafana, Datadog)        │
│  - Apply policies (OPA, Kyverno)                     │
└──────────┬──────────────────────────────────────────┘
           │
┌──────────▼──────────────────────────────────────────┐
│  Infrastructure & Services                           │
│  - K8s clusters, databases, S3, secrets, networking  │
└─────────────────────────────────────────────────────┘
```

**IDP ≠ tools.** An IDP is the glue layer that connects tools into a coherent workflow.

### 🔥 Q: What is Backstage?

**Backstage** (Spotify, CNCF graduated 2022) — Open-source developer portal framework.

**Core plugins:**
```yaml
# Backstage features
Software Catalog:
  - Service ownership registry
  - API documentation (OpenAPI, gRPC)
  - Dependencies (service A calls B, C)
  - Links (GitHub, PagerDuty, Grafana)

Software Templates (Scaffolder):
  - Golden path templates
  - One-click "Create Microservice"
  - Generates repo, CI/CD, K8s manifests, docs

TechDocs:
  - Docs-as-code (Markdown in repo)
  - Auto-published to Backstage
  - Versioned, searchable

Search:
  - Unified search across catalog, docs, logs, Slack

Plugins:
  - GitHub, ArgoCD, Jenkins, PagerDuty, Datadog, etc.
```

**Example: Create a new service from a template:**
```
1. Developer opens Backstage → Software Templates
2. Clicks "Create Microservice" template
3. Fills form:
   - Service name: payment-api
   - Owner team: payments
   - Language: Go
   - Database: PostgreSQL
4. Backstage:
   - Creates GitHub repo with scaffolded code
   - Sets up CI/CD (GitHub Actions + ArgoCD)
   - Provisions PostgreSQL via Crossplane
   - Configures observability (Prometheus, Grafana)
   - Adds service to catalog
5. Developer gets a link to the repo + docs
6. First deploy happens in 10 minutes
```

### ⭐ Q: Backstage vs Port vs Cortex — which to choose?

| Feature | Backstage | Port | Cortex |
|---------|-----------|------|--------|
| **Type** | Open-source | Commercial | Commercial |
| **Cost** | Free (self-hosted) | Paid (SaaS) | Paid (SaaS) |
| **Customization** | High (plugins, React) | Medium (config) | Low (opinionated) |
| **Setup effort** | High (DIY deployment) | Low (SaaS) | Low (SaaS) |
| **Catalog** | Strong | Strong | Strong |
| **Templates** | Strong | Medium | Weak |
| **Scorecards** | Plugin-based | Built-in | Built-in |
| **Multi-cloud** | Yes | Yes | Yes |
| **Community** | Large (CNCF) | Growing | Smaller |
| **Best for** | Eng-heavy orgs, full control | Fast setup, small teams | Scorecards, compliance |

**2025-26 trend:** Backstage dominates (Spotify, Netflix, Expedia, American Airlines); Port growing fast in SMBs.

### 🔥 Q: How do you measure platform success?

**Platform team OKRs:**

| Metric | Target | How to measure |
|--------|--------|---------------|
| **Adoption rate** | 80% of services on platform | Backstage catalog vs total repos |
| **Lead time to first deploy** | < 1 day (new service) | Backstage template timestamp → first production deploy |
| **Toil reduction** | 50% fewer infra tickets | Jira ticket volume before/after |
| **Developer satisfaction (DevEx)** | SPACE score > 70 | Quarterly survey |
| **MTTR** | < 30 min | PagerDuty / incident data |
| **Change failure rate** | < 5% | Failed deploys / total deploys |
| **Platform uptime** | 99.9% | ArgoCD / IDP control plane SLA |

**DevEx surveys (2025-26 hot topic):**
- **SPACE framework** — Satisfaction, Performance, Activity, Communication, Efficiency
- **Questions:** "How easy is it to provision a new service?", "Do you understand what the platform does?", "Would you recommend our platform to a friend?"
- **Tools:** Cortex, Jellyfish, LinearB

**Anti-pattern:** Measuring platform team output (PRs, deploys) instead of app team outcomes (lead time, satisfaction).

---

## GitOps Deep Dive

### 🔥 Q: What are the four GitOps principles?

```
1. Declarative — Entire system described declaratively (YAML, Terraform HCL)
2. Versioned and immutable — Canonical desired state versioned in Git
3. Pulled automatically — Software agents pull desired state from Git (not pushed by CI)
4. Continuously reconciled — Agents ensure actual state matches desired state (drift correction)
```

**Pull vs Push:**

```
Push Model (Traditional CI/CD):
Developer → Push → CI pipeline → kubectl apply → Cluster
              (CI has cluster credentials)

Pull Model (GitOps):
Developer → Push → Git repo
                      ↓
Cluster ← Poll ← GitOps agent (ArgoCD/Flux)
(Agent has Git credentials, cluster stays internal)
```

**GitOps advantages:**
- **Security** — Cluster credentials never leave the cluster
- **Audit trail** — Git history = full deployment history
- **Self-healing** — Drift detection and auto-correction
- **Disaster recovery** — Cluster state in Git, rebuild from scratch
- **Multi-cluster** — One Git repo → many clusters

### 🔥 Q: ArgoCD vs Flux — deep comparison.

| Dimension | ArgoCD | Flux |
|-----------|--------|------|
| **Architecture** | Monolithic controller | Microservices (GitOps Toolkit) |
| **UI** | Rich web UI (app-level, sync status, diff) | No UI (CLI only, K9s integration) |
| **Helm** | Native support, tracks upstream chart | HelmRelease CRD, Helm Controller |
| **Kustomize** | Native support | Native support |
| **Multi-cluster** | Easy (cluster add via UI/CLI) | Manual (kubeconfig per cluster, Flux per cluster) |
| **Image automation** | Via Argo CD Image Updater plugin | Built-in (ImageUpdateAutomation CRD) |
| **Drift detection** | Live diff in UI, auto-sync | Reconciliation loop logs |
| **Secrets** | Supports sealed-secrets, Vault, external-secrets | Supports sealed-secrets, SOPS, external-secrets |
| **RBAC** | AppProjects + RBAC | Kubernetes RBAC + multi-tenancy via namespaces |
| **Notifications** | Built-in (Slack, email, webhooks) | Via notification-controller |
| **Progressive delivery** | Via Argo Rollouts (separate project) | Via Flagger (separate project) |
| **Community** | CNCF graduated, larger | CNCF graduated, GitOps Toolkit composability |
| **Resource usage** | Heavier (6+ pods) | Lighter (3-4 controllers) |

**When to choose ArgoCD:**
- Multi-cluster management (100+ clusters)
- Non-technical users need UI (platform engineers, SREs)
- Centralized control plane (one ArgoCD for whole org)

**When to choose Flux:**
- GitOps-purist teams (want minimal control plane)
- Pull-request-driven workflows (Flux bot comments on PRs)
- Image automation out-of-box (Flux ImageUpdateAutomation)

**2025-26 trend:** ArgoCD at 70% market share (platform teams), Flux at 25% (GitOps enthusiasts), coexistence common (ArgoCD for apps, Flux for infra).

### ⭐ Q: ArgoCD architecture deep dive.

```
┌─────────────────────────────────────────────────────┐
│  Developer                                           │
│  Commits manifests to Git repo                       │
└──────────┬──────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────┐
│  Git Repository (GitHub, GitLab)                     │
│  - Kubernetes manifests (YAML)                       │
│  - Helm charts                                       │
│  - Kustomize overlays                                │
└──────────┬──────────────────────────────────────────┘
           │
           │ (poll or webhook)
           ▼
┌─────────────────────────────────────────────────────┐
│  ArgoCD (in Kubernetes cluster)                      │
│                                                      │
│  ┌─────────────────────────────────────────────┐   │
│  │  Application Controller                      │   │
│  │  - Compares Git desired state vs live state  │   │
│  │  - Detects drift (OutOfSync)                 │   │
│  │  - Syncs (applies manifests)                 │   │
│  │  - Health assessment (Healthy/Progressing)   │   │
│  └─────────────────────────────────────────────┘   │
│                                                      │
│  ┌─────────────────────────────────────────────┐   │
│  │  Repo Server                                 │   │
│  │  - Clones Git repos                          │   │
│  │  - Renders manifests (Helm, Kustomize)       │   │
│  │  - Caches rendered manifests                 │   │
│  └─────────────────────────────────────────────┘   │
│                                                      │
│  ┌─────────────────────────────────────────────┐   │
│  │  API Server                                  │   │
│  │  - Serves web UI                             │   │
│  │  - gRPC/REST API                             │   │
│  │  - RBAC enforcement                          │   │
│  └─────────────────────────────────────────────┘   │
│                                                      │
│  ┌─────────────────────────────────────────────┐   │
│  │  Redis (cache)                               │   │
│  └─────────────────────────────────────────────┘   │
└──────────┬──────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────┐
│  Kubernetes Cluster (target)                         │
│  Actual state reconciled with Git                    │
└─────────────────────────────────────────────────────┘
```

**Key concepts:**

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
      selfHeal: true        # Fix manual kubectl changes (drift)
    syncOptions:
    - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### 🔥 Q: GitOps at scale — app-of-apps vs ApplicationSets.

**Problem:** 500 microservices = 500 Application manifests to manage.

**Solution 1: App-of-apps pattern**

```
┌─────────────────────────────────────────────────────┐
│  Root Application (app-of-apps)                      │
│  Watches: apps/ directory                            │
└──────────┬──────────────────────────────────────────┘
           │
           ├──► Application: service-a
           ├──► Application: service-b
           ├──► Application: service-c
           ├──► Application: infra-nginx
           └──► Application: infra-cert-manager
```

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
    path: apps/
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Directory structure:**
```
apps/
├── root-app.yaml           (parent)
├── service-a.yaml          (child)
├── service-b.yaml          (child)
├── service-c.yaml          (child)
└── infra/
    ├── nginx-ingress.yaml
    └── cert-manager.yaml
```

**Solution 2: ApplicationSets (2023-2026)**

```yaml
# Dynamic app generation from Git directory structure
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: microservices
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/org/k8s-manifests.git
      revision: main
      directories:
      - path: apps/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/k8s-manifests.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

**ApplicationSet generators:**

| Generator | Use Case |
|-----------|----------|
| **Git** | Generate apps from Git directories |
| **Cluster** | Generate apps for each registered cluster (multi-cluster) |
| **List** | Static list of apps |
| **Matrix** | Combine generators (Git × Cluster) |
| **Pull Request** | Generate preview environments per PR |

**Advanced: Multi-cluster with ApplicationSet**

```yaml
# Deploy same app to all clusters
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: multi-cluster-app
spec:
  generators:
  - clusters:
      selector:
        matchLabels:
          env: production
  template:
    metadata:
      name: 'my-app-{{name}}'
    spec:
      source:
        repoURL: https://github.com/org/k8s-manifests.git
        path: apps/my-app
        targetRevision: main
      destination:
        server: '{{server}}'
        namespace: default
```

### ⭐ Q: GitOps repo structure — monorepo vs per-app vs per-environment?

```
Option 1: Monorepo (all apps, all environments)
k8s-manifests/
├── apps/
│   ├── service-a/
│   │   ├── base/
│   │   │   └── deployment.yaml
│   │   └── overlays/
│   │       ├── dev/
│   │       ├── staging/
│   │       └── production/
│   └── service-b/
└── infra/
    ├── nginx-ingress/
    └── cert-manager/

Pros: Single source of truth, cross-service changes (API version bumps)
Cons: Large blast radius, slower CI on big repos

Option 2: Per-app repos
service-a-manifests/
├── base/
└── overlays/
    ├── dev/
    ├── staging/
    └── production/

Pros: Team ownership, smaller blast radius
Cons: Hard to enforce standards, cross-service changes painful

Option 3: Per-environment repos (rare)
k8s-production/
├── service-a/
├── service-b/
└── infra/

Pros: Environment isolation
Cons: Hard to promote changes (dev → staging → prod)
```

**Best practice (2025-26):** Monorepo with Kustomize overlays, branch protection on main, PR-required for changes.

### 🔥 Q: GitOps secrets management — sealed secrets vs external secrets vs SOPS.

**Problem:** Can't commit Kubernetes Secrets (base64-encoded) to Git (anyone can decode).

**Solution 1: Sealed Secrets (Bitnami)**

```bash
# Encrypt secret with cluster's public key
kubeseal < secret.yaml > sealed-secret.yaml

# Commit sealed-secret.yaml to Git
git add sealed-secret.yaml
git commit -m "Add DB password"
git push

# ArgoCD syncs sealed-secret.yaml
# SealedSecret controller decrypts in-cluster → regular Secret
```

```yaml
# sealed-secret.yaml (safe to commit)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: db-credentials
  namespace: production
spec:
  encryptedData:
    password: AgB7... (encrypted)
```

**Pros:** Simple, no external dependencies  
**Cons:** Key rotation hard, no audit trail

**Solution 2: External Secrets Operator (ESO)**

```yaml
# ExternalSecret fetches from AWS Secrets Manager / Vault / GCP Secret Manager
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-credentials
  data:
  - secretKey: password
    remoteRef:
      key: production/db-password
```

**Pros:** Centralized secret management, audit trail, rotation  
**Cons:** External dependency (Vault/AWS), more complex

**Solution 3: SOPS (Mozilla)**

```bash
# Encrypt YAML with AWS KMS / GCP KMS / PGP
sops --encrypt --kms arn:aws:kms:us-east-1:123:key/abc secret.yaml > secret.enc.yaml

# Commit encrypted file to Git
git add secret.enc.yaml

# Flux decrypts on-the-fly (ArgoCD needs plugin)
```

**Pros:** Git-native, versioned, diff-friendly  
**Cons:** Flux-first class, ArgoCD needs plugin

**Comparison:**

| Tool | Encryption | Where secrets live | ArgoCD support | Flux support |
|------|------------|-------------------|---------------|--------------|
| **Sealed Secrets** | In-cluster public key | Git (encrypted) | ✅ Native | ✅ Native |
| **External Secrets** | External (Vault, AWS) | External store | ✅ Native | ✅ Native |
| **SOPS** | KMS / PGP | Git (encrypted) | Plugin | ✅ Native |

**2025-26 trend:** External Secrets Operator winning (centralized management, compliance).

### ⭐ Q: Promotion across environments — how to model dev → staging → prod in GitOps?

**Option 1: Branches**

```
Git branches:
├── dev        (dev cluster syncs from this)
├── staging    (staging cluster syncs)
└── main       (production cluster syncs)

Promotion:
dev branch → PR to staging → PR to main
```

**Pros:** Clear separation  
**Cons:** Merge conflicts, divergence

**Option 2: Directories (Kustomize overlays)**

```
apps/my-app/
├── base/
│   └── deployment.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml (image tag: latest)
    ├── staging/
    │   └── kustomization.yaml (image tag: v1.2.3)
    └── production/
        └── kustomization.yaml (image tag: v1.2.3)

Promotion:
Edit overlays/production/kustomization.yaml to update image tag
```

**Pros:** Single branch, fewer merge conflicts  
**Cons:** Accidental changes to prod

**Option 3: Separate repos**

```
k8s-dev/        (dev cluster syncs)
k8s-staging/    (staging cluster syncs)
k8s-production/ (production cluster syncs)

Promotion:
Copy manifests from dev repo → staging repo → production repo
```

**Pros:** Strongest isolation  
**Cons:** Hard to track changes across environments

**Best practice (2025-26):** Directories (Kustomize) with branch protection + required approvals on production overlay changes.

---

## Progressive Delivery & Rollouts

### 🔥 Q: What is progressive delivery?

**Progressive Delivery** = Continuous Delivery + fine-grained release control (canary, blue-green, feature flags).

```
Continuous Delivery:
  All changes that pass tests → deployable
  Manual gate before production

Progressive Delivery:
  All changes that pass tests → deployed to production
  Automated gradual rollout (canary 10% → 50% → 100%)
  Automated rollback if metrics degrade
```

**Key techniques:**

| Technique | What | When |
|-----------|------|------|
| **Canary** | Deploy to 10% of traffic, monitor, increase to 100% | Validate service health |
| **Blue-Green** | Deploy to parallel env (green), switch traffic from blue | Zero-downtime, instant rollback |
| **A/B Testing** | Split traffic by user attributes (region, ID) | Compare business metrics |
| **Feature Flags** | Runtime toggle for code paths | Decouple deploy from release |

### ⭐ Q: Argo Rollouts — automated canary with metrics.

**Argo Rollouts** = Kubernetes controller for advanced deployment strategies (canary, blue-green).

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
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
      canaryService: my-app-canary
      stableService: my-app-stable
      trafficRouting:
        istio:
          virtualService:
            name: my-app-vs
  template:
    spec:
      containers:
      - name: my-app
        image: my-app:v2
---
# AnalysisTemplate — defines success criteria
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
  - name: success-rate
    interval: 1m
    successCondition: result >= 0.99
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{status!~"5.."}[5m]))
          /
          sum(rate(http_requests_total[5m]))
```

**What happens:**
```
1. Deploy canary (v2) alongside stable (v1)
2. Shift 20% traffic to canary
3. Wait 5 minutes, run analysis (Prometheus query)
4. If success rate >= 99% → continue
5. Shift 50% traffic to canary
6. Run analysis again
7. If success → shift 100% (promote canary to stable)
8. If failure at any step → automatic rollback to stable
```

**Traffic routing:**
- **Istio / Linkerd** — VirtualService weight
- **Nginx / ALB** — Canary annotations
- **SMI (Service Mesh Interface)** — TrafficSplit

### 💡 Q: Argo Rollouts vs Flagger — which to choose?

| Dimension | Argo Rollouts | Flagger |
|-----------|---------------|---------|
| **Architecture** | Standalone controller | Part of Flux (optional standalone) |
| **Traffic routing** | Istio, Nginx, ALB, Linkerd, SMI | Istio, Linkerd, Contour, Gloo, nginx |
| **Metrics** | Prometheus, Datadog, New Relic | Prometheus |
| **Strategies** | Canary, blue-green | Canary, A/B, blue-green |
| **UI** | Argo CD UI integration | No UI (Flux CD) |
| **Analysis** | AnalysisTemplate (flexible) | Metrics thresholds |
| **Rollback** | Automatic | Automatic |
| **GitOps** | Works with ArgoCD | Flux-first, works with ArgoCD |
| **Community** | CNCF sandbox | CNCF sandbox |

**When to choose Argo Rollouts:**
- Using ArgoCD for GitOps
- Need UI for rollout status
- Complex analysis (multiple metrics, webhooks)

**When to choose Flagger:**
- Using Flux for GitOps
- Simple metric-based canary
- Lightweight (no UI needed)

### ⭐ Q: Feature flags vs canary deployments — how to combine them?

```
┌───────────────────────────────────────────────────────┐
│  Phase 1: Canary deployment (infra validation)        │
│  Deploy v2 to 10% of pods                              │
│  Monitor: CPU, memory, error rate, latency             │
│  Goal: Validate service health                         │
└────────────┬──────────────────────────────────────────┘
             │ (metrics healthy)
             ▼
┌───────────────────────────────────────────────────────┐
│  Phase 2: Feature flag rollout (user validation)       │
│  All pods running v2, but new feature off by default   │
│  Enable feature for 10% of users (flag: new-checkout)  │
│  Monitor: Conversion rate, user errors, support tickets│
│  Goal: Validate business impact                        │
└────────────┬──────────────────────────────────────────┘
             │ (business metrics good)
             ▼
┌───────────────────────────────────────────────────────┐
│  Phase 3: Full rollout                                 │
│  Increase canary to 100%                               │
│  Increase feature flag to 100%                         │
└───────────────────────────────────────────────────────┘
```

**Why both?**
- **Canary** validates infra/service doesn't crash
- **Feature flag** validates users like the new feature

**Tools:** LaunchDarkly, Unleash, Flagsmith, Split, AWS AppConfig

---

## Infrastructure as Software & Control Planes

### 🔥 Q: What is infrastructure as software vs infrastructure as code?

```
Infrastructure as Code (IaC):              Infrastructure as Software:
─────────────────────────────────          ──────────────────────────────────
Imperative or declarative scripts          Declarative API + reconciliation loop
terraform apply (one-time)                 Kubernetes operator (continuous)
State stored in S3/local                   State stored in K8s etcd
Drift detection manual (terraform plan)    Drift detection automatic (reconcile)
Examples: Terraform, Pulumi, CloudFormation Examples: Crossplane, AWS Controllers, operators
```

**Key shift:** From "run a script" to "declare desired state, let controller reconcile."

### 🔥 Q: What is Crossplane?

**Crossplane** = Infrastructure as Kubernetes resources. Provision cloud resources (AWS RDS, GCP GCS, Azure VMs) via kubectl.

```yaml
# Instead of Terraform HCL:
# Create AWS RDS instance via K8s manifest
apiVersion: database.aws.crossplane.io/v1beta1
kind: RDSInstance
metadata:
  name: my-postgres
spec:
  forProvider:
    engine: postgres
    engineVersion: "15.3"
    instanceClass: db.t3.medium
    allocatedStorage: 100
    masterUsername: admin
    masterPasswordSecretRef:
      name: db-password
      namespace: default
      key: password
    region: us-east-1
  writeConnectionSecretToRef:
    name: postgres-connection
    namespace: default
```

**Architecture:**

```
┌─────────────────────────────────────────────────────┐
│  Developer                                           │
│  kubectl apply -f rds.yaml                           │
└──────────┬──────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────┐
│  Kubernetes API                                      │
│  Stores desired state in etcd                        │
└──────────┬──────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────┐
│  Crossplane Provider (AWS)                           │
│  Watches RDSInstance resources                       │
│  Calls AWS API to create/update/delete RDS           │
└──────────┬──────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────┐
│  AWS (actual RDS instance)                           │
└─────────────────────────────────────────────────────┘
```

**Compositions (like Terraform modules):**

```yaml
# XRD (Composite Resource Definition) — abstract "database"
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.platform.example.com
spec:
  group: platform.example.com
  names:
    kind: XDatabase
    plural: xdatabases
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              size:
                type: string
                enum: [small, medium, large]
              region:
                type: string
---
# Composition — map XDatabase to AWS RDS
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xdatabase-aws
spec:
  compositeTypeRef:
    apiVersion: platform.example.com/v1alpha1
    kind: XDatabase
  resources:
  - name: rds
    base:
      apiVersion: database.aws.crossplane.io/v1beta1
      kind: RDSInstance
      spec:
        forProvider:
          engine: postgres
          engineVersion: "15.3"
    patches:
    - fromFieldPath: spec.size
      toFieldPath: spec.forProvider.instanceClass
      transforms:
      - type: map
        map:
          small: db.t3.small
          medium: db.t3.medium
          large: db.r5.xlarge
```

**App team usage:**

```yaml
# App team doesn't know AWS RDS details
apiVersion: platform.example.com/v1alpha1
kind: XDatabase
metadata:
  name: my-database
spec:
  size: medium
  region: us-east-1
```

**Crossplane vs Terraform:**

| Dimension | Terraform | Crossplane |
|-----------|-----------|------------|
| **Paradigm** | Imperative apply | Declarative reconciliation |
| **State** | S3 / Terraform Cloud | Kubernetes etcd |
| **Drift** | Manual `terraform plan` | Auto-detected, auto-fixed |
| **API** | CLI | kubectl / K8s API |
| **GitOps** | Via CI/CD | Native (ArgoCD, Flux) |
| **Multi-cloud** | Yes | Yes |
| **Learning curve** | Low | High (K8s knowledge) |
| **Maturity** | High (2014) | Growing (2018) |

**When to choose Crossplane:**
- Already using Kubernetes
- Want GitOps for infra + apps
- Need self-healing infra
- Building an internal platform (Crossplane = control plane)

**When to stick with Terraform:**
- No Kubernetes
- Team knows Terraform well
- Need mature provider ecosystem (Terraform has 3000+ providers)

---

## Service Mesh & Networking

### 🔥 Q: What is a service mesh and when do you need one?

**Service Mesh** = Infrastructure layer that handles service-to-service communication (mTLS, observability, traffic management, retries, circuit breaking).

```
Without service mesh:
Service A ──HTTP──► Service B
(No encryption, manual retry logic, no observability)

With service mesh:
Service A ──HTTP──► Sidecar Proxy (Envoy)
                       │
                       ├── mTLS
                       ├── Metrics
                       ├── Retries
                       ├── Circuit breaking
                       │
                       ▼
                    Sidecar Proxy (Envoy) ──► Service B
```

**When you need a service mesh:**
- 50+ microservices
- Zero-trust networking (mTLS everywhere)
- Advanced traffic management (canary, retries, circuit breaking)
- Centralized observability (no code changes)

**When you DON'T need a service mesh:**
- < 10 services (overhead > benefit)
- Monolith or simple architecture
- No compliance requirements for mTLS

### 🔥 Q: Istio vs Linkerd vs Cilium — which to choose?

| Dimension | Istio | Linkerd | Cilium |
|-----------|-------|---------|--------|
| **Architecture** | Sidecar (Envoy proxy per pod) | Sidecar (Linkerd2-proxy) | eBPF (kernel-level, no sidecar) |
| **Performance** | Medium overhead (~20ms latency, 0.5 CPU) | Low overhead (~10ms, 0.3 CPU) | Lowest (<5ms, 0.1 CPU) |
| **Features** | Most extensive (VirtualService, Gateways, AuthN/AuthZ) | Simpler, opinionated | Networking + security + observability |
| **mTLS** | Yes (auto, Spire integration) | Yes (auto) | Yes (via WireGuard) |
| **Multi-cluster** | Strong (multi-primary, primary-remote) | Good (service mirroring) | Good |
| **Observability** | Prometheus, Jaeger, Kiali | Prometheus, Grafana | Hubble (eBPF-based) |
| **Complexity** | High (many CRDs, configuration knobs) | Low (batteries-included) | Medium |
| **Ingress** | Istio Gateway | Any (nginx, Contour) | Cilium Gateway API |
| **CNI** | Works with any CNI | Works with any CNI | Cilium IS the CNI |
| **CNCF** | Graduated | Graduated | Graduated |
| **Maturity** | High (Google, IBM) | High (Buoyant) | Growing (Isovalent / Cisco) |

**Istio Ambient Mesh (2024-2026):**
- **No sidecars** — Ztunnel (per-node proxy) for mTLS, optional waypoint proxy for L7
- **Lower overhead** — Fewer resources than sidecar
- **Easier upgrade** — No pod restarts

**When to choose Istio:**
- Need full feature set (traffic splitting, AuthZ policies, multi-cluster)
- Team can handle complexity
- Already using Envoy

**When to choose Linkerd:**
- Want simplicity + good defaults
- Low latency / low resource usage critical
- Kubernetes-native (no external control plane)

**When to choose Cilium:**
- Want eBPF (no sidecars, best performance)
- Need CNI + service mesh + security (NetworkPolicy) in one
- Kubernetes 1.25+ with eBPF support

**2025-26 trend:** Cilium growing fast (40% YoY), Istio Ambient Mesh reducing sidecar overhead, Linkerd stable at simplicity-focused orgs.

---

## Multi-Tenancy & Cluster Fleet Management

### ⭐ Q: Namespace isolation vs cluster-per-team — which model?

| Dimension | Namespace Isolation | Cluster per Team |
|-----------|-------------------|------------------|
| **Cost** | Low (shared control plane) | High (N clusters) |
| **Security** | Medium (kernel shared) | High (full isolation) |
| **Blast radius** | High (noisy neighbor) | Low (team blast radius) |
| **Management** | Simple (one cluster) | Complex (N clusters) |
| **RBAC** | Kubernetes RBAC | Cloud IAM + K8s RBAC |
| **Compliance** | Hard for strict isolation | Easier (PCI, HIPAA per cluster) |
| **Upgrade risk** | High (all teams affected) | Low (teams upgrade independently) |

**Best practice (2025-26):** Namespace isolation for small orgs (<50 engineers), cluster-per-team for large orgs (>500 engineers) or compliance needs.

**Hybrid:** Cluster-per-environment (shared dev cluster, isolated prod clusters).

### 💡 Q: What is vCluster?

**vCluster** = Virtual Kubernetes clusters inside a namespace of a host cluster.

```
Host Cluster
├── Namespace: team-a
│   └── vCluster (virtual K8s API, virtual etcd)
│       ├── Pod A
│       └── Pod B
└── Namespace: team-b
    └── vCluster
        ├── Pod C
        └── Pod D
```

**Pros:**
- Cost-effective (no separate control planes)
- Full isolation (each vCluster has own API server)
- Teams can install CRDs, operators without affecting others

**Cons:**
- Overhead (virtual API server per vCluster)
- Complexity (debugging spans host + virtual)

**Use case:** Multi-tenant platforms, CI/CD ephemeral environments, developer sandboxes.

### 💡 Q: Cluster API (CAPI) — what is it and when to use it?

**Cluster API** = Kubernetes-native API for provisioning and managing Kubernetes clusters.

```yaml
# Declarative cluster creation
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: my-cluster
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks: [10.244.0.0/16]
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSCluster
    name: my-cluster
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: my-cluster-control-plane
```

**Use cases:**
- **Fleet management** — Manage 100+ clusters declaratively
- **Multi-cloud** — Same API for AWS, GCP, Azure, on-prem
- **Day 2 ops** — Upgrade, scale, repair clusters via kubectl

**Alternatives:** EKS (AWS), GKE (GCP), AKS (Azure) managed services (simpler for single-cloud).

### ⭐ Q: Policy as code — OPA/Gatekeeper vs Kyverno.

**Use case:** Enforce platform guardrails — "no :latest tags", "all images signed", "resource limits required".

**OPA (Open Policy Agent) + Gatekeeper:**

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
  container := input.request.object.spec.template.spec.containers[_]
  not container.resources.limits
  msg := sprintf("Container %v must have resource limits", [container.name])
}
```

**Kyverno (Kubernetes-native):**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: enforce
  rules:
  - name: check-resource-limits
    match:
      resources:
        kinds:
        - Deployment
    validate:
      message: "All containers must have resource limits"
      pattern:
        spec:
          template:
            spec:
              containers:
              - resources:
                  limits:
                    memory: "?*"
                    cpu: "?*"
```

**Comparison:**

| Dimension | OPA / Gatekeeper | Kyverno |
|-----------|-----------------|---------|
| **Language** | Rego (custom) | YAML (Kubernetes-native) |
| **Learning curve** | High (learn Rego) | Low (YAML) |
| **Flexibility** | Very high | Medium |
| **Mutating** | Via Gatekeeper mutation | Native (generate, mutate) |
| **Testing** | Conftest (separate tool) | Built-in (kyverno test) |
| **Use case** | Complex policies | Simple to medium policies |

**2025-26 trend:** Kyverno winning for platform teams (easier to write/maintain), OPA for security teams (complex policies).

---

## FinOps & Cost Management

### 🔥 Q: What is FinOps and why is it a platform concern?

**FinOps** = Cloud financial management — visibility, optimization, accountability for cloud costs.

**Why platform teams own it:**
- **Platform sets defaults** — Right-sized instance types, spot usage, autoscaling policies
- **Showback/chargeback** — Platform teams enable cost attribution (team, product, environment)
- **Guardrails** — Platform prevents runaway costs (resource quotas, budget alerts)

**FinOps maturity model:**

```
Level 1: Visibility
- What are we spending?
- Where is the spend? (service, team, region)

Level 2: Optimization
- Right-size overprovisioned resources
- Use spot instances / savings plans
- Delete unused resources (dev clusters on weekends)

Level 3: Accountability
- Showback: "Team A spent $10k this month"
- Chargeback: Costs allocated to team budgets
- Forecast: "At current rate, we'll hit budget in 2 weeks"
```

### ⭐ Q: Kubecost vs OpenCost — what's the difference?

**OpenCost** (CNCF sandbox) = Open-source Kubernetes cost monitoring.

```bash
# Install OpenCost
kubectl apply -f https://raw.githubusercontent.com/opencost/opencost/main/kubernetes/opencost.yaml

# Query API
curl http://localhost:9003/allocation/compute?window=7d
```

**Kubecost** = Commercial product (free tier available, built on OpenCost).

| Feature | OpenCost | Kubecost |
|---------|----------|----------|
| **Cost** | Free | Free tier + paid |
| **Granularity** | Pod, namespace, label | Pod, namespace, label, team, product |
| **Alerts** | No | Yes (Slack, email) |
| **Recommendations** | No | Yes (right-sizing, spot) |
| **Multi-cluster** | Manual | Built-in |
| **Showback/chargeback** | Basic | Advanced |
| **UI** | Basic | Rich (dashboards, reports) |

**When to choose OpenCost:**
- Small team, simple needs
- Want full control / self-hosted

**When to choose Kubecost:**
- Need multi-cluster cost management
- Want recommendations + alerts
- Budget for SaaS (or use free tier)

### ⭐ Q: Cost optimization techniques for platform teams.

```
┌─────────────────────────────────────────────────────┐
│  1. Right-sizing                                     │
│  - Vertical Pod Autoscaler (VPA) recommendations     │
│  - Analyze P95 CPU/memory usage → set limits         │
│  Savings: 20-40%                                     │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  2. Spot / Preemptible instances                     │
│  - Karpenter (AWS), GKE Autopilot (GCP)              │
│  - Stateless workloads on spot (80% cheaper)         │
│  Savings: 50-80% on compute                          │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  3. Cluster autoscaling                              │
│  - Scale down dev clusters nights/weekends           │
│  - Karpenter consolidation (bin-packing)             │
│  Savings: 30-50% on non-prod                         │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  4. Storage optimization                             │
│  - Lifecycle policies (S3 Intelligent-Tiering)       │
│  - Delete orphaned EBS volumes, snapshots            │
│  Savings: 10-20%                                     │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│  5. Reserved Instances / Savings Plans               │
│  - 1-year commitment for stable workloads            │
│  Savings: 30-60%                                     │
└─────────────────────────────────────────────────────┘
```

**Karpenter (AWS) example:**

```yaml
# NodePool for spot instances
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot", "on-demand"]
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64"]
      nodeClassRef:
        name: default
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h   # 30 days
```

**Cost guardrails:**

```yaml
# ResourceQuota per namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-a-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "100"
    requests.memory: 200Gi
    limits.cpu: "200"
    limits.memory: 400Gi
```

---

## DevEx & Golden Paths

### 🔥 Q: What is developer experience (DevEx) and how to measure it?

**DevEx** = How easy it is for developers to build, test, deploy, and operate software.

**SPACE framework (2025-26 standard):**

| Dimension | What | How to Measure |
|-----------|------|---------------|
| **S**atisfaction | Happiness, well-being | Survey: "I can be productive", "I understand what to work on" |
| **P**erformance | Outcomes achieved | DORA metrics (deploy freq, lead time, MTTR) |
| **A**ctivity | Work done | PRs, commits (NOT a target, context only) |
| **C**ommunication | Info flow | Survey: "I get timely answers", "Docs are useful" |
| **E**fficiency | Complete work with minimal friction | Survey: "Tools don't slow me down", "Few interruptions" |

**DevEx survey questions:**
- "How easy is it to provision a new service?" (1-5)
- "How confident are you deploying to production?" (1-5)
- "How often do you wait on other teams?" (daily/weekly/monthly)
- "Would you recommend our platform to a friend?" (NPS)

**Tools:** Cortex, Jellyfish, LinearB, DX (getdx.com)

### 🔥 Q: What is a golden path (paved road)?

**Golden Path** = The blessed, well-maintained, documented way to build and deploy software at your org.

**Example: Golden path for a new microservice**

```
Step 1: Backstage template "Create Microservice"
  ├── Fill form (name, owner, language)
  └── Click "Create"

Step 2: Backstage scaffolds:
  ├── GitHub repo with code skeleton
  ├── CI/CD (GitHub Actions + ArgoCD Application)
  ├── Dockerfile (approved base image)
  ├── Kubernetes manifests (Helm chart)
  ├── Observability (Prometheus metrics, logs to Grafana)
  ├── Security (SAST, DAST, SBOM in pipeline)
  └── Docs (TechDocs auto-published)

Step 3: Developer:
  ├── git clone
  ├── make test (runs locally)
  ├── git commit + push
  └── First deploy to dev (auto, < 10 minutes)

Step 4: Production deploy:
  ├── PR to main → CI runs
  ├── Merge → ArgoCD syncs to production
  └── Argo Rollouts canary (10% → 50% → 100%)
```

**Off the golden path:**
- Developer wants Rust (not in golden path) → Manual setup, no CI/CD template, no support
- Trade-off: Flexibility vs. cognitive load

**Best practice:** Golden path for 80% of use cases, escape hatch for 20%.

### 💡 Q: Platform scorecards — measuring platform adoption.

**Scorecard example (Backstage plugin):**

```yaml
# Scorecard criteria
service: payment-api
checks:
  - name: Has CI/CD
    status: ✅ pass
  - name: Has production deploy
    status: ✅ pass
  - name: Has README
    status: ✅ pass
  - name: Has observability (Prometheus)
    status: ❌ fail
  - name: Has SBOM
    status: ⚠️ warn (manual SBOM, not automated)
  - name: Image signed
    status: ❌ fail
  - name: Resource limits set
    status: ✅ pass
  - name: On-call schedule
    status: ✅ pass

Score: 6/8 (75%)
```

**How to use:**
- Track org-wide compliance (80% of services passing)
- Gamify (leaderboard)
- Block production deploys if critical checks fail (e.g., no SBOM)

---

## Scenario-Based Questions

### 🔥 Q: Design an internal developer platform for 500 engineers.

**Architecture:**

```
┌─────────────────────────────────────────────────────┐
│  Backstage (Developer Portal)                        │
│  - Service catalog (500 services)                    │
│  - Software templates (Go, Python, Node, Java)       │
│  - TechDocs (auto-published Markdown)                │
│  - Search (unified search across catalog, docs)      │
└──────────┬──────────────────────────────────────────┘
           │
┌──────────▼──────────────────────────────────────────┐
│  Control Plane                                       │
│  - Crossplane (provision AWS resources)              │
│  - ArgoCD (GitOps for K8s manifests)                 │
│  - GitHub Actions (CI pipelines)                     │
│  - OPA/Kyverno (policy guardrails)                   │
│  - External Secrets Operator (secrets from Vault)    │
└──────────┬──────────────────────────────────────────┘
           │
┌──────────▼──────────────────────────────────────────┐
│  Infrastructure                                      │
│  - EKS clusters (prod, staging, dev)                 │
│  - RDS, S3, SQS (via Crossplane)                     │
│  - Istio (service mesh for mTLS, observability)      │
│  - Prometheus, Grafana, Jaeger (observability)       │
│  - Kubecost (cost management)                        │
└─────────────────────────────────────────────────────┘
```

**Golden path workflow:**

```
1. Developer: Backstage → "Create Microservice" template
2. Fill form: Service name, team, language (Go/Python/Node/Java)
3. Backstage:
   - Creates GitHub repo (from template)
   - Generates Dockerfile (approved base image)
   - Creates GitHub Actions workflow (build, test, scan, push)
   - Creates ArgoCD Application (GitOps for deploy)
   - Creates Crossplane Composition (if DB/S3/queue needed)
   - Adds service to Backstage catalog
4. Developer: git clone, code, git push
5. CI: Build → Test → SBOM → Sign → Push to ECR
6. ArgoCD: Syncs to dev cluster (auto)
7. Developer: PR to main → ArgoCD syncs to staging → Manual approval → ArgoCD syncs to prod
8. Argo Rollouts: Canary deploy (10% → 50% → 100%)
```

**Platform team OKRs:**
- 80% of services using golden path (6 months)
- Lead time to first deploy < 1 day (from template click)
- DevEx survey score > 4/5
- 50% reduction in infra tickets

### ⭐ Q: Set up GitOps for 50 microservices across 3 environments and multiple clusters.

**Repo structure:**

```
k8s-manifests/
├── apps/
│   ├── service-a/
│   │   ├── base/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   └── kustomization.yaml
│   │   └── overlays/
│   │       ├── dev/
│   │       │   └── kustomization.yaml (replicas: 1, image: latest)
│   │       ├── staging/
│   │       │   └── kustomization.yaml (replicas: 2, image: v1.2.3)
│   │       └── production/
│   │           └── kustomization.yaml (replicas: 5, image: v1.2.3)
│   ├── service-b/
│   └── ... (48 more)
└── infra/
    ├── nginx-ingress/
    ├── cert-manager/
    └── external-secrets/
```

**ArgoCD setup:**

```yaml
# ApplicationSet for all apps across all clusters
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: microservices
  namespace: argocd
spec:
  generators:
  - matrix:
      generators:
      - git:
          repoURL: https://github.com/org/k8s-manifests.git
          revision: main
          directories:
          - path: apps/*
      - clusters:
          selector:
            matchLabels:
              env: '{{path.basename}}'
  template:
    metadata:
      name: '{{path.basename}}-{{name}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/org/k8s-manifests.git
        targetRevision: main
        path: '{{path}}/overlays/{{values.env}}'
      destination:
        server: '{{server}}'
        namespace: default
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

**Cluster registration:**

```bash
# Register clusters in ArgoCD
argocd cluster add dev-cluster --name dev --label env=dev
argocd cluster add staging-cluster --name staging --label env=staging
argocd cluster add prod-us-east-1 --name prod-us-east-1 --label env=production
argocd cluster add prod-eu-west-1 --name prod-eu-west-1 --label env=production
```

**Promotion workflow:**

```
1. Developer merges to main
2. CI builds image → tags as v1.2.3 → pushes to ECR
3. Developer updates apps/service-a/overlays/staging/kustomization.yaml:
   images:
   - name: service-a
     newTag: v1.2.3
4. ArgoCD syncs to staging clusters (auto)
5. Staging tests pass
6. Developer updates apps/service-a/overlays/production/kustomization.yaml:
   images:
   - name: service-a
     newTag: v1.2.3
7. PR reviewed + approved
8. ArgoCD syncs to production clusters (auto, with Argo Rollouts canary)
```

### ⭐ Q: Roll out a change safely with automated canary analysis.

**Scenario:** Deploy payment-api v2 with new payment provider integration.

**Setup:**

```yaml
# 1. Argo Rollout with analysis
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: payment-api
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 10           # 10% traffic
      - pause: {duration: 5m}
      - analysis:
          templates:
          - templateName: payment-success-rate
          - templateName: payment-latency
      - setWeight: 50
      - pause: {duration: 10m}
      - analysis:
          templates:
          - templateName: payment-success-rate
          - templateName: payment-latency
      - setWeight: 100
      canaryService: payment-api-canary
      stableService: payment-api-stable
      trafficRouting:
        istio:
          virtualService:
            name: payment-api-vs
  template:
    spec:
      containers:
      - name: payment-api
        image: payment-api:v2
---
# 2. AnalysisTemplate — success rate
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: payment-success-rate
spec:
  metrics:
  - name: success-rate
    interval: 1m
    count: 5
    successCondition: result >= 0.99
    failureLimit: 2
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(payment_requests_total{status="success"}[5m]))
          /
          sum(rate(payment_requests_total[5m]))
---
# 3. AnalysisTemplate — latency
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: payment-latency
spec:
  metrics:
  - name: p95-latency
    interval: 1m
    count: 5
    successCondition: result < 500   # ms
    failureLimit: 2
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          histogram_quantile(0.95,
            sum(rate(payment_request_duration_bucket[5m])) by (le)
          )
```

**Rollout timeline:**

```
T+0:  Deploy v2 (10% traffic, 90% stable)
T+5:  Analysis runs (5 checks over 5 minutes)
      Success rate: 99.2% ✅
      P95 latency: 420ms ✅
T+10: Increase to 50% traffic
T+15: Analysis runs
      Success rate: 98.8% ❌ (below 99%)
      ROLLBACK triggered automatically
T+16: Traffic shifted back to 100% stable (v1)
T+17: Alert sent to #payments-team Slack
```

**Rollback triggers:**
- Success rate < 99% (2 failures in 5 checks)
- P95 latency > 500ms (2 failures)
- Manual abort (kubectl argo rollouts abort payment-api)

### 🔥 Q: Give app teams self-service infra without cloud god-mode.

**Problem:** App teams need databases, S3 buckets, queues — but can't have AWS admin access (security risk).

**Solution: Crossplane Compositions with RBAC**

```yaml
# 1. Define abstract "Database" resource (app teams use this)
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.platform.example.com
spec:
  group: platform.example.com
  names:
    kind: XDatabase
    plural: xdatabases
  claimNames:
    kind: Database
    plural: databases
  versions:
  - name: v1alpha1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              size:
                type: string
                enum: [small, medium, large]
              engine:
                type: string
                enum: [postgres, mysql]
            required: [size, engine]
---
# 2. Composition — maps XDatabase → AWS RDS (platform team owns)
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xdatabase-aws
spec:
  compositeTypeRef:
    apiVersion: platform.example.com/v1alpha1
    kind: XDatabase
  resources:
  - name: rds
    base:
      apiVersion: database.aws.crossplane.io/v1beta1
      kind: RDSInstance
      spec:
        forProvider:
          engine: postgres
          engineVersion: "15.3"
          instanceClass: db.t3.medium
          allocatedStorage: 100
          region: us-east-1
          # Platform team controls: region, VPC, security groups
    patches:
    - fromFieldPath: spec.engine
      toFieldPath: spec.forProvider.engine
    - fromFieldPath: spec.size
      toFieldPath: spec.forProvider.instanceClass
      transforms:
      - type: map
        map:
          small: db.t3.small
          medium: db.t3.medium
          large: db.r5.xlarge
---
# 3. RBAC — app teams can create Database claims (not raw RDS)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: app-team-database
rules:
- apiGroups: ["platform.example.com"]
  resources: ["databases"]
  verbs: ["get", "list", "create", "update", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-database
  namespace: team-a
subjects:
- kind: Group
  name: team-a
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: app-team-database
  apiGroup: rbac.authorization.k8s.io
```

**App team usage:**

```yaml
# App team creates database (no AWS access needed)
apiVersion: platform.example.com/v1alpha1
kind: Database
metadata:
  name: my-database
  namespace: team-a
spec:
  size: medium
  engine: postgres
```

**Platform team controls:**
- VPC, security groups, backup schedule, region
- Instance classes (no db.r6g.16xlarge — too expensive)
- Policies (all DBs encrypted, backups enabled)

**Guardrails via Kyverno:**

```yaml
# Enforce DB size limits
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: restrict-db-size
spec:
  validationFailureAction: enforce
  rules:
  - name: no-large-dbs-in-dev
    match:
      resources:
        kinds:
        - Database
        namespaces:
        - dev-*
    validate:
      message: "Dev namespaces can only create small databases"
      pattern:
        spec:
          size: small
```

---

## Key Resources

- **Team Topologies (book)** — Matthew Skelton & Manuel Pais (foundational for platform teams)
- **platformengineering.org** — Community site, conference
- **internaldeveloperplatform.org** — IDP patterns, case studies
- **CNCF Platforms White Paper** — https://tag-app-delivery.cncf.io/whitepapers/platforms/
- **Backstage** — https://backstage.io
- **ArgoCD** — https://argo-cd.readthedocs.io
- **Flux** — https://fluxcd.io
- **Crossplane** — https://www.crossplane.io
- **Argo Rollouts** — https://argoproj.github.io/argo-rollouts/
- **Flagger** — https://flagger.app
- **SPACE DevEx Framework** — https://queue.acm.org/detail.cfm?id=3454124
- **Kubecost** — https://www.kubecost.com
- **Kyverno** — https://kyverno.io
