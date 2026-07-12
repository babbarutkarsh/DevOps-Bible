---
title: 2-Day Revision
nav_order: 2
description: "A condensed two-day cram sheet — the highest-yield facts, commands, and talking points across every DevOps Bible topic."
---

# 2-Day Revision Sheet
{: .no_toc }

A fast, high-signal recap for the 48 hours before an interview. **Day 1** covers
fundamentals, containers, and IaC. **Day 2** covers cloud, delivery,
observability, system design, and security. Skim the bold lines; drill into any
[full topic](../) you're shaky on.
{: .fs-5 .fw-300 }

<div id="rev-progress" class="rev-progress" aria-live="polite">
  <div class="rev-progress__head">
    <span class="rev-progress__label">Revision progress</span>
    <span class="rev-progress__count"><span id="rev-done">0</span> / <span id="rev-total">0</span> sections · <span id="rev-pct">0%</span></span>
  </div>
  <div class="rev-progress__track" role="progressbar" aria-valuemin="0" aria-valuemax="100" aria-valuenow="0">
    <div class="rev-progress__fill" id="rev-fill"></div>
  </div>
  <button type="button" id="rev-reset" class="rev-progress__reset">Reset</button>
</div>

1. TOC
{:toc}

---

## Day 1 — Foundations, Containers & IaC

### <label class="rev-h"><input type="checkbox" data-rev="linux"> Linux</label>

- **Boot:** BIOS/UEFI → bootloader (GRUB) → kernel → `initramfs` → `systemd` (PID 1) → targets.
- **Process vs thread:** processes have isolated memory; threads share the process address space. States: R, S, D (uninterruptible), Z (zombie), T.
- **Zombie** = child finished, parent hasn't `wait()`ed. Fix by reaping/parent restart; zombies hold only a PID slot.
- **OOM killer** picks by `oom_score` (memory use + `oom_score_adj`). Tune with `oom_score_adj`.
- **Inodes** store metadata + block pointers, not the name. Hard link = same inode; soft link = separate inode pointing at a path.
- **Permissions:** rwx = 4/2/1. SUID/SGID/sticky = leading 4/2/1. `sticky` on `/tmp` stops users deleting others' files.
- **Disk full triage:** `df -h` → `du -sh *` → check deleted-but-open files with `lsof | grep deleted`; also inode exhaustion (`df -i`).
- **Perf tools:** `top`/`htop`, `vmstat`, `iostat`, `pidstat`, `ss -tulpn`, `sar`. Load avg = runnable + uninterruptible tasks.
- **Signals:** SIGTERM(15) graceful, SIGKILL(9) forced, SIGHUP(1) reload.

### <label class="rev-h"><input type="checkbox" data-rev="networking"> Networking</label>

- **OSI recall:** L3 IP (routing), L4 TCP/UDP (ports), L7 HTTP. LB at L4 = fast/opaque, L7 = content-aware.
- **TCP** = connection, ordered, reliable (3-way handshake SYN/SYN-ACK/ACK). **UDP** = fire-and-forget (DNS, QUIC, streaming).
- **DNS order:** cache → `/etc/hosts` → resolver → root → TLD → authoritative. Record types: A/AAAA, CNAME, MX, TXT, NS, SOA.
- **HTTPS/TLS:** handshake negotiates cipher, validates cert chain, exchanges keys (ECDHE = forward secrecy), then symmetric encryption. TLS 1.3 = 1-RTT.
- **Subnetting:** `/24` = 256 addrs (254 usable). CIDR: bigger prefix = smaller network.
- **NAT vs proxy:** NAT rewrites IPs; forward proxy fronts clients, reverse proxy fronts servers.
- **Latency ladder:** memory ns → SSD µs → same-DC ms → cross-region 10s–100s ms.

### <label class="rev-h"><input type="checkbox" data-rev="docker"> Docker & Containers</label>

- **Containers vs VMs:** share host kernel (namespaces + cgroups), no guest OS → lighter, faster, less isolated.
- **Kernel primitives:** **namespaces** = isolation (pid, net, mnt, uts, ipc, user); **cgroups** = resource limits.
- **Multi-stage build:** compile in a fat builder stage, `COPY --from=builder` only the artifact into a slim/distroless final image.
- **Layer caching:** order Dockerfile least→most volatile; copy dependency manifests and install *before* copying source.
- **CMD vs ENTRYPOINT:** ENTRYPOINT = the executable, CMD = default args; exec form (`["cmd"]`) avoids a shell and forwards signals.
- **Shrink images:** slim/distroless base, multi-stage, `--no-install-recommends`, combine `RUN` layers, `.dockerignore`.
- **Networking modes:** bridge (default), host, none, overlay (multi-host). Port map = `-p host:container`.
- **Volumes** (managed, portable) vs **bind mounts** (host path) vs **tmpfs** (memory).

### <label class="rev-h"><input type="checkbox" data-rev="kubernetes"> Kubernetes</label>

- **Control plane:** API server (front door), etcd (state store), scheduler (placement), controller-manager (reconcile loops), cloud-controller.
- **Node:** kubelet (runs pods), kube-proxy (Service routing), container runtime (containerd).
- **`kubectl apply` flow:** authn/authz → admission → validate → write to etcd → scheduler binds → kubelet pulls & starts → status back to etcd.
- **Workloads:** Deployment (stateless), StatefulSet (stable identity + ordered), DaemonSet (one per node), Job/CronJob (batch).
- **Services:** ClusterIP (internal), NodePort, LoadBalancer, ExternalName. Ingress = L7 HTTP routing.
- **Probes:** liveness (restart), readiness (traffic gate), startup (slow starters).
- **Config:** ConfigMap (plain) vs Secret (base64, not encrypted at rest unless configured). Mount as env or volume.
- **Scheduling:** requests/limits, nodeSelector/affinity, taints & tolerations, topology spread. HPA scales on metrics.
- **Networking:** flat model — every pod gets a routable IP; CNI plugin implements it; NetworkPolicy for segmentation.

### <label class="rev-h"><input type="checkbox" data-rev="terraform"> Terraform / IaC</label>

- **Workflow:** `init` → `plan` → `apply` → `destroy`. Plan symbols: `+` create, `-` destroy, `~` update, `-/+` replace.
- **State:** maps config → real resources. Store **remote with locking** (S3 + DynamoDB, or TF Cloud). Never edit by hand; it can hold secrets.
- **Drift:** real infra diverges from state; `plan` detects, `apply` reconciles, `terraform refresh`/`-refresh-only` updates state.
- **`count` vs `for_each`:** `count` = index-based list (reindexing churns); `for_each` = stable keyed map (prefer for sets/maps).
- **Modules:** reusable units with inputs (variables) + outputs; pin versions; keep root thin.
- **Lifecycle:** `create_before_destroy`, `prevent_destroy`, `ignore_changes`.
- **State ops:** `terraform import`, `state mv`, `state rm`, `taint`/`-replace`.

---

## Day 2 — Cloud, Delivery, Observability & Design

### <label class="rev-h"><input type="checkbox" data-rev="aws"> AWS</label>

- **IAM:** users/roles/policies; prefer **roles** (assume, temp creds) over long-lived keys. Policy = effect + action + resource + condition. Least privilege.
- **VPC:** subnets (public = route to IGW, private = NAT), route tables, SGs (stateful, instance level), NACLs (stateless, subnet level).
- **Compute:** EC2 (VMs), ECS/EKS (containers), Lambda (serverless, event-driven, 15-min cap).
- **Storage:** S3 (object, 11 nines durability, lifecycle/versioning), EBS (block, per-AZ), EFS (shared NFS).
- **Data:** RDS (managed relational, Multi-AZ = HA, read replicas = scale reads), DynamoDB (NoSQL, single-digit ms).
- **Scaling/HA:** ALB (L7) / NLB (L4) + Auto Scaling Groups across AZs; Route 53 for DNS + health-based routing.
- **Cost:** right-size, Spot for stateless, Savings Plans/RIs for steady load, S3 tiering, tag for allocation.

### <label class="rev-h"><input type="checkbox" data-rev="cicd"> CI/CD & GitOps</label>

- **CI** = merge + build + test often; **CD** = automated release to prod (vs continuous delivery = to a deployable state).
- **Pipeline stages:** build → unit/integration test → security scan (SAST/DAST/SCA) → package → deploy → smoke test.
- **Deploy strategies:** rolling (default), blue-green (instant switch/rollback), canary (progressive %), feature flags.
- **GitOps:** Git is the single source of truth; ArgoCD/Flux continuously reconcile cluster state to the repo. Pull-based, auditable, easy rollback (git revert).
- **DevSecOps:** shift security left — scan deps, images, and IaC in the pipeline; sign artifacts; manage secrets in a vault.

### <label class="rev-h"><input type="checkbox" data-rev="git"> Git</label>

- **Merge vs rebase:** merge preserves history (merge commit); rebase rewrites for linear history. **Never rebase shared/pushed branches.**
- **Reset:** `--soft` (keep index+worktree), `--mixed` (keep worktree), `--hard` (discard). `revert` = safe undo via new commit.
- **Branching:** trunk-based (short-lived branches, feature flags) vs GitFlow (heavier, release branches).
- **Debugging:** `git bisect` binary-searches for the breaking commit; `git reflog` recovers "lost" commits.
- **Internals:** commits point to trees point to blobs; a branch is just a movable pointer to a commit.

### <label class="rev-h"><input type="checkbox" data-rev="monitoring"> Monitoring & Observability</label>

- **Three pillars:** metrics (Prometheus), logs (ELK/EFK, Loki), traces (Jaeger/Tempo, OpenTelemetry).
- **Prometheus:** pull-based, scrapes `/metrics`, PromQL, Alertmanager for routing/silencing. Metric types: counter, gauge, histogram, summary.
- **SLI/SLO/SLA:** SLI = measured signal; SLO = internal target; SLA = external contract. **Error budget** = 1 − SLO.
- **Golden signals:** latency, traffic, errors, saturation (USE = utilization/saturation/errors for resources).
- **DORA metrics:** deployment frequency, lead time, change failure rate, MTTR.
- **Alerting:** alert on symptoms (SLO burn) not causes; avoid noise; page only on actionable + urgent.

### <label class="rev-h"><input type="checkbox" data-rev="system-design"> System Design</label>

- **HA:** eliminate single points of failure — redundancy across AZs, health checks, auto-failover. Availability nines → downtime budget.
- **Scaling:** vertical (bigger box, limited) vs horizontal (more boxes, needs statelessness + LB). Shard/partition data to scale writes.
- **Caching:** client → CDN → app (Redis/Memcached) → DB. Patterns: cache-aside, write-through, write-back. Watch invalidation & stampede.
- **DB scaling:** read replicas (read scale), sharding (write scale), CQRS, denormalization. CAP: pick consistency vs availability under partition.
- **Queues:** decouple + absorb spikes (Kafka = log/stream, SQS/RabbitMQ = task queue). At-least-once → design idempotent consumers.
- **DR:** RPO (data loss tolerance) vs RTO (recovery time). Strategies: backup/restore → pilot light → warm standby → active-active.

### <label class="rev-h"><input type="checkbox" data-rev="security"> Security (DevSecOps)</label>

- **Secrets:** never in code/images; use Vault/KMS/Secrets Manager; rotate; inject at runtime.
- **Container security:** minimal base, non-root user, read-only FS, drop capabilities, scan images, sign + verify (cosign).
- **K8s security:** RBAC least privilege, NetworkPolicies, Pod Security Standards, admission control (OPA/Kyverno), no privileged pods.
- **Supply chain:** SBOM, pin/verify dependencies, provenance (SLSA), signed artifacts.
- **Crypto:** TLS in transit, encryption at rest (KMS); symmetric (AES) for data, asymmetric (RSA/ECDSA) for exchange/signing; hash+salt passwords (bcrypt/argon2).
- **Zero trust:** never trust by network location; authenticate + authorize every request; short-lived creds.

---

## Final-Hour Checklist

- [ ] Draw the **Kubernetes control-plane → node** flow from memory.
- [ ] Explain **TCP handshake + TLS handshake** end to end.
- [ ] Write an **optimized multi-stage Dockerfile** on a whiteboard.
- [ ] Reason about **Terraform state locking** and why it matters.
- [ ] Compare **rolling vs blue-green vs canary** and when to use each.
- [ ] Define **SLI/SLO/SLA + error budget** in one breath.
- [ ] Sketch a **highly available, scalable web app** on AWS (LB + ASG + Multi-AZ RDS + cache + CDN).
- [ ] State **RPO vs RTO** and the four DR tiers.

<style>
.rev-progress { margin: 1.25rem 0 0.5rem; }
.rev-progress__head { display: flex; justify-content: space-between; align-items: baseline; margin-bottom: 0.4rem; flex-wrap: wrap; gap: 0.3rem; }
.rev-progress__label { font-weight: 600; }
.rev-progress__count { font-size: 0.85em; opacity: 0.85; }
.rev-progress__track { width: 100%; height: 14px; background: rgba(128,128,128,0.25); border-radius: 999px; overflow: hidden; }
.rev-progress__fill { height: 100%; width: 0%; border-radius: 999px; background: linear-gradient(90deg, #2f81f7, #3fb950); transition: width 0.35s ease; }
.rev-progress__reset { margin-top: 0.5rem; font-size: 0.8em; background: none; border: 1px solid rgba(128,128,128,0.4); color: inherit; border-radius: 6px; padding: 0.2rem 0.6rem; cursor: pointer; opacity: 0.8; }
.rev-progress__reset:hover { opacity: 1; border-color: #2f81f7; }
.rev-h { cursor: pointer; }
.rev-h input { width: 15px; height: 15px; margin-right: 0.4rem; vertical-align: middle; accent-color: #3fb950; cursor: pointer; }
</style>

<script>
(function () {
  var KEY = "dob-revision-v1";
  var boxes = Array.prototype.slice.call(document.querySelectorAll('input[data-rev]'));
  if (!boxes.length) return;
  var fill = document.getElementById('rev-fill');
  var track = fill.parentElement;
  var doneEl = document.getElementById('rev-done');
  var totalEl = document.getElementById('rev-total');
  var pctEl = document.getElementById('rev-pct');

  function load() { try { return JSON.parse(localStorage.getItem(KEY)) || {}; } catch (e) { return {}; } }
  function save(s) { try { localStorage.setItem(KEY, JSON.stringify(s)); } catch (e) {} }
  function render() {
    var done = boxes.filter(function (b) { return b.checked; }).length;
    var total = boxes.length;
    var pct = total ? Math.round((done / total) * 100) : 0;
    fill.style.width = pct + '%';
    doneEl.textContent = done; totalEl.textContent = total; pctEl.textContent = pct + '%';
    track.setAttribute('aria-valuenow', pct);
  }

  var state = load();
  boxes.forEach(function (b) {
    var t = b.getAttribute('data-rev');
    b.checked = !!state[t];
    // keep clicks on the checkbox from toggling the TOC/heading anchor
    b.addEventListener('click', function (e) { e.stopPropagation(); });
    b.addEventListener('change', function () { state[t] = b.checked; save(state); render(); });
  });
  render();

  document.getElementById('rev-reset').addEventListener('click', function () {
    boxes.forEach(function (b) { b.checked = false; });
    state = {}; save(state); render();
  });
})();
</script>
