---
title: Ansible
nav_order: 31
description: "Ansible — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# Ansible & Configuration Management — DevOps Interview Preparation

## Table of Contents
- [Configuration Management Fundamentals](#configuration-management-fundamentals)
- [Ansible Architecture](#ansible-architecture)
- [Playbooks & Roles](#playbooks--roles)
- [Inventory Management](#inventory-management)
- [Variables & Facts](#variables--facts)
- [Ansible Vault](#ansible-vault)
- [Modules & Plugins](#modules--plugins)
- [Advanced Patterns](#advanced-patterns)
- [Ansible vs Others](#ansible-vs-others)
- [Scenario-Based Questions](#scenario-based-questions)

---

## Configuration Management Fundamentals

### Q: Why configuration management?

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

### Q: How does Ansible work?

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

### Q: What is idempotency in Ansible?

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

---

## Playbooks & Roles

### Q: Playbook structure and example.

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

### Q: Ansible Role structure.

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
```

---

## Inventory Management

### Q: Static vs Dynamic Inventory.

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

---

## Variables & Facts

### Q: Variable precedence (lowest to highest).

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

### Q: What are Ansible Facts?

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

---

## Ansible Vault

### Q: How to manage secrets in Ansible?

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

---

## Modules & Plugins

### Q: Most important Ansible modules.

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

### Q: command vs shell vs raw?

| Module | Shell | Pipes/Redirects | Use When |
|--------|-------|----------------|----------|
| `command` | No (`/bin/sh`) | No | Simple commands, more secure |
| `shell` | Yes (`/bin/sh -c`) | Yes | Need pipes, env vars, redirects |
| `raw` | SSH directly | N/A | No Python on target (bootstrap) |

**Rule:** Prefer specific modules (`apt`, `copy`, `template`) over `command`/`shell`. They're idempotent and handle edge cases.

---

## Advanced Patterns

### Q: Ansible strategies and error handling.

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

### Q: Ansible with Kubernetes.

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

## Ansible vs Others

### Q: When to use Ansible vs Terraform?

| Aspect | Ansible | Terraform |
|--------|---------|-----------|
| Primary use | **Configuration management** | **Infrastructure provisioning** |
| Approach | Procedural (tasks in order) | Declarative (desired state) |
| State | Stateless | Stateful (state file) |
| Agents | Agentless (SSH) | Agentless (API calls) |
| Best for | Install packages, configure servers, deploy apps | Create VPCs, EC2, RDS, K8s clusters |
| Drift detection | Re-run playbook | `terraform plan` |

**Use both together:**
- **Terraform** provisions infrastructure (VPC, EC2, RDS)
- **Ansible** configures the servers (install packages, deploy app, harden OS)

---

## Scenario-Based Questions

### Q: How do you handle rolling deployments with zero downtime using Ansible?

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

### Q: You need to configure 500 servers. How to optimize Ansible?

```
1. Use SSH pipelining (ansible.cfg):
   [ssh_connection]
   pipelining = True          # Reduces SSH operations

2. Increase parallelism:
   [defaults]
   forks = 50                 # Default is 5

3. Use async for long-running tasks:
   - command: /opt/long-task.sh
     async: 3600              # Max runtime
     poll: 0                  # Don't wait (fire and forget)

4. Disable gather_facts if not needed:
   gather_facts: no

5. Use fact caching:
   [defaults]
   fact_caching = jsonfile
   fact_caching_connection = /tmp/facts_cache
   fact_caching_timeout = 86400

6. Use mitogen strategy plugin (3-7x faster):
   strategy_plugins = /path/to/mitogen/ansible_mitogen/plugins/strategy
   strategy = mitogen_linear
```

---

## Key Resources

- **Ansible Documentation** — https://docs.ansible.com
- **Ansible for DevOps (book)** — Jeff Geerling
- **Ansible Galaxy** — https://galaxy.ansible.com (community roles)
- **Jeff Geerling's YouTube** — Excellent Ansible tutorials
- **Molecule** — Testing framework for Ansible roles
