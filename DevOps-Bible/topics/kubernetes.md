---
title: Kubernetes
nav_order: 21
description: "Kubernetes — DevOps interview preparation: concepts, most-asked questions, scenarios, and commands."
---

# Kubernetes — DevOps Interview Preparation

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

---

## Architecture & Components

### Q: Explain Kubernetes architecture.

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

### Q: Explain each control plane component.

| Component | Role | What Happens if it Fails |
|-----------|------|-------------------------|
| **kube-apiserver** | REST API frontend for all operations. All components communicate through it. Validates and persists state to etcd. | Cluster is unmanageable. Existing workloads continue running but no changes possible. |
| **etcd** | Distributed key-value store. Single source of truth for all cluster state. | Data loss risk. Cluster cannot function. (Always back up etcd!) |
| **kube-scheduler** | Watches for unscheduled pods, assigns them to nodes based on resource requirements, affinity, taints/tolerations. | New pods stay Pending. Existing pods unaffected. |
| **kube-controller-manager** | Runs controllers (reconciliation loops): Deployment, ReplicaSet, Node, Job, ServiceAccount, etc. | Cluster stops self-healing. No scaling, no rescheduling of failed pods. |
| **cloud-controller-manager** | Integrates with cloud provider APIs for LoadBalancers, Routes, Volumes. | Cloud resources not provisioned (LBs, EBS volumes). |

### Q: Explain each worker node component.

| Component | Role |
|-----------|------|
| **kubelet** | Agent on each node. Ensures containers in pods are running. Reports node status to API server. Pulls images, mounts volumes. |
| **kube-proxy** | Maintains network rules (iptables/IPVS) for Service → Pod routing. |
| **Container Runtime** | Actually runs containers. containerd (default), CRI-O, or any CRI-compliant runtime. |

### Q: What happens when you run `kubectl apply -f deployment.yaml`?

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

### Q: What is a Pod? Why not just containers?

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

### Q: What are Init Containers?

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

### Q: Explain Pod lifecycle and phases.

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

### Q: What is a static pod?

Static pods are managed directly by the **kubelet** on a specific node, without the API server.

- Defined as YAML files in `/etc/kubernetes/manifests/` (or configured path)
- Kubelet watches this directory and creates/manages pods automatically
- Control plane components (apiserver, etcd, scheduler, controller-manager) are typically static pods in kubeadm clusters
- API server creates a **mirror pod** (read-only) so they appear in `kubectl get pods`

---

## Workload Resources

### Q: Explain Deployment, ReplicaSet, DaemonSet, StatefulSet, and Job.

| Resource | Purpose | Use Case |
|----------|---------|----------|
| **Deployment** | Declarative updates for Pods. Manages ReplicaSets. | Stateless apps (web servers, APIs) |
| **ReplicaSet** | Ensures N pod replicas are running. | Don't use directly — Deployments manage these |
| **DaemonSet** | Ensures one pod per node (or subset). | Log collectors, monitoring agents, CNI plugins |
| **StatefulSet** | Ordered, stable pods with persistent identity. | Databases, Kafka, ZooKeeper, etcd |
| **Job** | Runs pods to completion. | Batch processing, migrations, one-time tasks |
| **CronJob** | Scheduled Jobs. | Backups, reports, cleanup tasks |

### Q: Explain StatefulSet guarantees.

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

### Q: DaemonSet use cases and updates.

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

### Q: Explain Kubernetes Service types.

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

### Q: How does Kubernetes DNS work?

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

### Q: Explain Kubernetes networking model (CNI).

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

---

## Ingress & Gateway API

### Q: What is Ingress?

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

### Q: Explain Kubernetes storage architecture.

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

---

## ConfigMaps & Secrets

### Q: ConfigMap vs Secret?

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

---

## RBAC & Security

### Q: Explain Kubernetes RBAC.

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

### Q: Kubernetes Security Best Practices?

1. **Pod Security Standards (PSS):**
   ```yaml
   # Namespace label to enforce security
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

3. **Network Policies** — Default deny, whitelist required traffic
4. **Image scanning** in CI/CD (Trivy, Snyk)
5. **Limit API server access** — Private endpoint, authorized networks
6. **Audit logging** — Enable K8s audit logs
7. **Rotate credentials** — ServiceAccount tokens, etcd certs

---

## Scheduling & Node Management

### Q: Explain taints, tolerations, and affinity.

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

### Q: What happens during node drain?

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

### Q: Explain requests, limits, and QoS classes.

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

### Q: Explain liveness, readiness, and startup probes.

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

### Q: Explain HPA, VPA, and Cluster Autoscaler.

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

---

## Deployment Strategies

### Q: Explain Kubernetes deployment strategies.

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

### Q: What is Helm and how does it work?

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

### Q: What are Custom Resource Definitions (CRDs) and Operators?

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

### Q: How do you monitor Kubernetes?

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

### Q: Pod is stuck in various states. How to debug?

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

### Q: Master troubleshooting commands cheat sheet.

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

### Q: Explain Network Policies.

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

## etcd

### Q: What is etcd and why is it critical?

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

## Scenario-Based Questions

### Q: Your Kubernetes cluster is running out of resources. What do you do?

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

### Q: A deployment rollout is stuck. How do you fix it?

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

### Q: How do you handle certificate rotation in Kubernetes?

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

## Key Resources

- **Kubernetes Official Docs** — https://kubernetes.io/docs/
- **Kubernetes the Hard Way** — Kelsey Hightower (learn by building from scratch)
- **CKA/CKAD/CKS Certification** — Best structured learning path
- **Killer.sh** — CKA/CKAD exam simulator
- **Kubernetes Patterns (book)** — Bilgin Ibryam & Roland Huß
- **Production Kubernetes (book)** — Josh Rosso et al.
- **Learnk8s** — https://learnk8s.io (excellent visual guides)
- **KWOK** — Kubernetes WithOut Kubelet (simulate large clusters for testing)
