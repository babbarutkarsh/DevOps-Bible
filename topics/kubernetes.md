---
title: Kubernetes
nav_order: 21
description: "Kubernetes — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# Kubernetes — DevOps Interview Preparation

> **Question frequency:** 🔥 very frequently asked · ⭐ important · 💡 good to know

## Table of Contents

- [Architecture & Components](#architecture--components)
- [Pods](#pods)
- [Workload Resources](#workload-resources)
- [Services & Networking](#services--networking)
- [Ingress & Gateway API](#ingress--gateway-api)
- [Storage](#storage)
- [ConfigMaps & Secrets](#configmaps--secrets)
- [RBAC & Security](#rbac--security)
- [Scheduling & Node Management](#scheduling--node-management)
- [Resource Management](#resource-management)
- [Health Checks & Self-Healing](#health-checks--self-healing)
- [Autoscaling](#autoscaling)
- [Deployment Strategies](#deployment-strategies)
- [Helm](#helm)
- [Operators & CRDs](#operators--crds)
- [Observability in Kubernetes](#observability-in-kubernetes)
- [Troubleshooting](#troubleshooting)
- [Network Policies](#network-policies)
- [Service Mesh](#service-mesh)
- [etcd](#etcd)
- [Multi-Cluster & Federation](#multi-cluster--federation)
- [Scenario-Based Questions](#scenario-based-questions)
- [Deep Troubleshooting & Debug Scenarios](#deep-troubleshooting--debug-scenarios)
- [Trending & Platform Engineering Questions (2025-26)](#trending--platform-engineering-questions-2025-26)

---

## Architecture & Components

### 🔥 Q: Explain Kubernetes architecture.

```
┌──────────────────────── Control Plane ─────────────────────────┐
│                                                                 │
│  ┌────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │ kube-api   │  │ etcd         │  │ kube-scheduler         │ │
│  │ server     │◄►│ (key-value   │  │ (assigns pods to nodes)│ │
│  │ (REST API) │  │  store)      │  └────────────────────────┘ │
│  └─────┬──────┘  └──────────────┘                              │
│        │                                                        │
│        │         ┌──────────────────────────────────────────┐  │
│        │         │ kube-controller-manager                   │  │
│        │         │ (Deployment, ReplicaSet, Node, Job, etc.) │  │
│        │         └──────────────────────────────────────────┘  │
│        │                                                        │
│        │         ┌──────────────────────────────────────────┐  │
│        │         │ cloud-controller-manager (optional)       │  │
│        │         │ (LB, routes, volumes for cloud providers) │  │
│        │         └──────────────────────────────────────────┘  │
└────────┼────────────────────────────────────────────────────────┘
         │
         │ kubelet watches API server
         ▼
┌──────────────────────── Worker Node ───────────────────────────┐
│                                                                 │
│  ┌────────────┐  ┌──────────────┐  ┌────────────────────────┐ │
│  │ kubelet    │  │ kube-proxy   │  │ Container Runtime      │ │
│  │ (node      │  │ (networking  │  │ (containerd / CRI-O)   │ │
│  │  agent)    │  │  rules)      │  │                        │ │
│  └────────────┘  └──────────────┘  └────────────────────────┘ │
│                                                                 │
│  ┌─── Pod ────┐  ┌─── Pod ────┐  ┌─── Pod ────────────────┐  │
│  │ Container  │  │ Container  │  │ Container A            │  │
│  │            │  │            │  │ Container B (sidecar)  │  │
│  └────────────┘  └────────────┘  └────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 🔥 Q: Explain each control plane component.

| Component | Role | What Happens if it Fails |
|-----------|------|-------------------------|
| **kube-apiserver** | REST API frontend for all operations. All components communicate through it. Validates and persists state to etcd. | Cluster is unmanageable. Existing workloads continue running but no changes possible. |
| **etcd** | Distributed key-value store. Single source of truth for all cluster state. | Data loss risk. Cluster cannot function. (Always back up etcd!) |
| **kube-scheduler** | Watches for unscheduled pods, assigns them to nodes based on resource requirements, affinity, taints/tolerations. | New pods stay Pending. Existing pods unaffected. |
| **kube-controller-manager** | Runs controllers (reconciliation loops): Deployment, ReplicaSet, Node, Job, ServiceAccount, etc. | Cluster stops self-healing. No scaling, no rescheduling of failed pods. |
| **cloud-controller-manager** | Integrates with cloud provider APIs for LoadBalancers, Routes, Volumes. | Cloud resources not provisioned (LBs, EBS volumes). |

### ⭐ Q: Explain each worker node component.

| Component | Role |
|-----------|------|
| **kubelet** | Agent on each node. Ensures containers in pods are running. Reports node status to API server. Pulls images, mounts volumes. |
| **kube-proxy** | Maintains network rules (iptables/IPVS) for Service → Pod routing. |
| **Container Runtime** | Actually runs containers. containerd (default), CRI-O, or any CRI-compliant runtime. |

### 🔥 Q: What happens when you run `kubectl apply -f deployment.yaml`?

```
1. kubectl sends POST/PUT to kube-apiserver
       │
2. API server authenticates (certs/tokens) → authorizes (RBAC) → validates
       │
3. API server persists the Deployment object to etcd
       │
4. Deployment controller (in controller-manager) detects new Deployment
   → Creates a ReplicaSet
       │
5. ReplicaSet controller detects new ReplicaSet
   → Creates Pod objects (desired state in etcd)
       │
6. Scheduler detects unscheduled Pods
   → Evaluates node resources, affinity, taints
   → Binds each Pod to a node (writes to etcd)
       │
7. Kubelet on the assigned node detects the new Pod binding
   → Pulls container image (via CRI)
   → Creates and starts containers
   → Sets up networking (via CNI plugin)
   → Mounts volumes (via CSI plugin)
       │
8. Kubelet reports Pod status back to API server
       │
9. kube-proxy updates iptables/IPVS rules if Pod is part of a Service
```

---

## Pods

### 🔥 Q: What is a Pod? Why not just containers?

A **Pod** is the smallest deployable unit in Kubernetes — a group of one or more containers that:
- Share the same **network namespace** (same IP, can reach each other via `localhost`)
- Share the same **IPC namespace**
- Can share **volumes**
- Are co-scheduled on the same node

**Why Pods and not just containers?**
- **Sidecar pattern:** Log collector alongside app container
- **Ambassador pattern:** Proxy container for external services
- **Adapter pattern:** Transform output of main container
- **Init containers:** Run setup tasks before main container starts

### ⭐ Q: What are Init Containers?

Init containers run **before** app containers start. They run to completion sequentially.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  initContainers:
  - name: wait-for-db
    image: busybox
    command: ['sh', '-c', 'until nc -z postgres 5432; do echo waiting; sleep 2; done']
  - name: run-migrations
    image: myapp:latest
    command: ['python', 'manage.py', 'migrate']
  containers:
  - name: app
    image: myapp:latest
    ports:
    - containerPort: 8080
```

**Use cases:**
- Wait for dependencies (database, external service)
- Run database migrations
- Download configuration from external source
- Set up permissions on shared volumes

### 🔥 Q: Explain Pod lifecycle and phases.

| Phase | Description |
|-------|-------------|
| **Pending** | Pod accepted but containers not yet created. Could be scheduling, image pulling. |
| **Running** | At least one container is running. |
| **Succeeded** | All containers exited with code 0 (common for Jobs). |
| **Failed** | All containers terminated, at least one exited non-zero. |
| **Unknown** | Node communication failure. |

**Container states:** `Waiting`, `Running`, `Terminated`

**Common reasons for Pending:**
- `Unschedulable` — No node has enough resources
- `ImagePullBackOff` — Can't pull image
- `PodExceedsFreeCPU/Memory` — Insufficient resources

### 💡 Q: What is a static pod?

Static pods are managed directly by the **kubelet** on a specific node, without the API server.

- Defined as YAML files in `/etc/kubernetes/manifests/` (or configured path)
- Kubelet watches this directory and creates/manages pods automatically
- Control plane components (apiserver, etcd, scheduler, controller-manager) are typically static pods in kubeadm clusters
- API server creates a **mirror pod** (read-only) so they appear in `kubectl get pods`

---

## Workload Resources

### 🔥 Q: Explain Deployment, ReplicaSet, DaemonSet, StatefulSet, and Job.

| Resource | Purpose | Use Case |
|----------|---------|----------|
| **Deployment** | Declarative updates for Pods. Manages ReplicaSets. | Stateless apps (web servers, APIs) |
| **ReplicaSet** | Ensures N pod replicas are running. | Don't use directly — Deployments manage these |
| **DaemonSet** | Ensures one pod per node (or subset). | Log collectors, monitoring agents, CNI plugins |
| **StatefulSet** | Ordered, stable pods with persistent identity. | Databases, Kafka, ZooKeeper, etcd |
| **Job** | Runs pods to completion. | Batch processing, migrations, one-time tasks |
| **CronJob** | Scheduled Jobs. | Backups, reports, cleanup tasks |

### ⭐ Q: Explain StatefulSet guarantees.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: "postgres"     # Required — headless service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:       # Each pod gets its own PVC
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: gp3
      resources:
        requests:
          storage: 50Gi
```

**StatefulSet guarantees:**

| Feature | Deployment | StatefulSet |
|---------|-----------|-------------|
| Pod names | Random (myapp-7b8f9d-x4k2l) | **Ordered** (postgres-0, postgres-1, postgres-2) |
| Startup order | Parallel | **Sequential** (0 → 1 → 2) |
| Shutdown order | Parallel | **Reverse** (2 → 1 → 0) |
| Network identity | Random | **Stable** DNS: `postgres-0.postgres.namespace.svc.cluster.local` |
| Storage | Shared or ephemeral | **Persistent** per-pod PVC (survives rescheduling) |
| Scaling | Any order | **Ordered** |

### ⭐ Q: DaemonSet use cases and updates.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: fluentd
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1       # Update one node at a time
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule    # Also run on control plane nodes
      containers:
      - name: fluentd
        image: fluent/fluentd:v1.16
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

---

## Services & Networking

### 🔥 Q: Explain Kubernetes Service types.

```
                    Internet
                       │
                       ▼
              ┌─── LoadBalancer ───┐  (Cloud LB → NodePort → ClusterIP → Pod)
              │    (External IP)   │
              └────────┬───────────┘
                       │
              ┌─── NodePort ───────┐  (Any Node IP:30000-32767 → ClusterIP → Pod)
              │  (Node IP:Port)    │
              └────────┬───────────┘
                       │
              ┌─── ClusterIP ──────┐  (Virtual IP, internal only)
              │  (10.96.x.x)       │
              └────────┬───────────┘
                       │
              ┌─── Pods ───────────┐
              │  (10.244.x.x)      │
              └────────────────────┘
```

| Type | Scope | How it Works |
|------|-------|-------------|
| **ClusterIP** (default) | Internal only | Virtual IP accessible within cluster. kube-proxy creates iptables/IPVS rules. |
| **NodePort** | External via node IP | Allocates port 30000-32767 on every node. Routes to ClusterIP. |
| **LoadBalancer** | External via cloud LB | Creates cloud provider LB → NodePort → ClusterIP. |
| **ExternalName** | DNS alias | Returns CNAME record. No proxying. |
| **Headless** | Direct pod access | ClusterIP: None. DNS returns pod IPs directly. Used by StatefulSets. |

```yaml
# ClusterIP Service
apiVersion: v1
kind: Service
metadata:
  name: my-api
spec:
  type: ClusterIP
  selector:
    app: my-api
  ports:
  - port: 80          # Service port
    targetPort: 8080   # Container port
    protocol: TCP

# Headless Service (for StatefulSets)
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  clusterIP: None     # Headless!
  selector:
    app: postgres
  ports:
  - port: 5432
```

### 🔥 Q: How does Kubernetes DNS work?

**CoreDNS** runs as a deployment in `kube-system` namespace.

**DNS records created automatically:**

| Object | DNS Name | Example |
|--------|----------|---------|
| Service | `<svc>.<ns>.svc.cluster.local` | `my-api.default.svc.cluster.local` |
| Pod | `<pod-ip-dashed>.<ns>.pod.cluster.local` | `10-244-1-5.default.pod.cluster.local` |
| StatefulSet Pod | `<pod-name>.<svc>.<ns>.svc.cluster.local` | `postgres-0.postgres.default.svc.cluster.local` |

```bash
# From inside a pod:
nslookup my-api                          # Works (same namespace)
nslookup my-api.other-namespace          # Cross-namespace
nslookup my-api.other-namespace.svc.cluster.local  # Fully qualified

# DNS search domains (from /etc/resolv.conf in pod):
# search default.svc.cluster.local svc.cluster.local cluster.local
```

### 🔥 Q: Explain Kubernetes networking model (CNI).

**Kubernetes networking requirements (the "flat network" model):**
1. Every Pod gets its own IP address
2. Pods on any node can communicate with all pods on any node without NAT
3. Agents on a node can communicate with all pods on that node

**CNI (Container Network Interface) plugins implement this:**

| Plugin | Approach | Features |
|--------|----------|---------|
| **Calico** | BGP routing or VXLAN overlay | Network policies, high performance |
| **Cilium** | eBPF-based | Network policies, observability, service mesh |
| **Flannel** | VXLAN overlay | Simple, lightweight |
| **Weave** | Mesh overlay | Easy setup, encryption |
| **AWS VPC CNI** | Native VPC IPs | Pods get real VPC IPs, best for EKS |

```
Node 1 (10.0.1.10)              Node 2 (10.0.2.10)
┌────────────────────┐           ┌────────────────────┐
│ Pod A (10.244.1.2) │           │ Pod C (10.244.2.2) │
│ Pod B (10.244.1.3) │           │ Pod D (10.244.2.3) │
│        │           │           │        │           │
│     cbr0/cni0      │           │     cbr0/cni0      │
│   (10.244.1.1)     │           │   (10.244.2.1)     │
└────────┬───────────┘           └────────┬───────────┘
         │                                │
    ┌────┴────────── Overlay / Routing ───┴────┐
    │              (VXLAN, BGP, etc.)           │
    └──────────────────────────────────────────┘
```

### ⭐ Q: Explain kube-proxy modes.

**kube-proxy** implements Service routing. Three modes:

| Mode | How it Works | Pros | Cons |
|------|-------------|------|------|
| **iptables** (default) | Creates iptables rules for each Service/Endpoint | Simple, widely supported | Performance degrades with many services (10k+ rules), no connection-level load balancing |
| **IPVS** | Uses Linux IPVS for load balancing | Better performance at scale, more load-balancing algorithms (rr, lc, dh, etc.) | Requires IPVS kernel modules |
| **eBPF** (Cilium) | BPF programs in kernel | Highest performance, smallest overhead, per-packet decisions | Requires modern kernel (5.x+), only Cilium/Calico eBPF mode |

```bash
# Check current mode
kubectl -n kube-system get cm kube-proxy -o yaml | grep mode

# IPVS example
spec:
  mode: ipvs
  ipvs:
    scheduler: "rr"   # round-robin, lc (least connection), dh (destination hashing)
```

**eBPF vs traditional:**
- **Traditional (iptables/IPVS):** Userspace kube-proxy writes kernel rules
- **eBPF (Cilium):** BPF programs loaded into kernel, no kube-proxy needed. Far faster, L7-aware.

### ⭐ Q: What is the Gateway API and how does it differ from Ingress?

**Gateway API** is the next-gen successor to Ingress (K8s 1.29+ GA).

| Feature | Ingress | Gateway API |
|---------|---------|-------------|
| **Role separation** | Single Ingress resource | Gateway (infra) + HTTPRoute/TCPRoute (app) — role-oriented |
| **Protocol support** | HTTP/HTTPS only | HTTP, HTTPS, TCP, UDP, gRPC, TLS passthrough |
| **Routing** | Simple path/host | Advanced (header, query param, method, weight-based) |
| **Multi-tenancy** | Weak | Strong — ReferenceGrant for cross-namespace access |
| **Vendor extensions** | Annotations (non-portable) | CRDs (typed, validated) |

```yaml
# Gateway (infra admin creates)
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: prod-gateway
  namespace: infra
spec:
  gatewayClassName: istio
  listeners:
  - name: http
    protocol: HTTP
    port: 80
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      certificateRefs:
      - name: prod-cert

---
# HTTPRoute (app team creates)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
  namespace: production
spec:
  parentRefs:
  - name: prod-gateway
    namespace: infra
  hostnames: ["app.example.com"]
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-service
      port: 80
      weight: 90
    - name: api-canary
      port: 80
      weight: 10
```

**Why Gateway API matters in 2025-26:**
- Standard for modern service mesh (Istio, Linkerd, Consul)
- Better multi-tenant isolation
- Role-oriented design maps to real org structure

---

## Ingress & Gateway API

### 🔥 Q: What is Ingress?

Ingress exposes HTTP/HTTPS routes from outside the cluster to services within.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "10"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: app-tls
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

**Ingress Controllers:** Nginx Ingress, Traefik, HAProxy, AWS ALB Ingress Controller, Istio Gateway, Contour

**Ingress vs LoadBalancer Service:**

| Feature | Ingress | LoadBalancer Service |
|---------|---------|---------------------|
| Layer | L7 (HTTP/HTTPS) | L4 (TCP/UDP) |
| Multiple services | Yes (path/host routing) | One LB per service |
| TLS termination | Yes | Depends on cloud LB |
| Cost | One LB for many services | One LB per service ($$$) |

---

## Storage

### 🔥 Q: Explain Kubernetes storage architecture.

```
┌─────────────────────────────────────────────────┐
│              Pod                                  │
│  ┌─────────────────┐                             │
│  │   Container     │                             │
│  │   ┌───────────┐ │                             │
│  │   │ mountPath │ │                             │
│  │   └─────┬─────┘ │                             │
│  └─────────┼───────┘                             │
│            │                                      │
│     ┌──────▼──────┐                              │
│     │   Volume    │                              │
│     └──────┬──────┘                              │
└────────────┼─────────────────────────────────────┘
             │
      ┌──────▼──────────────┐
      │  PersistentVolume   │  ← Actual storage (EBS, NFS, etc.)
      │  Claim (PVC)        │
      └──────┬──────────────┘
             │ bound to
      ┌──────▼──────────────┐
      │  PersistentVolume   │  ← Storage resource
      │  (PV)               │
      └──────┬──────────────┘
             │ provisioned by
      ┌──────▼──────────────┐
      │  StorageClass       │  ← Defines how to provision (gp3, io2, etc.)
      └─────────────────────┘
             │ implements
      ┌──────▼──────────────┐
      │  CSI Driver         │  ← Container Storage Interface plugin
      └─────────────────────┘
```

```yaml
# StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "5000"
  throughput: "250"
reclaimPolicy: Retain        # Retain or Delete
volumeBindingMode: WaitForFirstConsumer  # Wait until pod is scheduled
allowVolumeExpansion: true

---
# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
  - ReadWriteOnce            # RWO, ROX, RWX, RWOP
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 100Gi
```

**Access Modes:**

| Mode | Abbr | Description |
|------|------|-------------|
| ReadWriteOnce | RWO | Single node read/write |
| ReadOnlyMany | ROX | Multiple nodes read-only |
| ReadWriteMany | RWX | Multiple nodes read/write (NFS, EFS) |
| ReadWriteOncePod | RWOP | Single pod read/write (K8s 1.27+) |

### ⭐ Q: What is CSI and why does it matter?

**CSI (Container Storage Interface)** — Standardized API for storage providers. Replaces in-tree volume plugins.

**CSI architecture:**
```
┌─── CSI Driver (DaemonSet) ──────┐
│                                  │
│  Node Plugin (on each node)      │
│  - Mounts volumes to pods        │
│  - Volume staging/unstaging      │
│                                  │
└──────────────────────────────────┘

┌─── CSI Controller (Deployment) ──┐
│                                  │
│  Controller Plugin               │
│  - CreateVolume, DeleteVolume    │
│  - CreateSnapshot, ExpandVolume  │
│                                  │
└──────────────────────────────────┘
```

**Common CSI drivers:**
- AWS EBS CSI Driver
- GCE PD CSI Driver
- Azure Disk CSI Driver
- Longhorn (cloud-native distributed storage)
- OpenEBS
- Portworx

**CSI features:**
- Volume snapshots
- Volume cloning
- Volume expansion
- Topology awareness (zone constraints)

**VolumeSnapshot example:**
```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: db-snapshot
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: postgres-data

---
# Restore from snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-restored
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: fast-ssd
  dataSource:
    name: db-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  resources:
    requests:
      storage: 100Gi
```

---

## ConfigMaps & Secrets

### 🔥 Q: ConfigMap vs Secret?

| Feature | ConfigMap | Secret |
|---------|-----------|--------|
| Purpose | Non-sensitive configuration | Sensitive data (passwords, keys, tokens) |
| Encoding | Plain text | Base64 encoded (NOT encrypted by default!) |
| Size limit | 1MB | 1MB |
| etcd storage | Plain text | Can be encrypted at rest with `EncryptionConfiguration` |

```yaml
# ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_HOST: "postgres.default.svc.cluster.local"
  LOG_LEVEL: "info"
  config.yaml: |
    server:
      port: 8080
      workers: 4

---
# Secret
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  DATABASE_PASSWORD: cGFzc3dvcmQxMjM=    # base64 encoded
  API_KEY: c2VjcmV0a2V5                   # echo -n "secretkey" | base64

---
# Using in Pod
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: app
    image: myapp:latest
    # As environment variables
    envFrom:
    - configMapRef:
        name: app-config
    - secretRef:
        name: app-secrets
    # Individual keys
    env:
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: app-secrets
          key: DATABASE_PASSWORD
    # As files (volume mount)
    volumeMounts:
    - name: config
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: app-config
```

**Secret security best practices:**
- Enable **encryption at rest** in etcd
- Use **External Secrets Operator** with AWS Secrets Manager / HashiCorp Vault
- Use **Sealed Secrets** for GitOps (encrypted in Git, decrypted in cluster)
- Limit access with **RBAC** (don't let all service accounts read secrets)

### ⭐ Q: Explain External Secrets Operator and CSI Secrets Store.

**External Secrets Operator** — Syncs secrets from external secret stores into Kubernetes Secrets.

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secretsmanager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: app-sa

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-creds
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore
  target:
    name: db-secret          # K8s Secret to create
    creationPolicy: Owner
  data:
  - secretKey: password
    remoteRef:
      key: prod/postgres/password
```

**CSI Secrets Store Driver** — Mounts secrets directly as volumes (no K8s Secret object created).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  serviceAccountName: app-sa
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: secrets-store
      mountPath: "/mnt/secrets"
      readOnly: true
  volumes:
  - name: secrets-store
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "aws-secrets"

---
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: aws-secrets
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "prod/api/key"
        objectType: "secretsmanager"
```

**External Secrets vs CSI Secrets Store:**

| Feature | External Secrets Operator | CSI Secrets Store |
|---------|-------------------------|------------------|
| Creates K8s Secret? | Yes | No (direct mount) |
| RBAC on Secret? | Standard Secret RBAC | ServiceAccount bound to volume |
| Rotation | Polling interval | Mount refresh |
| Use case | App reads env vars or volume | App reads from file path |

---

## RBAC & Security

### 🔥 Q: Explain Kubernetes RBAC.

```
┌─── Subject ──────┐     ┌─── RoleBinding ───┐     ┌─── Role ──────────┐
│                   │     │                    │     │                    │
│ • User            │────►│ Binds subject to   │────►│ Rules:             │
│ • Group           │     │ a role in a        │     │ • resources: pods  │
│ • ServiceAccount  │     │ namespace          │     │ • verbs: get,list  │
│                   │     │                    │     │                    │
└───────────────────┘     └────────────────────┘     └────────────────────┘

Namespace-scoped: Role + RoleBinding
Cluster-scoped:   ClusterRole + ClusterRoleBinding
```

```yaml
# Role — namespace scoped
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]

---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: production
subjects:
- kind: User
  name: alice
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: monitoring
  namespace: monitoring
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

---
# ClusterRole — cluster-wide
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]

---
# ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: sre-team
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```

**RBAC verbs:** `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, `deletecollection`

### 🔥 Q: Kubernetes Security Best Practices?

1. **Pod Security Standards (PSS) / Pod Security Admission:**
   ```yaml
   # Namespace label to enforce security (replaces PodSecurityPolicy)
   apiVersion: v1
   kind: Namespace
   metadata:
     name: production
     labels:
       pod-security.kubernetes.io/enforce: restricted
       pod-security.kubernetes.io/warn: restricted
       pod-security.kubernetes.io/audit: restricted
   ```

   | Level | Description |
   |-------|-------------|
   | Privileged | No restrictions |
   | Baseline | Prevents known privilege escalations |
   | Restricted | Heavily restricted (non-root, read-only root FS, no capabilities) |

   **Note:** Pod Security Admission (PSA) replaced PodSecurityPolicy (PSP) in K8s 1.25. PSP is removed.

2. **Security Context:**
   ```yaml
   spec:
     securityContext:
       runAsNonRoot: true
       runAsUser: 1000
       fsGroup: 2000
       seccompProfile:
         type: RuntimeDefault
     containers:
     - name: app
       securityContext:
         allowPrivilegeEscalation: false
         readOnlyRootFilesystem: true
         capabilities:
           drop: ["ALL"]
   ```

3. **Admission Controllers** — Enforce policies before objects are persisted:
   - **ValidatingAdmissionWebhook** — Custom validation logic
   - **MutatingAdmissionWebhook** — Modify objects (inject sidecars, set defaults)
   - **Built-in:** PodSecurity, LimitRanger, ResourceQuota, NamespaceLifecycle

4. **OPA / Gatekeeper / Kyverno** — Policy-as-code enforcement:
   ```yaml
   # Kyverno policy example
   apiVersion: kyverno.io/v1
   kind: ClusterPolicy
   metadata:
     name: require-labels
   spec:
     validationFailureAction: enforce
     rules:
     - name: check-labels
       match:
         any:
         - resources:
             kinds:
             - Pod
       validate:
         message: "label 'team' is required"
         pattern:
           metadata:
             labels:
               team: "?*"
   ```

5. **Network Policies** — Default deny, whitelist required traffic
6. **Image scanning** in CI/CD (Trivy, Snyk, Grype)
7. **Limit API server access** — Private endpoint, authorized networks
8. **Audit logging** — Enable K8s audit logs
9. **Rotate credentials** — ServiceAccount tokens, etcd certs
10. **RBAC least privilege** — Never use `cluster-admin` for apps

### ⭐ Q: What replaced PodSecurityPolicy and why?

**PodSecurityPolicy (PSP)** was deprecated in 1.21 and **removed in 1.25**.

**Replaced by Pod Security Admission (PSA):**
- **PSP problems:** Complex, order-dependent, hard to reason about, cluster-wide scope
- **PSA benefits:** Namespace-scoped labels, three standard profiles (privileged/baseline/restricted), no controller needed (built-in admission controller)

**Migration path:**
```bash
# Analyze existing PSPs
kubectl get psp
kubectl describe psp restricted

# Map PSP policies to PSA levels
# - Unrestricted PSP → privileged
# - Restricted PSP → restricted
# - Middle ground → baseline

# Apply to namespaces
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

**For custom policies beyond PSA profiles, use OPA Gatekeeper or Kyverno.**

---

## Scheduling & Node Management

### 🔥 Q: Explain taints, tolerations, and affinity.

**Taints & Tolerations** — Prevent pods from scheduling on certain nodes:

```bash
# Taint a node
kubectl taint nodes node1 gpu=true:NoSchedule
# Only pods with matching toleration can be scheduled here
```

```yaml
# Pod toleration
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
```

| Taint Effect | Behavior |
|-------------|----------|
| `NoSchedule` | Don't schedule new pods (existing stay) |
| `PreferNoSchedule` | Soft version — try to avoid |
| `NoExecute` | Evict existing pods too |

**Node Affinity** — Attract pods to certain nodes:

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values: ["us-east-1a", "us-east-1b"]
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: instance-type
            operator: In
            values: ["m5.xlarge"]
```

**Pod Affinity / Anti-Affinity** — Co-locate or spread pods relative to other pods:

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values: ["postgres"]
        topologyKey: kubernetes.io/hostname
        # Ensures no two postgres pods on same node
```

### ⭐ Q: Explain Topology Spread Constraints.

**TopologySpreadConstraints** — Fine-grained control over pod distribution across topology domains (zones, nodes, regions).

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1                        # Max difference in pod count between zones
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule  # or ScheduleAnyway
    labelSelector:
      matchLabels:
        app: web
  - maxSkew: 2
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: web
```

**Use cases:**
- High availability — spread across zones to survive zone failure
- Even load — distribute pods evenly across nodes
- Better than podAntiAffinity for balanced spreading

### 💡 Q: Explain Pod Priority and Preemption.

**Priority Classes** — Assign relative importance to pods. Scheduler can evict lower-priority pods to make room for higher-priority ones.

```yaml
# PriorityClass
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000                # Higher value = higher priority
globalDefault: false
description: "High priority for production services"
preemptionPolicy: PreemptLowerPriority

---
# System priority classes (built-in):
# - system-cluster-critical (value: 2000000000)
# - system-node-critical (value: 2000001000)

---
# Use in Pod
spec:
  priorityClassName: high-priority
```

**How preemption works:**
1. Pod cannot be scheduled (insufficient resources)
2. Scheduler identifies lower-priority pods to evict
3. Lower-priority pods get graceful termination (default 30s)
4. Higher-priority pod scheduled after resources freed

**Best practices:**
- Use sparingly — only for truly critical workloads
- Combine with PodDisruptionBudgets to protect lower-priority but important pods
- Monitor preemption events: `kubectl get events --field-selector reason=Preempted`

### ⭐ Q: What happens during node drain?

```bash
kubectl drain node1 --ignore-daemonsets --delete-emptydir-data --grace-period=30
```

1. Node is **cordoned** (marked unschedulable)
2. Pods are **evicted** respecting:
   - PodDisruptionBudgets (PDBs)
   - Grace period (SIGTERM → wait → SIGKILL)
3. DaemonSet pods are ignored (they must stay)
4. Pods with emptyDir volumes lose that data
5. Evicted pods' controllers (Deployment, StatefulSet) create replacements on other nodes

---

## Resource Management

### 🔥 Q: Explain requests, limits, and QoS classes.

```yaml
containers:
- name: app
  resources:
    requests:              # Guaranteed minimum (used for scheduling)
      cpu: "250m"          # 250 millicores = 0.25 CPU
      memory: "256Mi"
    limits:                # Maximum allowed
      cpu: "1000m"         # 1 CPU
      memory: "512Mi"      # Container OOM-killed if exceeded
```

**What happens when limits are exceeded:**
- **CPU limit exceeded:** Container is **throttled** (not killed)
- **Memory limit exceeded:** Container is **OOM-killed**

**QoS Classes (determined automatically):**

| QoS Class | Condition | OOM Kill Priority |
|-----------|-----------|-------------------|
| **Guaranteed** | requests == limits for all containers | Last to be killed |
| **Burstable** | requests < limits (or only requests set) | Middle |
| **BestEffort** | No requests or limits set | First to be killed |

**LimitRange** — Default/min/max for a namespace:
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: production
spec:
  limits:
  - type: Container
    default:
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "2"
      memory: "2Gi"
```

**ResourceQuota** — Total limits for a namespace:
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "50"
    persistentvolumeclaims: "20"
```

---

## Health Checks & Self-Healing

### 🔥 Q: Explain liveness, readiness, and startup probes.

| Probe | Purpose | Failure Action |
|-------|---------|---------------|
| **Liveness** | Is the container healthy? | **Restarts** the container |
| **Readiness** | Can the container serve traffic? | **Removes** from Service endpoints |
| **Startup** | Has the container finished starting? | **Blocks** liveness/readiness until success |

```yaml
containers:
- name: app
  livenessProbe:
    httpGet:
      path: /healthz
      port: 8080
    initialDelaySeconds: 15
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 3

  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
    initialDelaySeconds: 5
    periodSeconds: 5
    failureThreshold: 3

  startupProbe:              # For slow-starting containers
    httpGet:
      path: /healthz
      port: 8080
    failureThreshold: 30     # 30 × 10s = 5 minutes to start
    periodSeconds: 10
```

**Probe types:**
- `httpGet` — HTTP GET request (200-399 = success)
- `tcpSocket` — TCP connection attempt
- `exec` — Run a command (exit 0 = success)
- `grpc` — gRPC health check (K8s 1.27+)

**Common mistake:** Making liveness probe check dependencies (DB, external APIs). If DB is down, all pods restart in a loop. **Liveness should only check the application itself.**

---

## Autoscaling

### 🔥 Q: Explain HPA, VPA, Cluster Autoscaler, and KEDA.

| Scaler | What it Scales | Based On |
|--------|---------------|----------|
| **HPA** (Horizontal Pod Autoscaler) | Number of pod replicas | CPU, memory, custom metrics |
| **VPA** (Vertical Pod Autoscaler) | Pod resource requests/limits | Historical resource usage |
| **Cluster Autoscaler** | Number of nodes | Pending pods (insufficient resources) |
| **KEDA** | Pod replicas (event-driven) | Queue length, HTTP requests, custom events |

```yaml
# HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 3
  maxReplicas: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300    # Wait 5 min before scaling down
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60               # Scale down max 10% per minute
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15               # Double pods every 15s if needed
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
```

### ⭐ Q: Explain VPA (Vertical Pod Autoscaler) in detail.

**VPA** adjusts CPU and memory requests/limits based on actual usage.

**VPA modes:**

| Mode | Behavior |
|------|----------|
| **Off** | Only provides recommendations (no changes) |
| **Initial** | Sets requests on pod creation only |
| **Recreate** | Updates requests by evicting and recreating pods |
| **Auto** | Recreate + Initial |

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  updatePolicy:
    updateMode: "Recreate"    # Off, Initial, Recreate, Auto
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 2
        memory: 2Gi
      controlledResources: ["cpu", "memory"]
```

**VPA caveats:**
- Evicting pods causes downtime — not suitable for single-replica apps
- Don't use VPA and HPA on CPU/memory together (conflicting signals)
- Use VPA for recommendations (`updateMode: Off`), apply manually

**Check VPA recommendations:**
```bash
kubectl describe vpa api-vpa
# Look at "Status.Recommendation.Container Recommendations"
```

### ⭐ Q: KEDA vs HPA?

**KEDA (Kubernetes Event-Driven Autoscaler)** — Scales pods based on event sources (queues, Kafka, HTTP, databases, cloud metrics).

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-scaler
spec:
  scaleTargetRef:
    name: worker
  minReplicaCount: 0        # Can scale to zero!
  maxReplicaCount: 50
  pollingInterval: 30
  cooldownPeriod: 300
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123/my-queue
      queueLength: "5"      # Target messages per replica
      awsRegion: us-east-1
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      metricName: http_requests_waiting
      threshold: "100"
      query: sum(rate(http_requests_total[1m]))
```

**HPA vs KEDA:**

| Feature | HPA | KEDA |
|---------|-----|------|
| Scale to zero | No (minReplicas: 1) | Yes |
| Event sources | CPU, memory, custom metrics | 60+ scalers (SQS, Kafka, Redis, HTTP, etc.) |
| Use case | Request-driven workloads | Event-driven, batch, async workers |

### ⭐ Q: Cluster Autoscaler vs Karpenter?

**Cluster Autoscaler** — Traditional node autoscaler. Watches for Pending pods, adds nodes from node groups.

**Karpenter** — AWS-native, faster, more flexible node autoscaler.

| Feature | Cluster Autoscaler | Karpenter |
|---------|-------------------|-----------|
| Speed | Slower (watches ASG) | Faster (direct EC2 API) |
| Node selection | Pre-defined node groups | Dynamic — picks best instance type per workload |
| Binpacking | Basic | Advanced — consolidation, efficient packing |
| Provider | Cloud-agnostic | AWS-native (Azure/GCP ports emerging) |
| Complexity | Simpler | More complex, requires IAM setup |

**Karpenter example:**
```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: general
spec:
  template:
    spec:
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot", "on-demand"]
      - key: kubernetes.io/arch
        operator: In
        values: ["amd64"]
      nodeClassRef:
        name: default
  limits:
    cpu: 1000
    memory: 1000Gi
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h
```

**When to use Karpenter:**
- AWS EKS
- Mixed workloads (varied CPU/memory needs)
- Spot instances heavily
- Want faster scale-up and better cost optimization

---

## Deployment Strategies

### 🔥 Q: Explain Kubernetes deployment strategies.

**1. Rolling Update (default):**
```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%          # Max extra pods during update
      maxUnavailable: 25%    # Max unavailable during update
```

**2. Recreate:**
```yaml
spec:
  strategy:
    type: Recreate           # Kill all old pods, then create new
    # Causes downtime! Use for DB migrations or incompatible versions
```

**3. Blue-Green (via Argo Rollouts or manual):**
```
                    ┌─── Blue (v1) ──── Running ───┐
Service ──►        │                                │
                    └─── Green (v2) ── Waiting ────┘

Switch Service selector from v1 to v2 after validation.
Instant rollback: switch back to v1.
Cost: 2x resources during deployment.
```

**4. Canary:**
```
                    ┌─── Stable (v1) ── 90% traffic ───┐
Service ──►        │                                     │
                    └─── Canary (v2) ── 10% traffic ────┘

Gradually shift traffic: 10% → 25% → 50% → 100%
Monitor metrics between each step.
Tools: Argo Rollouts, Flagger, Istio
```

```yaml
# Argo Rollouts Canary
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: my-app
spec:
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 5m}      # Wait and monitor
      - setWeight: 30
      - pause: {duration: 5m}
      - setWeight: 60
      - pause: {duration: 5m}
      - setWeight: 100
      canaryService: my-app-canary
      stableService: my-app-stable
      trafficRouting:
        nginx:
          stableIngress: my-app-ingress
```

---

## Helm

### 🔥 Q: What is Helm and how does it work?

**Helm** is the package manager for Kubernetes. It packages K8s manifests into **charts**.

```
mychart/
├── Chart.yaml          # Chart metadata (name, version, dependencies)
├── values.yaml         # Default configuration values
├── charts/             # Dependency charts
├── templates/          # K8s manifests with Go templates
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl    # Template helpers
│   └── NOTES.txt       # Post-install instructions
└── .helmignore
```

**Key commands:**
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm search repo nginx
helm install my-release bitnami/nginx -f custom-values.yaml -n production
helm upgrade my-release bitnami/nginx -f custom-values.yaml
helm rollback my-release 1              # Rollback to revision 1
helm list -n production
helm history my-release
helm uninstall my-release
helm template my-release bitnami/nginx  # Render templates locally (dry-run)
```

---

## Operators & CRDs

### ⭐ Q: What are Custom Resource Definitions (CRDs) and Operators?

**CRDs** extend the Kubernetes API with custom resources:
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.mycompany.com
spec:
  group: mycompany.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              engine:
                type: string
                enum: ["postgres", "mysql"]
              version:
                type: string
              replicas:
                type: integer
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames: ["db"]
```

**Operators** = CRDs + Custom Controllers. They encode operational knowledge into software.

**Popular Operators:**
- **Prometheus Operator** — Manages Prometheus, Alertmanager, ServiceMonitor CRDs
- **cert-manager** — Automates TLS certificate management
- **Strimzi** — Manages Kafka clusters
- **Zalando Postgres Operator** — Manages PostgreSQL clusters
- **ArgoCD** — GitOps continuous delivery

---

## Observability in Kubernetes

### 🔥 Q: How do you monitor Kubernetes?

```
┌──────────────── Observability Stack ─────────────────────┐
│                                                           │
│  Metrics:    Prometheus → Thanos/Mimir → Grafana          │
│  Logs:       Fluentd/Fluent Bit → Elasticsearch → Kibana  │
│              OR Loki → Grafana                             │
│  Traces:     OpenTelemetry → Jaeger/Tempo → Grafana       │
│  Events:     Kubernetes Events → Event Exporter           │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

**Key metrics to monitor:**
- **Cluster:** Node CPU/memory, pod count, API server latency
- **Pods:** CPU/memory usage vs requests/limits, restarts, OOM kills
- **Networking:** Request rate, error rate, latency (RED method)
- **Storage:** PVC usage, IOPS
- **etcd:** Leader elections, DB size, WAL fsync duration

---

## Troubleshooting

### 🔥 Q: Pod is stuck in various states. How to debug?

**CrashLoopBackOff:**
```bash
kubectl describe pod <pod>               # Check Events section
kubectl logs <pod> --previous             # Logs from crashed container
kubectl get events --sort-by='.lastTimestamp' -n <ns>

# Common causes:
# - Application error (check logs)
# - Missing config/secrets
# - Health check misconfiguration (liveness probe too aggressive)
# - OOM Kill (check describe for OOMKilled)
# - Wrong command/args
```

**ImagePullBackOff:**
```bash
kubectl describe pod <pod>
# Check:
# - Image name/tag correct?
# - Registry accessible from nodes?
# - ImagePullSecret configured?
# - Repository exists and image tag exists?

kubectl get secret regcred -o yaml         # Check pull secret
```

**Pending:**
```bash
kubectl describe pod <pod>
# Check Events for:
# "Insufficient cpu/memory" → Scale nodes or reduce requests
# "no nodes available" → Node issues
# "didn't match Pod's node affinity" → Fix affinity rules
# "PVC not bound" → StorageClass issues

kubectl get nodes -o wide
kubectl describe node <node>               # Check Allocatable vs Allocated
```

**Terminating (stuck):**
```bash
# Check for finalizers
kubectl get pod <pod> -o json | jq '.metadata.finalizers'

# Force delete (last resort)
kubectl delete pod <pod> --grace-period=0 --force
```

### 🔥 Q: Master troubleshooting commands cheat sheet.

```bash
# Cluster health
kubectl cluster-info
kubectl get nodes -o wide
kubectl get componentstatuses              # Deprecated but still useful
kubectl top nodes
kubectl top pods -n <namespace>

# Pod debugging
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> -c <container>
kubectl logs <pod> -n <ns> --previous     # Previous crashed instance
kubectl exec -it <pod> -n <ns> -- /bin/sh
kubectl port-forward <pod> 8080:80 -n <ns>

# Network debugging
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- /bin/bash
# Inside: curl, dig, nslookup, tcpdump, iperf, etc.

# Events
kubectl get events -n <ns> --sort-by='.lastTimestamp'
kubectl get events --field-selector type=Warning

# Resource usage
kubectl top pods -n <ns> --sort-by=memory
kubectl describe node <node> | grep -A 5 "Allocated"
```

---

## Network Policies

### ⭐ Q: Explain Network Policies.

```yaml
# Default deny all ingress in namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}           # Select all pods
  policyTypes:
  - Ingress                 # Deny all ingress

---
# Allow specific traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-from-web
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web
    - namespaceSelector:
        matchLabels:
          name: monitoring
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - protocol: TCP
      port: 5432
  - to:                      # Allow DNS
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
```

**Important:** Network Policies require a CNI that supports them (Calico, Cilium, Weave). **Flannel does NOT support Network Policies.**

---

## Service Mesh

### ⭐ Q: What is a service mesh and why use it?

**Service mesh** — Infrastructure layer that handles service-to-service communication (mTLS, observability, traffic management).

**Core features:**
- **mTLS** — Automatic mutual TLS between services
- **Traffic management** — Retries, timeouts, circuit breaking, traffic splitting
- **Observability** — Request-level metrics, traces, logs
- **Security** — Authorization policies at L7

**Popular service meshes:**

| Mesh | Architecture | Pros | Cons |
|------|-------------|------|------|
| **Istio** | Envoy sidecar | Feature-rich, mature | Complex, resource-heavy |
| **Linkerd** | Linkerd2-proxy sidecar | Lightweight, simple | Fewer features than Istio |
| **Consul** | Envoy sidecar | Multi-platform (K8s, VMs) | HashiCorp ecosystem lock-in |
| **Cilium Service Mesh** | eBPF (sidecarless) | Low overhead, fast | Less mature, fewer L7 features |

### 💡 Q: When would you NOT use a service mesh?

**Skip service mesh when:**
- Simple architecture (few services, no cross-cluster communication)
- Cost-sensitive (mesh adds 20-40% resource overhead for sidecar-based)
- Team lacks expertise (mesh adds operational complexity)
- Already have ingress/egress gateway handling mTLS

**Alternatives:**
- **Ingress controller + Network Policies** — For basic security
- **App-level mTLS** — Libraries like gRPC with TLS
- **Cloud-native solutions** — AWS App Mesh, GCP Traffic Director

**When mesh DOES make sense:**
- Many microservices (10+)
- Multi-cluster or multi-cloud
- Need uniform observability across services
- Compliance requires mTLS everywhere

---

## etcd

### 🔥 Q: What is etcd and why is it critical?

**etcd** is a distributed, consistent key-value store that stores ALL Kubernetes cluster state.

**Key operations:**
```bash
# Backup etcd (CRITICAL for disaster recovery)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-20240101.db -w table

# Restore (disaster recovery)
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-20240101.db \
  --data-dir=/var/lib/etcd-restored

# Health check
ETCDCTL_API=3 etcdctl endpoint health
ETCDCTL_API=3 etcdctl endpoint status -w table
```

**etcd best practices:**
- Run on **dedicated SSDs** (etcd is I/O sensitive)
- Always **odd number** of members (3 or 5) for quorum
- Regular automated **backups** (every hour minimum)
- Monitor: leader changes, DB size, WAL sync duration
- **Defragment** periodically: `etcdctl defrag`

---

## Multi-Cluster & Federation

### ⭐ Q: What are the patterns for multi-cluster Kubernetes?

**Common multi-cluster patterns:**

1. **High Availability** — Multiple clusters in different regions/zones
   - Use case: Survive region failures
   - Pattern: Active-active or active-passive with global load balancer

2. **Geographic distribution** — Clusters in different regions for latency
   - Use case: Serve users globally with low latency
   - Pattern: Regional clusters with local data

3. **Isolation** — Separate clusters for security/compliance
   - Use case: Multi-tenant with hard isolation, prod vs non-prod
   - Pattern: Cluster per tenant or environment

4. **Workload separation** — Different cluster types per workload
   - Use case: GPU workloads, batch processing, sensitive data
   - Pattern: Specialized clusters (GPU cluster, batch cluster)

### 💡 Q: Tools for multi-cluster management?

| Tool | Purpose |
|------|---------|
| **Cluster API** | Declarative cluster provisioning and lifecycle |
| **Kubefed (deprecated)** | Old federation approach |
| **ArgoCD / Flux** | GitOps across multiple clusters |
| **Rancher** | Multi-cluster UI and management |
| **Google Anthos / Azure Arc** | Cloud-vendor multi-cluster platforms |
| **Istio Multi-Cluster** | Service mesh across clusters |
| **Submariner** | Cross-cluster networking (pod-to-pod) |
| **Cilium Cluster Mesh** | Multi-cluster networking with eBPF |

### 💡 Q: How does cross-cluster service discovery work?

**Options:**

1. **DNS-based** — Global DNS points to regional clusters
   ```
   app.us.example.com → US cluster
   app.eu.example.com → EU cluster
   app.example.com → Global LB → nearest cluster
   ```

2. **Service Mesh** — Istio multi-cluster, service registry spans clusters
   ```yaml
   # ServiceEntry for remote cluster service
   apiVersion: networking.istio.io/v1beta1
   kind: ServiceEntry
   metadata:
     name: api-eu
   spec:
     hosts:
     - api.eu.svc.cluster.local
     ports:
     - number: 80
       name: http
       protocol: HTTP
     location: MESH_EXTERNAL
     resolution: DNS
     endpoints:
     - address: api.eu-cluster.example.com
   ```

3. **Submariner** — Direct pod-to-pod networking across clusters
   - Creates secure tunnels between clusters
   - Services discoverable across clusters: `<svc>.<ns>.svc.clusterset.local`

4. **External Services** — Traditional approach
   - Services in one cluster exposed via LoadBalancer
   - Other clusters access via external IP/DNS

### ⭐ Q: Multi-cluster disaster recovery strategy?

```
Region A (Primary)          Region B (DR)
┌─────────────────┐         ┌─────────────────┐
│ Prod Cluster    │────────▶│ DR Cluster      │
│ - Active traffic│  Sync   │ - Standby       │
│ - etcd backup   │         │ - Warm/Cold     │
└─────────────────┘         └─────────────────┘
        │                           │
        ▼                           ▼
   S3 / GCS Backups        S3 / GCS Backups
   (Cross-region replicated)
```

**DR strategy:**

1. **Backup regularly:**
   - etcd snapshots (hourly)
   - GitOps repo (all manifests)
   - Persistent data (volumes, databases)

2. **Replication:**
   - Cross-region etcd snapshots
   - Database replication (Aurora Global, GCP Spanner)
   - Object storage replication (S3 cross-region)

3. **Standby cluster:**
   - **Cold:** Spin up on failure (slow but cheap)
   - **Warm:** Cluster running, no traffic (medium cost/speed)
   - **Hot:** Active-active with traffic splitting (expensive but instant)

4. **Failover process:**
   ```bash
   # 1. Restore etcd in DR cluster
   etcdctl snapshot restore /backup/etcd-snapshot.db
   
   # 2. Apply manifests via GitOps
   kubectl apply -f manifests/
   
   # 3. Restore persistent data
   # From volume snapshots or database backups
   
   # 4. Switch DNS/Load balancer to DR cluster
   aws route53 change-resource-record-sets --hosted-zone-id Z123 --change-batch file://failover.json
   
   # 5. Verify applications
   kubectl get pods -A
   kubectl get svc -A
   ```

5. **Regular DR drills** — Test failover quarterly

---

## Scenario-Based Questions

### 🔥 Q: Your Kubernetes cluster is running out of resources. What do you do?

```
1. Assess current usage:
   kubectl top nodes
   kubectl top pods -A --sort-by=memory

2. Identify resource hogs:
   kubectl describe nodes | grep -A 5 "Allocated"
   # Look for pods without limits or with high usage

3. Quick fixes:
   - Set resource requests/limits on all pods
   - Add ResourceQuotas per namespace
   - Add LimitRanges for defaults
   - Kill unused/orphaned resources

4. Right-sizing:
   - Install VPA in recommend mode → get right-sizing suggestions
   - Use Goldilocks for VPA dashboard
   - Review and adjust requests/limits

5. Scale:
   - Enable Cluster Autoscaler
   - Add node pools with appropriate instance types
   - Use Spot/Preemptible instances for non-critical workloads

6. Long-term:
   - Implement Kubecost or OpenCost for cost visibility
   - Regular right-sizing reviews
   - Pod Priority and Preemption for critical workloads
```

### 🔥 Q: A deployment rollout is stuck. How do you fix it?

```bash
# 1. Check rollout status
kubectl rollout status deployment/myapp

# 2. Check rollout history
kubectl rollout history deployment/myapp

# 3. Describe deployment
kubectl describe deployment myapp
# Look for: "Progressing" condition = False

# 4. Check new ReplicaSet
kubectl get rs -l app=myapp
# Old RS still has pods? New RS pods failing?

# 5. Check pod events
kubectl describe pod <new-pod>
# ImagePullBackOff? CrashLoopBackOff? Resource issues?

# 6. Fix or rollback
kubectl rollout undo deployment/myapp                    # Rollback to previous
kubectl rollout undo deployment/myapp --to-revision=3    # Specific revision

# 7. Increase progress deadline if slow to start
spec:
  progressDeadlineSeconds: 600
```

### ⭐ Q: How do you handle certificate rotation in Kubernetes?

```bash
# Check certificate expiry
kubeadm certs check-expiration

# Renew all certificates
kubeadm certs renew all

# Restart control plane after renewal
# (kubeadm clusters use static pods, just move and replace manifests)

# For automated rotation:
# - Use cert-manager for application TLS
# - kubelet client certificate rotation is automatic (--rotate-certificates)
# - Service Account token rotation: use bound service account tokens (default in 1.24+)
```

---

## Deep Troubleshooting & Debug Scenarios

This section covers systematic debugging for common failure modes in production Kubernetes clusters.

### 🔥 Q: Pod stuck in CrashLoopBackOff — systematic debug approach.

```bash
# 1. Check pod status and recent events
kubectl get pod <pod> -n <ns>
kubectl describe pod <pod> -n <ns>
# Look at Events section for OOMKilled, error messages

# 2. Check logs from current and previous container
kubectl logs <pod> -n <ns> -c <container>
kubectl logs <pod> -n <ns> -c <container> --previous   # Crucial for crashes

# 3. Check if it's an OOM kill
kubectl describe pod <pod> -n <ns> | grep -A 5 "Last State"
# State: Terminated, Reason: OOMKilled → increase memory limit

# 4. Check liveness probe configuration
kubectl get pod <pod> -n <ns> -o yaml | grep -A 10 livenessProbe
# Is probe too aggressive? initialDelaySeconds too low?

# 5. Common causes:
# - Application crash on startup (missing config, bad code)
# - Liveness probe failing (app slow to start, probe misconfigured)
# - OOM kill (memory limit too low)
# - Missing dependencies (ConfigMap, Secret, volume, external service)
# - Wrong command/args in container spec

# 6. Quick fixes:
# - Increase initialDelaySeconds on liveness probe
# - Increase memory limit
# - Check and fix missing ConfigMap/Secret references
# - Exec into init container to debug dependencies
```

### 🔥 Q: ImagePullBackOff — what checks do you run?

```bash
# 1. Describe pod to see exact error
kubectl describe pod <pod> -n <ns>
# Look for: "Failed to pull image", "manifest unknown", "unauthorized"

# 2. Check image name and tag
kubectl get pod <pod> -n <ns> -o jsonpath='{.spec.containers[*].image}'
# Typo in image name? Tag exists?

# 3. Check imagePullSecrets
kubectl get pod <pod> -n <ns> -o jsonpath='{.spec.imagePullSecrets}'
kubectl get secret <pull-secret> -n <ns> -o yaml
# Secret exists? Correct namespace? Valid credentials?

# 4. Test image pull from node
ssh <node>
crictl pull <image>
# Or: docker pull <image> (if docker runtime)

# 5. Check network connectivity from node
# Can node reach registry? (gcr.io, docker.io, ECR, private registry)

# 6. Common causes:
# - Image doesn't exist or tag is wrong
# - Private registry, missing imagePullSecret
# - ImagePullSecret in wrong namespace
# - Registry authentication expired
# - Network policy blocking registry access
# - Node IAM role missing ECR permissions (EKS)

# 7. Quick fixes:
kubectl create secret docker-registry regcred \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<pass> \
  -n <ns>
```

### 🔥 Q: Pod stuck Pending — how do you diagnose?

```bash
# 1. Describe pod — Events section is key
kubectl describe pod <pod> -n <ns>

# 2. Common Pending reasons:
# - "Insufficient cpu" or "Insufficient memory"
# - "no nodes available to schedule pods"
# - "didn't match Pod's node affinity/selector"
# - "PersistentVolumeClaim is not bound"
# - "Taints on nodes, pod has no tolerations"

# 3. Check node resources
kubectl get nodes -o wide
kubectl top nodes
kubectl describe nodes | grep -A 5 "Allocated resources"
# Are nodes at capacity?

# 4. Check pod resource requests
kubectl get pod <pod> -n <ns> -o yaml | grep -A 5 requests
# Are requests too high?

# 5. Check PVC status (if using volumes)
kubectl get pvc -n <ns>
kubectl describe pvc <pvc> -n <ns>
# Stuck in Pending? StorageClass exists? CSI driver healthy?

# 6. Check node affinity/taints
kubectl get nodes --show-labels
kubectl describe node <node> | grep Taints
kubectl get pod <pod> -n <ns> -o yaml | grep -A 10 affinity

# 7. Check scheduler logs
kubectl logs -n kube-system <scheduler-pod>
# Any errors about this pod?

# 8. Fixes:
# - Insufficient resources → add nodes (Cluster Autoscaler/Karpenter) or reduce requests
# - Node affinity mismatch → fix affinity rules or node labels
# - PVC not bound → check StorageClass, CSI driver, provisioner logs
# - Taints → add tolerations or remove taints
```

### 🔥 Q: Pod OOMKilled — how do you investigate?

```bash
# 1. Confirm OOM kill
kubectl describe pod <pod> -n <ns> | grep -i oom
# Last State: Terminated, Reason: OOMKilled, Exit Code: 137

# 2. Check memory limit
kubectl get pod <pod> -n <ns> -o yaml | grep -A 3 "limits:"
# What was the memory limit?

# 3. Check actual memory usage before kill
kubectl top pod <pod> -n <ns>
# If metrics-server is running

# 4. Check application logs for memory leaks
kubectl logs <pod> -n <ns> --previous
# Look for: allocating large objects, unclosed connections, cache growth

# 5. Check if sidecar containers consuming memory
kubectl top pod <pod> -n <ns> --containers
# Which container hit the limit?

# 6. Long-term investigation — metrics
# Query Prometheus for memory usage trend:
container_memory_working_set_bytes{pod="<pod>"}
# Was memory steadily growing (leak) or sudden spike?

# 7. Fixes:
# - Increase memory limit
# - Add memory request equal to limit (QoS Guaranteed)
# - Fix memory leak in application
# - Optimize application memory usage (cache size, object pooling)
# - Consider VPA to right-size automatically
```

### 🔥 Q: Node in NotReady state — systematic debug.

```bash
# 1. Identify NotReady nodes
kubectl get nodes
# STATUS: NotReady

# 2. Describe node
kubectl describe node <node>
# Check Conditions section: DiskPressure, MemoryPressure, PIDPressure, NetworkUnavailable

# 3. Check kubelet status on the node
ssh <node>
sudo systemctl status kubelet
sudo journalctl -u kubelet -f
# Look for errors: certificate issues, API server unreachable, CNI failures

# 4. Check node resources
df -h              # Disk full? (logs, images)
free -h            # Out of memory?
top                # High CPU? Process runaway?

# 5. Check container runtime
sudo systemctl status containerd   # or docker/cri-o
sudo crictl ps     # Containers running?

# 6. Check CNI plugin
ls /etc/cni/net.d/
kubectl logs -n kube-system <cni-pod>   # Calico, Cilium, etc.

# 7. Common causes:
# - Disk pressure (logs, container images fill disk)
# - Kubelet not running or crashing
# - API server unreachable (network, certs)
# - CNI plugin failure
# - Node resource exhaustion
# - Clock skew / NTP issues

# 8. Fixes:
# - Disk full → clean up: docker/crictl image prune, clear /var/log
# - Restart kubelet: sudo systemctl restart kubelet
# - Check certificates: ls -l /var/lib/kubelet/pki/
# - Fix CNI: restart CNI pods, check node network config
# - Drain and replace node if hardware failure
```

### 🔥 Q: DNS resolution failing inside pods.

```bash
# 1. Test DNS from inside a pod
kubectl run -it --rm debug --image=nicolaka/netshoot -- /bin/bash
nslookup kubernetes.default.svc.cluster.local
nslookup google.com
# Does cluster DNS work? Does external DNS work?

# 2. Check CoreDNS pods
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
# Are they Running? Any errors?

# 3. Check CoreDNS service
kubectl get svc -n kube-system kube-dns
# ClusterIP should match /etc/resolv.conf in pods

# 4. Check pod's resolv.conf
kubectl exec <pod> -n <ns> -- cat /etc/resolv.conf
# nameserver should be kube-dns service ClusterIP
# search domains should include: <ns>.svc.cluster.local svc.cluster.local cluster.local

# 5. Check CoreDNS ConfigMap
kubectl get cm -n kube-system coredns -o yaml
# Any misconfigurations? Forward rules correct?

# 6. Test DNS query with dig
kubectl exec <pod> -n <ns> -- dig kubernetes.default.svc.cluster.local
# NOERROR? NXDOMAIN? SERVFAIL?

# 7. Common causes:
# - CoreDNS pods not running or crashing
# - CoreDNS service ClusterIP changed
# - Network policy blocking DNS (port 53 UDP)
# - CoreDNS out of resources (CPU/memory)
# - Upstream DNS unreachable (for external queries)

# 8. Fixes:
# - Scale CoreDNS: kubectl scale deployment coredns -n kube-system --replicas=3
# - Restart CoreDNS: kubectl rollout restart deployment coredns -n kube-system
# - Check network policies (allow DNS egress)
# - Increase CoreDNS resources
```

### 🔥 Q: Networking between pods not working.

```bash
# 1. Get pod IPs
kubectl get pod <pod1> -n <ns> -o wide
kubectl get pod <pod2> -n <ns> -o wide

# 2. Test connectivity
kubectl exec <pod1> -n <ns> -- ping <pod2-ip>
kubectl exec <pod1> -n <ns> -- curl <pod2-ip>:<port>
# Can ping? Can reach port?

# 3. Test service connectivity
kubectl get svc <service> -n <ns>
kubectl exec <pod1> -n <ns> -- curl <service>.<ns>.svc.cluster.local
# Service reachable?

# 4. Check endpoints
kubectl get endpoints <service> -n <ns>
# Are pod IPs listed? If empty, label selector mismatch

# 5. Check network policies
kubectl get networkpolicy -n <ns>
kubectl describe networkpolicy <policy> -n <ns>
# Is there a default-deny? Does it allow this traffic?

# 6. Check CNI plugin
kubectl get pods -n kube-system | grep -E 'calico|cilium|flannel|weave'
kubectl logs -n kube-system <cni-pod>
# CNI healthy?

# 7. Check kube-proxy
kubectl get pods -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system <kube-proxy-pod>
# iptables/IPVS rules being created?

# 8. Check iptables rules on node (if using iptables mode)
ssh <node>
sudo iptables-save | grep <service-ip>

# 9. Common causes:
# - Network policy blocking traffic
# - Service selector not matching pod labels
# - CNI plugin not running or misconfigured
# - kube-proxy not running
# - SecurityGroup / firewall rules (cloud)

# 10. Fixes:
# - Fix network policies (add allow rules)
# - Fix service label selector
# - Restart CNI pods
# - Restart kube-proxy
```

### 🔥 Q: PVC stuck in Pending — how to debug?

```bash
# 1. Check PVC status
kubectl get pvc <pvc> -n <ns>
kubectl describe pvc <pvc> -n <ns>
# Events section will show the error

# 2. Common reasons:
# - "waiting for first consumer to be created"
# - "no persistent volumes available"
# - "failed to provision volume"
# - StorageClass not found

# 3. Check StorageClass
kubectl get storageclass
kubectl describe storageclass <sc>
# Does it exist? Correct provisioner?

# 4. Check volumeBindingMode
kubectl get sc <sc> -o yaml | grep volumeBindingMode
# WaitForFirstConsumer → PVC waits until pod scheduled
# Immediate → PVC binds immediately

# 5. If WaitForFirstConsumer, check if pod created
kubectl get pods -n <ns>
# PVC won't bind until a pod using it is created

# 6. Check CSI driver pods
kubectl get pods -n kube-system | grep csi
kubectl logs -n kube-system <csi-controller-pod>
# CSI driver healthy? Any provision errors?

# 7. Check cloud provider provisioner
# AWS EBS: Check IAM permissions on node IAM role
# GCP PD: Check GCP service account permissions
# Azure Disk: Check Azure managed identity

# 8. Check for available PVs (if static provisioning)
kubectl get pv
# Any Available PVs matching the PVC request?

# 9. Fixes:
# - WaitForFirstConsumer → create the pod
# - Missing StorageClass → create it
# - CSI driver unhealthy → restart driver pods
# - IAM/permissions → fix cloud IAM
# - Static PV → create matching PV
```

### ⭐ Q: Certificate expired — cluster components failing.

```bash
# 1. Check certificate expiry
kubeadm certs check-expiration
# Shows expiry dates for all control plane certs

# 2. If kubectl not working
# SSH to control plane node
sudo kubeadm certs check-expiration

# 3. Check which component failing
kubectl get pods -n kube-system
# apiserver, etcd, controller-manager, scheduler failing?

# 4. Check kubelet cert
sudo ls -l /var/lib/kubelet/pki/
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates
# Kubelet cert expired?

# 5. Renew all certificates
sudo kubeadm certs renew all

# 6. Restart control plane components (static pods)
# Move manifest out and back to trigger restart
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
# Wait 10 seconds
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
# Repeat for etcd, controller-manager, scheduler

# 7. Restart kubelet
sudo systemctl restart kubelet

# 8. Update kubeconfig with new certs
sudo kubeadm init phase kubeconfig admin
cp /etc/kubernetes/admin.conf ~/.kube/config

# 9. Prevent future expiry
# - Set up automated cert renewal (kubeadm renews on upgrade)
# - Use cert-manager for application certs
# - Monitor cert expiry with Prometheus alerts
```

### ⭐ Q: etcd cluster unhealthy — how to diagnose and recover?

```bash
# 1. Check etcd pods
kubectl get pods -n kube-system -l component=etcd
kubectl describe pod <etcd-pod> -n kube-system

# 2. Check etcd member list
kubectl -n kube-system exec -it etcd-<node> -- sh
ETCDCTL_API=3 etcdctl member list \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 3. Check etcd endpoint health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 4. Check etcd logs
kubectl logs -n kube-system etcd-<node>
# Look for: leader election failures, disk I/O errors, member unreachable

# 5. Common issues:
# - Disk latency (etcd needs fast disk)
# - Leader election failures (network partition)
# - Quorum lost (majority of members down)
# - Disk space full
# - DB size too large (needs defragmentation)

# 6. Check etcd metrics
# wal_fsync_duration_seconds (should be <10ms)
# db_total_size_in_bytes (monitor growth)

# 7. Recovery steps:
# If quorum intact but one member unhealthy:
etcdctl member remove <member-id>
# Add member back after fixing

# If quorum lost (disaster):
# Restore from backup
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored

# 8. Prevention:
# - Regular automated backups (hourly minimum)
# - Monitor etcd metrics (fsync latency, DB size)
# - Use fast SSDs (gp3 with high IOPS)
# - Defragment regularly
```

---

## Trending & Platform Engineering Questions (2025-26)

These are questions emerging in senior DevOps/SRE/Platform Engineer loops at top companies.

### ⭐ Q: How would you design a multi-tenant Kubernetes platform?

**Key considerations:**

1. **Namespace-per-tenant** (soft isolation) or **Cluster-per-tenant** (hard isolation)?
   - Soft: Lower cost, shared control plane. Use for internal teams with trust.
   - Hard: Strong isolation, higher cost. Use for external customers or regulated data.

2. **Tenant isolation layers:**
   ```
   Resource Quotas (compute limits per tenant)
        ↓
   LimitRanges (default/max per pod)
        ↓
   Network Policies (default deny, whitelist)
        ↓
   RBAC (tenant admins can't see other tenants)
        ↓
   Pod Security Admission (enforce security standards)
        ↓
   OPA/Kyverno policies (custom tenant rules)
   ```

3. **Resource management:**
   - ResourceQuotas per namespace
   - Fair scheduling (priority classes, preemption)
   - Cost allocation (Kubecost, OpenCost by namespace)

4. **Networking:**
   - Default-deny NetworkPolicies
   - Ingress isolation (separate Ingress per tenant or path-based routing)
   - Service mesh for mTLS between tenants (optional)

5. **Security:**
   - Separate ServiceAccounts per tenant
   - RBAC: tenant-admin Role in their namespace only
   - Pod Security Standards: enforce restricted
   - Audit logs per tenant

6. **Observability:**
   - Tenant-scoped Grafana dashboards
   - Log aggregation with tenant labels
   - Trace context propagation

7. **Self-service:**
   - GitOps (ArgoCD/Flux) per tenant namespace
   - Crossplane or Terraform for tenant resource provisioning
   - Developer portal (Backstage) for self-service

**Interview answer:**
"I'd use namespace-per-tenant with ResourceQuotas, default-deny NetworkPolicies, tenant-scoped RBAC, and Pod Security Admission at restricted level. For cost visibility, I'd deploy Kubecost with namespace-level breakdowns. Self-service would be GitOps via ArgoCD with tenant admins managing their own namespaces. For hard isolation requirements, I'd use cluster-per-tenant with a management cluster running Cluster API."

### ⭐ Q: Explain eBPF and why it matters for Kubernetes networking.

**eBPF (Extended Berkeley Packet Filter)** — Run sandboxed programs in the Linux kernel without changing kernel code.

**Why eBPF for Kubernetes:**

| Traditional (iptables/IPVS) | eBPF (Cilium, Calico eBPF) |
|---------------------------|---------------------------|
| Userspace kube-proxy writes kernel rules | BPF programs run directly in kernel |
| O(n) rule traversal, degrades at scale | O(1) lookups, constant performance |
| L3/L4 only | L3/L4 + L7 (HTTP, gRPC) aware |
| No visibility into dropped packets | Full observability (Hubble) |

**Use cases:**
- **Networking:** kube-proxy replacement, fast packet filtering
- **Observability:** Network flow logs, latency tracking, dropped packets
- **Security:** Runtime security (Falco), network policies at L7
- **Service mesh:** Sidecarless service mesh (no Envoy proxy per pod)

**Cilium example features:**
```yaml
# L7 network policy (HTTP-aware)
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-l7-policy
spec:
  endpointSelector:
    matchLabels:
      app: api
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: web
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: GET
          path: "/api/v1/.*"
```

**Interview answer:**
"eBPF allows running programs directly in the kernel for networking, observability, and security without kernel changes. For Kubernetes, this means kube-proxy replacement with O(1) performance, L7-aware network policies, and deep observability like per-request latency and dropped packet analysis. Tools like Cilium use eBPF for both CNI and sidecarless service mesh, removing the Envoy proxy overhead."

### 💡 Q: What is a sidecarless service mesh and how does it compare to traditional (Istio)?

**Traditional service mesh (Istio, Linkerd 2.x):**
```
Pod
├── App container
└── Envoy sidecar (proxy)
    - mTLS
    - Traffic routing
    - Telemetry
    - Policy enforcement
```

**Sidecarless service mesh (Cilium Service Mesh, Istio Ambient Mode):**
```
Node
├── Shared proxy (per-node or eBPF)
└── Pods (no sidecar)
```

**Comparison:**

| Feature | Sidecar (Traditional) | Sidecarless (Ambient/Cilium) |
|---------|----------------------|----------------------------|
| **Resource overhead** | High (proxy per pod) | Low (shared proxy or eBPF) |
| **Latency** | Additional hop through sidecar | Lower (kernel-level or shared proxy) |
| **Operational complexity** | High (sidecar injection, upgrades) | Lower (no per-pod injection) |
| **Feature richness** | Full L7 features | L4 mTLS + limited L7 |
| **Maturity** | Mature | Emerging |

**Istio Ambient Mode (2024+):**
- L4 features (mTLS, telemetry) via ztunnel (per-node DaemonSet)
- Optional L7 features via waypoint proxy (per-namespace)
- No sidecar injection required

**When to use:**
- Sidecar: Need full L7 features (retries, circuit breaking, advanced routing)
- Sidecarless: Want lower overhead, simpler operations, L4 mTLS sufficient

**Interview answer:**
"Traditional service mesh injects an Envoy sidecar into every pod for mTLS, routing, and telemetry. This adds resource overhead and operational complexity. Sidecarless approaches like Cilium Service Mesh use eBPF in the kernel, and Istio Ambient Mode uses a per-node ztunnel for L4 features, only deploying per-namespace waypoint proxies when L7 features are needed. This reduces resource usage by 50-70% and simplifies operations."

### ⭐ Q: How do you approach Kubernetes cost optimization (FinOps)?

**Cost optimization layers:**

1. **Visibility:**
   - Deploy **Kubecost** or **OpenCost** for per-namespace/workload cost breakdown
   - Tag resources (labels) by team, app, environment
   - Track idle resources (pods with low CPU/memory usage)

2. **Right-sizing:**
   - VPA in recommend mode → identify over-provisioned pods
   - Goldilocks (VPA dashboard) for right-sizing recommendations
   - Regularly review and reduce requests/limits

3. **Autoscaling:**
   - HPA for workloads (avoid over-provisioning static replicas)
   - Cluster Autoscaler or Karpenter for node scaling
   - KEDA for event-driven scale-to-zero

4. **Spot/Preemptible instances:**
   - Use Spot for fault-tolerant workloads (batch, dev/test)
   - Node affinity for cost-sensitive workloads
   - Karpenter: `karpenter.sh/capacity-type: spot`

5. **Resource bin-packing:**
   - Use multiple instance types (Karpenter selects best)
   - Topology spread constraints for even distribution
   - Avoid resource fragmentation

6. **Storage optimization:**
   - Delete unused PVCs (orphaned after app deletion)
   - Use gp3 instead of io2 where appropriate
   - Lifecycle policies (delete old snapshots)

7. **Idle resource cleanup:**
   - Shut down non-prod environments off-hours
   - Scale dev/test to zero nights/weekends
   - Delete abandoned namespaces

**Interview answer:**
"I'd start with visibility — deploy Kubecost to identify cost by team and workload. Then right-size using VPA recommendations, eliminating over-provisioned requests. Next, implement autoscaling (HPA for pods, Karpenter for nodes with Spot) to match actual load. For workloads that can tolerate interruptions, use Spot instances. Finally, automate cleanup of idle resources and scale down non-prod environments off-hours. I'd set up monthly cost reviews with team leads and dashboard alerts for cost anomalies."

### 💡 Q: What is Karpenter's disruption/consolidation feature?

**Consolidation** — Karpenter actively looks for underutilized nodes and consolidates pods onto fewer nodes to reduce cost.

```yaml
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  disruption:
    consolidationPolicy: WhenUnderutilized   # or WhenEmpty
    consolidateAfter: 30s                    # How long to wait before consolidating
    expireAfter: 720h                        # Replace nodes after 30 days
  template:
    spec:
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot", "on-demand"]
```

**How it works:**
1. Karpenter detects a node is underutilized (<50% CPU/memory)
2. Checks if pods can be rescheduled to other nodes
3. Cordons and drains the node (respecting PDBs)
4. Terminates the node
5. May launch a smaller/cheaper instance if needed

**vs Cluster Autoscaler:**
- Cluster Autoscaler only scales down empty nodes after 10 minutes
- Karpenter actively consolidates underutilized nodes in 30 seconds

**Benefits:**
- 20-40% cost savings from better bin-packing
- Automatic replacement of older instance types
- Faster response to workload changes

### ⭐ Q: How would you implement progressive delivery in Kubernetes?

**Progressive delivery** — Gradually roll out changes with automated rollback on failure.

**Tools:**
- **Argo Rollouts** — Kubernetes-native progressive delivery
- **Flagger** — Works with Istio, Linkerd, Contour
- **Flux** — GitOps with Flagger integration

**Canary deployment with Argo Rollouts:**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api
spec:
  replicas: 10
  strategy:
    canary:
      steps:
      - setWeight: 10              # 10% traffic to canary
      - pause: {duration: 2m}
      - analysis:
          templates:
          - templateName: error-rate
        args:
        - name: service-name
          value: api
      - setWeight: 30
      - pause: {duration: 2m}
      - analysis:
          templates:
          - templateName: error-rate
      - setWeight: 60
      - pause: {duration: 2m}
      - setWeight: 100
      canaryService: api-canary      # Service for canary pods
      stableService: api-stable      # Service for stable pods
      trafficRouting:
        istio:
          virtualService:
            name: api
            routes:
            - primary

---
# Analysis Template (Prometheus query)
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: error-rate
spec:
  args:
  - name: service-name
  metrics:
  - name: error-rate
    interval: 1m
    successCondition: result < 0.05      # <5% error rate
    failureLimit: 3
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{service="{{args.service-name}}",status=~"5.."}[1m]))
          /
          sum(rate(http_requests_total{service="{{args.service-name}}"}[1m]))
```

**Blue-Green with instant rollback:**
```yaml
strategy:
  blueGreen:
    activeService: api-active
    previewService: api-preview
    autoPromotionEnabled: false     # Manual approval
    scaleDownDelaySeconds: 300      # Keep old version 5 min for rollback
```

**Interview answer:**
"I'd use Argo Rollouts for canary deployments with automated analysis. The rollout starts at 10% traffic to the new version, pauses to run Prometheus queries checking error rate and latency, then progressively increases to 30%, 60%, 100% if metrics are healthy. If error rate exceeds threshold at any step, it automatically rolls back. For critical services, I'd use blue-green with manual approval gates, keeping the old version running for instant rollback."

### 💡 Q: Explain the Cluster API and when you'd use it.

**Cluster API** — Kubernetes-native API for declaratively managing Kubernetes clusters (cluster-as-code).

```yaml
# Management cluster manages workload clusters
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: prod-us-east
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["10.244.0.0/16"]
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: prod-us-east-control-plane
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSCluster
    name: prod-us-east

---
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: prod-us-east-control-plane
spec:
  replicas: 3
  version: v1.29.0
  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: AWSMachineTemplate
      name: prod-us-east-control-plane
```

**Use cases:**
- **Multi-cluster management** — Manage 10s-100s of clusters from one place
- **Multi-tenant platform** — Cluster-per-tenant with declarative provisioning
- **Disaster recovery** — Quickly provision replacement clusters
- **Cluster lifecycle automation** — Upgrades, scaling, day-2 operations

**When to use:**
- Running your own Kubernetes platform (not just using EKS/GKE/AKS)
- Managing many clusters (fleet management)
- Need cloud-agnostic cluster provisioning

**When NOT to use:**
- Single cluster
- Using managed Kubernetes (EKS/GKE/AKS is simpler)

---

## Key Resources

**Official Documentation:**
- **Kubernetes Official Docs** — https://kubernetes.io/docs/
- **Kubernetes Blog** — https://kubernetes.io/blog/ (release notes, feature updates)
- **CNCF Landscape** — https://landscape.cncf.io/ (ecosystem overview)

**Learning & Certification:**
- **Kubernetes the Hard Way** — Kelsey Hightower (learn by building from scratch)
- **CKA/CKAD/CKS Certification** — Best structured learning path
- **Killer.sh** — CKA/CKAD exam simulator
- **Learnk8s** — https://learnk8s.io (excellent visual guides)
- **KodeKloud** — Hands-on labs for CKA/CKAD

**Books:**
- **Kubernetes Patterns (book)** — Bilgin Ibryam & Roland Huß
- **Production Kubernetes (book)** — Josh Rosso et al.
- **Kubernetes Best Practices** — Brendan Burns et al.

**Tools & Projects (2025-26 Relevant):**
- **Gateway API** — https://gateway-api.sigs.k8s.io/ (next-gen Ingress)
- **Karpenter** — https://karpenter.sh/ (efficient autoscaling)
- **Cilium** — https://cilium.io/ (eBPF networking, service mesh)
- **Argo Rollouts** — https://argoproj.github.io/rollouts/ (progressive delivery)
- **External Secrets Operator** — https://external-secrets.io/ (secret management)
- **Kubecost / OpenCost** — Cost visibility and optimization
- **KEDA** — https://keda.sh/ (event-driven autoscaling)
- **Kyverno** — https://kyverno.io/ (policy-as-code, simpler than OPA)
- **Crossplane** — https://crossplane.io/ (infrastructure-as-code via K8s)
- **KWOK** — Kubernetes WithOut Kubelet (simulate large clusters for testing)

**Community & News:**
- **Kubernetes Podcast** — https://kubernetespodcast.com/
- **r/kubernetes** — Active Reddit community
- **CNCF Slack** — kubernetes.slack.com (join via communityinviter.com/apps/kubernetes)
