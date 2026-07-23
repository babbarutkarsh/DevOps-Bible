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

  <div class="dob-hero">
    <div class="dob-rank">
      <div class="dob-rank__badge" id="dob-rank-badge">🌱</div>
      <div class="dob-rank__meta">
        <span class="dob-rank__title" id="dob-rank-title">Rookie</span>
        <span class="dob-rank__lvl">Level <span id="dob-level">1</span></span>
      </div>
    </div>
    <div class="dob-stats">
      <div class="dob-stat"><span class="dob-stat__num" id="dob-xp">0</span><span class="dob-stat__lbl">XP</span></div>
      <div class="dob-stat"><span class="dob-stat__num" id="dob-streak">0</span><span class="dob-stat__lbl">🔥 streak</span></div>
      <div class="dob-stat"><span class="dob-stat__num"><span id="dob-done">0</span>/<span id="dob-total">0</span></span><span class="dob-stat__lbl">topics</span></div>
    </div>
  </div>

  <div class="dob-progress__head">
    <span class="dob-progress__label" id="dob-next-label">Progress to next level</span>
    <span class="dob-progress__count"><span id="dob-pct">0%</span> complete</span>
  </div>
  <div class="dob-progress__track" role="progressbar" aria-valuemin="0" aria-valuemax="100" aria-valuenow="0">
    <div class="dob-progress__fill" id="dob-fill"></div>
  </div>

  <div class="dob-badges" id="dob-badges" aria-label="Achievements"></div>

  <button type="button" id="dob-reset" class="dob-progress__reset">Reset progress</button>
</div>

<div id="dob-toast" class="dob-toast" aria-live="assertive"></div>

---

## Topics

Use the sidebar (or the search bar at the top) to jump into any topic. Tick a
box as you finish each one — the bar above tracks your overall completion and is
saved in your browser.

Inside each topic, every question is tagged by how often it shows up in real
interview loops: **🔥 very frequently asked · ⭐ important · 💡 good to know** —
so you can drill the 🔥 lines first when time is short.

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
  <li><label class="dob-item"><input type="checkbox" data-topic="golang"><span><a href="topics/golang.html"><strong>Go (Golang)</strong></a> — Concurrency (goroutines, channels, context), worker pools, race detection, GC tuning, building static container binaries</span></label></li>
  <li><label class="dob-item"><input type="checkbox" data-topic="scripting"><span><a href="topics/scripting.html"><strong>Shell Scripting &amp; Python</strong></a> — Bash (set -euo pipefail, grep/sed/awk, traps) and Python automation (subprocess, boto3, retries) with coding-round problems</span></label></li>
  <li><label class="dob-item"><input type="checkbox" data-topic="dsa-for-sre"><span><a href="topics/dsa-for-sre.html"><strong>DSA for DevOps &amp; SRE</strong></a> — The coding round: arrays/strings/hashing patterns, 25+ worked problems in Python &amp; Go, plus ops-flavored problems (log parsing, LRU, rate limiter)</span></label></li>
</ul>

<h3>Modern Platform &amp; AI Infrastructure</h3>
<ul class="dob-list">
  <li><label class="dob-item"><input type="checkbox" data-topic="platform-engineering"><span><a href="topics/platform-engineering.html"><strong>Platform Engineering &amp; GitOps</strong></a> — IDPs (Backstage), ArgoCD vs Flux, progressive delivery, Crossplane, service mesh, multi-tenancy, FinOps</span></label></li>
  <li><label class="dob-item"><input type="checkbox" data-topic="genai-llmops"><span><a href="topics/genai-llmops.html"><strong>GenAI, LLMOps &amp; AI Infra</strong></a> — MLOps vs LLMOps, model serving (vLLM/KServe), GPU scheduling &amp; cost, RAG &amp; vector DBs, AI reliability</span></label></li>
</ul>

<h3>Interview Rounds</h3>
<ul class="dob-list">
  <li><label class="dob-item"><input type="checkbox" data-topic="behavioral-sre"><span><a href="topics/behavioral-sre.html"><strong>Behavioral &amp; SRE Culture</strong></a> — STAR framework, blameless postmortems, on-call/incident command, SLO &amp; error-budget discussions, sample senior-level stories</span></label></li>
</ul>

</div>

<script>
(function () {
  "use strict";
  var KEY = "dob-progress-v1";   // { topicId: true }  (kept compatible)
  var META = "dob-meta-v1";      // { streak, lastDay, badges: {id:true} }
  var XP_PER_TOPIC = 100;

  var boxes = Array.prototype.slice.call(document.querySelectorAll('#dob-topics input[type=checkbox]'));
  if (!boxes.length) return;

  // Rank ladder — index by level. Level = floor(done / 2) + 1, capped.
  var RANKS = [
    { t: "Rookie",             b: "🌱" },
    { t: "Junior Engineer",    b: "🔧" },
    { t: "DevOps Engineer",    b: "⚙️" },
    { t: "Platform Engineer",  b: "🚀" },
    { t: "Senior Engineer",    b: "🛡️" },
    { t: "Staff Engineer",     b: "🧠" },
    { t: "Principal / Architect", b: "👑" }
  ];

  // Achievement definitions — fn(done, total, checkedSet) => unlocked?
  var BADGES = [
    { id: "first",     icon: "🎯", name: "First Blood",        test: function (d) { return d >= 1; } },
    { id: "five",      icon: "⭐", name: "Getting Serious",    test: function (d) { return d >= 5; } },
    { id: "container", icon: "🐳", name: "Container Captain",  test: function (d, t, s) { return s.docker && s.kubernetes; } },
    { id: "iac",       icon: "🏗️", name: "IaC Wizard",         test: function (d, t, s) { return s.terraform && s.ansible; } },
    { id: "cloud",     icon: "☁️", name: "Cloud Native",       test: function (d, t, s) { return s.aws && s.kubernetes; } },
    { id: "obs",       icon: "📈", name: "Observability Pro",  test: function (d, t, s) { return s.monitoring && s["system-design"]; } },
    { id: "half",      icon: "🔥", name: "Halfway There",      test: function (d, t) { return t && d >= Math.ceil(t / 2); } },
    { id: "streak3",   icon: "📅", name: "3-Day Streak",       streak: 3 },
    { id: "streak7",   icon: "🗓️", name: "Weekly Warrior",     streak: 7 },
    { id: "complete",  icon: "👑", name: "Bible Complete",     test: function (d, t) { return t && d >= t; } }
  ];

  var els = {
    fill:  document.getElementById('dob-fill'),
    done:  document.getElementById('dob-done'),
    total: document.getElementById('dob-total'),
    pct:   document.getElementById('dob-pct'),
    xp:    document.getElementById('dob-xp'),
    level: document.getElementById('dob-level'),
    streak:document.getElementById('dob-streak'),
    rankT: document.getElementById('dob-rank-title'),
    rankB: document.getElementById('dob-rank-badge'),
    nextL: document.getElementById('dob-next-label'),
    badges:document.getElementById('dob-badges'),
    toast: document.getElementById('dob-toast')
  };
  var track = els.fill.parentElement;

  function readJSON(k) { try { return JSON.parse(localStorage.getItem(k)) || {}; } catch (e) { return {}; } }
  function writeJSON(k, v) { try { localStorage.setItem(k, JSON.stringify(v)); } catch (e) {} }

  var state = readJSON(KEY);
  var meta = readJSON(META);
  if (!meta.badges) meta.badges = {};

  function todayStr() { var d = new Date(); return d.getFullYear() + '-' + (d.getMonth() + 1) + '-' + d.getDate(); }
  function dayNumber(s) { var p = s.split('-'); return Math.floor(Date.UTC(+p[0], p[1] - 1, +p[2]) / 86400000); }

  // Update streak whenever there is study activity today.
  function touchStreak() {
    var today = todayStr();
    if (meta.lastDay === today) return;
    if (meta.lastDay && dayNumber(today) - dayNumber(meta.lastDay) === 1) {
      meta.streak = (meta.streak || 0) + 1;
    } else {
      meta.streak = 1;
    }
    meta.lastDay = today;
    writeJSON(META, meta);
  }

  function toast(msg) {
    if (!els.toast) return;
    els.toast.textContent = msg;
    els.toast.classList.add('show');
    clearTimeout(toast._t);
    toast._t = setTimeout(function () { els.toast.classList.remove('show'); }, 3200);
  }

  function rankFor(level) { return RANKS[Math.min(level - 1, RANKS.length - 1)]; }

  var lastLevel = null;

  function render(animate) {
    var checkedSet = {};
    var done = 0;
    boxes.forEach(function (b) { if (b.checked) { done++; checkedSet[b.getAttribute('data-topic')] = true; } });
    var total = boxes.length;
    var pct = total ? Math.round((done / total) * 100) : 0;
    var xp = done * XP_PER_TOPIC;
    var level = Math.floor(done / 2) + 1;
    var rank = rankFor(level);

    // progress-to-next-level bar (2 topics per level), or overall at max rank
    var maxLevel = RANKS.length;
    var barPct, barLabel;
    if (level >= maxLevel) {
      barPct = pct; barLabel = "Overall completion";
    } else {
      var into = done % 2;
      barPct = Math.round((into / 2) * 100);
      barLabel = (2 - into) + " topic" + (2 - into === 1 ? "" : "s") + " to next rank";
    }

    els.fill.style.width = barPct + '%';
    track.setAttribute('aria-valuenow', barPct);
    els.done.textContent = done;
    els.total.textContent = total;
    els.pct.textContent = pct + '%';
    els.xp.textContent = xp.toLocaleString();
    els.level.textContent = level;
    els.streak.textContent = meta.streak || 0;
    els.rankT.textContent = rank.t;
    els.rankB.textContent = rank.b;
    els.nextL.textContent = barLabel;

    // level-up celebration
    if (lastLevel !== null && level > lastLevel) {
      toast("⬆️ Level " + level + " — you're now a " + rank.t + "!");
      if (els.rankB) { els.rankB.classList.remove('pop'); void els.rankB.offsetWidth; els.rankB.classList.add('pop'); }
    }
    lastLevel = level;

    renderBadges(done, total, checkedSet, animate);
  }

  function renderBadges(done, total, checkedSet, announce) {
    els.badges.innerHTML = '';
    BADGES.forEach(function (bd) {
      var unlocked = bd.streak ? (meta.streak || 0) >= bd.streak : bd.test(done, total, checkedSet);
      if (unlocked && announce && !meta.badges[bd.id]) {
        meta.badges[bd.id] = true;
        writeJSON(META, meta);
        toast(bd.icon + " Achievement unlocked: " + bd.name);
      }
      var el = document.createElement('span');
      el.className = 'dob-badge' + (unlocked ? ' is-on' : '');
      el.title = bd.name + (unlocked ? '' : ' (locked)');
      el.innerHTML = '<span class="dob-badge__ico">' + bd.icon + '</span><span class="dob-badge__name">' + bd.name + '</span>';
      els.badges.appendChild(el);
    });
  }

  // init checkboxes
  boxes.forEach(function (b) {
    var t = b.getAttribute('data-topic');
    b.checked = !!state[t];
    b.addEventListener('change', function () {
      state[t] = b.checked;
      writeJSON(KEY, state);
      if (b.checked) touchStreak();   // studying activity today
      render(true);
    });
  });

  // On load: if there's any progress, keep streak fresh for today's visit
  var hasProgress = boxes.some(function (b) { return b.checked; });
  if (hasProgress) touchStreak();
  render(false);

  document.getElementById('dob-reset').addEventListener('click', function () {
    if (!confirm('Reset all progress, XP, streak and badges?')) return;
    boxes.forEach(function (b) { b.checked = false; });
    state = {};
    meta = { badges: {} };
    writeJSON(KEY, state);
    writeJSON(META, meta);
    lastLevel = null;
    render(false);
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
