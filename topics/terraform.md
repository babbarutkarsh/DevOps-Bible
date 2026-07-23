---
title: Terraform
nav_order: 30
description: "Terraform — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# Terraform / Infrastructure as Code — DevOps Interview Preparation

> **Question frequency:** 🔥 very frequently asked · ⭐ important · 💡 good to know

## Table of Contents
- [IaC Fundamentals](#iac-fundamentals)
- [Terraform Architecture](#terraform-architecture)
- [State Management](#state-management)
- [HCL Language](#hcl-language)
- [Modules](#modules)
- [Variables & Outputs](#variables--outputs)
- [Providers & Data Sources](#providers--data-sources)
- [Workspaces & Backends](#workspaces--backends)
- [Import & Migration](#import--migration)
- [Functions & Expressions](#functions--expressions)
- [Testing & Policy](#testing--policy)
- [Terraform in CI/CD](#terraform-in-cicd)
- [OpenTofu & Terraform Licensing](#opentofu--terraform-licensing)
- [Best Practices](#best-practices)
- [Scenario-Based Questions](#scenario-based-questions)
- [Advanced Scenarios](#advanced-scenarios)

---

## IaC Fundamentals

### 🔥 Q: What is Infrastructure as Code?

Managing infrastructure through configuration files instead of manual processes.

**Benefits:** Version control, reproducibility, consistency, speed, collaboration, auditability.

**Declarative vs Imperative:**
| Approach | Description | Tools |
|----------|-------------|-------|
| **Declarative** | Define desired state; tool figures out how | Terraform, CloudFormation, Pulumi |
| **Imperative** | Step-by-step instructions | Ansible, Shell scripts |

---

## Terraform Architecture

### 🔥 Q: How does Terraform work?

```
terraform init  → Download providers, initialize backend
terraform plan  → Compare desired state vs current state (dry run)
terraform apply → Execute changes to reach desired state
terraform destroy → Remove all managed resources
```

```
.tf files ──► Terraform Core ──► State File
(desired)     (DAG builder,       (actual)
               plan engine)
                    │
                    ▼ gRPC
               Providers (AWS, GCP, Azure, K8s)
                    │
                    ▼ API calls
               Real Infrastructure
```

**DAG (Directed Acyclic Graph):** Terraform builds a dependency graph to parallelize independent resources and order dependent ones. View with `terraform graph`.

### ⭐ Q: Explain plan output symbols.

| Symbol | Meaning |
|--------|---------|
| `+` | Create |
| `-` | Destroy |
| `~` | Update in-place |
| `-/+` | Destroy and recreate (force replacement) |
| `<=` | Read (data source) |

---

## State Management

### 🔥 Q: What is Terraform state?

A JSON file mapping config to real resources: `aws_instance.web → i-1234567890abcdef0`

**Why it matters:**
- Maps config to real resources
- Tracks metadata and dependencies
- Performance (caches attributes)
- Drift detection

**Losing state is catastrophic** — Terraform would try to recreate everything.

### 🔥 Q: Remote state with locking (production setup).

```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "production/vpc/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

Always enable: **remote storage**, **versioning**, **encryption**, **locking**.

### ⭐ Q: State operations cheat sheet.

```bash
terraform state list                          # List resources
terraform state show aws_instance.web         # Show details
terraform state mv aws_instance.web aws_instance.app  # Rename
terraform state rm aws_instance.web           # Remove (resource still exists)
terraform import aws_instance.web i-12345     # Import existing resource
terraform apply -replace=aws_instance.web     # Force replacement
terraform apply -refresh-only                 # Sync state with reality
terraform state pull > backup.tfstate         # Download remote state
terraform state push backup.tfstate           # Upload state (DANGEROUS)
```

### 🔥 Q: What is state drift?

Someone changed infrastructure outside Terraform (console, CLI). Detect with `terraform plan`. Fix by either accepting drift (`-refresh-only`) or reverting it (`terraform apply`).

### 🔥 Q: Sensitive data in state — what's the risk?

State files contain **plaintext secrets** (DB passwords, API keys). Even with `sensitive = true`, values are visible in state JSON.

**Mitigation:**
- Encrypt state at rest (S3 encryption, TFC/HCP Terraform encryption)
- Lock down state access (IAM policies, RBAC)
- Use external secret stores (Vault, AWS Secrets Manager) and reference ARNs
- Never commit state to git
- Rotate secrets if state is leaked

```hcl
# WRONG: password ends up in state
resource "aws_db_instance" "db" {
  password = var.db_password  # In state plaintext!
}

# BETTER: use Secrets Manager
data "aws_secretsmanager_secret_version" "db" {
  secret_id = "prod/db/master"
}
resource "aws_db_instance" "db" {
  password = data.aws_secretsmanager_secret_version.db.secret_string
}
```

### ⭐ Q: State locking with S3 + DynamoDB — how does it work?

```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "production/vpc/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"  # Must have LockID (String) as primary key
  }
}
```

**Lock flow:**
1. `terraform apply` → writes lock record to DynamoDB with LockID = `<bucket>/<key>`
2. Second apply → DynamoDB returns ConditionalCheckFailedException
3. First apply finishes → deletes lock record
4. Second apply retries → acquires lock

**Force-unlock (CAREFUL):** `terraform force-unlock <lock-id>` — only if process died.

---

## HCL Language

### ⭐ Q: Resource lifecycle rules.

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type

  lifecycle {
    create_before_destroy = true    # Zero-downtime replacements
    prevent_destroy       = true    # Protect critical resources
    ignore_changes        = [tags]  # Ignore external changes
    replace_triggered_by  = [aws_security_group.web.id]
  }
}
```

### 🔥 Q: for_each vs count?

```hcl
# count — index-based (AVOID: removing item shifts all indices)
resource "aws_instance" "server" {
  count         = 3
  instance_type = var.instance_type
  tags = { Name = "server-${count.index}" }
}

# for_each — key-based (PREFERRED: stable references)
resource "aws_instance" "server" {
  for_each      = toset(["web", "api", "worker"])
  instance_type = var.instance_type
  tags = { Name = "server-${each.key}" }
}
# Removing "api" only destroys that resource. Others untouched.
```

### ⭐ Q: Dynamic blocks?

```hcl
variable "ingress_rules" {
  default = [
    { port = 80,  cidr = "0.0.0.0/0" },
    { port = 443, cidr = "0.0.0.0/0" },
  ]
}

resource "aws_security_group" "web" {
  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = [ingress.value.cidr]
    }
  }
}
```

### ⭐ Q: CMD, ENTRYPOINT equivalent — `depends_on` vs implicit?

```hcl
# Implicit dependency (preferred — Terraform auto-detects references)
resource "aws_instance" "web" {
  subnet_id = aws_subnet.main.id  # Terraform knows to create subnet first
}

# Explicit dependency (when there's no reference but order matters)
resource "aws_instance" "web" {
  depends_on = [aws_iam_role_policy.s3_access]
}
```

---

## Modules

### 🔥 Q: Module structure and usage.

```
modules/vpc/
├── main.tf
├── variables.tf
├── outputs.tf
└── versions.tf

environments/
├── production/main.tf    # Calls modules with prod values
├── staging/main.tf
└── dev/main.tf
```

```hcl
module "vpc" {
  source = "../../modules/vpc"
  # Or: source = "terraform-aws-modules/vpc/aws"
  # Or: source = "git::https://github.com/org/modules.git//vpc?ref=v1.2.0"

  environment     = "production"
  vpc_cidr        = "10.0.0.0/16"
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
}

module "eks" {
  source     = "../../modules/eks"
  vpc_id     = module.vpc.vpc_id              # Cross-module reference
  subnet_ids = module.vpc.private_subnet_ids
}
```

**Best practices:** Pin versions, keep modules small, validate inputs, test with Terratest.

### ⭐ Q: Module versioning — how do you manage updates safely?

```hcl
# Pin to exact version (safest for prod)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"  # Exact pin
}

# Pin to minor version (auto patch updates)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.1.0"  # >=5.1.0, <5.2.0
}

# Git with ref (for internal modules)
module "vpc" {
  source = "git::https://github.com/myorg/terraform-modules.git//vpc?ref=v2.3.1"
}
```

**Workflow:**
1. Dev: test new module version in staging
2. `terraform init -upgrade` to pull new version
3. Review `terraform plan` diff carefully
4. Promote to prod only after staging validation

### 💡 Q: Public vs private module registry?

| Feature | Public Registry | Private Registry |
|---------|----------------|------------------|
| URL | registry.terraform.io | app.terraform.io/org/registry |
| Authentication | None | Required |
| Best for | Open-source modules | Internal company modules |
| Versioning | Git tags | Git tags or TFC versioning |
| Examples | AWS VPC, GKE modules | Internal platform modules |

---

## Variables & Outputs

### ⭐ Q: Variable types and validation.

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Must be dev, staging, or prod."
  }
}

variable "vpc_config" {
  type = object({
    cidr    = string
    azs     = list(string)
    enable  = optional(bool, true)
  })
}

variable "db_password" {
  type      = string
  sensitive = true   # Won't show in plan output
}
```

**Variable precedence (highest → lowest):**
1. `-var` / `-var-file` CLI flags
2. `*.auto.tfvars`
3. `terraform.tfvars`
4. `TF_VAR_name` environment variables
5. Default values

---

## Providers & Data Sources

### ⭐ Q: Multiple providers (multi-account/region).

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

provider "aws" {
  alias  = "shared"
  assume_role { role_arn = "arn:aws:iam::111111111111:role/Terraform" }
}

resource "aws_instance" "west_server" {
  provider = aws.west
  ami      = "ami-west-123"
}
```

### ⭐ Q: Data sources — read existing infrastructure.

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
}

data "aws_caller_identity" "current" {}

resource "aws_instance" "web" {
  ami = data.aws_ami.ubuntu.id
  tags = { Account = data.aws_caller_identity.current.account_id }
}
```

---

## Workspaces & Backends

### 🔥 Q: Workspaces — multiple states for same config.

```bash
terraform workspace new staging
terraform workspace select production
terraform workspace list
```

```hcl
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "production" ? "t3.large" : "t3.small"
  tags = { Environment = terraform.workspace }
}
```

**Why workspaces are an anti-pattern for environments:**

| Issue | Why It's Bad |
|-------|--------------|
| **Blast radius** | One typo → destroy prod instead of dev |
| **Different configs** | Prod needs more resources, different modules |
| **State isolation** | All envs share same backend config |
| **IAM/RBAC** | Can't restrict who can deploy to prod |
| **Code review** | Hard to review prod vs dev in same file |

**Preferred:** Separate directories with environment-specific configs.

```
terraform/
├── environments/
│   ├── production/
│   │   ├── main.tf          # Different backend, stricter settings
│   │   ├── terraform.tfvars # Prod values
│   ├── staging/
│   │   ├── main.tf
│   │   ├── terraform.tfvars
```

**Workspaces are OK for:** Short-lived feature branch testing, Terraform Cloud workspaces (different concept).

### ⭐ Q: Available backends.

| Backend | Locking | Best For |
|---------|---------|----------|
| **S3 + DynamoDB** | DynamoDB | AWS teams |
| **GCS** | Built-in | GCP teams |
| **Azure Blob** | Built-in | Azure teams |
| **Terraform Cloud** | Built-in | Any cloud, best UX |
| **Consul/PostgreSQL** | Built-in | Self-hosted |

---

## Import & Migration

### 🔥 Q: Import existing infrastructure.

```bash
# Traditional: write config first, then import
terraform import aws_instance.web i-1234567890abcdef0
terraform plan  # Adjust config until no diff
```

```hcl
# Import block (Terraform 1.5+) — generate config automatically
import {
  to = aws_instance.web
  id = "i-1234567890abcdef0"
}
# terraform plan -generate-config-out=generated.tf
```

### ⭐ Q: Refactoring with moved blocks (Terraform 1.1+).

```hcl
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}
```

**Use cases:** Rename resources, move into/out of modules, restructure without recreation.

### ⭐ Q: Provisioners — when and why to avoid them?

```hcl
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = "t3.micro"

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
    ]
  }
}
```

**Why avoid:**
- Breaks idempotency (run twice = different state)
- Not tracked in state (Terraform doesn't know if provisioner failed)
- Better alternatives: Packer (bake AMI), cloud-init, Ansible, configuration management

**When OK:** One-time bootstrap, local-exec for notifications, debugging.

```hcl
# Acceptable use: trigger external workflow
resource "null_resource" "notify" {
  provisioner "local-exec" {
    command = "curl -X POST https://hooks.slack.com/... -d 'Deploy complete'"
  }
}
```

---

## Functions & Expressions

### ⭐ Q: Must-know functions.

```hcl
# String
upper("hello")                          # "HELLO"
format("Hello, %s!", var.name)          # String formatting
join(",", ["a", "b", "c"])              # "a,b,c"
replace("hello-world", "-", "_")        # "hello_world"
trimspace("  hello  ")                  # "hello"

# Collection
length(var.list)                        # List/map length
lookup(var.map, "key", "default")       # Map lookup with default
flatten([["a"], ["b", "c"]])            # ["a", "b", "c"]
merge(map1, map2)                       # Merge maps
keys(var.map)                           # Map keys
values(var.map)                         # Map values
contains(["a", "b"], "a")              # true
distinct(["a", "a", "b"])              # ["a", "b"]
zipmap(keys, values)                    # Create map from two lists

# Filesystem
file("${path.module}/config.json")      # Read file
templatefile("template.tpl", { name = "app" })
fileexists("path/file")

# Encoding
jsonencode(value)  / jsondecode(string)
base64encode(str)  / base64decode(str)
yamlencode(value)  / yamldecode(string)

# IP/CIDR
cidrsubnet("10.0.0.0/16", 8, 1)       # "10.0.1.0/24"
cidrhost("10.0.1.0/24", 5)            # "10.0.1.5"

# Type conversion
tostring(42)  tolist(set)  tomap(obj)  tonumber("42")  tobool("true")

# Conditional
var.env == "prod" ? "t3.large" : "t3.small"

# For expressions
[for s in var.list : upper(s)]                    # Transform list
{for k, v in var.map : k => upper(v)}             # Transform map
[for s in var.list : s if s != ""]                 # Filter
```

---

## Testing & Policy

### 🔥 Q: How do you test Terraform code?

**Levels of testing:**

| Level | Tool | What It Tests |
|-------|------|---------------|
| **Syntax** | `terraform fmt -check` | Code formatting |
| **Validation** | `terraform validate` | Config syntax, required args |
| **Static analysis** | tfsec, checkov, terrascan | Security misconfigs (S3 public, no encryption) |
| **Plan testing** | `terraform test` (1.6+) | Assertions on plan without apply |
| **Integration** | Terratest (Go), kitchen-terraform | Real apply to test account, verify resources |
| **Policy-as-code** | Sentinel, OPA | Enforce standards (tagging, naming, approved AMIs) |

### 💡 Q: Terraform test (native testing, 1.6+).

```hcl
# tests/vpc.tftest.hcl
run "validate_vpc" {
  command = plan

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR must be 10.0.0.0/16"
  }

  assert {
    condition     = length(aws_subnet.private) == 3
    error_message = "Must have exactly 3 private subnets"
  }
}

run "apply_and_verify" {
  command = apply

  assert {
    condition     = can(regex("^vpc-", aws_vpc.main.id))
    error_message = "VPC ID must start with vpc-"
  }
}
```

```bash
terraform test  # Runs all .tftest.hcl files in tests/
```

### 💡 Q: Policy-as-code — Sentinel vs OPA?

| Feature | Sentinel | OPA |
|---------|----------|-----|
| Language | Sentinel DSL | Rego |
| Terraform integration | Terraform Cloud/Enterprise native | Via conftest or custom |
| Use case | Enforce "no t3.nano in prod", "all S3 encrypted" | General-purpose policy (K8s, API, Terraform) |
| Ecosystem | Terraform-specific | Broader (CNCF project) |

**Sentinel example (Terraform Cloud):**

```hcl
import "tfplan/v2" as tfplan

main = rule {
  all tfplan.resource_changes as _, rc {
    rc.type is "aws_s3_bucket" implies
      rc.change.after.server_side_encryption_configuration is not null
  }
}
```

**OPA/Conftest example:**

```rego
package terraform

deny[msg] {
  r := input.resource_changes[_]
  r.type == "aws_instance"
  r.change.after.instance_type == "t3.nano"
  msg := "t3.nano is not allowed in production"
}
```

### ⭐ Q: Security scanning tools comparison.

| Tool | Language | Focus | CI Integration |
|------|----------|-------|----------------|
| **tfsec** | Go | Static security | Fast, GitHub Actions |
| **checkov** | Python | Broad coverage (Terraform, K8s, Docker) | More checks, slower |
| **terrascan** | Go | Policy-as-code, multiple IaC | OPA-based |
| **trivy** | Go | Misconfigs + vulnerabilities | Also scans containers |

---

## Terraform in CI/CD

### 🔥 Q: How do you run Terraform in a CI/CD pipeline?

```yaml
# GitHub Actions example
name: Terraform
on:
  pull_request:
    paths: ['terraform/**']
  push:
    branches: [main]

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - uses: hashicorp/setup-terraform@v3

    - name: Terraform Init
      run: terraform init
      working-directory: terraform/

    - name: Terraform Format Check
      run: terraform fmt -check -recursive

    - name: Terraform Validate
      run: terraform validate

    - name: Terraform Plan
      run: terraform plan -out=tfplan
      # On PR: post plan as comment

    - name: Terraform Apply
      if: github.ref == 'refs/heads/main'
      run: terraform apply -auto-approve tfplan
```

**Key CI/CD practices:**
- `plan` on PR, `apply` on merge to main
- Use `-out=tfplan` to save plan, apply exact plan
- Never `apply -auto-approve` without a saved plan
- Use OIDC for authentication (no long-lived credentials)
- Run `terraform fmt -check` and `terraform validate`
- Use tools: **tflint**, **tfsec/trivy**, **checkov**, **infracost**

### ⭐ Q: Atlantis vs Terraform Cloud vs HCP Terraform?

| Feature | Atlantis | Terraform Cloud | HCP Terraform |
|---------|----------|-----------------|---------------|
| **Deployment** | Self-hosted | SaaS | SaaS (new name for TFC) |
| **Plan on PR** | ✅ (as PR comment) | ✅ | ✅ |
| **State storage** | Your backend | Terraform-managed | Terraform-managed |
| **RBAC** | Basic | Advanced | Advanced + Sentinel |
| **Cost** | Free (you host) | Free tier + paid | Free tier + paid |
| **Sentinel/OPA** | ❌ | ✅ Sentinel | ✅ Sentinel |
| **Best for** | Self-hosted teams | Teams wanting managed solution | Enterprise (same as TFC) |

**HCP Terraform = Terraform Cloud rebrand (2024).** Same product, new name under HashiCorp Cloud Platform.

### 💡 Q: CI/CD with Terragrunt — DRY config?

Terragrunt wraps Terraform to keep config DRY across environments.

```hcl
# terragrunt.hcl (root)
remote_state {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "${path_relative_to_include()}/terraform.tfstate"
    region = "us-east-1"
  }
}

# environments/production/vpc/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../modules/vpc"
}

inputs = {
  environment = "production"
  vpc_cidr    = "10.0.0.0/16"
}
```

```bash
cd environments/production/vpc
terragrunt plan   # Wraps terraform plan + auto-configures backend
terragrunt apply
```

**Benefits:** DRY backend config, automatic dependency ordering, run-all command.
**Drawback:** Extra layer of complexity.

---

## OpenTofu & Terraform Licensing

### 🔥 Q: What is OpenTofu and why does it exist?

**Timeline:**
- Aug 2023: HashiCorp changes Terraform license from MPL to BSL (Business Source License)
- BSL prohibits competitive SaaS offerings using Terraform
- Sept 2023: Community forks Terraform → **OpenTofu** (Linux Foundation project)
- OpenTofu 1.6+ is license-compatible with Terraform 1.5.x

**OpenTofu:**
- Open-source fork (MPL license, always free)
- API-compatible with Terraform
- Maintained by Linux Foundation
- Growing adoption in 2025-2026

### ⭐ Q: OpenTofu vs Terraform — should you switch?

| Aspect | Terraform | OpenTofu |
|--------|-----------|----------|
| **License** | BSL (proprietary) | MPL (open-source) |
| **Compatibility** | N/A | 1:1 with Terraform 1.5.x |
| **Provider support** | All providers | All providers (same registry) |
| **Enterprise** | HCP Terraform / TFE | Community + vendors (Spacelift, etc.) |
| **Roadmap** | HashiCorp-controlled | Community-driven |
| **2026 adoption** | Still dominant | Growing (AWS, Cloudflare using it) |

**When to consider OpenTofu:**
- License concerns (building competitive product)
- Preference for open-source governance
- Already on Terraform 1.5.x (easy migration)

**Migration:** `alias tofu=terraform` → most commands work identically.

### ⭐ Q: Terraform vs Pulumi vs Crossplane (2025-2026)?

| Feature | Terraform/OpenTofu | Pulumi | Crossplane |
|---------|-------------------|--------|------------|
| **Language** | HCL | TypeScript, Python, Go, etc. | YAML (K8s CRDs) |
| **Approach** | Declarative config | Imperative code with state | K8s-native reconciliation |
| **State** | tfstate file | Pulumi Cloud or self-managed | K8s etcd |
| **Multi-cloud** | ✅ Best | ✅ Good | ✅ Good |
| **Testing** | Terratest, terraform test | Native language tests | K8s tests |
| **Learning curve** | Moderate (new DSL) | Low (use existing language) | High (K8s knowledge required) |
| **Best for** | General IaC, broad ecosystem | Developers wanting familiar languages | K8s-centric orgs, GitOps |
| **2026 trend** | Still #1 | Growing for dev teams | Growing in K8s-native orgs |

**Interview answer:** "Terraform has the largest ecosystem and is battle-tested. Pulumi is gaining traction with dev teams who prefer real programming languages. Crossplane is strong in Kubernetes-native environments. Choice depends on team expertise and existing tooling."

---

## Best Practices

### 🔥 Q: Terraform best practices checklist.

**Code Organization:**
- One state per environment per component (not one giant state)
- Use modules for reusable components
- Consistent file naming: `main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`

**State:**
- Remote state with locking, encryption, versioning
- Small blast radius — split state by component
- Never edit state files manually

**Security:**
- No secrets in `.tf` files or state — use Vault, AWS Secrets Manager
- Use `sensitive = true` on variables/outputs
- OIDC authentication in CI/CD
- Scan with tfsec/checkov

**Operations:**
- Pin provider and module versions
- Use `terraform fmt` and `terraform validate`
- Always review `plan` before `apply`
- Use `prevent_destroy` on critical resources
- Tag everything

**Team:**
- PR-based workflow with plan output in comments
- Use Terraform Cloud / Atlantis for collaboration
- Document modules with README and examples

---

## Scenario-Based Questions

### 🔥 Q: You ran `terraform apply` and it failed midway. What now?

State is partially updated. Some resources were created, others weren't.

```bash
# 1. DON'T PANIC. State tracks what was created.
terraform plan    # Shows remaining changes

# 2. Fix the error (permissions, quota, config)
# 3. Re-run apply — Terraform will only do remaining work
terraform apply

# 4. If resource is in a bad state:
terraform apply -replace=aws_instance.broken

# 5. If totally stuck:
terraform state rm aws_instance.broken  # Remove from state
# Manually clean up the resource, then re-apply
```

### 🔥 Q: Two team members are running `terraform apply` at the same time.

With **state locking** (DynamoDB/Terraform Cloud): Second person gets lock error.

Without locking: **State corruption** — both read same state, both write different versions.

**Solution:** Always use remote state with locking. Use CI/CD to serialize applies.

### ⭐ Q: How do you handle a resource that Terraform wants to destroy but shouldn't?

```bash
# Option 1: Remove from state (Terraform forgets it)
terraform state rm aws_instance.legacy

# Option 2: Ignore in config
lifecycle { prevent_destroy = true }

# Option 3: Import the correct resource
terraform import aws_instance.legacy i-correct-id
```

### 💡 Q: How do you move from one Terraform state to another?

```bash
# 1. Move resource between states
terraform state mv -state-out=../new-project/terraform.tfstate \
  aws_instance.web aws_instance.web

# 2. Or use moved blocks for refactoring within same state
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}
```

### ⭐ Q: How do you manage 200 microservices' infrastructure?

**Anti-pattern:** One giant monolithic Terraform state.

**Paved-path approach:**

```
terraform/
├── shared/                    # Shared foundation (once)
│   ├── network/               # VPC, subnets
│   ├── platform/              # EKS/GKE cluster, observability
│   ├── security/              # IAM roles, security groups
│
├── services/                  # Per-service infra
│   ├── service-a/
│   │   ├── main.tf            # DB, cache, SQS for service-a
│   │   └── backend.tf
│   ├── service-b/
│   │   └── ...
│
├── modules/                   # Reusable modules
│   ├── microservice/          # Standard service module
│   │   ├── database.tf
│   │   ├── cache.tf
│   │   └── queues.tf
```

**Key strategies:**
1. **Small blast radius:** Each service has its own state
2. **Standardized modules:** DRY via `module "standard" { source = "../../modules/microservice" }`
3. **Terragrunt or Atlantis:** Manage multi-service orchestration
4. **Remote state data sources:** Services reference shared infra
5. **CI/CD:** Auto-plan on service code changes, auto-apply on merge

```hcl
# service-a/main.tf
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "shared/network/terraform.tfstate"
  }
}

module "service" {
  source    = "../../modules/microservice"
  vpc_id    = data.terraform_remote_state.network.outputs.vpc_id
  subnet_ids = data.terraform_remote_state.network.outputs.private_subnets
}
```

---

## Advanced Scenarios

### 🔥 Q: State file got corrupted / accidentally deleted. What do you do?

**If versioning is enabled (S3 versioning, TFC always versions):**

```bash
# S3 example
aws s3api list-object-versions --bucket mycompany-terraform-state \
  --prefix production/vpc/terraform.tfstate

aws s3api get-object --bucket mycompany-terraform-state \
  --key production/vpc/terraform.tfstate \
  --version-id <previous-version-id> terraform.tfstate

terraform state push terraform.tfstate  # Restore
```

**If no backup:**
1. Manually list all resources (AWS Console, `aws resourcegroupstaggingapi`)
2. Recreate state with imports:

```bash
# Create new minimal config
cat > recover.tf <<EOF
resource "aws_instance" "web" {}
resource "aws_vpc" "main" {}
EOF

# Import each resource
terraform import aws_instance.web i-1234567890abcdef0
terraform import aws_vpc.main vpc-abcdef123456

# Adjust config until plan shows no diff
terraform plan
```

**Prevention:** Always enable versioning + backups.

### ⭐ Q: Refactor monolithic state into modules with zero downtime.

**Scenario:** 100 resources in one state → split into separate modules/states.

**Approach:**

```bash
# 1. Create new module structure
mkdir -p modules/network modules/compute

# 2. Move resources in state (doesn't touch real infra)
terraform state mv aws_vpc.main module.network.aws_vpc.main
terraform state mv aws_subnet.private[0] module.network.aws_subnet.private[0]
# ... repeat for all network resources

# 3. Adjust code to match
cat > main.tf <<EOF
module "network" {
  source = "./modules/network"
}
module "compute" {
  source = "./modules/compute"
  vpc_id = module.network.vpc_id
}
EOF

# 4. Plan should show "No changes" (just moved blocks)
terraform plan

# 5. If splitting into separate states:
terraform state mv -state-out=../network/terraform.tfstate \
  module.network module.network
```

**Use `moved` blocks for complex refactoring:**

```hcl
moved {
  from = aws_instance.server[0]
  to   = module.compute.aws_instance.server["web"]
}

moved {
  from = aws_instance.server[1]
  to   = module.compute.aws_instance.server["api"]
}
```

### ⭐ Q: Two engineers ran `terraform apply` simultaneously (no locking).

**What happens:** State corruption. Both read state v100, both write different v101.

**Symptoms:**
- Resources missing from state
- Plan shows unexpected recreations
- Inconsistent state

**Recovery:**

```bash
# 1. Stop all applies immediately
# 2. Pull latest state
terraform state pull > corrupted.tfstate

# 3. Restore from backup (before corruption)
aws s3api list-object-versions --bucket mycompany-terraform-state \
  --prefix production/terraform.tfstate
aws s3api get-object --version-id <before-collision> good.tfstate

# 4. Compare corrupted vs good, identify missing resources
diff <(terraform state list -state=corrupted.tfstate) \
     <(terraform state list -state=good.tfstate)

# 5. Re-import missing resources or push good state
terraform state push good.tfstate

# 6. FIX LOCKING before any more applies
```

**Prevention:** Always use state locking (DynamoDB, TFC, etc.).

### 💡 Q: Migrate resources between Terraform states (zero downtime).

**Scenario:** Move `aws_rds_instance.db` from state A to state B.

```bash
# In state A directory
terraform state rm aws_rds_instance.db  # Removes from A (doesn't delete resource)

# In state B directory
terraform import aws_rds_instance.db my-db-instance-id

# Adjust config in B until plan shows no changes
terraform plan  # Should be no-op

# Verify resource wasn't recreated
aws rds describe-db-instances --db-instance-identifier my-db-instance-id
```

**Using `terraform state mv` between states:**

```bash
terraform state mv \
  -state=../state-a/terraform.tfstate \
  -state-out=../state-b/terraform.tfstate \
  aws_rds_instance.db aws_rds_instance.db
```

### 🔥 Q: Secret (password, API key) ended up in Terraform state. How do you remediate?

**Immediate:**
1. Rotate the leaked secret
2. Lock down state access (audit who downloaded)
3. Purge state history if possible (S3 delete all versions — CAREFUL)

**Long-term fix:**

```hcl
# WRONG: secret in config → in state
resource "aws_db_instance" "db" {
  password = var.db_password
}

# RIGHT: externalize secret
data "aws_secretsmanager_secret_version" "db_master" {
  secret_id = "prod/db/master"
}

resource "aws_db_instance" "db" {
  password = data.aws_secretsmanager_secret_version.db_master.secret_string
}
```

**For sensitive state fields:**

```hcl
# Mark outputs as sensitive (won't show in logs)
output "db_password" {
  value     = aws_db_instance.db.password
  sensitive = true
}
```

**Tools:** `terraform-compliance`, checkov rule to detect secrets in variables.

### 💡 Q: plan vs apply internals — what's the difference?

| Phase | Plan | Apply |
|-------|------|-------|
| **Refresh** | Read current state from real infra | Same |
| **Graph** | Build dependency DAG | Use plan's DAG |
| **Diff** | Desired vs current → show changes | Execute changes from plan |
| **Write** | No changes to real infra | Create/update/delete resources |
| **State update** | No state write | Write new state |

**Saved plan:** `terraform plan -out=tfplan` → binary file with exact changes.

```bash
terraform plan -out=tfplan        # Save plan
terraform show -json tfplan       # Inspect as JSON
terraform apply tfplan            # Apply exact plan (no new diff)
```

**Why saved plans matter in CI/CD:**
- Plan on PR → human reviews → apply exact plan on merge
- Prevents "plan showed X, apply did Y" drift

---

## Key Resources

**Official:**
- **Terraform Docs** — https://developer.hashicorp.com/terraform/docs
- **Terraform Registry** — https://registry.terraform.io (modules & providers)
- **OpenTofu Docs** — https://opentofu.org/docs
- **HCP Terraform (Terraform Cloud)** — https://app.terraform.io

**Books & Guides:**
- **Terraform Up & Running (3rd ed, 2024)** — Yevgeniy Brikman (THE Terraform book)
- **Terraform Best Practices** — https://www.terraform-best-practices.com

**Tools:**
- **tfsec / checkov / trivy** — Security scanning
- **tflint** — Linter for Terraform
- **Infracost** — Cloud cost estimates from plans
- **Terragrunt** — DRY Terraform wrapper
- **Terratest** — Go testing framework
- **Atlantis** — Self-hosted PR automation

**Platforms:**
- **HCP Terraform / Terraform Cloud** — Managed state, RBAC, Sentinel
- **Spacelift** — GitOps for Terraform
- **env0** — Policy-driven IaC platform

**Certification:**
- **HashiCorp Certified: Terraform Associate (003)** — Updated 2024, covers 1.6+ features
