---
title: Docker
nav_order: 20
description: "Docker — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# Docker & Containers — DevOps Interview Preparation

> **Question frequency:** 🔥 very frequently asked · ⭐ important · 💡 good to know

## Table of Contents

- [Container Fundamentals](#container-fundamentals)
- [Docker Architecture](#docker-architecture)
- [Image Internals & Layers](#image-internals--layers)
- [Dockerfile Best Practices](#dockerfile-best-practices)
- [BuildKit & Advanced Builds](#buildkit--advanced-builds)
- [Docker Networking](#docker-networking)
- [Docker Storage & Volumes](#docker-storage--volumes)
- [Docker Compose](#docker-compose)
- [Image Management & Registries](#image-management--registries)
- [Container Security](#container-security)
- [Container Runtimes & OCI](#container-runtimes--oci)
- [Docker Performance & Resource Management](#docker-performance--resource-management)
- [Logging & Monitoring](#logging--monitoring)
- [Troubleshooting & Debugging](#troubleshooting--debugging)
- [Scenario-Based Questions](#scenario-based-questions)
- [Trending Topics (2025-2026)](#trending-topics-2025-2026)

---

## Container Fundamentals

### 🔥 Q: What are containers and how are they different from VMs?

```
Virtual Machines:                     Containers:
┌─────────────────────┐              ┌─────────────────────┐
│   App A  │  App B   │              │   App A  │  App B   │
├──────────┼──────────┤              ├──────────┼──────────┤
│ Bins/Libs│Bins/Libs │              │ Bins/Libs│Bins/Libs │
├──────────┼──────────┤              ├──────────┴──────────┤
│ Guest OS │ Guest OS │              │   Container Runtime  │
├──────────┴──────────┤              ├─────────────────────┤
│     Hypervisor      │              │     Host OS Kernel   │
├─────────────────────┤              ├─────────────────────┤
│     Host OS         │              │     Hardware         │
├─────────────────────┤              └─────────────────────┘
│     Hardware        │
└─────────────────────┘
```

| Feature | VM | Container |
|---------|-----|-----------|
| Isolation | Hardware-level (hypervisor) | OS-level (namespaces, cgroups) |
| Boot time | Minutes | Milliseconds to seconds |
| Size | GBs (includes full OS) | MBs (shares host kernel) |
| Performance | Near-native with overhead | Near-native, minimal overhead |
| Density | 10s per host | 100s-1000s per host |
| Kernel | Own kernel | Shares host kernel |
| Security | Stronger isolation | Weaker isolation (shared kernel) |

### ⭐ Q: How does a container differ from a VM at the kernel level?

**VMs:**
- Hardware virtualization via hypervisor (KVM, ESXi, Hyper-V)
- Each VM runs its **own kernel**
- Strong isolation boundary (separate kernel address space)
- Hypervisor translates guest kernel calls to host hardware

**Containers:**
- OS-level virtualization via **kernel features** (namespaces + cgroups)
- All containers **share the host kernel**
- Weaker isolation (same kernel, different namespaces)
- Container runtime (runc) invokes kernel APIs directly

**Why this matters:**
- Can't run Windows containers on Linux kernel (different syscall interface)
- Kernel exploits can escape container to host
- Containers are processes on the host — `ps aux` on host shows container PIDs

### 🔥 Q: What Linux kernel features enable containers?

1. **Namespaces** — Isolation of system resources (process thinks it's alone):

| Namespace | Isolates | Flag | What it does |
|-----------|----------|------|--------------|
| PID | Process IDs | `CLONE_NEWPID` | Container sees its own PID tree (PID 1 = container init) |
| NET | Network stack | `CLONE_NEWNET` | Own IPs, routes, iptables, sockets |
| MNT | Mount points | `CLONE_NEWNS` | Own filesystem tree (can't see host mounts) |
| UTS | Hostname | `CLONE_NEWUTS` | Own hostname/domain name |
| IPC | Inter-process communication | `CLONE_NEWIPC` | Own message queues, semaphores, shared memory |
| USER | UID/GID mappings | `CLONE_NEWUSER` | Map container root (UID 0) to non-root host UID (rootless) |
| CGROUP | Cgroup root | `CLONE_NEWCGROUP` | Own cgroup hierarchy view |

```bash
# See namespaces of a running container
docker inspect <container> --format '{{.State.Pid}}'  # Get host PID
ls -l /proc/<pid>/ns/                                  # List namespace symlinks
```

2. **Cgroups (Control Groups)** — Resource limits and accounting:

| Cgroup v2 Controller | Controls |
|---------------------|----------|
| `cpu` | CPU time shares, quotas |
| `memory` | Memory limits, swap, OOM behavior |
| `io` | Block device I/O (read/write IOPS, bandwidth) |
| `pids` | Max number of processes (prevents fork bombs) |
| `cpuset` | Pin to specific CPU cores |

```bash
# Inspect cgroup limits on host
docker inspect <container> --format '{{.HostConfig.Memory}}'
cat /sys/fs/cgroup/system.slice/docker-<container-id>.scope/memory.max
```

3. **Union Filesystems** — Layer multiple filesystems (OverlayFS, AUFS)

4. **seccomp** — System call filtering (blocks ~300 of ~400 syscalls by default)

5. **AppArmor / SELinux** — Mandatory access control

6. **Capabilities** — Break root into granular privileges (CAP_NET_BIND_SERVICE, CAP_SYS_ADMIN, etc.)

### ⭐ Q: What is an OCI (Open Container Initiative)?

The **OCI** defines industry standards for containers:

- **Runtime Specification** — How to run a container (lifecycle, configuration)
- **Image Specification** — Container image format (layers, manifest, config)
- **Distribution Specification** — How to distribute images (registry API)

This means you can build an image with Docker and run it with any OCI-compliant runtime (containerd, CRI-O, Podman).

---

## Docker Architecture

### 🔥 Q: Explain Docker's architecture.

```
┌────────────────────────────────────────────────────────┐
│                     Docker Client                       │
│                   (docker CLI)                          │
└───────────────────────┬────────────────────────────────┘
                        │ REST API (Unix socket / TCP)
                        ▼
┌────────────────────────────────────────────────────────┐
│                   Docker Daemon (dockerd)                │
│  ┌──────────────┐ ┌────────────┐ ┌──────────────────┐ │
│  │ Image Mgmt   │ │ Network    │ │ Volume Mgmt      │ │
│  └──────────────┘ └────────────┘ └──────────────────┘ │
└───────────────────────┬────────────────────────────────┘
                        │ gRPC
                        ▼
┌────────────────────────────────────────────────────────┐
│                   containerd                            │
│           (Container lifecycle management)              │
└───────────────────────┬────────────────────────────────┘
                        │
                        ▼
┌────────────────────────────────────────────────────────┐
│                   runc (OCI runtime)                    │
│        (Actually creates/runs the container)            │
│        Uses namespaces, cgroups, seccomp               │
└────────────────────────────────────────────────────────┘
```

**Key components:**
- **Docker Client** — CLI tool that sends commands to daemon
- **Docker Daemon (dockerd)** — Manages containers, images, networks, volumes
- **containerd** — Industry-standard container runtime (manages container lifecycle)
- **runc** — Low-level OCI runtime (creates the actual container process)
- **Docker Registry** — Stores images (Docker Hub, ECR, GCR, Harbor)

### ⭐ Q: What is the difference between `docker run`, `docker create`, and `docker start`?

| Command | What it does |
|---------|-------------|
| `docker create` | Creates a container (writable layer) but doesn't start it |
| `docker start` | Starts an existing stopped container |
| `docker run` | `docker create` + `docker start` + `docker attach` (if interactive) |

---

## Image Internals & Layers

### 🔥 Q: What is the difference between an image and a container?

```
Image (read-only template):        Container (running instance):
┌─────────────────────────┐       ┌─────────────────────────┐
│  Layer 3: ADD app.js    │       │  Writable Layer         │ ← Changes here
├─────────────────────────┤       ├─────────────────────────┤
│  Layer 2: RUN npm i     │       │  Layer 3: ADD app.js    │ ← Read-only
├─────────────────────────┤       ├─────────────────────────┤
│  Layer 1: FROM node     │       │  Layer 2: RUN npm i     │ ← Read-only
└─────────────────────────┘       ├─────────────────────────┤
     (on disk)                     │  Layer 1: FROM node     │ ← Read-only
                                   └─────────────────────────┘
                                        (in memory + disk)
```

| Concept | Image | Container |
|---------|-------|-----------|
| Definition | Immutable template (blueprint) | Running instance of an image |
| Storage | Stored in registry (Docker Hub, ECR, GCR) | Exists on host with writable layer |
| State | Static (no state) | Dynamic (has state, running processes) |
| Layers | Read-only layers only | Image layers + writable layer on top |
| Lifecycle | Built once, pulled many times | Created → Started → Stopped → Removed |

```bash
# One image can spawn many containers
docker run -d --name web1 nginx
docker run -d --name web2 nginx
docker run -d --name web3 nginx
# All share the same nginx image layers (Copy-on-Write)
```

### ⭐ Q: Explain OverlayFS and Copy-on-Write in detail.

**OverlayFS** is the default union filesystem (storage driver) for Docker. It merges multiple directories into a single unified view.

```
Host filesystem:
├── /var/lib/docker/overlay2/
│   ├── l/                        # Short symlinks to layers
│   ├── abc123.../
│   │   ├── diff/                 # Layer content
│   │   ├── work/                 # OverlayFS working dir
│   │   └── merged/               # Unified view (what container sees)
```

**How Copy-on-Write works:**

```
Container wants to modify /app/config.json:
1. File exists in lower (image) layer  → Read-only
2. Container writes to file
3. OverlayFS copies file UP to writable layer
4. Container modifies the COPY (original unchanged)
5. When reading, writable layer shadows lower layer

Performance implications:
- First write to large file → Slow (full copy)
- Subsequent writes → Fast (file already in writable layer)
- Database files in container? → Use volumes (bypass CoW)
```

**View OverlayFS mounts:**
```bash
docker inspect <container> --format '{{.GraphDriver.Data}}'
# Shows LowerDir (image layers), UpperDir (writable), MergedDir (unified view)
```

### ⭐ Q: How are image layers stored and identified?

**Content-addressable storage:** Each layer is identified by its SHA256 digest of its contents.

```bash
# Image manifest (JSON)
docker manifest inspect nginx:alpine
{
  "schemaVersion": 2,
  "config": {
    "digest": "sha256:abc123...",
    "size": 1234
  },
  "layers": [
    {"digest": "sha256:layer1...", "size": 3456789},
    {"digest": "sha256:layer2...", "size": 1234567}
  ]
}
```

**Layer sharing:**
```dockerfile
# Image A
FROM alpine:3.19
RUN apk add curl

# Image B
FROM alpine:3.19     # ← Shares layer with Image A
RUN apk add wget
```

Both images share the `alpine:3.19` base layer. Only one copy stored on disk.

```bash
# See shared layers
docker system df -v
# Lists images and shared size
```

**Why this matters:**
- Faster pulls (only download new layers)
- Efficient disk usage (deduplication)
- Cache hits across builds

---

## Dockerfile Best Practices

### 🔥 Q: Write an optimized multi-stage Dockerfile.

**Bad example (large image, security issues):**
```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y nodejs npm
COPY . /app
WORKDIR /app
RUN npm install
EXPOSE 3000
CMD ["node", "server.js"]
# Result: ~800MB image, running as root, includes build tools
```

**Good example (multi-stage, optimized):**
```dockerfile
# Stage 1: Build
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:20-alpine AS production
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=builder --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --from=builder --chown=appuser:appgroup /app/package.json ./

USER appuser
EXPOSE 3000
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1
CMD ["node", "dist/server.js"]
# Result: ~150MB image, non-root user, no build tools
```

### 🔥 Q: Explain Dockerfile layer caching and optimization.

Each instruction creates a layer. Docker caches layers and reuses them if the instruction and context haven't changed.

**Optimization rules:**

1. **Order matters** — Put rarely changing instructions first:
   ```dockerfile
   # Good: Dependencies change less often than source code
   COPY package*.json ./
   RUN npm ci
   COPY . .

   # Bad: Any source code change invalidates npm install cache
   COPY . .
   RUN npm ci
   ```

2. **Combine RUN commands** — Reduce layers:
   ```dockerfile
   # Good: Single layer, smaller image
   RUN apt-get update && \
       apt-get install -y --no-install-recommends curl wget && \
       rm -rf /var/lib/apt/lists/*

   # Bad: 3 layers, apt cache persists
   RUN apt-get update
   RUN apt-get install -y curl wget
   ```

3. **Use `.dockerignore`:**
   ```
   .git
   node_modules
   *.md
   .env
   Dockerfile
   docker-compose.yml
   .github
   ```

4. **Use specific tags** — `node:20-alpine` not `node:latest`

5. **Use `COPY` over `ADD`** — `ADD` does extra things (tar extraction, URL downloads)

### 🔥 Q: What is the difference between CMD, ENTRYPOINT, and RUN?

| Instruction | When it runs | Purpose | Overridable |
|------------|-------------|---------|-------------|
| `RUN` | Build time | Execute commands, create layers | N/A |
| `CMD` | Container start | Default command/args | Yes, via `docker run <image> <command>` |
| `ENTRYPOINT` | Container start | Defines the executable | Only via `--entrypoint` |

**Interaction between CMD and ENTRYPOINT:**
```dockerfile
# CMD only — easy to override
CMD ["nginx", "-g", "daemon off;"]
# docker run myimage              → nginx -g daemon off;
# docker run myimage /bin/bash    → /bin/bash

# ENTRYPOINT only — always runs
ENTRYPOINT ["nginx", "-g", "daemon off;"]
# docker run myimage              → nginx -g daemon off;
# docker run myimage -v           → nginx -g daemon off; -v  (appended!)

# ENTRYPOINT + CMD — best practice for parameterized containers
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8080"]
# docker run myimage              → python app.py --port 8080
# docker run myimage --port 9090  → python app.py --port 9090
```

### ⭐ Q: What are the different forms of CMD/ENTRYPOINT?

| Form | Syntax | Shell | Signal Handling |
|------|--------|-------|----------------|
| **Exec form** (preferred) | `["executable", "arg1"]` | No shell (`/bin/sh -c` not used) | PID 1, receives signals directly |
| **Shell form** | `executable arg1` | Runs via `/bin/sh -c` | Shell is PID 1, signals may not reach app |

**Always use exec form** in production — shell form prevents proper signal handling (SIGTERM for graceful shutdown).

### ⭐ Q: What is a distroless image and why use it?

**Distroless** images contain only your application and its runtime dependencies — **no shell, package manager, or OS utilities**.

```dockerfile
# Traditional alpine image
FROM node:20-alpine       # Includes: apk, sh, ash, vi, etc.
# Result: ~180MB, large attack surface

# Distroless image
FROM gcr.io/distroless/nodejs20-debian12
COPY --from=builder /app .
USER nonroot:nonroot
CMD ["/app/server.js"]
# Result: ~130MB, minimal attack surface
```

| Feature | Alpine | Distroless |
|---------|--------|-----------|
| Shell | ✅ sh, ash | ❌ No shell |
| Package manager | ✅ apk | ❌ None |
| Debugging | Easy (`docker exec -it sh`) | Hard (no shell — use ephemeral containers) |
| Security | Good | **Better** (smaller attack surface) |
| Size | Small (~10-50MB base) | **Smallest** (only runtime) |
| CVEs | More packages = more CVEs | Fewer packages = fewer CVEs |

**When to use distroless:**
- Production images (no debugging needed)
- Security-sensitive workloads
- Supply chain compliance (minimal SBOM)

**How to debug distroless containers:**
```bash
# Can't docker exec (no shell!) — use ephemeral debug container
kubectl debug -it pod-name --image=busybox --target=container-name

# Or use multi-stage with debug target
FROM gcr.io/distroless/nodejs20-debian12 AS production
# ...

FROM node:20-alpine AS debug
COPY --from=production / /
CMD ["sh"]

# Build debug variant when needed:
# docker build --target=debug -t myapp:debug .
```

### ⭐ Q: What is a .dockerignore file and why is it critical?

`.dockerignore` excludes files from the **build context** sent to Docker daemon. Without it, every `COPY . .` sends **everything** (node_modules, .git, logs, etc.).

```
# .dockerignore example
# Version control
.git
.gitignore
.github

# Dependencies (install fresh in image)
node_modules
vendor
__pycache__

# Documentation
*.md
docs/

# Environment files (secrets!)
.env
.env.*
*.key
*.pem

# Build artifacts
dist/
build/
target/

# IDE
.vscode
.idea
*.swp

# Docker files
Dockerfile*
docker-compose*.yml

# CI/CD
.circleci
.travis.yml
Jenkinsfile

# OS
.DS_Store
Thumbs.db
```

**Impact:**
```bash
# Without .dockerignore:
COPY . .  # Sends 500MB (includes node_modules, .git, etc.)

# With .dockerignore:
COPY . .  # Sends 5MB (only source code)
```

**Why this matters:**
- Faster builds (smaller context = faster upload to daemon)
- Better caching (context hash changes less often)
- **Security** (prevents accidentally copying .env, SSH keys into image)
- Smaller images (if using `COPY . .` before cleanup)

---

## BuildKit & Advanced Builds

### ⭐ Q: What is BuildKit and how does it improve Docker builds?

**BuildKit** is the next-generation Docker build engine (default since Docker 23.0). It's faster, more efficient, and supports advanced features.

**Enable BuildKit:**
```bash
# Environment variable (older Docker)
DOCKER_BUILDKIT=1 docker build .

# Or set in daemon.json (persistent)
{
  "features": {
    "buildkit": true
  }
}
```

**Key improvements over legacy builder:**

| Feature | Legacy Builder | BuildKit |
|---------|---------------|----------|
| Parallelization | Sequential | **Parallel independent stages** |
| Unused stage skip | Builds all stages | **Skips unused stages** |
| Cache mounts | No | **Yes** (`--mount=type=cache`) |
| Secret mounts | No | **Yes** (`--mount=type=secret`) |
| SSH forwarding | No | **Yes** (for private Git repos) |
| Multi-platform | Separate builds | **Single build** (`docker buildx`) |
| Output formats | Image only | Image, tarball, registry, local filesystem |

### ⭐ Q: How do BuildKit cache mounts work and why use them?

**Cache mounts** persist directories across builds (package manager caches, Go build cache, etc.) without baking them into the final image.

```dockerfile
# Traditional approach (cache baked into layer)
RUN pip install -r requirements.txt
# Every requirements.txt change → re-download ALL packages

# BuildKit cache mount (cache persists across builds)
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
# Cache persists → only download CHANGED packages
```

**Common cache mount targets:**

```dockerfile
# Python
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt

# Node.js
RUN --mount=type=cache,target=/root/.npm \
    npm ci

# Go
RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -o /app

# Rust
RUN --mount=type=cache,target=/usr/local/cargo/registry \
    --mount=type=cache,target=/app/target \
    cargo build --release

# Maven
RUN --mount=type=cache,target=/root/.m2 \
    mvn package

# Gradle
RUN --mount=type=cache,target=/root/.gradle \
    gradle build
```

**Why this matters:**
- **10-100x faster rebuilds** (no re-download)
- Smaller final image (cache not baked in)
- Works across branches (cache shared on build host)

### 🔥 Q: How do you handle secrets in Dockerfile builds?

**❌ NEVER do this:**
```dockerfile
# Secret ends up in image layer!
ARG GITHUB_TOKEN
RUN git clone https://${GITHUB_TOKEN}@github.com/private/repo.git
# Anyone can extract: docker history myimage
```

**✅ Use BuildKit secret mounts:**
```dockerfile
# Dockerfile
RUN --mount=type=secret,id=github_token \
    git clone https://$(cat /run/secrets/github_token)@github.com/private/repo.git

# Build command
docker build --secret id=github_token,src=$HOME/.github_token .
```

The secret is:
- Mounted temporarily during RUN
- **Not** stored in any image layer
- Not in docker history
- Not in final image

**For SSH keys (private Git repos):**
```dockerfile
# Dockerfile
RUN --mount=type=ssh \
    git clone git@github.com:private/repo.git

# Build command
docker build --ssh default .
# Uses your SSH agent
```

### ⭐ Q: How do you build multi-architecture images (ARM + x86)?

**Use `docker buildx` (BuildKit's multi-platform builder):**

```bash
# 1. Create a multi-platform builder
docker buildx create --name multiarch --use
docker buildx inspect --bootstrap

# 2. Build and push for multiple architectures
docker buildx build \
  --platform linux/amd64,linux/arm64,linux/arm/v7 \
  --tag myregistry/myapp:latest \
  --push \
  .

# Result: Single manifest with 3 image variants
# Docker automatically pulls the right one for the host arch
```

**View manifest:**
```bash
docker manifest inspect myregistry/myapp:latest
{
  "manifests": [
    {"platform": {"architecture": "amd64", "os": "linux"}},
    {"platform": {"architecture": "arm64", "os": "linux"}},
    {"platform": {"architecture": "arm", "os": "linux", "variant": "v7"}}
  ]
}
```

**Why this matters:**
- Deploy to mixed fleets (x86 servers, ARM Graviton, Raspberry Pi)
- Apple Silicon Macs (M1/M2/M3) need ARM images
- K8s clusters with mixed node types

**Cross-compilation tips:**
```dockerfile
# Use --platform in FROM to pin base image arch
FROM --platform=$BUILDPLATFORM golang:1.22 AS builder
ARG TARGETPLATFORM
ARG TARGETARCH
RUN GOARCH=$TARGETARCH go build -o /app

# BuildKit automatically sets TARGETPLATFORM, TARGETARCH
```

---

## Docker Networking

### 🔥 Q: Explain Docker networking modes.

| Network Driver | Description | Use Case |
|---------------|-------------|----------|
| **bridge** (default) | Virtual bridge network, containers get private IPs | Single-host, isolated containers |
| **host** | Container shares host network stack | Maximum network performance |
| **none** | No networking | Security-sensitive workloads |
| **overlay** | Multi-host networking (VXLAN) | Docker Swarm, multi-host |
| **macvlan** | Container gets its own MAC address on physical network | Legacy apps needing direct LAN access |

**Bridge network deep dive:**
```
┌──────────── Host ─────────────────────────────────┐
│                                                     │
│  ┌──── Container A ────┐  ┌──── Container B ────┐ │
│  │  eth0: 172.17.0.2   │  │  eth0: 172.17.0.3   │ │
│  └─────────┬───────────┘  └─────────┬───────────┘ │
│            │ veth pair                │ veth pair    │
│            ▼                         ▼              │
│  ┌──────────────────────────────────────────────┐  │
│  │            docker0 (bridge)                   │  │
│  │            172.17.0.1                         │  │
│  └──────────────────┬───────────────────────────┘  │
│                     │ NAT (iptables)                │
│                     ▼                               │
│              eth0 (host NIC)                        │
└─────────────────────────────────────────────────────┘
```

**User-defined bridge networks (recommended over default bridge):**
```bash
docker network create mynet

# Containers on same user-defined network get automatic DNS resolution
docker run --name api --network mynet myapi
docker run --name web --network mynet myweb

# From web container:
curl http://api:8080/health    # DNS resolves 'api' to container IP
```

### ⭐ Q: How does Docker port mapping work?

```bash
docker run -p 8080:3000 myapp
# Host:8080 → Container:3000

# Under the hood, Docker creates iptables rules:
# -A DOCKER -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:3000
```

| Flag | Meaning |
|------|---------|
| `-p 8080:3000` | Map host 8080 → container 3000 (all interfaces) |
| `-p 127.0.0.1:8080:3000` | Only localhost |
| `-p 8080:3000/udp` | UDP mapping |
| `-P` | Map all EXPOSE ports to random host ports |

---

## Docker Storage & Volumes

### ⭐ Q: Explain Docker storage drivers and copy-on-write.

**Container filesystem uses Copy-on-Write (CoW):**

```
┌─────────────────────────┐
│  Container Writable Layer│  ← Changes go here (ephemeral)
├─────────────────────────┤
│  Image Layer 4 (app)    │  ← Read-only
├─────────────────────────┤
│  Image Layer 3 (deps)   │  ← Read-only
├─────────────────────────┤
│  Image Layer 2 (config) │  ← Read-only
├─────────────────────────┤
│  Image Layer 1 (base OS)│  ← Read-only
└─────────────────────────┘
```

When a container modifies a file from a lower layer, it's **copied up** to the writable layer. Original is unchanged.

**Storage drivers:** overlay2 (default, recommended), devicemapper, btrfs, zfs

### 🔥 Q: Volumes vs Bind Mounts vs tmpfs?

| Type | Managed by Docker | Location | Persistence | Use Case |
|------|------------------|----------|-------------|----------|
| **Volume** | Yes | `/var/lib/docker/volumes/` | Persists | Databases, shared data |
| **Bind Mount** | No | Anywhere on host | Depends on host | Dev environments, config files |
| **tmpfs** | N/A | RAM only | Lost on stop | Sensitive data, performance |

```bash
# Named volume
docker volume create mydata
docker run -v mydata:/app/data myapp

# Bind mount
docker run -v /host/path:/container/path myapp

# tmpfs
docker run --tmpfs /app/tmp:rw,size=100m myapp

# Read-only mount
docker run -v mydata:/app/data:ro myapp
```

---

## Docker Compose

### 🔥 Q: Explain Docker Compose and write a production-like example.

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
      target: production
    ports:
      - "8080:3000"
    environment:
      - NODE_ENV=production
      - DB_HOST=postgres
      - REDIS_HOST=redis
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s
    networks:
      - frontend
      - backend

  postgres:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: myapp
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password
    secrets:
      - db_password
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U myapp"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - backend

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certs:/etc/ssl/certs:ro
    depends_on:
      - app
    networks:
      - frontend

volumes:
  pgdata:
  redisdata:

networks:
  frontend:
  backend:

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

**Key Docker Compose commands:**
```bash
docker compose up -d                    # Start in background
docker compose down                     # Stop and remove containers
docker compose down -v                  # Also remove volumes
docker compose logs -f app              # Follow app logs
docker compose exec app sh              # Shell into running container
docker compose ps                       # List running services
docker compose build --no-cache         # Rebuild without cache
docker compose pull                     # Pull latest images
docker compose config                   # Validate and view merged config
```

---

## Image Management & Registries

### ⭐ Q: How do Docker image layers work?

```bash
# View layers of an image
docker history nginx:alpine

IMAGE          CREATED       CREATED BY                                      SIZE
a1b2c3d4e5     2 weeks ago   CMD ["nginx" "-g" "daemon off;"]                0B
b2c3d4e5f6     2 weeks ago   EXPOSE map[80/tcp:{}]                           0B
c3d4e5f6g7     2 weeks ago   COPY conf /etc/nginx/                           1.2kB
d4e5f6g7h8     2 weeks ago   RUN /bin/sh -c apk add --no-cache nginx         5.4MB
e5f6g7h8i9     2 weeks ago   /bin/sh -c #(nop) ADD file:abc...              7.8MB
```

**Content-addressable storage:** Each layer is identified by its SHA256 digest. Layers are shared across images.

### 🔥 Q: How to reduce Docker image size?

1. **Use Alpine/Distroless base images:**
   ```
   ubuntu:22.04     → 77MB
   node:20          → 1.1GB
   node:20-alpine   → 180MB
   node:20-slim     → 250MB
   gcr.io/distroless/nodejs20-debian12 → 130MB
   ```

2. **Multi-stage builds** — Copy only artifacts, leave build tools behind

3. **Minimize layers and clean up in same RUN:**
   ```dockerfile
   RUN apt-get update && \
       apt-get install -y --no-install-recommends package && \
       rm -rf /var/lib/apt/lists/*
   ```

4. **Use `.dockerignore`** to exclude unnecessary files

5. **Use `docker image prune`** to clean up dangling images

### ⭐ Q: How do you scan images for vulnerabilities?

```bash
# Docker Scout (built-in)
docker scout cves myimage:latest
docker scout recommendations myimage:latest

# Trivy (popular open-source scanner)
trivy image myimage:latest
trivy image --severity HIGH,CRITICAL myimage:latest

# Grype
grype myimage:latest

# Snyk
snyk container test myimage:latest
```

---

## Container Security

### 🔥 Q: Docker security best practices?

1. **Don't run as root:**
   ```dockerfile
   RUN addgroup -S app && adduser -S app -G app
   USER app
   ```

2. **Use read-only root filesystem:**
   ```bash
   docker run --read-only --tmpfs /tmp myapp
   ```

3. **Drop all capabilities, add only needed:**
   ```bash
   docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
   ```

4. **No privileged mode in production:**
   ```bash
   # NEVER in production:
   docker run --privileged myapp

   # This gives container full host access!
   ```

5. **Limit resources:**
   ```bash
   docker run --memory=512m --cpus=1.0 --pids-limit=100 myapp
   ```

6. **Use seccomp profiles:**
   ```bash
   docker run --security-opt seccomp=profile.json myapp
   ```

7. **Scan images in CI/CD pipeline**

8. **Sign images** — Docker Content Trust / Cosign
   ```bash
   export DOCKER_CONTENT_TRUST=1
   docker push myimage:latest    # Signed push
   ```

9. **No secrets in images:**
   ```dockerfile
   # BAD
   ENV API_KEY=mysecret

   # GOOD — Use runtime secrets
   # docker run -e API_KEY=$(vault read -field=key secret/app) myapp
   ```

### ⭐ Q: What is Docker socket mounting and why is it dangerous?

```bash
docker run -v /var/run/docker.sock:/var/run/docker.sock myapp
```

This gives the container **full control over the Docker daemon** — it can:
- Start/stop any container
- Mount host filesystems
- Effectively gain root access to the host

**When needed** (CI/CD, monitoring): Use alternatives like Docker-in-Docker (dind), Kaniko for builds, or limit access with a Docker socket proxy.

---

## Container Runtimes & OCI

### 💡 Q: Explain the OCI runtime specification in detail.

The **OCI Runtime Spec** defines a standard for how to run a container, ensuring portability across runtimes.

**Key components:**

1. **`config.json`** — Container configuration:
```json
{
  "ociVersion": "1.0.0",
  "process": {
    "args": ["/bin/sh"],
    "env": ["PATH=/usr/local/bin:/usr/bin"],
    "cwd": "/",
    "user": {"uid": 1000, "gid": 1000}
  },
  "root": {"path": "rootfs", "readonly": false},
  "hostname": "mycontainer",
  "mounts": [...],
  "linux": {
    "namespaces": [
      {"type": "pid"}, {"type": "network"}, {"type": "mount"}
    ],
    "resources": {
      "memory": {"limit": 536870912},
      "cpu": {"shares": 1024}
    }
  }
}
```

2. **Runtime lifecycle operations:**
```bash
# Low-level OCI runtime (runc) commands:
runc create <container-id>    # Create container (don't start)
runc start <container-id>     # Start the process
runc kill <container-id>      # Send signal
runc delete <container-id>    # Clean up
```

3. **Bundle directory structure:**
```
container-bundle/
├── config.json        # OCI config
└── rootfs/            # Container root filesystem
    ├── bin/
    ├── etc/
    └── ...
```

**Why this matters:**
- Build image with Docker → Run with Podman, Kubernetes CRI-O, or any OCI runtime
- Standardized security (seccomp profiles, AppArmor, SELinux)
- Predictable behavior across platforms

### ⭐ Q: Explain the container runtime landscape.

```
High-level runtimes (CRI implementations):
├── containerd        ← Used by Docker, default in Kubernetes
├── CRI-O             ← Red Hat's lightweight CRI runtime for K8s
└── Docker Engine     ← Uses containerd under the hood

Low-level runtimes (OCI implementations):
├── runc              ← Default, reference implementation
├── crun              ← Faster, written in C (Red Hat)
├── gVisor (runsc)    ← Google's sandboxed runtime (extra isolation)
├── Kata Containers   ← Lightweight VMs as containers
└── Firecracker       ← AWS's microVM (used by Lambda, Fargate)
```

### 🔥 Q: Why did Kubernetes remove dockershim and what does it mean?

**Timeline:**
- Kubernetes v1.20 (Dec 2020): dockershim deprecated
- Kubernetes v1.24 (May 2022): dockershim removed

**Why:**
```
Old (with dockershim):                  New (CRI-native):
kubelet                                 kubelet
  ↓                                      ↓ CRI (gRPC)
dockershim (translation layer)         containerd
  ↓ Docker API                           ↓ OCI
Docker Engine (dockerd)                runc
  ↓ gRPC
containerd
  ↓ OCI
runc
```

**Problems with Docker:**
- Extra layer (dockershim) = maintenance burden
- Docker daemon required = extra attack surface
- Docker CLI not needed in K8s (kubelet talks CRI)
- Docker's features (docker build, docker compose) unused in K8s

**Migration:**
- Replace Docker Engine with **containerd** or **CRI-O**
- Existing images **still work** (OCI-compliant)
- `docker build` still works for development
- In prod, use containerd + nerdctl (or buildah/kaniko for builds)

**Key takeaway:** Kubernetes doesn't need Docker. It needs:
1. A CRI-compatible runtime (containerd, CRI-O)
2. OCI-compliant images (Docker/Podman/buildah all produce these)

### ⭐ Q: containerd vs Docker Engine in production?

| Feature | Docker Engine | containerd |
|---------|--------------|------------|
| Purpose | Developer experience | Production container runtime |
| Daemon | dockerd + containerd | containerd only |
| CLI | `docker` | `ctr` (low-level) or `nerdctl` (Docker-compatible) |
| Image builds | Built-in (`docker build`) | External (BuildKit, buildah, kaniko) |
| Docker Compose | Built-in | `nerdctl compose` or external |
| Kubernetes CRI | Via dockershim (removed) | **Native CRI plugin** |
| Resource usage | Higher (extra daemon) | **Lower** |
| Security | More features = larger attack surface | **Minimal** |
| Adoption | Docker Hub, local dev | K8s default, cloud prod |

**When to use Docker Engine:**
- Local development (Docker Desktop)
- Legacy workflows tied to `docker` CLI
- Docker Compose-heavy environments

**When to use containerd:**
- Kubernetes clusters (default since v1.24)
- Production servers (no need for Docker daemon)
- Minimal attack surface required

### ⭐ Q: Docker vs Podman?

| Feature | Docker | Podman |
|---------|--------|--------|
| Daemon | Yes (dockerd) | **Daemonless** (fork/exec model) |
| Root | Requires root daemon | **Rootless by default** |
| CLI compatibility | `docker ...` | `podman ...` (drop-in: `alias docker=podman`) |
| Pods | No native concept | Yes (like K8s pods) |
| Compose | `docker compose` | `podman-compose` or `podman compose` |
| Systemd integration | Limited | **Native** (`podman generate systemd`) |
| Socket | Unix/TCP socket | Optional socket via `podman system service` |
| Security | Daemon runs as root | Each container forked by user |
| K8s integration | Via CRI-O or containerd | Via CRI-O |

**Rootless mode:**
```bash
# Podman (rootless by default)
podman run nginx
# Container runs as your UID (no root daemon)

# Docker rootless (opt-in, complex setup)
dockerd-rootless-setuptool.sh install
docker context use rootless
```

**Why Podman:**
- Security (no root daemon = smaller attack surface)
- Systemd native (generate `.service` files for containers)
- Red Hat / RHEL / Fedora default
- Drop-in Docker replacement

**Why Docker:**
- Better developer experience (Docker Desktop)
- Larger ecosystem (Docker Hub integration)
- Mature tooling (Docker Compose, buildx)
- Industry default

---

## Docker Performance & Resource Management

### 🔥 Q: How to limit container resources?

```bash
# Memory
docker run --memory=512m --memory-swap=1g myapp
# --memory: Hard limit
# --memory-swap: Memory + swap limit (-1 = unlimited swap)
# --memory-reservation: Soft limit (for scheduling)
# --oom-kill-disable: Dangerous — prevents OOM killer

# CPU
docker run --cpus=1.5 myapp                    # 1.5 CPUs worth of time
docker run --cpu-shares=512 myapp              # Relative weight (default 1024)
docker run --cpuset-cpus="0,1" myapp           # Pin to specific CPUs

# I/O
docker run --blkio-weight=500 myapp            # Relative I/O weight (10-1000)
docker run --device-read-bps /dev/sda:10mb myapp

# PIDs
docker run --pids-limit=100 myapp              # Prevent fork bombs
```

### ⭐ Q: How do you monitor Docker containers?

```bash
# Real-time stats
docker stats
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"

# Container resource usage
docker top <container>
docker inspect <container> | jq '.[0].State'

# Logs
docker logs <container>
docker logs -f --tail 100 <container>          # Follow last 100 lines
docker logs --since "2024-01-01" <container>

# Events
docker events                                   # Real-time Docker events

# Prometheus metrics (enable in daemon.json)
# "metrics-addr": "127.0.0.1:9323"
```

---

## Logging & Monitoring

### ⭐ Q: What are Docker logging drivers and how to choose one?

**Logging drivers** determine where container logs (stdout/stderr) go.

```bash
# View default logging driver
docker info --format '{{.LoggingDriver}}'
# Usually: json-file

# Configure per-container
docker run --log-driver=syslog nginx

# Configure daemon-wide (daemon.json)
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

**Common logging drivers:**

| Driver | Use Case | Storage | Log Rotation |
|--------|----------|---------|--------------|
| `json-file` (default) | Local dev, small deployments | `/var/lib/docker/containers/<id>/<id>-json.log` | Manual (`max-size`, `max-file`) |
| `syslog` | Centralized logging (rsyslog) | Remote syslog server | Syslog handles |
| `journald` | Systemd-based systems | systemd journal | journald handles |
| `fluentd` | Kubernetes, microservices | Fluentd aggregator | External |
| `awslogs` | AWS ECS, Fargate | CloudWatch Logs | AWS retention |
| `gcplogs` | Google Cloud Run, GKE | Cloud Logging | GCP retention |
| `splunk` | Enterprise logging | Splunk | Splunk retention |
| `none` | No logging (performance) | Nowhere | N/A |

**Production logging setup:**
```bash
# json-file with rotation (prevents disk fill)
docker run \
  --log-driver json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  --log-opt labels=app,env \
  --log-opt env=NODE_ENV \
  myapp

# Or Fluentd for centralized logging
docker run \
  --log-driver fluentd \
  --log-opt fluentd-address=fluentd.example.com:24224 \
  --log-opt tag="docker.{{.Name}}" \
  myapp
```

**Read logs:**
```bash
docker logs <container>                    # All logs
docker logs -f <container>                 # Follow (tail -f)
docker logs --tail 100 <container>         # Last 100 lines
docker logs --since "2025-01-01" <container>
docker logs --timestamps <container>       # Add timestamps
```

**Why logging matters:**
- `json-file` without rotation → **fills disk** → host crashes
- `none` in prod → no debugging when things break
- Centralized logging (Fluentd, CloudWatch) → aggregate across containers

### ⭐ Q: How to monitor Docker resource usage?

```bash
# Real-time stats (all containers)
docker stats
docker stats --no-stream                   # One-time snapshot
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"

# Per-container inspection
docker top <container>                     # Processes inside container
docker inspect <container> | jq '.[0].State'

# System-wide disk usage
docker system df
docker system df -v                        # Verbose (per-image, per-container)

# Prometheus metrics (enable in daemon.json)
{
  "metrics-addr": "0.0.0.0:9323",
  "experimental": true
}
# Scrape http://<host>:9323/metrics with Prometheus
```

**cAdvisor for advanced monitoring:**
```bash
docker run -d \
  --name=cadvisor \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --publish=8080:8080 \
  gcr.io/cadvisor/cadvisor:latest

# Metrics at http://localhost:8080/metrics
```

---

## Troubleshooting & Debugging

### 🔥 Q: Container exits immediately after starting. How to debug?

```bash
# 1. Check exit code
docker ps -a                                    # STATUS shows exit code
docker inspect <container> --format='{{.State.ExitCode}}'

# Common exit codes:
# 0   — Normal exit (CMD completed)
# 1   — Application error
# 137 — OOM killed (128 + SIGKILL=9)
# 139 — Segfault (128 + SIGSEGV=11)
# 143 — SIGTERM (128 + SIGTERM=15)

# 2. Check logs
docker logs <container>

# 3. Run interactively to debug
docker run -it --entrypoint /bin/sh myimage

# 4. Check the CMD/ENTRYPOINT
docker inspect myimage --format='{{.Config.Cmd}} {{.Config.Entrypoint}}'

# 5. Common causes:
# - No foreground process (daemon mode instead of foreground)
# - Missing dependencies or config files
# - Permission issues (non-root user can't bind to port 80)
# - Signal handling (PID 1 not handling SIGTERM)
```

### ⭐ Q: Docker build is slow. How to speed it up?

```bash
# 1. Optimize layer caching (order Dockerfile correctly)
# 2. Use .dockerignore to reduce build context
# 3. Use BuildKit
DOCKER_BUILDKIT=1 docker build .

# 4. Use cache mounts for package managers
RUN --mount=type=cache,target=/root/.cache/pip pip install -r requirements.txt
RUN --mount=type=cache,target=/root/.npm npm ci

# 5. Multi-stage builds to parallelize
FROM base AS deps
RUN install-deps

FROM base AS build
RUN build-app

FROM deps AS final
COPY --from=build /app .

# 6. Use remote cache
docker build --cache-from myregistry/myimage:cache .
```

### 🔥 Q: How do you debug a running container?

**Method 1: Exec into the container (if shell exists):**
```bash
docker exec -it <container> /bin/sh
# Or bash, if available:
docker exec -it <container> /bin/bash

# Run commands inside container:
ps aux                      # Running processes
netstat -tlnp               # Listening ports
cat /etc/resolv.conf        # DNS config
env                         # Environment variables
df -h                       # Disk usage
top                         # Resource usage
```

**Method 2: Inspect container metadata:**
```bash
docker inspect <container>
docker inspect <container> --format='{{.NetworkSettings.IPAddress}}'
docker inspect <container> --format='{{json .State}}' | jq
```

**Method 3: View logs:**
```bash
docker logs -f --tail 100 <container>
```

**Method 4: Attach to container (see stdout live):**
```bash
docker attach <container>
# Ctrl+P, Ctrl+Q to detach without stopping
```

**Method 5: Debug distroless/minimal images (no shell):**
```bash
# Kubernetes ephemeral debug container
kubectl debug -it <pod> --image=busybox --target=<container>

# Docker doesn't have native ephemeral containers
# Workaround: nsenter from host
docker inspect <container> --format '{{.State.Pid}}'
nsenter -t <pid> -n -u -p -m /bin/sh
# Now you're in the container's namespaces with host's /bin/sh
```

**Method 6: Copy files out for inspection:**
```bash
docker cp <container>:/app/logs /tmp/logs
docker cp <container>:/etc/nginx/nginx.conf .
```

**Method 7: Network debugging:**
```bash
# From host, test container network
docker inspect <container> --format='{{.NetworkSettings.IPAddress}}'
curl http://<container-ip>:8080

# From another container on same network
docker run --rm --network container:<container> nicolaka/netshoot
# netshoot includes: curl, dig, tcpdump, iperf, nmap, etc.
```

### ⭐ Q: Container can't reach another container. How to troubleshoot?

```bash
# 1. Check both containers are on same network
docker inspect <container1> --format='{{json .NetworkSettings.Networks}}' | jq
docker inspect <container2> --format='{{json .NetworkSettings.Networks}}' | jq

# 2. Check network exists
docker network ls
docker network inspect <network>

# 3. Test connectivity from container1 to container2
docker exec <container1> ping <container2>
docker exec <container1> curl http://<container2>:8080

# 4. Check DNS resolution (user-defined networks only)
docker exec <container1> nslookup <container2>
docker exec <container1> cat /etc/resolv.conf

# 5. Check firewall rules (iptables)
sudo iptables -L DOCKER -n
sudo iptables -L DOCKER-USER -n

# 6. Check if port is actually listening inside container2
docker exec <container2> netstat -tlnp

# Common issues:
# - Containers on default bridge (no DNS) → use user-defined network
# - Container2 listening on 127.0.0.1 instead of 0.0.0.0
# - Firewall blocking DOCKER-USER chain
# - Wrong network specified in docker-compose.yml
```

### ⭐ Q: Container is OOMKilled. How to investigate and fix?

```bash
# 1. Confirm OOM kill
docker inspect <container> --format='{{.State.OOMKilled}}'
# true = kernel killed due to memory exhaustion

# 2. Check host kernel logs
dmesg | grep -i oom
dmesg | grep -i "out of memory"
# Shows which process was killed and why

# 3. Check container memory limit
docker inspect <container> --format='{{.HostConfig.Memory}}'
# 0 = unlimited
# If set, was it too low?

# 4. Check actual memory usage before crash
docker stats <container> --no-stream

# 5. Profile memory inside container
docker exec <container> cat /proc/meminfo
docker exec <container> ps aux --sort=-%mem | head -10

# 6. Root causes:
# - Memory limit too low for workload
# - Memory leak in application
# - No GC tuning (Java -Xmx, Go GOGC, Node --max-old-space-size)
# - Loading large files into memory
# - Too many worker threads/processes

# 7. Fixes:
# Increase memory limit
docker run --memory=1g --memory-swap=2g myapp

# Or tune application memory (Java example)
docker run --memory=1g -e JAVA_OPTS="-Xmx800m -Xms800m" myapp
# Leave headroom for non-heap (200MB)

# Enable memory reservation (soft limit)
docker run --memory=1g --memory-reservation=512m myapp

# Monitor over time
watch -n 1 docker stats <container>
```

### ⭐ Q: Image is 2GB. How to shrink it?

```bash
# 1. Check current size
docker images myapp

# 2. See layer breakdown
docker history myapp:latest
dive myapp:latest          # Interactive layer explorer

# 3. Optimization strategies:

# Strategy A: Use smaller base image
FROM ubuntu:22.04          → 77MB
FROM debian:12-slim        → 46MB
FROM alpine:3.19           → 7MB
FROM gcr.io/distroless/... → 20-50MB (static or dynamic)

# Strategy B: Multi-stage build (build artifacts only)
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN go build -o /app/server
# Result: ~1GB

FROM alpine:3.19
COPY --from=builder /app/server /server
CMD ["/server"]
# Result: ~15MB

# Strategy C: Clean up in same RUN (single layer)
# BAD (each RUN = layer, bloat persists):
RUN apt-get update
RUN apt-get install -y wget curl
RUN rm -rf /var/lib/apt/lists/*    # Too late! Bloat already in layer 2

# GOOD (cleanup in same RUN):
RUN apt-get update && \
    apt-get install -y --no-install-recommends wget curl && \
    rm -rf /var/lib/apt/lists/*

# Strategy D: Don't include build tools in final image
# Move gcc, make, dev headers to builder stage only

# Strategy E: .dockerignore (reduce context)
# Exclude node_modules, .git, tests, docs

# Strategy F: Use .dockerignore and COPY selectively
# Don't COPY . . blindly — copy only what's needed

# 4. Measure improvement
docker images myapp
```

---

## Scenario-Based Questions

### ⭐ Q: Your Docker container is consuming too much memory. Investigate and fix.

```bash
# 1. Identify memory usage
docker stats <container>

# 2. Check if OOM killed
docker inspect <container> --format='{{.State.OOMKilled}}'
dmesg | grep -i oom

# 3. Check memory limit
docker inspect <container> --format='{{.HostConfig.Memory}}'
# 0 = unlimited

# 4. Profile memory inside container
docker exec <container> cat /proc/meminfo
docker exec <container> top

# 5. Set appropriate limits
docker update --memory=512m --memory-swap=1g <container>

# 6. Root causes:
# - Memory leak in application
# - No garbage collection tuning (Java -Xmx, Go GOGC)
# - Caching without eviction
# - Too many worker processes/threads
```

### ⭐ Q: How do you do zero-downtime deployments with Docker Compose?

```bash
# Option 1: Rolling update with scale
docker compose up -d --scale app=3 --no-recreate
# Then update and remove old containers one by one

# Option 2: Blue-Green with Nginx
# 1. Start new version on different port
docker compose -f docker-compose.blue.yml up -d

# 2. Health check new version
curl http://localhost:8081/health

# 3. Switch Nginx upstream to new port
# 4. Stop old version

# Option 3: Use Docker Swarm or Kubernetes for proper orchestration
# Docker Swarm:
docker service update --image myapp:v2 --update-parallelism 1 --update-delay 10s myservice
```

### 🔥 Q: How do you handle secrets in Docker?

```bash
# BAD: Environment variables (visible in docker inspect)
docker run -e DB_PASSWORD=secret myapp

# BETTER: Docker secrets (Swarm mode)
echo "secret" | docker secret create db_password -
docker service create --secret db_password myapp
# Available at /run/secrets/db_password inside container

# BETTER: Mount from external secret manager
docker run -v /path/to/secrets:/run/secrets:ro myapp

# BEST: Use external secret manager (Vault, AWS Secrets Manager)
# Application fetches secrets at runtime via API
```

### ⭐ Q: How do you secure the container image supply chain?

**Full supply chain security:**

```
┌─────────────────────────────────────────────────────────┐
│ 1. Base Image Selection                                 │
│    ✓ Trusted sources (official images, minimal distros) │
│    ✓ Verified publishers (Docker Official Images)       │
│    ✓ Distroless / Wolfi / Chainguard                    │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│ 2. Build Time                                            │
│    ✓ Scan base image (Trivy, Grype)                     │
│    ✓ Pin dependencies (package-lock.json, go.sum)       │
│    ✓ SBOM generation (Syft)                             │
│    ✓ Sign image (Cosign, Notary)                        │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│ 3. Registry Storage                                      │
│    ✓ Private registry (Harbor, ECR, GCR, ACR)           │
│    ✓ Immutable tags (never overwrite :v1.2.3)           │
│    ✓ Image retention policies                           │
│    ✓ Registry scanning (auto-scan on push)              │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│ 4. Deploy Time                                           │
│    ✓ Verify signature (admission controller)            │
│    ✓ Policy enforcement (OPA, Kyverno)                  │
│    ✓ Runtime scanning (Falco)                           │
│    ✓ Only pull from trusted registries                  │
└─────────────────────────────────────────────────────────┘
```

**Practical implementation:**

```bash
# 1. Generate SBOM (Software Bill of Materials)
syft myapp:latest -o cyclonedx-json > sbom.json
# Lists all packages and versions in the image

# 2. Scan for vulnerabilities
trivy image --severity HIGH,CRITICAL myapp:latest
grype myapp:latest

# 3. Sign image with Cosign
cosign generate-key-pair
cosign sign --key cosign.key myregistry/myapp:v1.2.3

# 4. Verify signature at deploy time
cosign verify --key cosign.pub myregistry/myapp:v1.2.3

# 5. Kubernetes admission controller (enforce signature)
# Using Sigstore Policy Controller or OPA Gatekeeper
```

**Registry best practices:**

```yaml
# Immutable tags (never overwrite)
✅ myapp:v1.2.3-abc123           # Semantic version + git SHA
✅ myapp:sha256:<digest>          # Content-addressable
❌ myapp:latest                   # Mutable, don't use in prod
❌ myapp:v1                       # Mutable (v1.0.0 vs v1.9.9)

# Tag strategy for CI/CD:
main → myapp:main-abc123
PR #42 → myapp:pr-42-def456
Release → myapp:v1.2.3           # Immutable, never overwritten
          myapp:v1.2             # Mutable (latest patch)
          myapp:v1               # Mutable (latest minor)
          myapp:latest           # Mutable (latest release)
```

**Image provenance (SLSA):**
```bash
# Attach build provenance to image
docker buildx build \
  --provenance=true \
  --sbom=true \
  --tag myapp:v1.2.3 \
  --push .

# Verify provenance
cosign verify-attestation --type slsaprovenance myapp:v1.2.3
```

---

## Trending Topics (2025-2026)

### ⭐ Q: What are rootless containers and why use them?

**Rootless containers** run without any root privileges — not just non-root user inside container, but **no root daemon** on the host.

**Traditional (rootful) Docker:**
```
Host:
├── dockerd (runs as root UID 0)        ← Attack surface
├── Container A (process appears as root in container)
│   └── But restricted via USER directive, capabilities, etc.
```

**Rootless Docker/Podman:**
```
Host:
├── dockerd or podman (runs as user UID 1000)  ← No root!
├── Container A (process UID 1000 on host)
│   └── Appears as UID 0 inside container via user namespaces
```

**How it works:**
- **User namespaces** map container UID 0 (root) → host UID 1000+ (non-root)
- No privileged daemon
- Can't bind to ports < 1024 (without CAP_NET_BIND_SERVICE)
- Storage driver: fuse-overlayfs (since OverlayFS needs root)

**Setup rootless Docker:**
```bash
# Install rootless Docker
dockerd-rootless-setuptool.sh install

# Or use Podman (rootless by default)
podman run nginx
```

**Why rootless:**
- **Security**: Container escape doesn't give root on host
- **Multi-tenancy**: Users can run their own containers without admin
- **Zero-trust**: Assume daemon is compromised → blast radius limited

**Limitations:**
- No port binding < 1024 (use 8080 instead of 80)
- Performance overhead (user namespace translation)
- Some storage drivers unsupported
- cgroup v2 required for resource limits

### 💡 Q: What are Wolfi and Chainguard images?

**Wolfi** is a Linux distribution (like Alpine) designed **specifically for containers**, built by Chainguard.

| Feature | Alpine | Distroless | Wolfi/Chainguard |
|---------|--------|-----------|------------------|
| Base size | ~7MB | ~20-50MB | ~10-30MB |
| Package manager | apk | None | apk (Wolfi packages) |
| C library | musl | glibc (debian) or none | glibc |
| Shell | Yes (ash) | No | No (Wolfi base has none) |
| CVE response | Good | Google-managed | **Industry-leading** (same-day) |
| SBOM | Manual | Google | **Built-in** |
| Provenance | None | Google | **Signed by default** |
| Updates | Community | Google | Chainguard |

**Use cases:**

```dockerfile
# Traditional Alpine
FROM alpine:3.19
RUN apk add python3 py3-pip
# Result: musl libc, works but some binaries need recompile

# Chainguard Python (distroless-like, glibc)
FROM cgr.dev/chainguard/python:latest
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
CMD ["python", "app.py"]
# Result: minimal, glibc, no shell, signed, SBOM included

# Wolfi base (if you need to install packages)
FROM cgr.dev/chainguard/wolfi-base
RUN apk add python3 py3-pip
# Result: minimal, glibc, apk available
```

**Why Chainguard/Wolfi:**
- **Supply chain security**: Every package signed + SBOM
- **Fast CVE patching**: Chainguard commits to same-day critical patches
- **Compliance**: Built-in SLSA provenance
- **Industry adoption**: Google, Microsoft, Snowflake using them

**Free vs Paid:**
- Free tier: Chainguard Images (public, community-supported)
- Paid tier: Guaranteed SLAs, private images, custom builds

### 💡 Q: What is BuildKit's experimental inline cache?

**Problem:** Remote cache (`--cache-from`) requires separate cache image push.

**Solution:** Inline cache embeds cache metadata **inside the image itself**.

```bash
# Old approach (separate cache image)
docker build --cache-from myregistry/myapp:cache -t myregistry/myapp:latest .
docker tag myregistry/myapp:latest myregistry/myapp:cache
docker push myregistry/myapp:cache    # Separate push for cache

# New approach (inline cache)
docker buildx build \
  --cache-to type=inline \
  --tag myregistry/myapp:latest \
  --push .

# Later, reuse cache from the same image
docker buildx build \
  --cache-from myregistry/myapp:latest \
  --tag myregistry/myapp:v2 \
  --push .
```

**Why inline cache:**
- No separate cache image to manage
- Simpler CI/CD (one push instead of two)
- Cache travels with the image
- Ideal for multi-branch workflows (cache per branch)

**Alternatives:**

```bash
# Registry cache (BuildKit's cache storage API)
docker buildx build \
  --cache-to type=registry,ref=myregistry/myapp:cache \
  --cache-from type=registry,ref=myregistry/myapp:cache \
  .

# Local cache (fast, but not shared across machines)
docker buildx build \
  --cache-to type=local,dest=/tmp/cache \
  --cache-from type=local,src=/tmp/cache \
  .

# GitHub Actions cache
docker buildx build \
  --cache-to type=gha \
  --cache-from type=gha \
  .
```

### 💡 Q: How do you optimize for cold start (serverless / FaaS)?

**Cold start** = time from "start container" to "application ready" (critical for Lambda, Cloud Run, Fargate).

**Optimization strategies:**

1. **Minimize image size:**
   ```dockerfile
   # Smaller image = faster pull
   FROM gcr.io/distroless/nodejs20-debian12
   # Alpine is small but may have slower musl libc
   ```

2. **Layer caching:**
   - Push images to same region as FaaS platform
   - Pre-pull common base images

3. **Reduce dependencies:**
   ```dockerfile
   # Don't install dev dependencies
   RUN npm ci --only=production
   RUN pip install --no-cache-dir -r requirements.txt
   ```

4. **Lazy loading (application-level):**
   ```javascript
   // BAD: Load everything upfront
   const heavyModule = require('./heavy');

   // GOOD: Lazy load
   exports.handler = async (event) => {
     const heavyModule = await import('./heavy');
     // ...
   };
   ```

5. **Use FaaS-optimized images:**
   ```dockerfile
   # AWS Lambda base images (pre-warmed)
   FROM public.ecr.aws/lambda/nodejs:20

   # Google Cloud Run optimized
   FROM gcr.io/buildpacks/builder:v1
   ```

6. **Keep functions warm (if allowed):**
   ```bash
   # Periodic pings to prevent cold starts
   # (incurs cost, but reduces latency)
   ```

7. **Use provisioned concurrency (AWS Lambda):**
   ```bash
   aws lambda put-provisioned-concurrency-config \
     --function-name myfunction \
     --provisioned-concurrent-executions 10
   ```

**Typical cold start times (2025):**
- AWS Lambda (Container image, 256MB): 1-3 seconds
- Google Cloud Run (minimal image): 0.5-1 second
- Azure Container Instances: 2-5 seconds

---

## Key Resources

### Official Documentation
- **Docker Documentation** — https://docs.docker.com
- **Dockerfile Best Practices** — https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
- **BuildKit Reference** — https://docs.docker.com/build/buildkit/
- **OCI Specifications** — https://github.com/opencontainers/runtime-spec

### Security
- **Docker Security Cheat Sheet** — OWASP Docker Security
- **CIS Docker Benchmark** — https://www.cisecurity.org/benchmark/docker
- **Trivy** — Vulnerability scanner: https://github.com/aquasecurity/trivy
- **Grype** — Vulnerability scanner: https://github.com/anchore/grype
- **Cosign** — Container signing: https://github.com/sigstore/cosign
- **Syft** — SBOM generation: https://github.com/anchore/syft

### Tools
- **Dive** — Explore Docker image layers: https://github.com/wagoodman/dive
- **Hadolint** — Dockerfile linter: https://github.com/hadolint/hadolint
- **Docker Slim** — Minify images: https://github.com/slimtoolkit/slim
- **Skopeo** — Inspect images without pulling: https://github.com/containers/skopeo
- **Podman** — Daemonless Docker alternative: https://podman.io

### Image Registries
- **Harbor** — Open-source registry: https://goharbor.io
- **Chainguard Images** — Minimal, secure base images: https://chainguard.dev
- **Google Distroless** — https://github.com/GoogleContainerTools/distroless
- **Wolfi OS** — Container-optimized Linux: https://wolfi.dev

### Books & Learning
- **Container Security** — Liz Rice (O'Reilly)
- **Docker Deep Dive** — Nigel Poulton
- **Kubernetes Patterns** — Bilgin Ibryam & Roland Huß (O'Reilly)
- **Building Secure and Reliable Systems** — Google SRE (O'Reilly)
