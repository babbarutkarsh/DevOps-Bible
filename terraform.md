# Terraform / Infrastructure as Code — DevOps Interview Preparation

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
- [Terraform in CI/CD](#terraform-in-cicd)
- [Best Practices](#best-practices)
- [Scenario-Based Questions](#scenario-based-questions)

---

## IaC Fundamentals

### Q: What is Infrastructure as Code?

Managing infrastructure through configuration files instead of manual processes.

**Benefits:** Version control, reproducibility, consistency, speed, collaboration, auditability.

**Declarative vs Imperative:**
| Approach | Description | Tools |
|----------|-------------|-------|
| **Declarative** | Define desired state; tool figures out how | Terraform, CloudFormation, Pulumi |
| **Imperative** | Step-by-step instructions | Ansible, Shell scripts |

---

## Terraform Architecture

### Q: How does Terraform work?

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

### Q: Explain plan output symbols.

| Symbol | Meaning |
|--------|---------|
| `+` | Create |
| `-` | Destroy |
| `~` | Update in-place |
| `-/+` | Destroy and recreate (force replacement) |
| `<=` | Read (data source) |

---

## State Management

### Q: What is Terraform state?

A JSON file mapping config to real resources: `aws_instance.web → i-1234567890abcdef0`

**Why it matters:**
- Maps config to real resources
- Tracks metadata and dependencies
- Performance (caches attributes)
- Drift detection

**Losing state is catastrophic** — Terraform would try to recreate everything.

### Q: Remote state with locking (production setup).

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

### Q: State operations cheat sheet.

```bash
terraform state list                          # List resources
terraform state show aws_instance.web         # Show details
terraform state mv aws_instance.web aws_instance.app  # Rename
terraform state rm aws_instance.web           # Remove (resource still exists)
terraform import aws_instance.web i-12345     # Import existing resource
terraform apply -replace=aws_instance.web     # Force replacement
terraform apply -refresh-only                 # Sync state with reality
```

### Q: What is state drift?

Someone changed infrastructure outside Terraform (console, CLI). Detect with `terraform plan`. Fix by either accepting drift (`-refresh-only`) or reverting it (`terraform apply`).

---

## HCL Language

### Q: Resource lifecycle rules.

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

### Q: for_each vs count?

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

### Q: Dynamic blocks?

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

### Q: CMD, ENTRYPOINT equivalent — `depends_on` vs implicit?

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

### Q: Module structure and usage.

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

---

## Variables & Outputs

### Q: Variable types and validation.

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

### Q: Multiple providers (multi-account/region).

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

### Q: Data sources — read existing infrastructure.

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

### Q: Workspaces — multiple states for same config.

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

**Not recommended for environment separation** — use separate directories instead for better isolation.

### Q: Available backends.

| Backend | Locking | Best For |
|---------|---------|----------|
| **S3 + DynamoDB** | DynamoDB | AWS teams |
| **GCS** | Built-in | GCP teams |
| **Azure Blob** | Built-in | Azure teams |
| **Terraform Cloud** | Built-in | Any cloud, best UX |
| **Consul/PostgreSQL** | Built-in | Self-hosted |

---

## Import & Migration

### Q: Import existing infrastructure.

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

### Q: Refactoring with moved blocks (Terraform 1.1+).

```hcl
moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}
```

---

## Functions & Expressions

### Q: Must-know functions.

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

## Terraform in CI/CD

### Q: How do you run Terraform in a CI/CD pipeline?

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

---

## Best Practices

### Q: Terraform best practices checklist.

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

### Q: You ran `terraform apply` and it failed midway. What now?

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

### Q: Two team members are running `terraform apply` at the same time.

With **state locking** (DynamoDB/Terraform Cloud): Second person gets lock error.

Without locking: **State corruption** — both read same state, both write different versions.

**Solution:** Always use remote state with locking. Use CI/CD to serialize applies.

### Q: How do you handle a resource that Terraform wants to destroy but shouldn't?

```bash
# Option 1: Remove from state (Terraform forgets it)
terraform state rm aws_instance.legacy

# Option 2: Ignore in config
lifecycle { prevent_destroy = true }

# Option 3: Import the correct resource
terraform import aws_instance.legacy i-correct-id
```

### Q: How do you move from one Terraform state to another?

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

### Q: Terraform vs CloudFormation vs Pulumi?

| Feature | Terraform | CloudFormation | Pulumi |
|---------|-----------|---------------|--------|
| Language | HCL | JSON/YAML | Python, Go, TS, etc. |
| Multi-cloud | Yes | AWS only | Yes |
| State | Self-managed or TF Cloud | AWS-managed | Pulumi Cloud |
| Ecosystem | Largest provider ecosystem | Deep AWS integration | Growing |
| Learning curve | Moderate | Moderate | Lower for devs |
| Drift detection | `plan` | Drift detection feature | `preview` |

---

## Key Resources

- **Terraform Docs** — https://developer.hashicorp.com/terraform/docs
- **Terraform Registry** — https://registry.terraform.io (modules & providers)
- **Terraform Up & Running (book)** — Yevgeniy Brikman (THE Terraform book)
- **Terraform Best Practices** — https://www.terraform-best-practices.com
- **HashiCorp Certified: Terraform Associate** — Great cert for structured learning
- **tfsec** — Static security analysis
- **Infracost** — Cloud cost estimates from Terraform plans
- **Spacelift / Terraform Cloud / Atlantis** — Collaboration platforms
