---
title: Home
layout: home
nav_order: 1
description: "DevOps Bible — the ultimate one-stop resource for DevOps & Platform Engineer interview preparation."
permalink: /
---

# DevOps Bible
{: .fs-9 }

The ultimate one-stop resource for DevOps & Platform Engineer interview
preparation. Covers the most asked topics at FAANG, top MNCs, and high-growth
startups.
{: .fs-6 .fw-300 }

[Get started](topics/linux.html){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[2-Day Revision](topics/revision.html){: .btn .fs-5 .mb-4 .mb-md-0 .mr-2 }
[View on GitHub](https://github.com/babbarutkarsh/DevOps-Bible){: .btn .fs-5 .mb-4 .mb-md-0 }

<div id="dob-progress" class="dob-progress" aria-live="polite">
  <div class="dob-progress__head">
    <span class="dob-progress__label">Your progress</span>
    <span class="dob-progress__count"><span id="dob-done">0</span> / <span id="dob-total">0</span> topics · <span id="dob-pct">0%</span></span>
  </div>
  <div class="dob-progress__track" role="progressbar" aria-valuemin="0" aria-valuemax="100" aria-valuenow="0">
    <div class="dob-progress__fill" id="dob-fill"></div>
  </div>
  <button type="button" id="dob-reset" class="dob-progress__reset">Reset progress</button>
</div>

---

## Topics

Use the sidebar (or the search bar at the top) to jump into any topic. Tick a
box as you finish each one — the bar above tracks your overall completion and is
saved in your browser.

<div class="dob-topics" id="dob-topics">

<h3>Core Infrastructure</h3>
<ul class="dob-list">
  <li><label class="dob-item"><input type="checkbox" data-topic="linux"><span><a href="topics/linux.html"><strong>Linux</strong></a> — Boot process, filesystems, process mgmt, memory, performance troubleshooting, security hardening, systemd, kernel tuning</span></label></li>
  <li><label class="dob-item"><input type="checkbox" data-topic="networking"><span><a href="topics/networking.html"><strong>Networking</strong></a> — OSI/TCP-IP, DNS, HTTP/TLS, load balancing, proxies, CDN, subnetting, VPN, firewalls, cloud networking</span></label></li>
  <li><label class="dob-item"><input type="checkbox" data-topic="security"><span><a href="topics/security.html"><strong>Security (DevSecOps)</strong></a> — Secrets management, container security, K8s security, supply chain, encryption, zero trust, incident response</span></label></li>
</ul>

<h3>Containers &amp; Orchestration</h3>
<ul class="dob-list">
  <li><label class="dob-item"><input type="checkbox" data-topic="docker"><span><a href="topics/docker.html"><strong>Docker</strong></a> — Container fundamentals, Dockerfile best practices, networking, volumes, Compose, registries, security, runtimes</span></label></li>
  <li><label class="dob-item"><input type="checkbox" data-topic="kubernetes"><span><a href="topics/kubernetes.html"><strong>Kubernetes</strong></a> — Architecture, workloads, services, ingress, storage, RBAC, scheduling, autoscaling, deployment strategies, troubleshooting, etcd, network policies</span></label></li>
</ul>

<h3>Infrastructure as Code &amp; Configuration</h3>
<ul class="dob-list">
  <li><label class="dob-item"><input type="checkbox" data-topic="terraform"><span><a href="topics/terraform.html"><strong>Terraform</strong></a> — State management, modules, providers, workspaces, backends, import, functions, CI/CD integration, best practices</span></label></li>
  <li><label class="dob-item"><input type="checkbox" data-topic="ansible"><span><a href="topics/ansible.html"><strong>Ansible</strong></a> — Playbooks, roles, inventory, variables, Vault, modules, rolling deployments, optimization</span></label></li>
</ul>

<h3>Cloud &amp; Services</h3>
<ul class="dob-list">
  <li><label class="dob-item"><input type="checkbox" data-topic="aws"><span><a href="topics/aws.html"><strong>AWS</strong></a> — IAM, VPC, EC2, ECS/EKS, S3, RDS, Route 53, ALB/NLB, Auto Scaling, Lambda, CloudWatch, cost optimization</span></label></li>
</ul>

<h3>CI/CD &amp; GitOps</h3>
<ul class="dob-list">
  <li><label class="dob-item"><input type="checkbox" data-topic="cicd"><span><a href="topics/cicd.html"><strong>CI/CD</strong></a> — Pipeline design, GitHub Actions, Jenkins, GitLab CI, ArgoCD, GitOps, deployment strategies, DevSecOps</span></label></li>
  <li><label class="dob-item"><input type="checkbox" data-topic="git"><span><a href="topics/git.html"><strong>Git</strong></a> — Branching strategies, merge vs rebase, internals, hooks, bisect, scenario-based troubleshooting</span></label></li>
</ul>

<h3>Observability &amp; Reliability</h3>
<ul class="dob-list">
  <li><label class="dob-item"><input type="checkbox" data-topic="monitoring"><span><a href="topics/monitoring.html"><strong>Monitoring</strong></a> — Prometheus, Grafana, ELK/EFK, Loki, alerting, distributed tracing, SLIs/SLOs/SLAs, DORA metrics, incident management</span></label></li>
  <li><label class="dob-item"><input type="checkbox" data-topic="system-design"><span><a href="topics/system-design.html"><strong>System Design</strong></a> — HA patterns, scalability, caching, database scaling, message queues, disaster recovery, capacity planning</span></label></li>
</ul>

<h3>Programming &amp; Scripting</h3>
<ul class="dob-list">
  <li><label class="dob-item"><input type="checkbox" data-topic="golang"><span><a href="topics/golang.html"><strong>Go (Golang)</strong></a> — Go for DevOps tooling and interviews</span></label></li>
  <li><label class="dob-item"><input type="checkbox" data-topic="scripting"><span><a href="topics/scripting.html"><strong>Shell Scripting &amp; Python</strong></a> — Bash essentials and Python for automation</span></label></li>
</ul>

</div>

<style>
.dob-progress { margin: 1.5rem 0 0.5rem; }
.dob-progress__head { display: flex; justify-content: space-between; align-items: baseline; margin-bottom: 0.4rem; flex-wrap: wrap; gap: 0.3rem; }
.dob-progress__label { font-weight: 600; }
.dob-progress__count { font-size: 0.85em; opacity: 0.85; }
.dob-progress__track { width: 100%; height: 14px; background: rgba(128,128,128,0.25); border-radius: 999px; overflow: hidden; }
.dob-progress__fill { height: 100%; width: 0%; border-radius: 999px; background: linear-gradient(90deg, #2f81f7, #3fb950); transition: width 0.35s ease; }
.dob-progress__reset { margin-top: 0.5rem; font-size: 0.8em; background: none; border: 1px solid rgba(128,128,128,0.4); color: inherit; border-radius: 6px; padding: 0.2rem 0.6rem; cursor: pointer; opacity: 0.8; }
.dob-progress__reset:hover { opacity: 1; border-color: #2f81f7; }
.dob-list { list-style: none; margin-left: 0; padding-left: 0; }
.dob-item { display: flex; align-items: flex-start; gap: 0.6rem; padding: 0.25rem 0; cursor: pointer; }
.dob-item input { margin-top: 0.35rem; width: 16px; height: 16px; flex: 0 0 auto; cursor: pointer; accent-color: #3fb950; }
.dob-item input:checked + span { opacity: 0.6; }
.dob-item input:checked + span a strong { text-decoration: line-through; }
</style>

<script>
(function () {
  var KEY = "dob-progress-v1";
  var boxes = Array.prototype.slice.call(document.querySelectorAll('#dob-topics input[type=checkbox]'));
  if (!boxes.length) return;
  var fill = document.getElementById('dob-fill');
  var track = fill.parentElement;
  var doneEl = document.getElementById('dob-done');
  var totalEl = document.getElementById('dob-total');
  var pctEl = document.getElementById('dob-pct');

  function load() {
    try { return JSON.parse(localStorage.getItem(KEY)) || {}; } catch (e) { return {}; }
  }
  function save(state) {
    try { localStorage.setItem(KEY, JSON.stringify(state)); } catch (e) {}
  }
  function render() {
    var done = boxes.filter(function (b) { return b.checked; }).length;
    var total = boxes.length;
    var pct = total ? Math.round((done / total) * 100) : 0;
    fill.style.width = pct + '%';
    doneEl.textContent = done;
    totalEl.textContent = total;
    pctEl.textContent = pct + '%';
    track.setAttribute('aria-valuenow', pct);
  }

  var state = load();
  boxes.forEach(function (b) {
    var t = b.getAttribute('data-topic');
    b.checked = !!state[t];
    b.addEventListener('change', function () {
      state[t] = b.checked;
      save(state);
      render();
    });
  });
  render();

  document.getElementById('dob-reset').addEventListener('click', function () {
    boxes.forEach(function (b) { b.checked = false; });
    state = {};
    save(state);
    render();
  });
})();
</script>

---

## How to Use This

1. **Start with fundamentals** — Linux, Networking, Docker
2. **Move to orchestration** — Kubernetes (most asked topic in DevOps interviews)
3. **Cover IaC** — Terraform is asked in almost every interview
4. **Cloud + CI/CD** — AWS services, pipeline design, GitOps
5. **System Design** — Infrastructure design questions for senior roles
6. **Security & Monitoring** — Expected knowledge for platform engineers
