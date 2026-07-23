---
title: Security (DevSecOps)
nav_order: 12
description: "Security (DevSecOps) — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# DevSecOps & Security — DevOps Interview Preparation

> **Question frequency:** 🔥 very frequently asked · ⭐ important · 💡 good to know

## Table of Contents
- [Security Fundamentals](#security-fundamentals)
- [Secrets Management](#secrets-management)
- [Container Security](#container-security)
- [Kubernetes Security](#kubernetes-security)
- [Network Security](#network-security)
- [Supply Chain Security](#supply-chain-security)
- [CI/CD Security](#cicd-security)
- [Compliance & Auditing](#compliance--auditing)
- [Encryption](#encryption)
- [Identity & Access Management](#identity--access-management)
- [Incident Response](#incident-response)
- [Trending Topics (2025-2026)](#trending-topics-2025-2026)
- [Scenario-Based Questions](#scenario-based-questions)

---

## Security Fundamentals

### 🔥 Q: What is the CIA Triad?

| Principle | Meaning | DevOps Example |
|-----------|---------|----------------|
| **Confidentiality** | Only authorized access | Encryption, RBAC, secrets management |
| **Integrity** | Data is not tampered | Checksums, signing, audit logs |
| **Availability** | System is accessible | HA, DDoS protection, backups |

### 🔥 Q: What is Shift-Left Security?

Moving security practices **earlier** in the development lifecycle:

```
Traditional:  Code → Build → Test → Deploy → [Security Audit] → Production
Shift-Left:   [Security] → Code → [Scan] → Build → [Scan] → Test → Deploy → Monitor
```

| Stage | Security Practice | Tools |
|-------|------------------|-------|
| **Code** | SAST, secret detection, linting | SonarQube, Semgrep, GitLeaks |
| **Dependencies** | SCA (vulnerability scanning) | Snyk, Dependabot, Renovate |
| **Build** | Image scanning, SBOM | Trivy, Grype, Syft |
| **Config** | IaC scanning | tfsec, Checkov, kube-score |
| **Deploy** | Admission controllers, policies | OPA/Gatekeeper, Kyverno |
| **Runtime** | DAST, runtime protection | Falco, OWASP ZAP, Sysdig |

### 🔥 Q: Principle of Least Privilege.

Grant the **minimum permissions** necessary for a task:

```
Bad:  IAM policy → Action: "*", Resource: "*"
Good: IAM policy → Action: "s3:GetObject", Resource: "arn:aws:s3:::my-bucket/*"

Bad:  Container runs as root
Good: Container runs as non-root user with specific capabilities

Bad:  Service account has cluster-admin
Good: Service account has namespace-scoped role for specific resources
```

---

## Secrets Management

### 🔥 Q: How should secrets be managed in DevOps?

**Never do:**
- Hardcode secrets in source code
- Store secrets in environment variables in Dockerfiles
- Commit `.env` files to Git
- Use default passwords
- Share secrets via Slack/email

**Secret management hierarchy (best to worst):**

```
1. External Secret Manager (Vault, AWS SM)    ← BEST
   App fetches secrets at runtime via API
   Auto-rotation, audit trail, fine-grained access

2. Kubernetes External Secrets Operator
   Syncs secrets from Vault/AWS SM into K8s Secrets
   Encrypted at rest in etcd

3. Sealed Secrets (for GitOps)
   Encrypted in Git, decrypted only in cluster
   kubeseal encrypts with cluster's public key

4. Kubernetes Secrets (base64, NOT encrypted by default)
   Must enable EncryptionConfiguration for etcd

5. Environment variables
   Visible in process list, docker inspect
   ← WORST (but better than hardcoded)
```

### ⭐ Q: HashiCorp Vault basics.

```
┌─── Vault Server ─────────────────────────────┐
│                                                │
│  ┌── Secret Engines ──┐  ┌── Auth Methods ──┐ │
│  │ KV (key-value)     │  │ Kubernetes       │ │
│  │ Database (dynamic) │  │ AWS IAM          │ │
│  │ PKI (certificates) │  │ OIDC/JWT         │ │
│  │ Transit (encrypt)  │  │ AppRole          │ │
│  └────────────────────┘  └──────────────────┘ │
│                                                │
│  ┌── Policies ────────┐  ┌── Audit Log ─────┐ │
│  │ path "secret/*" {  │  │ Every operation   │ │
│  │   capabilities =   │  │ is logged         │ │
│  │   ["read","list"]  │  │                   │ │
│  │ }                  │  │                   │ │
│  └────────────────────┘  └──────────────────┘ │
└────────────────────────────────────────────────┘
```

**Key concepts:**
- **Dynamic secrets** — Generated on-demand, auto-expire (DB credentials, AWS keys)
- **Lease & TTL** — Secrets have expiration, must be renewed
- **Auto-rotation** — Vault rotates secrets automatically
- **Seal/Unseal** — Vault starts sealed, needs threshold of key shares to unseal

```bash
# Basic Vault operations
vault kv put secret/myapp db_password=s3cret
vault kv get secret/myapp
vault kv get -field=db_password secret/myapp

# Dynamic database secret
vault read database/creds/my-role
# Returns: username=v-token-my-role-xxx, password=A1B2C3, lease_duration=1h
```

### ⭐ Q: Secret rotation strategies.

**Why rotate?** Limit blast radius if compromised, comply with policies, prevent long-lived credential abuse.

| Rotation Type | How | Tools |
|--------------|-----|-------|
| **Manual** | Change secret, update systems | Runbook-driven (error-prone) |
| **Scheduled** | Cron/Lambda rotates at interval | AWS Secrets Manager auto-rotation |
| **Event-driven** | Rotate on detection of compromise | Vault lease revocation, IAM key disable |
| **Zero-standing** | No long-lived secrets, generate on-demand | Vault dynamic secrets, IAM role assumption |

**Best practice:** Vault dynamic secrets (no rotation needed, TTL expires) > automated rotation > manual.

```bash
# AWS Secrets Manager automatic rotation
aws secretsmanager rotate-secret --secret-id prod/db/password \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456:function:rotate-db

# External Secrets Operator (ESO) — sync from Vault to K8s
kubectl apply -f - <<EOF
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-secret
  data:
  - secretKey: password
    remoteRef:
      key: secret/data/db
      property: password
EOF
```

---

## Container Security

### 🔥 Q: Container security checklist.

```
1. Image Security:
   □ Use minimal base images (Alpine, Distroless)
   □ Scan for CVEs (Trivy, Snyk, Grype)
   □ Pin image tags (no :latest)
   □ Sign images (Cosign, Docker Content Trust)
   □ Use private registries
   □ SBOM generation (Syft, Docker Scout)

2. Build Security:
   □ Multi-stage builds (no build tools in prod)
   □ No secrets in images (use build secrets with --mount=type=secret)
   □ No root user (USER directive)
   □ Read-only root filesystem where possible
   □ Lint Dockerfiles (Hadolint)

3. Runtime Security:
   □ Drop all capabilities, add specific ones (CAP_NET_BIND_SERVICE, etc.)
   □ No privileged containers
   □ Resource limits (CPU, memory, PIDs)
   □ Seccomp profiles (restrict syscalls)
   □ AppArmor/SELinux profiles
   □ Read-only root filesystem + tmpfs for writable paths
   □ No host namespace sharing (--pid=host, --network=host, --ipc=host)
   □ Run as non-root user with explicit UID

4. Supply Chain:
   □ SBOM generation (Syft)
   □ Image provenance (SLSA)
   □ Vulnerability scanning in CI/CD
   □ Admission policies (only signed images)
   □ Dependency pinning and hash verification
```

### ⭐ Q: What is Distroless and why use it?

**Distroless** images contain only the application and its runtime dependencies — no shell, no package manager, no OS utilities.

| Image | Size | Shell | Package Mgr | Attack Surface |
|-------|------|-------|-------------|----------------|
| ubuntu:22.04 | 77MB | Yes | Yes | Large |
| alpine:3.19 | 7MB | Yes | Yes | Small |
| distroless/static | 2MB | No | No | Minimal |
| scratch | 0MB | No | No | Zero (just your binary) |

**Trade-off:** Can't exec into the container for debugging. Use debug variants or ephemeral containers.

```dockerfile
# Multi-stage build with distroless
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 go build -o myapp

FROM gcr.io/distroless/static-debian12:nonroot
COPY --from=builder /app/myapp /myapp
USER 65532:65532
ENTRYPOINT ["/myapp"]
```

### ⭐ Q: Explain Linux capabilities and why drop them.

**Linux capabilities** break root's all-or-nothing power into fine-grained privileges.

```
Default Docker container runs with ~15 capabilities:
CAP_CHOWN, CAP_DAC_OVERRIDE, CAP_FOWNER, CAP_SETGID, CAP_SETUID,
CAP_NET_BIND_SERVICE, CAP_KILL, CAP_AUDIT_WRITE, etc.

Privileged container: ALL 40+ capabilities → full root power
```

**Security best practice:** Drop all, add only what's needed.

```yaml
# Kubernetes example
securityContext:
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE  # Only if binding to ports <1024
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
```

```bash
# Docker example
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE \
  --read-only --user 1000:1000 \
  --security-opt=no-new-privileges:true myapp:v1
```

### 💡 Q: Seccomp and AppArmor profiles.

**Seccomp (Secure Computing Mode):** Restrict syscalls a container can make.

```
Default Docker: ~300 syscalls allowed
Custom profile: Allow only the 40-50 your app actually needs
RuntimeDefault (K8s): Reasonable baseline, blocks dangerous syscalls
```

**AppArmor:** Mandatory Access Control — restrict file access, network, capabilities.

```yaml
# Kubernetes pod with seccomp + AppArmor
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: runtime/default
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:v1
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: [ALL]
      readOnlyRootFilesystem: true
      runAsNonRoot: true
```

---

## Kubernetes Security

### 🔥 Q: Kubernetes security layers.

```
Layer 1: Cluster Infrastructure
├── Private API server endpoint
├── Node OS hardening (CIS benchmarks)
├── etcd encryption at rest
└── Network segmentation

Layer 2: Authentication & Authorization
├── OIDC/SSO integration (no static tokens)
├── RBAC (least privilege)
├── Namespace isolation
└── Service account management

Layer 3: Admission Control
├── Pod Security Standards (restricted)
├── OPA Gatekeeper / Kyverno policies
├── Image policy (signed, from approved registries)
└── Resource quotas and limit ranges

Layer 4: Pod Security
├── Security contexts (non-root, read-only, no privilege escalation)
├── Network policies (default deny)
├── Service mesh (mTLS between pods)
└── Seccomp/AppArmor profiles

Layer 5: Runtime Security
├── Falco (syscall monitoring)
├── Container runtime security
├── Audit logging
└── Vulnerability scanning
```

### ⭐ Q: Pod Security Standards (PSS) — replacement for PodSecurityPolicy.

**Pod Security Admission** replaced deprecated PodSecurityPolicy in K8s 1.25.

| Level | Use Case | Restrictions |
|-------|----------|-------------|
| **Privileged** | Unrestricted, no enforcement | None |
| **Baseline** | Minimally restrictive, prevents known escalations | No hostNetwork/hostPID/hostIPC, no privileged, no hostPath |
| **Restricted** | Hardened, defense-in-depth | Baseline + must run as non-root, drop ALL caps, no privilege escalation, seccomp RuntimeDefault |

```yaml
# Enforce at namespace level
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

```bash
# Cluster-wide default via admission config
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: "baseline"
      audit: "restricted"
      warn: "restricted"
```

### ⭐ Q: What is OPA Gatekeeper vs Kyverno?

Both are Kubernetes admission controllers for policy enforcement.

| Feature | OPA Gatekeeper | Kyverno |
|---------|---------------|---------|
| **Language** | Rego (declarative policy language) | YAML (Kubernetes-native) |
| **Learning Curve** | Steeper (learn Rego) | Easier (just YAML) |
| **Flexibility** | Very flexible, any logic | Good for common K8s policies |
| **Mutation** | Yes (via Mutation webhooks) | Yes (built-in) |
| **Generation** | No | Yes (generate NetworkPolicy, etc.) |
| **Use Case** | Complex policies, cross-resource validation | Quick K8s-native policies |

**Gatekeeper example:**

```yaml
# ConstraintTemplate — Define the policy
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items: { type: string }
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8srequiredlabels
      violation[{"msg": msg}] {
        provided := {label | input.review.object.metadata.labels[label]}
        required := {label | label := input.parameters.labels[_]}
        missing := required - provided
        count(missing) > 0
        msg := sprintf("Missing required labels: %v", [missing])
      }

---
# Constraint — Apply the policy
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Namespace"]
  parameters:
    labels: ["team", "environment"]
```

**Kyverno example:**

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-team-label
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-namespace-labels
    match:
      resources:
        kinds:
        - Namespace
    validate:
      message: "Namespaces must have 'team' and 'environment' labels"
      pattern:
        metadata:
          labels:
            team: "?*"
            environment: "?*"
```

### 🔥 Q: Kubernetes Network Policies.

**Default:** All pods can talk to all pods (flat network).
**Best practice:** Default deny, explicit allow.

```yaml
# 1. Default deny all ingress in namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress

---
# 2. Allow specific ingress to app pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080

---
# 3. Allow egress only to DNS and specific services
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53  # DNS
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

### ⭐ Q: Runtime security with Falco.

**Falco** monitors syscalls and Kubernetes audit logs for suspicious behavior.

```
Detection examples:
- Container spawns shell (unexpected)
- Process reads sensitive files (/etc/shadow)
- Container runs as root
- Privileged container launched
- Network connection to known bad IP
- Crypto mining activity
```

```yaml
# Falco rule example
- rule: Unexpected Shell Spawned in Container
  desc: Detect shell spawned in container (potential breakin)
  condition: >
    spawned_process and
    container and
    proc.name in (shell_binaries) and
    not proc.pname in (known_parent_processes)
  output: >
    Shell spawned in container
    (user=%user.name container_id=%container.id image=%container.image.repository
    proc=%proc.cmdline)
  priority: WARNING

- rule: Read Sensitive File
  desc: Detect reads of sensitive files
  condition: >
    open_read and
    fd.name in (/etc/shadow, /etc/sudoers, /root/.ssh/id_rsa)
  output: >
    Sensitive file read (user=%user.name file=%fd.name proc=%proc.cmdline)
  priority: CRITICAL
```

```bash
# Deploy Falco via Helm
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm install falco falcosecurity/falco \
  --namespace falco-system --create-namespace \
  --set driver.kind=ebpf
```

---

## Network Security

### ⭐ Q: mTLS (Mutual TLS) explained.

**Regular TLS:** Client verifies server identity.
**mTLS:** Both client AND server verify each other's identity.

```
Standard TLS:
Client ──► Server shows certificate ──► Client verifies ──► Encrypted channel

mTLS:
Client ──► Server shows certificate ──► Client verifies
Server ──► Client shows certificate ──► Server verifies
       ──► Both authenticated ──► Encrypted channel
```

**In Kubernetes:** Service meshes (Istio, Linkerd) implement mTLS between all pods automatically — zero-trust networking.

### ⭐ Q: DDoS protection strategies.

| Layer | Attack Type | Mitigation |
|-------|------------|------------|
| **L3/L4** | SYN flood, UDP flood | AWS Shield, rate limiting at NLB |
| **L7** | HTTP flood, slowloris | WAF rules, rate limiting, CAPTCHA |
| **DNS** | DNS amplification | Route 53 / CloudFlare |
| **Application** | API abuse | Rate limiting, auth, API gateway |

**Defense in depth:**
```
CloudFlare/CloudFront (CDN + DDoS absorption)
    → WAF (L7 filtering)
        → ALB (health checks, connection limits)
            → Auto Scaling (absorb legitimate traffic)
                → Rate limiting (per-user/per-IP)
```

---

## Supply Chain Security

### 🔥 Q: What is software supply chain security?

Ensuring every component in your software delivery pipeline is trusted — from source to production.

**Attack vectors:**
- Compromised dependencies (typosquatting, dependency confusion)
- Malicious commits (SolarWinds-style)
- Build system compromise
- Unsigned/unverified artifacts
- Vulnerable base images

**SLSA Framework (Supply-chain Levels for Software Artifacts):**

| Level | Requirements |
|-------|-------------|
| **SLSA 1** | Build process exists and produces provenance |
| **SLSA 2** | Version-controlled, authenticated build service, signed provenance |
| **SLSA 3** | Hardened build platform, non-falsifiable provenance, isolated build |
| **SLSA 4** | Two-person review, hermetic/reproducible builds, all deps verified |

**Key practices:**
- **SBOM** (Software Bill of Materials) — Know what's in your images
- **Image signing** — Cosign, Notary, Docker Content Trust
- **Dependency pinning** — Lock files, hash verification
- **Provenance attestation** — Prove where artifacts came from
- **Admission policies** — Only allow signed images from trusted registries
- **Dependency scanning** — Detect known vulnerabilities (Dependabot, Renovate, Snyk)

```bash
# Sign an image with Cosign
cosign sign --key cosign.key myregistry/myapp:v1.0

# Keyless signing with Sigstore (OIDC-based, no key management)
cosign sign myregistry/myapp:v1.0
# Uses Fulcio (ephemeral cert) + Rekor (transparency log)

# Verify signature
cosign verify --key cosign.pub myregistry/myapp:v1.0

# Verify keyless signature
cosign verify myregistry/myapp:v1.0 \
  --certificate-identity user@example.com \
  --certificate-oidc-issuer https://accounts.google.com

# Generate SBOM
syft myregistry/myapp:v1.0 -o spdx-json > sbom.json

# Scan SBOM for vulnerabilities
grype sbom:sbom.json

# Attach SBOM as attestation
cosign attest --predicate sbom.json --key cosign.key myregistry/myapp:v1.0

# Generate SLSA provenance
cosign attest --predicate provenance.json --key cosign.key myregistry/myapp:v1.0
```

### 💡 Q: Sigstore and keyless signing.

**Problem:** Key management is hard — keys can leak, expire, need rotation.

**Sigstore solution:** Keyless signing via short-lived certificates + transparency log.

```
Flow:
1. Developer authenticates via OIDC (GitHub, Google, etc.)
2. Fulcio issues short-lived certificate (valid 10 minutes)
3. Developer signs artifact with ephemeral key
4. Signature + certificate recorded in Rekor (transparency log)
5. Verifier checks: signature valid + certificate in log + OIDC identity matches

No long-lived keys to manage!
```

**Components:**
- **Fulcio:** Certificate Authority issuing short-lived certs
- **Rekor:** Transparency log (tamper-proof audit trail)
- **Cosign:** CLI tool for signing and verification

```bash
# Sign with keyless (uses your OIDC identity)
cosign sign myregistry/myapp:v1.0
# Browser opens for GitHub auth, cert issued, signature recorded

# Verify
cosign verify myregistry/myapp:v1.0 \
  --certificate-identity github-actions[bot]@users.noreply.github.com \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

### ⭐ Q: Dependency confusion and typosquatting.

**Dependency Confusion:** Internal package name collides with public package name.

```
Your app: import "mycompany-utils"
Attacker: Publishes "mycompany-utils" to public npm/PyPI with higher version
Your build: Installs attacker's package instead of internal one
```

**Defense:**
- Use private registries with authentication
- Configure package manager to prefer private registry
- Namespace internal packages (`@mycompany/utils`)
- Block public registry access from CI

**Typosquatting:** Attacker registers package with similar name.

```
Legit:   requests (Python)
Attacker: reqeusts, request, python-requests
```

**Defense:**
- Dependency pinning with hash verification (lock files)
- Automated dependency scanning (Dependabot alerts)
- Code review of dependency changes
- Use curated/approved package lists

```json
// npm .npmrc — prefer private registry
@mycompany:registry=https://npm.internal.com
registry=https://npm.internal.com
```

```python
# pip.conf — private registry only
[global]
index-url = https://pypi.internal.com/simple
trusted-host = pypi.internal.com
```

### ⭐ Q: SBOM mandates and compliance.

**Why SBOM matters:**
- U.S. Executive Order 14028 (2021) requires SBOMs for federal software
- Identify vulnerable components quickly (Log4Shell response)
- License compliance
- Supply chain transparency

**SBOM formats:**
- **SPDX** (Software Package Data Exchange) — Linux Foundation standard
- **CycloneDX** — OWASP standard
- **SWID** (Software Identification Tags) — ISO standard

```bash
# Generate SBOM with Syft
syft myapp:v1.0 -o spdx-json=sbom.spdx.json
syft myapp:v1.0 -o cyclonedx-json=sbom.cdx.json

# Scan SBOM
grype sbom:sbom.spdx.json

# Attach SBOM to image (Sigstore attestation)
cosign attest --predicate sbom.spdx.json --type spdx myapp:v1.0

# Query SBOM
cat sbom.spdx.json | jq '.packages[] | select(.name=="log4j")'
```

---

## CI/CD Security

### 🔥 Q: Security scanning in CI/CD pipeline.

**Multi-layer scanning approach:**

```
┌─── CI/CD Pipeline Security ────────────────────────┐
│                                                     │
│  1. Pre-commit                                      │
│     - Git hooks: GitLeaks, detect-secrets          │
│     - Local SAST: SonarLint, ESLint security       │
│                                                     │
│  2. PR / Commit                                     │
│     - SAST: SonarQube, Semgrep, CodeQL             │
│     - Secret scanning: TruffleHog, GitGuardian     │
│     - Dependency scan: Snyk, Dependabot            │
│     - IaC scanning: tfsec, Checkov, kube-score     │
│                                                     │
│  3. Build                                           │
│     - Container scan: Trivy, Grype, Snyk           │
│     - SBOM generation: Syft                        │
│     - Image signing: Cosign                        │
│     - License compliance: FOSSA                    │
│                                                     │
│  4. Pre-deploy                                      │
│     - DAST: OWASP ZAP, Burp                        │
│     - Policy enforcement: OPA, Kyverno             │
│     - Signature verification                       │
│                                                     │
│  5. Runtime                                         │
│     - Vulnerability monitoring: Falco, Sysdig      │
│     - RASP: Contrast Security, Signal Sciences     │
│     - Continuous scanning: Aqua, Prisma Cloud      │
└─────────────────────────────────────────────────────┘
```

**SAST vs DAST vs SCA:**

| Type | What | When | Tools |
|------|------|------|-------|
| **SAST** | Static code analysis | Build time | SonarQube, Semgrep, CodeQL, Checkmarx |
| **DAST** | Dynamic runtime testing | After deploy (staging) | OWASP ZAP, Burp Suite, Acunetix |
| **SCA** | Software Composition Analysis (deps) | Build time | Snyk, Dependabot, WhiteSource, BlackDuck |
| **IAST** | Interactive (instrumented runtime) | Test/staging | Contrast Security, Seeker |
| **RASP** | Runtime protection | Production | Contrast Protect, Signal Sciences |

### 🔥 Q: Secure CI/CD best practices.

```
Pipeline Security Checklist:

1. Pipeline as Code
   - Version-controlled (Git)
   - Code-reviewed (PR approval)
   - Immutable (tagged versions)

2. Secrets Management
   - Never hardcode secrets in pipeline config
   - Use secret managers (Vault, AWS Secrets Manager)
   - Mask secrets in logs
   - Rotate CI/CD service account credentials

3. Access Control
   - Least privilege for CI/CD service accounts
   - MFA for pipeline modifications
   - Audit logs for all pipeline runs
   - Branch protection (no direct commits to main)

4. Build Environment
   - Ephemeral build agents (destroy after use)
   - Isolated build containers
   - No persistent state between builds
   - Verified base images for build agents

5. Artifact Integrity
   - Sign all artifacts (containers, binaries, packages)
   - Generate and attach SBOMs
   - Store in private registries with scanning
   - Verify signatures before deploy

6. Supply Chain
   - Dependency pinning with hash verification
   - Private mirrors for public dependencies
   - Two-person approval for dependency updates
   - SLSA provenance attestation

7. Deployment Controls
   - Multi-stage (dev → staging → prod)
   - Approval gates for production
   - Rollback capability
   - Deployment windows (change freeze)
```

### ⭐ Q: GitHub Actions security.

**Common vulnerabilities:**
- Secret leakage in logs
- Injection via untrusted input
- Malicious actions from marketplace
- Over-permissioned GITHUB_TOKEN

```yaml
# BAD: Command injection
- name: Echo PR title
  run: echo "${{ github.event.pull_request.title }}"
# If PR title is "; rm -rf /", it executes!

# GOOD: Use environment variables
- name: Echo PR title
  env:
    PR_TITLE: ${{ github.event.pull_request.title }}
  run: echo "$PR_TITLE"

# BAD: Default GITHUB_TOKEN has write permissions
permissions: write-all

# GOOD: Least privilege
permissions:
  contents: read
  pull-requests: write

# Pin actions to SHA (not tag, tags are mutable)
- uses: actions/checkout@8e5e7e5ab8b370d6c329ec480221332ada57f0ab  # v3.5.2
```

**Secret scanning:**

```yaml
- name: Secret scan
  uses: trufflesecurity/trufflehog@main
  with:
    path: ./
    base: ${{ github.event.repository.default_branch }}
    head: HEAD
```

---

## Encryption

### 🔥 Q: Encryption at rest vs in transit.

| Type | What | How | Examples |
|------|------|-----|---------|
| **At Rest** | Data stored on disk | AES-256, KMS | EBS encryption, S3 SSE, etcd encryption |
| **In Transit** | Data moving over network | TLS 1.3, mTLS | HTTPS, mTLS in service mesh |

### ⭐ Q: KMS (Key Management Service) concepts.

```
Envelope Encryption:
1. KMS generates a Data Encryption Key (DEK)
2. DEK encrypts your data
3. KMS master key (CMK) encrypts the DEK
4. Store: encrypted data + encrypted DEK
5. To decrypt: KMS decrypts DEK → DEK decrypts data

Why: You can encrypt large data without sending it to KMS.
     Only the small DEK goes to KMS.
```

---

## Compliance & Auditing

### ⭐ Q: Compliance frameworks (SOC 2, PCI-DSS, GDPR) — DevOps perspective.

| Framework | Focus | DevOps Responsibilities |
|-----------|-------|------------------------|
| **SOC 2** | Trust services (security, availability, confidentiality) | Audit logs, access controls, incident response, change management |
| **PCI-DSS** | Credit card data | Encrypt data at rest/transit, network segmentation, vulnerability scanning, access logs |
| **GDPR** | Personal data privacy (EU) | Data encryption, right to erasure, data portability, breach notification |
| **HIPAA** | Healthcare data (US) | Encryption, audit logs, access controls, BAA agreements |
| **FedRAMP** | US federal cloud services | NIST 800-53 controls, continuous monitoring, FIPS 140-2 crypto |

**Common DevOps controls:**
- Audit logging (who did what, when)
- Access control (RBAC, least privilege)
- Encryption (at rest, in transit)
- Backup and disaster recovery
- Vulnerability management
- Incident response plan
- Change management (PR approval, audit trail)

```bash
# Enable Kubernetes audit logging
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
- level: Metadata
  resources:
  - group: ""
    resources: ["secrets", "configmaps"]
- level: Request
  verbs: ["create", "update", "patch", "delete"]
```

### ⭐ Q: CIS Benchmarks.

**Center for Internet Security (CIS) Benchmarks** — consensus-based security configuration guides.

```
Available for:
- Kubernetes
- Docker
- Linux (Ubuntu, RHEL, etc.)
- Cloud platforms (AWS, GCP, Azure)
- Databases (PostgreSQL, MySQL, MongoDB)
```

**Example CIS Kubernetes recommendations:**
- 4.2.1: Minimize admission of privileged containers
- 4.2.6: Ensure containers run as non-root
- 5.1.3: Ensure RBAC is enabled
- 5.7.3: Apply Security Context to pods

```bash
# Scan Kubernetes cluster against CIS benchmark
kube-bench run --targets=node,master

# Docker CIS benchmark scan
docker run --rm --net host --pid host --userns host --cap-add audit_control \
  -v /etc:/etc:ro -v /var/lib:/var/lib:ro -v /var/run/docker.sock:/var/run/docker.sock:ro \
  aquasec/docker-bench-security
```

---

## Identity & Access Management

### 🔥 Q: Authentication vs Authorization.

| Concept | Question | K8s Implementation |
|---------|----------|-------------------|
| **Authentication** | Who are you? | Certificates, OIDC, tokens |
| **Authorization** | What can you do? | RBAC, ABAC, webhook |
| **Admission** | Is this allowed? | Admission controllers, OPA |

### 🔥 Q: RBAC (Role-Based Access Control) in Kubernetes.

```
Four core resources:
1. Role / ClusterRole — Define permissions
2. RoleBinding / ClusterRoleBinding — Grant permissions to subjects

Scope:
- Role / RoleBinding: Namespace-scoped
- ClusterRole / ClusterRoleBinding: Cluster-scoped
```

```yaml
# 1. Role — read-only access to pods in "production" namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

---
# 2. RoleBinding — grant "pod-reader" to user "jane"
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jane-read-pods
  namespace: production
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# 3. ClusterRole — view nodes (cluster-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-viewer
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list"]

---
# 4. ClusterRoleBinding — grant to service account
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-view-nodes
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: node-viewer
  apiGroup: rbac.authorization.k8s.io
```

**Best practices:**
- Grant least privilege (only the verbs/resources needed)
- Use namespace-scoped Roles when possible
- Avoid `*` in verbs or resources
- Don't grant `cluster-admin` to service accounts
- Audit RBAC with `kubectl auth can-i`

```bash
# Check if current user can delete pods
kubectl auth can-i delete pods --namespace production

# Check if service account can create deployments
kubectl auth can-i create deployments --as=system:serviceaccount:production:myapp
```

### 🔥 Q: IAM for cloud platforms (AWS, GCP, Azure).

**Principle:** Identity-based access, not IP-based.

| Platform | Service | Key Concepts |
|---------|---------|--------------|
| **AWS** | IAM | Users, Groups, Roles, Policies (JSON), AssumeRole, MFA |
| **GCP** | Cloud IAM | Service accounts, Roles (primitive/predefined/custom), IAM policies, Workload Identity |
| **Azure** | Azure AD | Managed identities, Role assignments, RBAC, Conditional Access |

**Best practices:**
- Use roles/service accounts, not user credentials in apps
- Enable MFA for human users
- Rotate access keys regularly (or use short-lived tokens)
- Audit with CloudTrail/Cloud Audit Logs/Activity Log
- Use managed identities (no keys in code)

```json
// AWS IAM policy — least privilege
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "s3:GetObject",
      "s3:PutObject"
    ],
    "Resource": "arn:aws:s3:::my-app-bucket/*"
  }]
}
```

```bash
# GCP Workload Identity (K8s pod assumes GCP service account)
gcloud iam service-accounts add-iam-policy-binding \
  myapp@project.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:project.svc.id.goog[namespace/k8s-sa]"
```

### ⭐ Q: Zero Trust Architecture.

**Old model:** Trust everything inside the network perimeter (castle and moat).
**Zero Trust:** "Never trust, always verify" regardless of network location.

**Principles:**
1. Verify explicitly — Authenticate and authorize every request
2. Least privilege — Minimum access needed
3. Assume breach — Segment, encrypt, monitor everything

**Implementation:**
- Identity-based access (not IP-based)
- mTLS everywhere (service mesh)
- Microsegmentation (network policies, firewalls)
- Short-lived credentials (no permanent tokens)
- Continuous monitoring and anomaly detection
- Device trust (posture checks)

**Example: Zero Trust for microservices**

```
Traditional:
  - All services in same VPC → implicit trust
  - Any service can call any service

Zero Trust:
  - Every request authenticated (mTLS)
  - Authorization policy for each service-to-service call
  - Network policies (default deny)
  - Short-lived tokens (no static API keys)
  - Audit every request
```

---

## Incident Response

### 🔥 Q: Security incident response plan.

```
1. PREPARATION
   - Incident response runbooks
   - On-call rotation
   - Security monitoring (GuardDuty, Falco)
   - Backup verification

2. IDENTIFICATION
   - Detect via alerts, audit logs, IDS
   - Classify severity
   - Determine scope (which systems, data affected?)

3. CONTAINMENT
   - Short-term: Isolate affected systems (SG rules, network policies)
   - Long-term: Patch vulnerability, revoke compromised credentials
   - Preserve evidence (disk snapshots, logs)

4. ERADICATION
   - Remove malware/backdoors
   - Patch vulnerabilities
   - Rebuild compromised systems from known-good images

5. RECOVERY
   - Restore from clean backups
   - Monitor closely for recurrence
   - Gradually restore services

6. LESSONS LEARNED
   - Blameless post-mortem
   - Update runbooks
   - Improve detection
   - Security training
```

---

## Trending Topics (2025-2026)

### 💡 Q: eBPF for security monitoring.

**eBPF (Extended Berkeley Packet Filter):** Run sandboxed programs in the Linux kernel without changing kernel code.

**Security use cases:**
- Syscall monitoring (like Falco)
- Network traffic inspection
- Process execution tracking
- File access monitoring
- Zero-overhead observability

**Why trending:**
- Kernel-level visibility with minimal performance impact
- No kernel modules needed (safer than LKMs)
- Runtime security without sidecars

**Tools using eBPF:**
- Falco (runtime security)
- Cilium (networking + security)
- Tetragon (runtime enforcement)
- Tracee (threat detection)
- Pixie (observability)

```bash
# Deploy Tetragon (eBPF-based runtime enforcement)
helm install tetragon cilium/tetragon -n kube-system

# Policy: block execution of shells in containers
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: block-shell-exec
spec:
  kprobes:
  - call: "sys_execve"
    args:
    - index: 0
      type: "string"
    selectors:
    - matchBinaries:
      - operator: "In"
        values:
        - "/bin/sh"
        - "/bin/bash"
    matchActions:
    - action: Sigkill
```

### 🔥 Q: OWASP Top 10 awareness (2021 updated list).

| Rank | Category | Mitigation |
|------|----------|-----------|
| **A01** | Broken Access Control | RBAC, least privilege, server-side checks |
| **A02** | Cryptographic Failures | TLS 1.3, AES-256, proper key management |
| **A03** | Injection (SQL, command, etc.) | Parameterized queries, input validation, ORM |
| **A04** | Insecure Design | Threat modeling, secure SDLC |
| **A05** | Security Misconfiguration | Harden defaults, disable debug, patch regularly |
| **A06** | Vulnerable Components | SCA scanning, dependency updates, SBOM |
| **A07** | Identification/Auth Failures | MFA, rate limiting, secure session management |
| **A08** | Software/Data Integrity Failures | Code signing, SBOM, supply chain security |
| **A09** | Logging/Monitoring Failures | Centralized logging, alerting, SIEM |
| **A10** | Server-Side Request Forgery | Validate URLs, whitelist destinations, network segmentation |

**DevOps relevance:**
- A05 (Misconfiguration): Default passwords, open S3 buckets, debug mode in prod
- A06 (Vulnerable Components): Unpatched Log4j, outdated base images
- A08 (Integrity Failures): Unsigned containers, compromised build pipeline
- A09 (Logging Failures): No audit logs, no alerting on security events

### 💡 Q: AI/ML security considerations.

**Model Security:**
- Model poisoning (training data tampering)
- Adversarial attacks (crafted inputs to fool model)
- Model extraction (steal proprietary model via API)
- Prompt injection (LLM-specific)

**Infrastructure Security:**
- Secure model storage (S3 encryption, access control)
- API rate limiting and authentication
- Input validation and sanitization
- PII in training data (GDPR compliance)
- Model versioning and provenance

**DevOps practices:**
- Version control for models (DVC, MLflow)
- Sign models (Cosign)
- Scan model dependencies
- Isolate training environments
- Monitor inference for anomalies

---

## Scenario-Based Questions

### 🔥 Q: A Kubernetes pod was compromised. What do you do?

```
1. CONTAIN immediately:
   - Apply restrictive NetworkPolicy (deny all egress)
   - Don't delete the pod (preserve evidence)
   - Cordon the node: kubectl cordon <node>

2. INVESTIGATE:
   - kubectl logs <pod>
   - kubectl describe pod <pod>
   - Check audit logs (API server)
   - Check Falco alerts (syscall anomalies)
   - Exec into pod (if safe): check processes, network connections
   - Snapshot the node's disk for forensics

3. ASSESS blast radius:
   - What ServiceAccount did the pod use?
   - What secrets did it have access to?
   - What network access did it have?
   - Was there lateral movement?

4. ERADICATE:
   - Rotate all secrets the pod had access to
   - Patch the vulnerability (image, config)
   - Rebuild from clean image

5. PREVENT:
   - Enforce Pod Security Standards (restricted)
   - Network policies (default deny)
   - Limit ServiceAccount permissions
   - Runtime security (Falco)
   - Regular vulnerability scanning
```

### ⭐ Q: You found that Docker images are being pulled from an untrusted registry. Fix it.

```yaml
# 1. OPA Gatekeeper policy to only allow trusted registries
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: trusted-repos
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
  parameters:
    repos:
    - "mycompany.azurecr.io/"
    - "123456789.dkr.ecr.us-east-1.amazonaws.com/"
    - "gcr.io/mycompany/"
```

```bash
# 2. Enable image signature verification (Cosign + policy controller)
# 3. Use ImagePullSecrets to require authentication
# 4. Audit: find all pods using untrusted images
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}: {.spec.containers[*].image}{"\n"}{end}'
```

### 🔥 Q: Secret leaked to a public GitHub repo. Respond.

```
IMMEDIATE ACTIONS (first 15 minutes):
1. REVOKE the secret immediately
   - Rotate credentials
   - Disable API keys
   - Invalidate tokens
   
2. ASSESS blast radius
   - What systems did this secret access?
   - Check audit logs for unauthorized usage
   - Were there any anomalous API calls?
   - Was the secret actually accessed by bad actors?
   
3. CONTAIN
   - If compromise detected, isolate affected systems
   - Enable additional alerting
   - Rotate all related credentials (not just the leaked one)

SHORT-TERM (next few hours):
4. INVESTIGATE
   - When was it committed? (git log)
   - Who committed it? (intent vs mistake)
   - Which commits contain it? (all branches/tags)
   - Was repo public when secret added, or made public later?
   - GitHub secret scanning alerts (check)
   
5. REMEDIATE
   - Remove from Git history (BFG Repo-Cleaner, git filter-repo)
   - Force-push to all branches
   - Delete and re-create any tags containing it
   - Notify all forks (if any) — they have the secret too!
   - Invalidate all clones (force everyone to re-clone)

LONG-TERM (next week):
6. PREVENT
   - Enable pre-commit hooks (GitLeaks, detect-secrets)
   - Enable GitHub secret scanning
   - Use secret managers (Vault, AWS Secrets Manager)
   - Developer training
   - Add to IR runbook
```

```bash
# Remove secret from Git history
git filter-repo --path config/secrets.yml --invert-paths
git push --force --all
git push --force --tags

# Scan repo for secrets
trufflehog git file://. --json
gitleaks detect --source . --verbose
```

### ⭐ Q: Container escape detected. How do you prevent and detect?

**Container escape:** Process breaks out of container isolation and gains host access.

**Prevention:**
```yaml
# 1. Secure pod security context
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  capabilities:
    drop: [ALL]
  readOnlyRootFilesystem: true
  seccompProfile:
    type: RuntimeDefault

# 2. Pod Security Standards (Restricted)
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted

# 3. No privileged containers
spec:
  containers:
  - name: app
    securityContext:
      privileged: false  # Never set to true

# 4. No host namespace sharing
spec:
  hostNetwork: false
  hostPID: false
  hostIPC: false

# 5. AppArmor/SELinux profiles
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/app: runtime/default
```

**Detection:**
```yaml
# Falco rule for container escape attempts
- rule: Container Escape Detected
  desc: Detect container escape attempts
  condition: >
    spawned_process and
    container and
    proc.name in (docker, runc, containerd, ctr) and
    not container.privileged = true
  output: >
    Container escape attempt (user=%user.name container=%container.id 
    image=%container.image.repository proc=%proc.cmdline)
  priority: CRITICAL

# Tetragon policy (eBPF-based enforcement)
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: detect-container-escape
spec:
  kprobes:
  - call: "cap_capable"
    args:
    - index: 1
      type: "int"
    selectors:
    - matchCapabilities:
      - operator: "In"
        values: ["CAP_SYS_ADMIN", "CAP_SYS_PTRACE"]
    matchActions:
    - action: Post
```

**Response:**
1. Isolate the node (cordon, drain)
2. Capture forensics (disk snapshot, memory dump)
3. Kill the container/pod
4. Rebuild node from known-good image
5. Audit all containers on that node
6. Check for persistence mechanisms

### ⭐ Q: Supply-chain attack (like SolarWinds/xz). How to defend?

**Attack:** Compromise a widely-used dependency to distribute malware.

**Defense in depth:**

```
1. DEPENDENCY VERIFICATION
   - Pin dependencies to exact versions with hash verification
   - Use lock files (package-lock.json, Pipfile.lock, go.sum)
   - Verify signatures of packages (npm verify, go mod verify)
   - Use private mirrors for critical dependencies

2. SBOM + CONTINUOUS MONITORING
   - Generate SBOM for all artifacts
   - Monitor for newly disclosed vulnerabilities in dependencies
   - Automated dependency updates (Dependabot, Renovate)
   - Track provenance (where did this come from?)

3. BUILD ISOLATION
   - Hermetic/reproducible builds (no network access)
   - Build in ephemeral environments
   - Verify build inputs (no fetching :latest)
   - Use build cache with hash verification

4. MULTI-PARTY APPROVAL
   - Two-person review for dependency updates
   - Automated tests must pass
   - Security scan before merge
   - Separate approvers for code vs dependencies

5. RUNTIME DETECTION
   - Monitor for unexpected network connections
   - File integrity monitoring (FIM)
   - Behavioral analysis (Falco, Tetragon)
   - Anomaly detection (unusual syscalls)

6. SEGMENTATION
   - Network policies (default deny)
   - Limit blast radius (microservices, least privilege)
   - Secrets isolation (per-service)
```

```yaml
# Example: OPA policy requiring signed dependencies
package kubernetes.admission

deny[msg] {
  input.request.kind.kind == "Pod"
  image := input.request.object.spec.containers[_].image
  not is_signed(image)
  msg := sprintf("Image %v is not signed", [image])
}

is_signed(image) {
  # Call Cosign verification
  cosign_verify(image)
}
```

### 🔥 Q: Design secrets management for 100 microservices.

**Requirements:**
- Secrets never in code, config, env vars in Dockerfile
- Automatic rotation
- Audit trail
- Different secrets per environment (dev/staging/prod)
- Least privilege (service A can't read service B's secrets)

**Architecture:**

```
┌─── Secrets Management Architecture ───────────────────────┐
│                                                            │
│  ┌─── Vault Cluster (HA) ─────────────────────────────┐   │
│  │                                                     │   │
│  │  ┌── KV Secrets Engine ──┐  ┌── Dynamic Secrets ─┐│   │
│  │  │ /prod/svc-a/db        │  │ Database creds     ││   │
│  │  │ /prod/svc-a/api-key   │  │ AWS IAM keys       ││   │
│  │  │ /prod/svc-b/...       │  │ Auto-rotation      ││   │
│  │  └───────────────────────┘  └────────────────────┘│   │
│  │                                                     │   │
│  │  ┌── Policies ────────────┐  ┌── Audit Log ──────┐│   │
│  │  │ path "prod/svc-a/*" {  │  │ Splunk/ELK        ││   │
│  │  │   capabilities=["read"]│  │                   ││   │
│  │  │ }                      │  │                   ││   │
│  │  └────────────────────────┘  └───────────────────┘│   │
│  └─────────────────────────────────────────────────────┘   │
│                           ▲                                │
│                           │ External Secrets Operator      │
│                           │                                │
│  ┌─── Kubernetes ─────────┴────────────────────────────┐   │
│  │                                                      │   │
│  │  Namespace: svc-a                                    │   │
│  │  ┌── ExternalSecret ──┐   ┌── K8s Secret ─────────┐ │   │
│  │  │ vault-db-creds     │ → │ db-credentials        │ │   │
│  │  │ refreshInterval:1h │   │ (synced from Vault)   │ │   │
│  │  └────────────────────┘   └───────────────────────┘ │   │
│  │                                                      │   │
│  │  ┌── Pod ──────────────────────────────────────────┐ │   │
│  │  │ ServiceAccount: svc-a                           │ │   │
│  │  │ (Vault role: can only read /prod/svc-a/*)      │ │   │
│  │  │                                                 │ │   │
│  │  │ App reads secret from:                          │ │   │
│  │  │ - Volume mount (K8s secret)                     │ │   │
│  │  │ - OR direct Vault API call                      │ │   │
│  │  └─────────────────────────────────────────────────┘ │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────┘
```

**Implementation:**

```yaml
# 1. External Secrets Operator — SecretStore (one per namespace)
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: svc-a
spec:
  provider:
    vault:
      server: "https://vault.internal.com"
      path: "prod"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "svc-a"

---
# 2. ExternalSecret — sync from Vault to K8s Secret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: svc-a
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: db-secret
    creationPolicy: Owner
  dataFrom:
  - extract:
      key: svc-a/database

---
# 3. Pod consumes secret
apiVersion: v1
kind: Pod
metadata:
  name: svc-a
  namespace: svc-a
spec:
  serviceAccountName: svc-a
  containers:
  - name: app
    image: myapp:v1
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
```

**Vault policies (per service):**

```hcl
# Policy for svc-a
path "prod/data/svc-a/*" {
  capabilities = ["read", "list"]
}

# Policy for svc-b
path "prod/data/svc-b/*" {
  capabilities = ["read", "list"]
}

# Kubernetes auth role
path "auth/kubernetes/role/svc-a" {
  bound_service_account_names = ["svc-a"]
  bound_service_account_namespaces = ["svc-a"]
  policies = ["svc-a-policy"]
  ttl = 1h
}
```

**Rotation strategy:**
- **Static secrets:** Scheduled rotation via Vault (weekly/monthly)
- **Dynamic secrets:** Short TTL (1-4 hours), auto-revoked on lease expiry
- **Emergency rotation:** Revoke all leases, force refresh

### 🔥 Q: IAM credentials compromised. Contain it.

**Indicators:**
- CloudTrail shows API calls from unknown IP
- GuardDuty alerts on unusual behavior
- MFA bypass attempts
- Resource creation in unexpected regions

**Response:**

```bash
# IMMEDIATE (first 5 minutes):
# 1. Disable the compromised IAM user/role
aws iam attach-user-policy --user-name compromised-user \
  --policy-arn arn:aws:iam::aws:policy/AWSDenyAll

# 2. Rotate access keys
aws iam delete-access-key --user-name compromised-user --access-key-id AKIAXXXXXXX
aws iam create-access-key --user-name compromised-user

# 3. Invalidate sessions (for assumed roles)
aws iam update-assume-role-policy --role-name compromised-role \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Deny","Principal":"*","Action":"sts:AssumeRole"}]}'

# 4. Enable MFA requirement immediately
aws iam attach-user-policy --user-name compromised-user \
  --policy-arn arn:aws:iam::123456789:policy/RequireMFA

# SHORT-TERM (next hour):
# 5. Audit CloudTrail logs
aws cloudtrail lookup-events --lookup-attributes AttributeKey=Username,AttributeValue=compromised-user \
  --start-time 2025-01-01T00:00:00Z --max-results 1000

# 6. Check for persistence mechanisms
# - New IAM users created?
aws iam list-users --query 'Users[?CreateDate>=`2025-01-01`]'
# - Backdoor roles?
aws iam list-roles --query 'Roles[?CreateDate>=`2025-01-01`]'
# - Lambda functions (data exfil)?
aws lambda list-functions --query 'Functions[?LastModified>=`2025-01-01`]'

# 7. Terminate unauthorized resources
aws ec2 describe-instances --filters "Name=tag:UnauthorizedByAdmin,Values=true" \
  --query 'Reservations[].Instances[].InstanceId' | xargs -I {} aws ec2 terminate-instances --instance-ids {}

# LONG-TERM (next week):
# 8. Rotate all credentials user had access to
# 9. Review IAM policies (least privilege)
# 10. Enable GuardDuty if not already
# 11. Mandatory MFA for all users
# 12. Session duration limits
# 13. Incident post-mortem
```

### 🔥 Q: Harden a Kubernetes cluster.

**Comprehensive hardening checklist:**

```
1. CONTROL PLANE SECURITY
   □ Private API server endpoint (no public internet)
   □ API server audit logging enabled
   □ Admission controllers enabled (NodeRestriction, PodSecurityPolicy successor)
   □ RBAC enabled (disable ABAC)
   □ Disable anonymous auth (--anonymous-auth=false)
   □ etcd encryption at rest
   □ etcd TLS for peer/client communication
   □ Rotate certificates regularly

2. NODE SECURITY
   □ CIS Kubernetes benchmark compliance
   □ Minimal OS (Bottlerocket, Flatcar, hardened Linux)
   □ Disable SSH access (use SSM/bastion if needed)
   □ Immutable infrastructure (nodes as cattle)
   □ Auto-patching (security updates)
   □ Kernel hardening (seccomp, AppArmor)

3. POD SECURITY
   □ Pod Security Standards (Restricted)
   □ No privileged containers
   □ Run as non-root
   □ Read-only root filesystem
   □ Drop all capabilities
   □ Resource limits (prevent DoS)
   □ No host namespace sharing

4. NETWORK SECURITY
   □ Network policies (default deny ingress/egress)
   □ Private cluster (nodes no public IP)
   □ Service mesh for mTLS (Istio, Linkerd)
   □ Ingress controller with WAF
   □ Egress filtering (no internet access except allowlist)

5. SECRETS MANAGEMENT
   □ Encrypt secrets at rest in etcd
   □ External secrets (Vault, AWS Secrets Manager)
   □ No secrets in env vars (use volumes)
   □ Short-lived service account tokens

6. IMAGE SECURITY
   □ Private container registry
   □ Image scanning (Trivy, Snyk)
   □ Signed images only (admission controller)
   □ Distroless/minimal base images
   □ No :latest tags

7. ACCESS CONTROL
   □ RBAC least privilege
   □ Namespace isolation
   □ No cluster-admin for apps
   □ Service account per app
   □ Audit who-can-do-what regularly

8. MONITORING & RESPONSE
   □ Runtime security (Falco, Tetragon)
   □ Centralized logging (ELK, Splunk)
   □ Alerting on security events
   □ Incident response runbooks
   □ Regular security audits

9. COMPLIANCE
   □ CIS Kubernetes benchmark scan (kube-bench)
   □ Penetration testing
   □ Vulnerability scanning (Trivy, Aqua)
   □ Policy enforcement (OPA, Kyverno)
```

```bash
# Audit current cluster security posture
kube-bench run --targets=master,node

# Check RBAC permissions
kubectl auth can-i --list --as=system:serviceaccount:default:myapp

# Find pods running as root
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.securityContext.runAsUser == 0 or .spec.containers[].securityContext.runAsUser == 0) | .metadata.namespace + "/" + .metadata.name'

# Find privileged pods
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.containers[].securityContext.privileged == true) | .metadata.namespace + "/" + .metadata.name'

# Scan images for vulnerabilities
trivy image --severity HIGH,CRITICAL myregistry/myapp:v1.0
```

---

## Key Resources

### Standards & Frameworks
- **OWASP Top 10** — https://owasp.org/www-project-top-ten/
- **CIS Benchmarks** — https://www.cisecurity.org/cis-benchmarks (K8s, Docker, Linux)
- **NIST Cybersecurity Framework** — https://www.nist.gov/cyberframework
- **SLSA Framework** — https://slsa.dev (supply-chain security levels)
- **NIST 800-190** — Application Container Security Guide

### Books
- **Container Security** — Liz Rice
- **Kubernetes Security** — Liz Rice & Michael Hausenblas
- **Hacking Kubernetes** — Andrew Martin & Michael Hausenblas
- **Practical Cloud Security** — Chris Dotson

### Tools & Projects
- **Falco** — https://falco.org (runtime security, eBPF/kernel modules)
- **Trivy** — https://trivy.dev (vulnerability scanner for containers, IaC, SBOM)
- **Cosign** — https://sigstore.dev (keyless signing, Sigstore project)
- **OPA Gatekeeper** — https://open-policy-agent.github.io/gatekeeper/
- **Kyverno** — https://kyverno.io (Kubernetes-native policy engine)
- **Vault** — https://vaultproject.io (secrets management)
- **External Secrets Operator** — https://external-secrets.io
- **Syft** — https://github.com/anchore/syft (SBOM generation)
- **Tetragon** — https://tetragon.io (eBPF runtime enforcement)
- **GitLeaks** — https://github.com/gitleaks/gitleaks (secret detection)

### Learning Platforms
- **Kubernetes Security Specialist (CKS)** — CNCF certification
- **TryHackMe** — Hands-on security labs
- **HackTheBox** — Penetration testing practice
- **Securing DevOps (book)** — Julien Vehent
