---
title: Security (DevSecOps)
nav_order: 12
description: "Security (DevSecOps) — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# DevSecOps & Security — DevOps Interview Preparation

## Table of Contents
- [Security Fundamentals](#security-fundamentals)
- [Secrets Management](#secrets-management)
- [Container Security](#container-security)
- [Kubernetes Security](#kubernetes-security)
- [Network Security](#network-security)
- [Supply Chain Security](#supply-chain-security)
- [Compliance & Auditing](#compliance--auditing)
- [Encryption](#encryption)
- [Identity & Access Management](#identity--access-management)
- [Incident Response](#incident-response)
- [Scenario-Based Questions](#scenario-based-questions)

---

## Security Fundamentals

### Q: What is the CIA Triad?

| Principle | Meaning | DevOps Example |
|-----------|---------|----------------|
| **Confidentiality** | Only authorized access | Encryption, RBAC, secrets management |
| **Integrity** | Data is not tampered | Checksums, signing, audit logs |
| **Availability** | System is accessible | HA, DDoS protection, backups |

### Q: What is Shift-Left Security?

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

### Q: Principle of Least Privilege.

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

### Q: How should secrets be managed in DevOps?

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

### Q: HashiCorp Vault basics.

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

---

## Container Security

### Q: Container security checklist.

```
1. Image Security:
   □ Use minimal base images (Alpine, Distroless)
   □ Scan for CVEs (Trivy, Snyk)
   □ Pin image tags (no :latest)
   □ Sign images (Cosign, Docker Content Trust)
   □ Use private registries

2. Build Security:
   □ Multi-stage builds (no build tools in prod)
   □ No secrets in images (use build secrets)
   □ No root user (USER directive)
   □ Read-only root filesystem
   □ Lint Dockerfiles (Hadolint)

3. Runtime Security:
   □ Drop all capabilities, add specific ones
   □ No privileged containers
   □ Resource limits (CPU, memory, PIDs)
   □ Seccomp profiles
   □ AppArmor/SELinux profiles
   □ Read-only root filesystem + tmpfs for writable paths
   □ No host namespace sharing

4. Supply Chain:
   □ SBOM generation (Syft)
   □ Image provenance (SLSA)
   □ Vulnerability scanning in CI/CD
   □ Admission policies (only signed images)
```

### Q: What is Distroless and why use it?

**Distroless** images contain only the application and its runtime dependencies — no shell, no package manager, no OS utilities.

| Image | Size | Shell | Package Mgr | Attack Surface |
|-------|------|-------|-------------|----------------|
| ubuntu:22.04 | 77MB | Yes | Yes | Large |
| alpine:3.19 | 7MB | Yes | Yes | Small |
| distroless/static | 2MB | No | No | Minimal |
| scratch | 0MB | No | No | Zero (just your binary) |

**Trade-off:** Can't exec into the container for debugging. Use debug variants or ephemeral containers.

---

## Kubernetes Security

### Q: Kubernetes security layers.

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

### Q: What is OPA Gatekeeper?

**Open Policy Agent (OPA) Gatekeeper** is a Kubernetes admission controller that enforces policies:

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

---

## Network Security

### Q: mTLS (Mutual TLS) explained.

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

### Q: DDoS protection strategies.

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

### Q: What is software supply chain security?

Ensuring every component in your software delivery pipeline is trusted.

**SLSA Framework (Supply-chain Levels for Software Artifacts):**

| Level | Requirements |
|-------|-------------|
| **SLSA 1** | Build process exists and produces provenance |
| **SLSA 2** | Version-controlled, authenticated build service |
| **SLSA 3** | Hardened build platform, non-falsifiable provenance |
| **SLSA 4** | Two-person review, hermetic builds |

**Key practices:**
- **SBOM** (Software Bill of Materials) — Know what's in your images
- **Image signing** — Cosign, Notary
- **Dependency pinning** — Lock file, hash verification
- **Provenance** — Prove where artifacts came from
- **Admission policies** — Only allow signed images from trusted registries

```bash
# Sign an image with Cosign
cosign sign --key cosign.key myregistry/myapp:v1.0

# Verify
cosign verify --key cosign.pub myregistry/myapp:v1.0

# Generate SBOM
syft myregistry/myapp:v1.0 -o spdx-json > sbom.json

# Scan SBOM for vulnerabilities
grype sbom:sbom.json
```

---

## Encryption

### Q: Encryption at rest vs in transit.

| Type | What | How | Examples |
|------|------|-----|---------|
| **At Rest** | Data stored on disk | AES-256, KMS | EBS encryption, S3 SSE, etcd encryption |
| **In Transit** | Data moving over network | TLS 1.3, mTLS | HTTPS, mTLS in service mesh |

### Q: KMS (Key Management Service) concepts.

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

## Identity & Access Management

### Q: Authentication vs Authorization.

| Concept | Question | K8s Implementation |
|---------|----------|-------------------|
| **Authentication** | Who are you? | Certificates, OIDC, tokens |
| **Authorization** | What can you do? | RBAC, ABAC, webhook |
| **Admission** | Is this allowed? | Admission controllers, OPA |

### Q: Zero Trust Architecture.

**Old model:** Trust everything inside the network perimeter (castle and moat).
**Zero Trust:** "Never trust, always verify" regardless of network location.

**Principles:**
1. Verify explicitly — Authenticate and authorize every request
2. Least privilege — Minimum access needed
3. Assume breach — Segment, encrypt, monitor everything

**Implementation:**
- Identity-based access (not IP-based)
- mTLS everywhere
- Service mesh for policy enforcement
- Short-lived credentials (no permanent tokens)
- Continuous monitoring and anomaly detection

---

## Incident Response

### Q: Security incident response plan.

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

## Scenario-Based Questions

### Q: A Kubernetes pod was compromised. What do you do?

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

### Q: You found that Docker images are being pulled from an untrusted registry. Fix it.

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

---

## Key Resources

- **OWASP Top 10** — https://owasp.org/www-project-top-ten/
- **CIS Benchmarks** — https://www.cisecurity.org/cis-benchmarks (K8s, Docker, Linux)
- **NIST Cybersecurity Framework** — https://www.nist.gov/cyberframework
- **Container Security (book)** — Liz Rice
- **Kubernetes Security (book)** — Liz Rice & Michael Hausenblas
- **SLSA Framework** — https://slsa.dev
- **Falco** — https://falco.org (runtime security)
- **Hacking Kubernetes (book)** — Andrew Martin & Michael Hausenblas
