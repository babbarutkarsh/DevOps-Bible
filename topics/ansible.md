---
title: Ansible
nav_order: 31
description: "Ansible — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# Ansible & Configuration Management — DevOps Interview Preparation

> **Question frequency:** 🔥 very frequently asked · ⭐ important · 💡 good to know

## Table of Contents
- [Configuration Management Fundamentals](#configuration-management-fundamentals)
- [Ansible Architecture](#ansible-architecture)
- [Playbooks & Roles](#playbooks--roles)
- [Inventory Management](#inventory-management)
- [Variables & Facts](#variables--facts)
- [Ansible Vault](#ansible-vault)
- [Modules & Plugins](#modules--plugins)
- [Advanced Patterns](#advanced-patterns)
- [Performance & Scale](#performance--scale)
- [Testing & Validation](#testing--validation)
- [Ansible vs Others](#ansible-vs-others)
- [Scenario-Based Questions](#scenario-based-questions)
- [Modern Ansible (2025-2026)](#modern-ansible-2025-2026)

---

## Configuration Management Fundamentals

### 🔥 Q: Why configuration management?

**Problem:** Manual server configuration leads to:
- Configuration drift (servers diverge over time)
- Snowflake servers (unique, unreproducible)
- Slow provisioning, human errors

**Solution:** Define desired state in code, apply consistently.

| Tool | Language | Architecture | Best For |
|------|----------|-------------|----------|
| **Ansible** | YAML/Python | Agentless (SSH) | Config mgmt, orchestration, ad-hoc |
| **Puppet** | Puppet DSL | Agent-based | Large-scale config management |
| **Chef** | Ruby | Agent-based | Complex config, dev-friendly |
| **SaltStack** | YAML/Python | Agent or agentless | Speed, event-driven |

---

## Ansible Architecture

### 🔥 Q: How does Ansible work?

```
┌─── Control Node ────────────────────────────┐
│  ansible / ansible-playbook                   │
│  ┌───────────┐  ┌──────────┐  ┌───────────┐ │
│  │ Inventory │  │Playbooks │  │  Modules   │ │
│  │ (hosts)   │  │ (YAML)   │  │ (Python)   │ │
│  └───────────┘  └──────────┘  └───────────┘ │
└──────────────────────┬──────────────────────┘
                       │ SSH (no agent needed!)
          ┌────────────┼────────────┐
          ▼            ▼            ▼
    ┌──────────┐ ┌──────────┐ ┌──────────┐
    │ Managed  │ │ Managed  │ │ Managed  │
    │ Node 1   │ │ Node 2   │ │ Node 3   │
    └──────────┘ └──────────┘ └──────────┘

Process:
1. Ansible reads inventory (which hosts)
2. Reads playbook (what to do)
3. Generates Python scripts from modules
4. Copies scripts to remote hosts via SSH
5. Executes scripts on remote hosts
6. Collects results and reports
7. Removes temporary files
```

**Key advantages of agentless:**
- No software to install/maintain on managed nodes
- No daemon, no ports, no agent upgrades
- Uses existing SSH infrastructure
- Lower attack surface

### 🔥 Q: What is idempotency in Ansible?

Running the same playbook multiple times produces the same result. If the desired state is already achieved, Ansible makes no changes.

```yaml
# Idempotent — safe to run repeatedly
- name: Ensure nginx is installed
  apt:
    name: nginx
    state: present      # Won't reinstall if already present

# NOT idempotent (avoid)
- name: Append to config
  shell: echo "line" >> /etc/config
  # This adds a new line EVERY time you run it!

# Fixed with lineinfile (idempotent)
- name: Ensure line in config
  lineinfile:
    path: /etc/config
    line: "line"
    state: present
```

**How modules ensure idempotency:**
- Check current state before making changes
- Compare desired state vs. current state
- Only execute if change is needed
- Return `changed: false` if already compliant

**Check mode (dry-run):**
```bash
# Test what would change without applying
ansible-playbook site.yml --check

# Combine with diff to see proposed changes
ansible-playbook site.yml --check --diff
```

---

## Playbooks & Roles

### 🔥 Q: Playbook structure and example.

```yaml
# site.yml — Main playbook
---
- name: Configure web servers
  hosts: webservers
  become: yes                    # Run as root (sudo)
  gather_facts: yes
  vars:
    app_port: 8080
    app_user: appuser

  pre_tasks:
  - name: Update apt cache
    apt:
      update_cache: yes
      cache_valid_time: 3600

  roles:
  - common
  - nginx
  - app

  tasks:
  - name: Ensure app user exists
    user:
      name: "{{ app_user }}"
      shell: /bin/bash
      state: present

  - name: Deploy application config
    template:
      src: app.conf.j2
      dest: /etc/app/config.yml
      owner: "{{ app_user }}"
      mode: '0644'
    notify: restart app

  handlers:
  - name: restart app
    systemd:
      name: myapp
      state: restarted
```

### ⭐ Q: Ansible Role structure.

```
roles/
└── nginx/
    ├── tasks/
    │   └── main.yml          # Main task list
    ├── handlers/
    │   └── main.yml          # Handlers (restart, reload)
    ├── templates/
    │   └── nginx.conf.j2     # Jinja2 templates
    ├── files/
    │   └── ssl.crt           # Static files
    ├── vars/
    │   └── main.yml          # Role variables (high priority)
    ├── defaults/
    │   └── main.yml          # Default variables (low priority, overridable)
    ├── meta/
    │   └── main.yml          # Dependencies, metadata
    └── README.md
```

```yaml
# roles/nginx/tasks/main.yml
---
- name: Install nginx
  apt:
    name: nginx
    state: present

- name: Deploy nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: reload nginx

- name: Ensure nginx is running
  systemd:
    name: nginx
    state: started
    enabled: yes

# roles/nginx/handlers/main.yml
---
- name: reload nginx
  systemd:
    name: nginx
    state: reloaded

# roles/nginx/defaults/main.yml
---
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_keepalive_timeout: 65

# roles/nginx/meta/main.yml
---
dependencies:
  - role: common
  - role: firewall
    vars:
      firewall_allowed_tcp_ports: [80, 443]
```

### ⭐ Q: How to use Ansible Galaxy for role reuse?

```bash
# Install role from Galaxy
ansible-galaxy install geerlingguy.nginx

# Install from requirements.yml
cat > requirements.yml <<EOF
roles:
  - name: geerlingguy.nginx
    version: 3.1.4
  - src: https://github.com/myorg/custom-role.git
    name: custom-role
    version: main
EOF

ansible-galaxy install -r requirements.yml

# Create new role skeleton
ansible-galaxy init my-role

# Search for roles
ansible-galaxy search nginx --platforms EL
```

**Using installed roles:**
```yaml
# site.yml
- hosts: webservers
  roles:
    - geerlingguy.nginx      # From Galaxy
    - custom-role             # From Git
    - role: my-local-role     # From roles/ directory
      vars:
        custom_var: value
```

---

## Inventory Management

### 🔥 Q: Static vs Dynamic Inventory.

**Static inventory (INI or YAML):**
```ini
# inventory/production.ini
[webservers]
web1.example.com ansible_host=10.0.1.10
web2.example.com ansible_host=10.0.1.11

[databases]
db1.example.com ansible_host=10.0.2.10

[all:vars]
ansible_user=deploy
ansible_ssh_private_key_file=~/.ssh/deploy_key

[production:children]
webservers
databases
```

**Dynamic inventory** — Query cloud APIs for current hosts:
```bash
# AWS EC2 plugin
# inventory/aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
filters:
  tag:Environment: production
keyed_groups:
  - key: tags.Role
    prefix: role
  - key: placement.availability_zone
    prefix: az
compose:
  ansible_host: private_ip_address
```

```bash
# Use dynamic inventory
ansible-playbook -i inventory/aws_ec2.yml site.yml

# List discovered hosts
ansible-inventory -i inventory/aws_ec2.yml --list
```

### ⭐ Q: Inventory groups and host/group variables.

```yaml
# inventory/production/hosts.yml
all:
  children:
    production:
      children:
        webservers:
          hosts:
            web1.prod.com:
              ansible_host: 10.0.1.10
            web2.prod.com:
              ansible_host: 10.0.1.11
        databases:
          hosts:
            db1.prod.com:
              ansible_host: 10.0.2.10
              mysql_port: 3306

# inventory/production/group_vars/all.yml
ansible_user: deploy
ansible_ssh_private_key_file: ~/.ssh/prod_key
ntp_server: ntp.prod.internal

# inventory/production/group_vars/webservers.yml
nginx_version: 1.24.0
app_port: 8080

# inventory/production/host_vars/web1.prod.com.yml
nginx_worker_connections: 2048    # Override for this specific host
```

**Common inventory patterns:**
```bash
# Target specific groups
ansible-playbook -i inventory site.yml --limit webservers

# Target patterns
ansible all -i inventory -m ping                    # All hosts
ansible 'web*' -i inventory -m ping                 # Wildcard
ansible webservers:databases -i inventory -m ping   # Union (OR)
ansible webservers:&production -i inventory -m ping # Intersection (AND)
ansible webservers:!staging -i inventory -m ping    # Exclude (NOT)
```

---

## Variables & Facts

### ⭐ Q: Variable precedence (lowest to highest).

```
1.  Role defaults (roles/x/defaults/main.yml)
2.  Inventory group_vars/all
3.  Inventory group_vars/<group>
4.  Inventory host_vars/<host>
5.  Playbook group_vars/all
6.  Playbook group_vars/<group>
7.  Playbook host_vars/<host>
8.  Play vars
9.  Play vars_prompt
10. Play vars_files
11. Role vars (roles/x/vars/main.yml)
12. Block vars
13. Task vars
14. set_fact / registered vars
15. Extra vars (-e) ← HIGHEST PRIORITY (always wins)
```

### ⭐ Q: What are Ansible Facts?

Facts are system information gathered automatically from managed nodes:

```yaml
# Access facts in playbooks
- debug:
    msg: "OS: {{ ansible_distribution }} {{ ansible_distribution_version }}"
    # → "OS: Ubuntu 22.04"

- debug:
    msg: "IP: {{ ansible_default_ipv4.address }}"
    # → "IP: 10.0.1.10"

- debug:
    msg: "RAM: {{ ansible_memtotal_mb }} MB"
    # → "RAM: 16384 MB"

# Disable fact gathering (faster for tasks that don't need it)
- hosts: all
  gather_facts: no

# Custom facts: place in /etc/ansible/facts.d/*.fact on managed nodes
```

### ⭐ Q: Using register and set_fact.

```yaml
# Register captures task output
- name: Check if app is running
  command: pgrep -f myapp
  register: app_running
  ignore_errors: yes

- name: Start app if not running
  systemd:
    name: myapp
    state: started
  when: app_running.rc != 0

# set_fact creates custom variables
- name: Calculate memory threshold
  set_fact:
    memory_threshold: "{{ (ansible_memtotal_mb * 0.8) | int }}"

- debug:
    msg: "Memory threshold: {{ memory_threshold }} MB"

# Registered variables persist across tasks
- name: Get current git commit
  command: git rev-parse HEAD
  register: git_commit
  changed_when: false

- name: Deploy with git commit in config
  template:
    src: version.j2
    dest: /etc/app/version.txt
  vars:
    version: "{{ git_commit.stdout }}"
```

---

## Ansible Vault

### 🔥 Q: How to manage secrets in Ansible?

```bash
# Create encrypted file
ansible-vault create secrets.yml

# Encrypt existing file
ansible-vault encrypt vars/passwords.yml

# Edit encrypted file
ansible-vault edit secrets.yml

# Decrypt
ansible-vault decrypt secrets.yml

# View without decrypting
ansible-vault view secrets.yml

# Run playbook with vault password
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass

# Encrypt single variable
ansible-vault encrypt_string 'mysecret' --name 'db_password'
# Produces:
# db_password: !vault |
#   $ANSIBLE_VAULT;1.1;AES256
#   3838...encrypted...3838
```

```yaml
# Use in playbook
vars_files:
  - vars/secrets.yml    # Vault-encrypted file

tasks:
- name: Configure database
  template:
    src: db.conf.j2
    dest: /etc/app/db.conf
  vars:
    password: "{{ db_password }}"    # From encrypted file
```

### 💡 Q: Managing secrets across environments.

```yaml
# Structure for multi-environment secrets
inventory/
├── production/
│   ├── hosts.yml
│   └── group_vars/
│       └── all/
│           ├── vars.yml              # Unencrypted vars
│           └── vault.yml             # Encrypted secrets (ansible-vault)
├── staging/
│   └── group_vars/
│       └── all/
│           ├── vars.yml
│           └── vault.yml
└── vault_pass.sh                     # Script to fetch vault password

# vault_pass.sh (executable)
#!/bin/bash
# Fetch from AWS Secrets Manager, HashiCorp Vault, etc.
aws secretsmanager get-secret-value --secret-id ansible-vault-prod \
  --query SecretString --output text

# Use with playbook
ansible-playbook -i inventory/production site.yml \
  --vault-password-file inventory/vault_pass.sh
```

**Multiple vault IDs for different secrets:**
```bash
# Encrypt with labeled vault IDs
ansible-vault encrypt --vault-id dev@prompt secrets_dev.yml
ansible-vault encrypt --vault-id prod@prompt secrets_prod.yml

# Run with multiple vault passwords
ansible-playbook site.yml \
  --vault-id dev@~/.vault_pass_dev \
  --vault-id prod@~/.vault_pass_prod
```

---

## Modules & Plugins

### 🔥 Q: Most important Ansible modules.

| Category | Module | Purpose |
|----------|--------|---------|
| **Package** | `apt`, `yum`, `dnf`, `pip` | Install packages |
| **File** | `file`, `copy`, `template`, `lineinfile` | Manage files |
| **Service** | `systemd`, `service` | Manage services |
| **User** | `user`, `group`, `authorized_key` | User management |
| **Command** | `command`, `shell`, `raw`, `script` | Execute commands |
| **Cloud** | `ec2`, `s3_bucket`, `rds` | AWS resource management |
| **K8s** | `k8s`, `helm` | Kubernetes management |
| **Network** | `uri`, `get_url`, `firewalld` | Network tasks |
| **Archive** | `unarchive`, `archive` | Compression |
| **Crypto** | `openssl_certificate`, `openssl_privatekey` | TLS certs |

### ⭐ Q: command vs shell vs raw?

| Module | Shell | Pipes/Redirects | Use When |
|--------|-------|----------------|----------|
| `command` | No (`/bin/sh`) | No | Simple commands, more secure |
| `shell` | Yes (`/bin/sh -c`) | Yes | Need pipes, env vars, redirects |
| `raw` | SSH directly | N/A | No Python on target (bootstrap) |

**Rule:** Prefer specific modules (`apt`, `copy`, `template`) over `command`/`shell`. They're idempotent and handle edge cases.

---

## Advanced Patterns

### ⭐ Q: Ansible strategies and error handling.

```yaml
# Block/Rescue/Always (try/catch/finally)
- block:
  - name: Try to deploy
    command: /opt/deploy.sh
  rescue:
  - name: Rollback on failure
    command: /opt/rollback.sh
  always:
  - name: Send notification
    slack:
      msg: "Deployment {{ 'succeeded' if not ansible_failed_task else 'failed' }}"

# Conditionals
- name: Install on Debian only
  apt: name=nginx state=present
  when: ansible_os_family == "Debian"

# Loops
- name: Create users
  user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
  loop:
    - { name: alice, groups: admin }
    - { name: bob, groups: developers }

# Delegate to another host
- name: Remove from load balancer
  uri:
    url: "http://lb.example.com/api/deregister/{{ inventory_hostname }}"
  delegate_to: localhost

# Rolling updates
- hosts: webservers
  serial: "25%"          # Update 25% at a time
  max_fail_percentage: 0  # Stop if any host fails
```

### 💡 Q: Advanced error handling patterns.

```yaml
# failed_when — define custom failure conditions
- name: Check application health
  uri:
    url: http://localhost:8080/health
  register: health_check
  failed_when: 
    - health_check.status != 200
    - "'healthy' not in health_check.json.status"

# changed_when — control when task reports changed
- name: Get current config
  command: cat /etc/app/config
  register: config
  changed_when: false    # Never report as changed

# ignore_errors — continue even if task fails
- name: Try to stop old service
  systemd:
    name: old-service
    state: stopped
  ignore_errors: yes

# any_errors_fatal — stop all hosts on any failure
- hosts: databases
  any_errors_fatal: yes
  tasks:
    - name: Critical database migration
      command: /opt/migrate.sh

# Tags for selective execution
- name: Deploy application
  copy:
    src: app.jar
    dest: /opt/app/
  tags:
    - deploy
    - app

- name: Configure monitoring
  template:
    src: metrics.yml.j2
    dest: /etc/metrics.yml
  tags:
    - monitoring
    - config

# Run only tagged tasks
# ansible-playbook site.yml --tags "deploy"
# ansible-playbook site.yml --skip-tags "monitoring"
```

### 💡 Q: run_once and delegation patterns.

```yaml
# run_once — execute task on only one host in the group
- name: Create shared database
  postgresql_db:
    name: myapp
    state: present
  run_once: yes
  delegate_to: "{{ groups['databases'][0] }}"

# Loop over all hosts from one controller
- name: Update DNS for all web servers
  route53:
    zone: example.com
    record: "{{ item }}.example.com"
    type: A
    value: "{{ hostvars[item].ansible_default_ipv4.address }}"
  loop: "{{ groups['webservers'] }}"
  delegate_to: localhost
  run_once: yes

# Delegate facts to other hosts
- name: Get LB IP
  set_fact:
    lb_ip: "{{ hostvars['lb1.example.com'].ansible_default_ipv4.address }}"
  delegate_facts: yes
  delegate_to: "{{ item }}"
  loop: "{{ groups['webservers'] }}"
```

### 💡 Q: Ansible with Kubernetes.

```yaml
- name: Deploy to Kubernetes
  hosts: localhost
  tasks:
  - name: Deploy application
    kubernetes.core.k8s:
      state: present
      definition:
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: myapp
          namespace: production
        spec:
          replicas: 3
          selector:
            matchLabels:
              app: myapp
          template:
            metadata:
              labels:
                app: myapp
            spec:
              containers:
              - name: app
                image: "myapp:{{ app_version }}"

  - name: Install Helm chart
    kubernetes.core.helm:
      name: prometheus
      chart_ref: prometheus-community/kube-prometheus-stack
      release_namespace: monitoring
      create_namespace: yes
      values_files:
      - values/prometheus.yml
```

---

## Performance & Scale

### ⭐ Q: How to optimize Ansible for large-scale deployments?

```ini
# ansible.cfg
[defaults]
forks = 50                      # Parallel hosts (default: 5)
host_key_checking = False       # Skip SSH host key check
gathering = smart               # Smart fact caching
fact_caching = jsonfile
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 86400    # 24 hours

[ssh_connection]
pipelining = True               # Reduce SSH operations (requires no requiretty in sudoers)
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
control_path = ~/.ansible/cp/%%h-%%p-%%r
```

**Async tasks for long-running operations:**
```yaml
- name: Run long deployment script
  command: /opt/deploy-app.sh
  async: 3600              # Max runtime (1 hour)
  poll: 0                  # Don't wait, fire-and-forget
  register: deploy_job

# Later, check status
- name: Wait for deployment to complete
  async_status:
    jid: "{{ deploy_job.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 60
  delay: 10
```

**Strategies for parallelism:**
```yaml
# Linear (default) — wait for all hosts before next task
- hosts: all
  strategy: linear

# Free — hosts proceed independently (fastest)
- hosts: all
  strategy: free

# Batch processing
- hosts: all
  serial:
    - 1          # First host
    - "10%"      # Then 10% of remaining
    - "25%"      # Then 25% of remaining
    - "100%"     # Then all remaining
```

### 💡 Q: Performance tuning best practices.

| Technique | Configuration | Impact |
|-----------|--------------|--------|
| **SSH pipelining** | `pipelining = True` | 25-50% faster |
| **ControlPersist** | `ssh_args = -o ControlPersist=60s` | Reuse SSH connections |
| **Increase forks** | `forks = 50` (or more) | More parallel execution |
| **Disable gather_facts** | `gather_facts: no` | Skip if not needed |
| **Fact caching** | `fact_caching = jsonfile` | Avoid re-gathering |
| **Async tasks** | `async: 3600` + `poll: 0` | Non-blocking long tasks |
| **Free strategy** | `strategy: free` | No inter-host waiting |
| **Mitogen plugin** | `strategy = mitogen_linear` | 3-7x faster overall |

```yaml
# Profile playbook execution
ANSIBLE_CALLBACK_WHITELIST=profile_tasks ansible-playbook site.yml

# Shows timing for each task — helps identify bottlenecks
```

---

## Testing & Validation

### ⭐ Q: How to test Ansible playbooks?

**Molecule — testing framework for roles:**
```bash
# Install
pip install molecule molecule-docker

# Initialize new role with molecule
molecule init role my-role --driver-name docker

# Molecule directory structure
my-role/
├── molecule/
│   └── default/
│       ├── molecule.yml       # Molecule config
│       ├── converge.yml       # Playbook to test
│       └── verify.yml         # Verification tasks
├── tasks/
└── defaults/
```

```yaml
# molecule/default/molecule.yml
---
driver:
  name: docker
platforms:
  - name: ubuntu2204
    image: geerlingguy/docker-ubuntu2204-ansible
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: ansible

# molecule/default/converge.yml
---
- name: Converge
  hosts: all
  roles:
    - role: my-role

# molecule/default/verify.yml
---
- name: Verify
  hosts: all
  tasks:
    - name: Check nginx is running
      systemd:
        name: nginx
      register: nginx_status
    - assert:
        that: nginx_status.status.ActiveState == 'active'
```

**Molecule test cycle:**
```bash
molecule create       # Create test instances
molecule converge     # Run the role
molecule verify       # Run verification tests
molecule destroy      # Clean up
molecule test         # Full cycle: create → converge → verify → destroy

# Test against multiple OS
molecule test --all
```

### ⭐ Q: Linting and validation.

```bash
# ansible-lint — best practices checker
pip install ansible-lint
ansible-lint playbook.yml

# yamllint — YAML syntax checker
pip install yamllint
yamllint playbook.yml

# Syntax check
ansible-playbook --syntax-check site.yml

# Dry-run with diff
ansible-playbook --check --diff site.yml

# CI/CD integration
# .github/workflows/ansible-test.yml
name: Ansible Tests
on: [push]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run ansible-lint
        run: ansible-lint playbooks/
      - name: Run molecule tests
        run: molecule test
```

---

## Ansible vs Others

### 🔥 Q: When to use Ansible vs Terraform?

| Aspect | Ansible | Terraform |
|--------|---------|-----------|
| Primary use | **Configuration management** | **Infrastructure provisioning** |
| Approach | Procedural (tasks in order) | Declarative (desired state) |
| State | Stateless | Stateful (state file) |
| Agents | Agentless (SSH) | Agentless (API calls) |
| Best for | Install packages, configure servers, deploy apps | Create VPCs, EC2, RDS, K8s clusters |
| Drift detection | Re-run playbook | `terraform plan` |
| Mutable infra | Yes (updates in place) | No (immutable, replace) |

**Use both together:**
- **Terraform** provisions infrastructure (VPC, EC2, RDS)
- **Ansible** configures the servers (install packages, deploy app, harden OS)

```bash
# Typical workflow
terraform apply                           # Provision EC2 instances
terraform output -json > inventory.json   # Export IPs
ansible-playbook -i inventory.json site.yml  # Configure servers
```

### 🔥 Q: Ansible vs Chef vs Puppet — when to use each?

| Feature | Ansible | Chef | Puppet |
|---------|---------|------|--------|
| Architecture | **Agentless (SSH push)** | Agent-based (pull) | Agent-based (pull) |
| Language | YAML | Ruby DSL | Puppet DSL |
| Learning curve | Low | High | Medium |
| Setup | Minimal (SSH only) | Master + agents | Master + agents |
| Speed | Slower (SSH overhead) | Fast (local agent) | Fast (local agent) |
| Windows support | Yes (WinRM) | Yes | Yes |
| Best for | Ad-hoc, small-to-mid scale, simplicity | Dev-friendly, complex logic | Enterprise, compliance |
| State | Stateless | Convergence-based | Catalog-based |

**When to choose:**
- **Ansible** — Quick setup, no agents, simplicity, orchestration
- **Chef** — Complex configuration logic, dev-heavy teams, test-driven infra
- **Puppet** — Large scale (1000+ nodes), compliance/reporting, enterprise support

### ⭐ Q: Push vs Pull architecture trade-offs.

**Ansible (Push):**
- ✅ Control node initiates changes (better control)
- ✅ No agents to maintain on nodes
- ✅ Immediate execution when you run playbook
- ❌ Control node must reach all managed nodes
- ❌ Doesn't auto-correct drift (must re-run)

**Chef/Puppet (Pull):**
- ✅ Agents self-heal (auto-correct drift)
- ✅ Distributed load (agents run independently)
- ✅ Works behind firewalls (agents pull from master)
- ❌ Agent software to install/maintain
- ❌ Agent can fail, lag, or miss updates
- ❌ More complex setup (master + agents + CA)

---

## Scenario-Based Questions

### 🔥 Q: How do you handle rolling deployments with zero downtime using Ansible?

```yaml
- hosts: webservers
  serial: 1                   # One server at a time
  max_fail_percentage: 0

  pre_tasks:
  - name: Deregister from load balancer
    uri:
      url: "http://lb/api/deregister/{{ inventory_hostname }}"
    delegate_to: localhost

  - name: Wait for connections to drain
    pause:
      seconds: 30

  roles:
  - deploy-app

  post_tasks:
  - name: Health check
    uri:
      url: "http://{{ inventory_hostname }}:8080/health"
      status_code: 200
    retries: 10
    delay: 5

  - name: Register with load balancer
    uri:
      url: "http://lb/api/register/{{ inventory_hostname }}"
    delegate_to: localhost
```

### ⭐ Q: Deploy to 50 servers with zero downtime and automated rollback.

```yaml
- hosts: webservers
  serial:
    - 1              # Canary (first server)
    - "10%"          # 10% batch
    - "25%"          # 25% batch
    - "100%"         # Remaining
  max_fail_percentage: 10    # Rollback if >10% fail

  vars:
    app_version: "{{ version | default('latest') }}"
    health_check_retries: 10
    rollback_on_failure: yes

  pre_tasks:
  - name: Remove from LB
    uri:
      url: "http://lb.internal/api/pool/remove/{{ inventory_hostname }}"
      method: POST
    delegate_to: localhost

  - name: Drain connections
    wait_for:
      timeout: 30

  tasks:
  - block:
    - name: Stop application
      systemd:
        name: myapp
        state: stopped

    - name: Backup current version
      command: "cp -r /opt/app /opt/app.backup.{{ ansible_date_time.epoch }}"

    - name: Deploy new version
      unarchive:
        src: "https://artifacts.internal/myapp-{{ app_version }}.tar.gz"
        dest: /opt/app
        remote_src: yes

    - name: Start application
      systemd:
        name: myapp
        state: started

    - name: Health check
      uri:
        url: "http://{{ inventory_hostname }}:8080/health"
        status_code: 200
      register: health
      until: health.status == 200
      retries: "{{ health_check_retries }}"
      delay: 5

    rescue:
    - name: Rollback to previous version
      command: "rsync -a --delete /opt/app.backup.*/ /opt/app/"
      when: rollback_on_failure

    - name: Restart with old version
      systemd:
        name: myapp
        state: restarted
      when: rollback_on_failure

    - fail:
        msg: "Deployment failed, rollback complete"

  post_tasks:
  - name: Add back to LB
    uri:
      url: "http://lb.internal/api/pool/add/{{ inventory_hostname }}"
      method: POST
    delegate_to: localhost

  - name: Cleanup old backups
    shell: "ls -t /opt/app.backup.* | tail -n +4 | xargs rm -rf"
```

### ⭐ Q: Make a non-idempotent shell task idempotent.

**Problem:**
```yaml
# NOT idempotent — runs every time
- name: Add cron job
  shell: echo "0 2 * * * /opt/backup.sh" >> /etc/crontab
```

**Solutions:**
```yaml
# Solution 1: Use appropriate module
- name: Add cron job (idempotent)
  cron:
    name: "Nightly backup"
    minute: "0"
    hour: "2"
    job: "/opt/backup.sh"
    user: root

# Solution 2: Use lineinfile
- name: Add line to config
  lineinfile:
    path: /etc/crontab
    line: "0 2 * * * /opt/backup.sh"
    state: present

# Solution 3: Check before executing
- name: Check if config exists
  stat:
    path: /etc/app/initialized
  register: app_init

- name: Initialize app (only once)
  shell: /opt/initialize.sh
  when: not app_init.stat.exists

- name: Mark as initialized
  file:
    path: /etc/app/initialized
    state: touch
  when: not app_init.stat.exists

# Solution 4: Use changed_when for reporting
- name: Idempotent command
  shell: |
    grep -q "setting=value" /etc/config || \
    echo "setting=value" >> /etc/config
  register: result
  changed_when: "'setting=value' not in result.stdout"
```

### ⭐ Q: Ansible playbook is slow — how to speed it up?

**Diagnosis:**
```bash
# Profile tasks to find bottlenecks
ANSIBLE_CALLBACK_WHITELIST=profile_tasks ansible-playbook site.yml
```

**Common bottlenecks and fixes:**
```yaml
# 1. Unnecessary fact gathering (5-10s per host)
- hosts: all
  gather_facts: no    # Skip if not needed

# 2. Low parallelism (default forks=5)
# ansible.cfg
[defaults]
forks = 50

# 3. No SSH pipelining
[ssh_connection]
pipelining = True

# 4. Not using ControlPersist
ssh_args = -o ControlMaster=auto -o ControlPersist=60s

# 5. Long tasks blocking others
- name: Long database restore
  command: /opt/restore-db.sh
  async: 3600
  poll: 0
  register: restore_job

# 6. Serial execution when free would work
- hosts: all
  strategy: free      # Hosts don't wait for each other

# 7. Package cache not updated efficiently
- name: Update cache once
  apt:
    update_cache: yes
    cache_valid_time: 3600    # Cache for 1 hour

# 8. Repeated template rendering
- name: Deploy config
  template:
    src: config.j2
    dest: /etc/app/config
  run_once: yes
  delegate_to: localhost

- name: Copy to all hosts
  copy:
    src: /tmp/config
    dest: /etc/app/config
```

### 💡 Q: Manage 200 servers across 3 environments (dev/staging/prod).

**Structure:**
```
ansible-project/
├── ansible.cfg
├── inventory/
│   ├── dev/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       ├── all.yml
│   │       └── webservers.yml
│   ├── staging/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       └── all.yml
│   └── production/
│       ├── hosts.yml
│       ├── group_vars/
│       │   ├── all.yml
│       │   └── all/
│       │       ├── vars.yml
│       │       └── vault.yml        # Encrypted secrets
│       └── host_vars/
│           └── web-prod-01.yml
├── roles/
│   ├── common/
│   ├── nginx/
│   └── app/
├── playbooks/
│   ├── site.yml
│   ├── webservers.yml
│   └── databases.yml
└── requirements.yml
```

```yaml
# inventory/production/hosts.yml
all:
  children:
    webservers:
      hosts:
        web-prod-[01:10].example.com:    # 10 web servers
    appservers:
      hosts:
        app-prod-[01:15].example.com:    # 15 app servers
    databases:
      hosts:
        db-prod-primary.example.com:
        db-prod-replica-[01:02].example.com:

# inventory/production/group_vars/all.yml
environment: production
ansible_user: deploy
ansible_become: yes
ntp_server: ntp.prod.internal
monitoring_enabled: yes
log_level: warn

# inventory/dev/group_vars/all.yml
environment: development
ansible_user: ubuntu
ntp_server: ntp.dev.internal
monitoring_enabled: no
log_level: debug

# Run against specific environment
ansible-playbook -i inventory/production playbooks/site.yml

# Deploy only web servers in staging
ansible-playbook -i inventory/staging playbooks/webservers.yml --limit staging-webservers
```

### 💡 Q: A task must run only once across a 10-server cluster.

```yaml
# run_once executes on first host only
- name: Initialize shared database
  postgresql_db:
    name: myapp
    state: present
  run_once: yes
  delegate_to: "{{ groups['databases'][0] }}"

# Create cluster-wide lock
- name: Acquire cluster lock
  shell: |
    if mkdir /tmp/cluster.lock 2>/dev/null; then
      echo "lock_acquired"
    else
      echo "lock_exists"
    fi
  register: lock
  delegate_to: "{{ groups['all'][0] }}"
  run_once: yes

- name: Run migration (only if lock acquired)
  command: /opt/run-migration.sh
  when: lock.stdout == "lock_acquired"
  run_once: yes
  delegate_to: "{{ groups['all'][0] }}"

- name: Release lock
  file:
    path: /tmp/cluster.lock
    state: absent
  delegate_to: "{{ groups['all'][0] }}"
  run_once: yes
```

### ⭐ Q: Dynamic inventory from AWS — how to configure and use?

```yaml
# inventory/aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
  - us-west-2
filters:
  tag:Managed: ansible
  instance-state-name: running
keyed_groups:
  - key: tags.Environment
    prefix: env
  - key: tags.Role
    prefix: role
  - key: placement.availability_zone
    prefix: az
compose:
  ansible_host: private_ip_address
  ansible_user: "'ec2-user'"
hostnames:
  - tag:Name
  - private-ip-address
```

```bash
# List discovered hosts
ansible-inventory -i inventory/aws_ec2.yml --list
ansible-inventory -i inventory/aws_ec2.yml --graph

# Run playbook
ansible-playbook -i inventory/aws_ec2.yml site.yml

# Target dynamically created groups
ansible-playbook -i inventory/aws_ec2.yml site.yml --limit env_production
ansible-playbook -i inventory/aws_ec2.yml site.yml --limit role_webserver
```

```yaml
# Use in playbook
- hosts: role_webserver:&env_production    # Intersection
  tasks:
    - name: Deploy to production web servers
      # ...
```

---

## Modern Ansible (2025-2026)

### ⭐ Q: What are Ansible Collections?

Collections are distribution format for Ansible content (roles, modules, plugins). Replaced the monolithic `ansible` package.

```bash
# Install collections
ansible-galaxy collection install community.general
ansible-galaxy collection install amazon.aws

# Install from requirements
cat > requirements.yml <<EOF
collections:
  - name: community.general
    version: ">=6.0.0"
  - name: kubernetes.core
    version: 2.4.0
  - name: ansible.posix
EOF

ansible-galaxy collection install -r requirements.yml
```

```yaml
# Use collection modules in playbooks
- hosts: all
  tasks:
    - name: Manage AWS resources
      amazon.aws.ec2_instance:
        state: present
        instance_type: t3.medium
        image_id: ami-12345678

    - name: Deploy to Kubernetes
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Pod
          # ...

# Or use FQCN (Fully Qualified Collection Name)
- community.general.docker_container:
    name: myapp
    image: nginx:latest
```

**Collection structure:**
```
my_namespace.my_collection/
├── plugins/
│   ├── modules/
│   ├── inventory/
│   ├── lookup/
│   └── filter/
├── roles/
├── playbooks/
└── galaxy.yml
```

### 💡 Q: What is AWX / Ansible Automation Platform?

**AWX** (open-source) / **Ansible Automation Platform** (Red Hat commercial):
- Web UI for running playbooks
- RBAC (role-based access control)
- Inventory management (static, dynamic, synced)
- Credential management (vault-encrypted)
- Job scheduling and workflows
- API for CI/CD integration
- Audit logging
- Multi-tenancy

```bash
# Install AWX (Kubernetes)
helm repo add awx-operator https://ansible.github.io/awx-operator/
helm install awx-operator awx-operator/awx-operator -n awx --create-namespace

# Create AWX instance
kubectl apply -f - <<EOF
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  service_type: NodePort
EOF
```

**API usage:**
```bash
# Launch job via API
curl -X POST https://awx.example.com/api/v2/job_templates/10/launch/ \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"extra_vars": "{\"version\": \"1.2.3\"}"}'

# Check job status
curl https://awx.example.com/api/v2/jobs/123/ \
  -H "Authorization: Bearer $TOKEN"
```

### 💡 Q: Execution Environments (EE) in modern Ansible.

**Problem:** Dependency hell (Python packages, collections, OS packages).

**Solution:** Containerized execution environments with all dependencies.

```yaml
# execution-environment.yml
---
version: 3
build_arg_defaults:
  ANSIBLE_GALAXY_CLI_COLLECTION_OPTS: '-v'

dependencies:
  galaxy: requirements.yml
  python: requirements.txt
  system: bindep.txt

images:
  base_image:
    name: quay.io/ansible/creator-base:latest

additional_build_steps:
  append_final:
    - RUN pip install netaddr jmespath
    - COPY custom-ca.crt /etc/pki/ca-trust/source/anchors/
```

```bash
# Build EE with ansible-builder
pip install ansible-builder
ansible-builder build --tag my-ee:1.0

# Use with ansible-navigator
ansible-navigator run site.yml --execution-environment-image my-ee:1.0

# Use with AWX
# Upload image to registry, configure in AWX
```

### 💡 Q: Ansible and GitOps — best practices.

```
ansible-gitops/
├── .github/
│   └── workflows/
│       ├── lint.yml           # ansible-lint + yamllint
│       ├── test.yml           # molecule tests
│       └── deploy.yml         # Deploy on merge to main
├── inventory/
│   ├── production/
│   └── staging/
├── playbooks/
├── roles/
└── ansible.cfg
```

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run ansible-lint
        run: ansible-lint playbooks/

      - name: Dry-run
        run: ansible-playbook playbooks/site.yml --check
        env:
          ANSIBLE_VAULT_PASSWORD: ${{ secrets.VAULT_PASSWORD }}

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to staging
        run: ansible-playbook -i inventory/staging playbooks/site.yml
      
      - name: Smoke tests
        run: ansible-playbook playbooks/smoke-tests.yml

      - name: Deploy to production
        run: ansible-playbook -i inventory/production playbooks/site.yml
        env:
          ANSIBLE_VAULT_PASSWORD: ${{ secrets.VAULT_PASSWORD_PROD }}
```

### ⭐ Q: Using Ansible with Terraform — integration patterns.

**Pattern 1: Terraform → Ansible (provision then configure)**
```hcl
# terraform/main.tf
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-12345678"
  instance_type = "t3.medium"
  
  provisioner "local-exec" {
    command = "ansible-playbook -i '${self.private_ip},' ../ansible/configure.yml"
  }
}

# Export inventory for Ansible
output "inventory" {
  value = {
    webservers = {
      hosts = aws_instance.web[*].private_ip
    }
  }
}

resource "local_file" "ansible_inventory" {
  content = templatefile("inventory.tpl", {
    web_ips = aws_instance.web[*].private_ip
  })
  filename = "../ansible/inventory/terraform.yml"
}
```

**Pattern 2: Ansible with Terraform module**
```yaml
# Use Ansible to orchestrate Terraform
- name: Provision infrastructure
  community.general.terraform:
    project_path: ./terraform
    state: present
    variables:
      region: us-east-1
      instance_count: 3
  register: tf_output

- name: Configure servers
  hosts: "{{ tf_output.outputs.web_ips.value }}"
  tasks:
    - name: Install nginx
      apt: name=nginx state=present
```

**Pattern 3: Dynamic inventory from Terraform state**
```python
#!/usr/bin/env python3
# terraform-inventory.py
import json
import subprocess

result = subprocess.run(['terraform', 'output', '-json'], capture_output=True)
tf_output = json.loads(result.stdout)

inventory = {
    'webservers': {
        'hosts': tf_output['web_ips']['value']
    },
    '_meta': {
        'hostvars': {}
    }
}
print(json.dumps(inventory))
```

### 💡 Q: Ansible + Kubernetes — modern deployment patterns.

```yaml
# Deploy app with k8s module
- name: Deploy to Kubernetes
  hosts: localhost
  connection: local
  tasks:
    - name: Create namespace
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Namespace
          metadata:
            name: myapp

    - name: Deploy from kustomize
      kubernetes.core.k8s:
        state: present
        namespace: myapp
        src: k8s/overlays/production

    - name: Wait for deployment
      kubernetes.core.k8s_info:
        kind: Deployment
        name: myapp
        namespace: myapp
        wait: yes
        wait_condition:
          type: Available
          status: "True"
        wait_timeout: 300

    - name: Run database migration job
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: batch/v1
          kind: Job
          metadata:
            name: db-migrate
            namespace: myapp
          spec:
            template:
              spec:
                containers:
                - name: migrate
                  image: myapp:{{ version }}
                  command: ["python", "manage.py", "migrate"]
                restartPolicy: Never

# Manage multiple clusters
- name: Deploy to all regions
  hosts: localhost
  vars:
    clusters:
      - name: us-east-1
        context: prod-us-east-1
      - name: eu-west-1
        context: prod-eu-west-1
  tasks:
    - name: Deploy to cluster
      kubernetes.core.k8s:
        state: present
        kubeconfig: ~/.kube/config
        context: "{{ item.context }}"
        namespace: myapp
        src: deployment.yml
      loop: "{{ clusters }}"
```

---

## Key Resources

- **Ansible Documentation** — https://docs.ansible.com
- **Ansible for DevOps (book)** — Jeff Geerling
- **Ansible Galaxy** — https://galaxy.ansible.com (community roles)
- **Jeff Geerling's YouTube** — Excellent Ansible tutorials
- **Molecule** — Testing framework for Ansible roles
- **Ansible Collections Index** — https://docs.ansible.com/ansible/latest/collections/
- **AWX Project** — https://github.com/ansible/awx
- **ansible-builder** — Build execution environments
- **ansible-lint** — https://ansible-lint.readthedocs.io
- **Ansible Community Forum** — https://forum.ansible.com
