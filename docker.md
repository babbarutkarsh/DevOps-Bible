# Docker & Containers — DevOps Interview Preparation

## Table of Contents

- [Container Fundamentals](#container-fundamentals)
- [Docker Architecture](#docker-architecture)
- [Dockerfile Best Practices](#dockerfile-best-practices)
- [Docker Networking](#docker-networking)
- [Docker Storage & Volumes](#docker-storage--volumes)
- [Docker Compose](#docker-compose)
- [Image Management & Registries](#image-management--registries)
- [Container Security](#container-security)
- [Container Runtimes](#container-runtimes)
- [Docker Performance & Resource Management](#docker-performance--resource-management)
- [Troubleshooting](#troubleshooting)
- [Scenario-Based Questions](#scenario-based-questions)

---

## Container Fundamentals

### Q: What are containers and how are they different from VMs?

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

### Q: What Linux kernel features enable containers?

1. **Namespaces** — Isolation of system resources:

| Namespace | Isolates | Flag |
|-----------|----------|------|
| PID | Process IDs | `CLONE_NEWPID` |
| NET | Network stack | `CLONE_NEWNET` |
| MNT | Mount points | `CLONE_NEWNS` |
| UTS | Hostname | `CLONE_NEWUTS` |
| IPC | Inter-process communication | `CLONE_NEWIPC` |
| USER | UID/GID mappings | `CLONE_NEWUSER` |
| CGROUP | Cgroup root | `CLONE_NEWCGROUP` |

2. **Cgroups** — Resource limits (CPU, memory, I/O, PIDs)

3. **Union Filesystems** — Layer multiple filesystems (OverlayFS, AUFS)

4. **seccomp** — System call filtering

5. **AppArmor / SELinux** — Mandatory access control

### Q: What is an OCI (Open Container Initiative)?

The **OCI** defines industry standards for containers:

- **Runtime Specification** — How to run a container (lifecycle, configuration)
- **Image Specification** — Container image format (layers, manifest, config)
- **Distribution Specification** — How to distribute images (registry API)

This means you can build an image with Docker and run it with any OCI-compliant runtime (containerd, CRI-O, Podman).

---

## Docker Architecture

### Q: Explain Docker's architecture.

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

### Q: What is the difference between `docker run`, `docker create`, and `docker start`?

| Command | What it does |
|---------|-------------|
| `docker create` | Creates a container (writable layer) but doesn't start it |
| `docker start` | Starts an existing stopped container |
| `docker run` | `docker create` + `docker start` + `docker attach` (if interactive) |

---

## Dockerfile Best Practices

### Q: Write an optimized multi-stage Dockerfile.

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

### Q: Explain Dockerfile layer caching and optimization.

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

### Q: What is the difference between CMD, ENTRYPOINT, and RUN?

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

### Q: What are the different forms of CMD/ENTRYPOINT?

| Form | Syntax | Shell | Signal Handling |
|------|--------|-------|----------------|
| **Exec form** (preferred) | `["executable", "arg1"]` | No shell (`/bin/sh -c` not used) | PID 1, receives signals directly |
| **Shell form** | `executable arg1` | Runs via `/bin/sh -c` | Shell is PID 1, signals may not reach app |

**Always use exec form** in production — shell form prevents proper signal handling (SIGTERM for graceful shutdown).

---

## Docker Networking

### Q: Explain Docker networking modes.

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

### Q: How does Docker port mapping work?

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

### Q: Explain Docker storage drivers and copy-on-write.

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

### Q: Volumes vs Bind Mounts vs tmpfs?

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

### Q: Explain Docker Compose and write a production-like example.

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

### Q: How do Docker image layers work?

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

### Q: How to reduce Docker image size?

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

### Q: How do you scan images for vulnerabilities?

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

### Q: Docker security best practices?

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

### Q: What is Docker socket mounting and why is it dangerous?

```bash
docker run -v /var/run/docker.sock:/var/run/docker.sock myapp
```

This gives the container **full control over the Docker daemon** — it can:
- Start/stop any container
- Mount host filesystems
- Effectively gain root access to the host

**When needed** (CI/CD, monitoring): Use alternatives like Docker-in-Docker (dind), Kaniko for builds, or limit access with a Docker socket proxy.

---

## Container Runtimes

### Q: Explain the container runtime landscape.

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

### Q: Docker vs Podman?

| Feature | Docker | Podman |
|---------|--------|--------|
| Daemon | Yes (dockerd) | **Daemonless** |
| Root | Requires root daemon | Rootless by default |
| CLI compatibility | `docker ...` | `podman ...` (drop-in compatible) |
| Pods | No native concept | Yes (like K8s pods) |
| Compose | `docker compose` | `podman-compose` or `podman compose` |
| Systemd integration | Limited | Native (generate systemd units) |
| Fork/exec model | Client → daemon → containerd → runc | Direct fork/exec |

---

## Docker Performance & Resource Management

### Q: How to limit container resources?

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

### Q: How do you monitor Docker containers?

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

## Troubleshooting

### Q: Container exits immediately after starting. How to debug?

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

### Q: Docker build is slow. How to speed it up?

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

---

## Scenario-Based Questions

### Q: Your Docker container is consuming too much memory. Investigate and fix.

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

### Q: How do you do zero-downtime deployments with Docker Compose?

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

### Q: How do you handle secrets in Docker?

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

---

## Key Resources

- **Docker Documentation** — https://docs.docker.com
- **Dockerfile Best Practices** — https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
- **Docker Security Cheat Sheet** — OWASP Docker Security
- **Dive** — Tool for exploring Docker image layers: https://github.com/wagoodman/dive
- **Hadolint** — Dockerfile linter: https://github.com/hadolint/hadolint
- **Container Security (book)** — Liz Rice
- **Docker Deep Dive (book)** — Nigel Poulton
