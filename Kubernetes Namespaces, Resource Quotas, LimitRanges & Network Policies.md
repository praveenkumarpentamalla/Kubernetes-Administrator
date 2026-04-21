## Table of Contents

1. [Introduction — Why Namespaces Are the Foundation of Multi-Tenancy](#1-introduction)
2. [Core Identity Table — Namespace Ecosystem Components](#2-core-identity-table)
3. [Understanding the Need for Namespaces](#3-understanding-need-for-namespaces)
4. [The Controller Pattern — Watch → Compare → Act → Loop](#4-controller-pattern)
5. [Lab: Creating Namespaces — All Methods](#5-lab-creating-namespaces)
6. [Understanding Resource Quotas in Kubernetes](#6-understanding-resource-quotas)
7. [Lab: Setting ResourceQuota on Namespaces](#7-lab-resourcequota)
8. [Understanding LimitRanges in Kubernetes](#8-understanding-limitranges)
9. [Lab: Setting LimitRanges on Namespaces](#9-lab-limitranges)
10. [Controlling Communication Using Network Policies](#10-network-policies)
11. [Lab: Limiting Pod Communication Using Network Policies](#11-lab-network-policies)
12. [Built-in Controllers Deep Dive](#12-built-in-controllers)
13. [Internal Working Concepts — Informers, Work Queues & Reconciliation](#13-internal-concepts)
14. [API Server and etcd Interaction](#14-api-server-etcd)
15. [Leader Election — HA for Namespace Infrastructure](#15-leader-election)
16. [Performance Tuning](#16-performance-tuning)
17. [Security Hardening Practices](#17-security-hardening)
18. [Monitoring and Observability](#18-monitoring)
19. [Troubleshooting Namespace, Quota, and Network Issues](#19-troubleshooting)
20. [Disaster Recovery Concepts](#20-disaster-recovery)
21. [Comparison — kube-apiserver vs kube-scheduler vs kube-controller-manager](#21-comparison)
22. [ASCII Architecture Diagram](#22-ascii-diagram)
23. [Real-World Production Use Cases](#23-production-use-cases)
24. [Best Practices for Production Environments](#24-best-practices)
25. [Common Mistakes and Pitfalls](#25-mistakes-pitfalls)
26. [Interview Questions — Beginner to Advanced](#26-interview-questions)
27. [Cheat Sheet — Commands, Flags & Manifests](#27-cheat-sheet)
28. [Key Takeaways & Summary](#28-key-takeaways)

---

## 1. Introduction — Why Namespaces Are the Foundation of Multi-Tenancy {#1-introduction}

A Kubernetes cluster is a shared compute fabric. In enterprise environments, dozens of teams, applications, and lifecycle stages coexist on the same hardware — all competing for the same pool of CPU, memory, and network. Without structured isolation, a single team's poorly-configured batch job can consume all cluster memory, a developer's accidental `kubectl delete pods --all` can destroy production workloads, and there is no way to audit who consumed which resources.

**Namespaces** are Kubernetes's built-in answer to this challenge. They provide:

- **Logical partitioning** — virtual clusters within a physical cluster
- **Access control scoping** — RBAC applies independently per namespace
- **Resource governance** — ResourceQuota limits aggregate consumption
- **Default constraints** — LimitRange enforces per-object sizing
- **Traffic isolation** — NetworkPolicy controls inter-pod communication

Together, these four mechanisms form the **multi-tenancy stack** — the layer between raw cluster compute and safe, accountable, isolated workload execution.

### The Governance Stack in One View

```
┌─────────────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                               │
│                                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │ team-a-prod  │  │ team-b-prod  │  │ platform-monitoring      │  │
│  │              │  │              │  │                          │  │
│  │ RBAC:        │  │ RBAC:        │  │ RBAC: SREs only          │  │
│  │ team-a only  │  │ team-b only  │  │                          │  │
│  │              │  │              │  │ Quota:                   │  │
│  │ Quota:       │  │ Quota:       │  │ 2 CPU, 4Gi               │  │
│  │ 8 CPU, 16Gi  │  │ 4 CPU, 8Gi   │  │                          │  │
│  │              │  │              │  │ LimitRange:              │  │
│  │ LimitRange:  │  │ LimitRange:  │  │ max 500m/pod             │  │
│  │ max 2CPU/pod │  │ max 1CPU/pod │  │                          │  │
│  │              │  │              │  │ NetworkPolicy:           │  │
│  │ NetPol:      │  │ NetPol:      │  │ allow internal only      │  │
│  │ deny-all +   │  │ deny-all +   │  │                          │  │
│  │ allow ingress│  │ allow egress │  │                          │  │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### The Production Cost of Missing Namespace Controls

| Risk Without Controls | Mechanism That Prevents It |
|---|---|
| Team A consumes all cluster CPU, starves Team B | ResourceQuota |
| Developer runs `kubectl delete pods --all` in production | RBAC (role scoped to dev namespace) |
| Container starts with no limits, OOM-kills node processes | LimitRange (injects default limits) |
| Pod in dev namespace talks to production database | NetworkPolicy (deny cross-ns by default) |
| No accountability for resource chargeback | Namespace labels + Prometheus aggregation |
| One slow team blocks others via runaway Jobs | ResourceQuota (pod count + CPU cap) |

---

## 2. Core Identity Table — Namespace Ecosystem Components {#2-core-identity-table}

| Component | Kind / Binary | API Group | Namespace Scoped | Role |
|---|---|---|---|---|
| **Namespace** | `namespace` | `core/v1` | **No** (cluster-scoped) | Logical partition; scope container for all namespaced resources |
| **ResourceQuota** | `resourcequota` | `core/v1` | Yes | Limits aggregate resource consumption in a namespace |
| **LimitRange** | `limitrange` | `core/v1` | Yes | Sets per-object resource defaults, minimums, and maximums |
| **NetworkPolicy** | `networkpolicy` | `networking.k8s.io/v1` | Yes | Declares allowed ingress/egress traffic rules for pods |
| **RBAC Role** | `role` | `rbac.authorization.k8s.io` | Yes | Permission rules scoped to one namespace |
| **RBAC ClusterRole** | `clusterrole` | `rbac.authorization.k8s.io` | No | Cluster-wide permission rules |
| **RoleBinding** | `rolebinding` | `rbac.authorization.k8s.io` | Yes | Binds Role to subjects within a namespace |
| **kube-apiserver** | Binary | N/A | N/A | Validates quota; enforces LimitRange via admission; stores all namespace objects |
| **kube-controller-manager** | Binary | 10257 | N/A | Namespace controller (lifecycle); ResourceQuota controller (status tracking) |
| **kube-scheduler** | Binary | 10259 | N/A | Schedules pods; reads node capacity; unaware of quotas directly |
| **CNI Plugin** | DaemonSet | N/A | N/A | Enforces NetworkPolicy via iptables/eBPF on every node |
| **ResourceQuota Admission** | Plugin (in API server) | N/A | N/A | Synchronous enforcement at request time |
| **LimitRanger Admission** | Plugin (in API server) | N/A | N/A | Injects defaults and validates min/max per object |
| **etcd** | Binary | N/A | N/A | Stores all namespace objects; only API server accesses it |

---

## 3. Understanding the Need for Namespaces {#3-understanding-need-for-namespaces}

### 3.1 What Namespaces Provide

Namespaces create **virtual clusters** within a physical cluster. They provide four critical properties:

**1. Name scoping** — identical resource names can coexist in different namespaces:
```
namespace: production  → Deployment "api-server" (v2.1.0)
namespace: staging     → Deployment "api-server" (v2.2.0-rc1)
namespace: development → Deployment "api-server" (v2.3.0-dev)
```

**2. Access control boundaries** — RBAC is applied per-namespace:
```
Jane: admin in team-a, view in team-b
Bob:  admin in team-b, no access to team-a
```

**3. Resource governance** — quotas apply independently:
```
team-a-production: 8 CPU, 16Gi RAM, 50 pods
team-b-production: 4 CPU,  8Gi RAM, 20 pods
```

**4. DNS scoping** — Services discoverable by namespace:
```bash
# From a pod in namespace: production
curl http://my-service                                     # Same namespace (short name)
curl http://my-service.staging.svc.cluster.local           # Cross-namespace (FQDN)
curl http://my-service.staging                             # Cross-namespace (short-form)
```

### 3.2 Namespaced vs Cluster-Scoped Resources

| Namespaced Resources | Cluster-Scoped Resources |
|---|---|
| Pod, Deployment, StatefulSet, DaemonSet, Job | Node, Namespace itself |
| Service, Endpoints, EndpointSlice | PersistentVolume, StorageClass |
| ConfigMap, Secret, ServiceAccount | ClusterRole, ClusterRoleBinding |
| PersistentVolumeClaim | CustomResourceDefinition (CRD) |
| Role, RoleBinding | IngressClass, APIService |
| ResourceQuota, LimitRange | RuntimeClass |
| NetworkPolicy, Ingress | PodSecurityPolicy (deprecated) |

### 3.3 The Four System Namespaces

| Namespace | Purpose | Who Manages It |
|---|---|---|
| `default` | Resources created without explicit namespace | Users |
| `kube-system` | Kubernetes system components (DNS, scheduler, kube-proxy) | Kubernetes |
| `kube-public` | Publicly readable data (cluster-info ConfigMap) | Kubernetes |
| `kube-node-lease` | Node heartbeat Lease objects for fast failure detection | Kubernetes |

### 3.4 Multi-Tenancy Patterns in Production

| Pattern | Namespace Per | Isolation Level | Best For |
|---|---|---|---|
| **By environment** | dev / staging / production | Medium | Small orgs, single-team |
| **By team** | team-a / team-b / platform | High | Multi-team enterprises |
| **By application** | app1 / app2 / app3 | Medium | Microservice orgs |
| **Hierarchical** | team-a-prod / team-a-staging | High | Large enterprise |
| **By tier** | frontend / backend / data | Low | Monorepo architecture |

---

## 4. The Controller Pattern — Watch → Compare → Act → Loop {#4-controller-pattern}

The Kubernetes control plane is built around the **reconciliation loop** — a continuous cycle of observing cluster state and taking action to match desired state. This pattern is used by the Namespace controller, ResourceQuota controller, and all other built-in controllers.

### 4.1 The Four-Phase Loop

```
┌─────────────────────────────────────────────────────────────────────┐
│              CONTROLLER RECONCILIATION LOOP                         │
│                                                                     │
│  1. WATCH                                                           │
│  ──────────────────────────────────────────────────────────────     │
│  Informer streams ADDED/MODIFIED/DELETED events from API server     │
│  via persistent HTTP watch connection (long-poll or HTTP/2 push).   │
│  Events enqueued as keys: "namespace/resource-name"                 │
│                                                                     │
│  2. COMPARE                                                         │
│  ──────────────────────────────────────────────────────────────     │
│  Worker goroutine dequeues key.                                     │
│  Reads DESIRED state from object .spec field (from local cache).    │
│  Reads ACTUAL state from cluster (from local cache or API call).    │
│  Computes delta: what action is needed?                             │
│                                                                     │
│  3. ACT                                                             │
│  ──────────────────────────────────────────────────────────────     │
│  Makes API calls to create/update/delete resources.                 │
│  Updates .status field with observed state.                         │
│  If error: workQueue.AddRateLimited(key)  [exponential backoff]     │
│                                                                     │
│  4. LOOP                                                            │
│  ──────────────────────────────────────────────────────────────     │
│  workQueue.Forget(key) on success.                                  │
│  Returns to WATCH for next event.                                   │
│  Periodic full resync (10-30 min) catches any missed events.        │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 Namespace Controller Reconciliation

```
NAMESPACE CONTROLLER LOOP:

WATCH:    Namespace ADDED/MODIFIED/DELETED events
COMPARE:  
  If namespace.status.phase == Active → nothing to do
  If namespace.status.phase == Terminating → are child resources deleted?
ACT:
  Namespace ADDED   → Start watching child resources
  Namespace DELETED → Set phase=Terminating
                   → Delete all namespaced resources in cascade order:
                     services → pods → configmaps → secrets → PVCs → ...
                   → When all deleted → remove kubernetes finalizer
                   → etcd deletes namespace object
LOOP:     Return to WATCH
```

### 4.3 ResourceQuota Controller Reconciliation

```
RESOURCEQUOTA CONTROLLER LOOP:

WATCH:    Pod/Service/PVC events in each namespace; ResourceQuota changes
COMPARE:  
  Current quota.status.used vs actual resource consumption
  Sum all pod CPU/memory requests, count all objects
ACT:
  PATCH ResourceQuota.status.used with recalculated totals
  (Note: this is STATUS tracking only — enforcement is done by Admission)
LOOP:
  Triggered by: Pod create/delete, ResourceQuota changes
  Also: periodic resync every 5 minutes regardless of events

CRITICAL INSIGHT:
  Admission Controller = synchronous enforcement at request time
  ResourceQuota Controller = asynchronous status reconciliation
  They work together for accurate quota tracking and enforcement.
```

---

## 5. Lab: Creating Namespaces — All Methods {#5-lab-creating-namespaces}

### 5.1 Method 1: Imperative CLI

```bash
# Create a single namespace
kubectl create namespace development
kubectl create namespace staging
kubectl create namespace production

# Verify creation
kubectl get namespaces
# NAME              STATUS   AGE
# default           Active   10d
# development       Active   3s
# kube-node-lease   Active   10d
# kube-public       Active   10d
# kube-system       Active   10d
# production        Active   1s
# staging           Active   2s

# Inspect a namespace
kubectl describe namespace production
```

### 5.2 Method 2: YAML Manifest (Recommended for Production)

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: team-a-production
  labels:
    # Labels are critical for NetworkPolicy namespaceSelector
    # and Prometheus aggregation by namespace
    team: team-a
    environment: production
    cost-center: "engineering-101"
    # Auto-added by API server but shown for clarity:
    kubernetes.io/metadata.name: team-a-production

  annotations:
    description: "Production workloads for Team A — API services"
    contact: "team-a@company.com"
    slack-channel: "#team-a-alerts"
    created-by: "platform-team"
    namespace.company.com/owner: "team-a"
    namespace.company.com/budget: "team-a-production-quota"
EOF

# Create namespaces for all environments using a loop
for env in dev staging production; do
  cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: team-a-${env}
  labels:
    team: team-a
    environment: ${env}
    kubernetes.io/metadata.name: team-a-${env}
  annotations:
    contact: "team-a@company.com"
EOF
done

# Verify
kubectl get namespaces -l team=team-a
```

### 5.3 Setting Default Namespace in kubectl Context

```bash
# Switch default namespace for current context
kubectl config set-context --current --namespace=team-a-production

# Verify
kubectl config view --minify | grep namespace
# namespace: team-a-production

# Now all commands use this namespace by default
kubectl get pods   # Same as: kubectl get pods -n team-a-production

# Switch back
kubectl config set-context --current --namespace=default

# Use kubectx/kubens for fast switching (install kubectx package)
kubens team-a-production
kubens default
```

### 5.4 Namespace Lifecycle Operations

```bash
# Check namespace status
kubectl get namespace team-a-dev \
  -o jsonpath='{.status.phase}'
# Active

# View all resources in a namespace
kubectl get all -n team-a-production
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -I{} sh -c "kubectl get {} -n team-a-production 2>/dev/null | grep -v '^$'"

# Delete a namespace (CASCADES to ALL resources within!)
kubectl delete namespace team-a-dev
# WARNING: This deletes every Pod, Deployment, Service, Secret, etc.

# Watch namespace termination progress
kubectl get namespace team-a-dev -w
# team-a-dev   Terminating   0s
# (disappears when all resources cleaned up)

# Fix namespace stuck in Terminating (check finalizers)
kubectl get namespace stuck-ns \
  -o jsonpath='{.metadata.finalizers}'
# ["kubernetes"] = namespace controller hasn't finished cleanup

# Emergency: force remove finalizer (USE ONLY WHEN STUCK)
kubectl patch namespace stuck-ns \
  -p '{"metadata":{"finalizers":[]}}' \
  --type=merge
```

---

## 6. Understanding Resource Quotas in Kubernetes {#6-understanding-resource-quotas}

### 6.1 ResourceQuota Controls Aggregate Namespace Limits

ResourceQuota enforces limits on the **total** resource consumption across ALL objects in a namespace. It answers: "What is this namespace's budget?"

```yaml
# ResourceQuota tracks AGGREGATE usage, not per-pod usage
# If quota.hard.requests.cpu = 10:
# This means: SUM of ALL pod CPU requests in namespace <= 10 cores
# It does NOT mean each pod is limited to 10 cores (that's LimitRange)
```

### 6.2 ResourceQuota Categories

| Category | Resource Names | Description |
|---|---|---|
| **Compute** | `requests.cpu`, `limits.cpu`, `requests.memory`, `limits.memory` | CPU and RAM constraints |
| **Storage** | `requests.storage`, `persistentvolumeclaims` | Disk consumption |
| **Object Count** | `pods`, `services`, `secrets`, `configmaps`, `deployments` | API object limits |
| **Service Types** | `services.loadbalancers`, `services.nodeports` | Service type limits |
| **Extended Resources** | `requests.nvidia.com/gpu` | Custom hardware resources |

### 6.3 How ResourceQuota Enforcement Works — Two-Part System

```
PART 1: ADMISSION ENFORCEMENT (synchronous, in kube-apiserver)
─────────────────────────────────────────────────────────────────
User: kubectl apply -f pod.yaml (requests: cpu=500m, memory=256Mi)
         │
         ▼ HTTPS :6443
┌────────────────────────────────────────────────────────────────┐
│  kube-apiserver                                                │
│                                                                │
│  1. Authentication + Authorization (RBAC)                     │
│         │                                                      │
│  2. ResourceQuota Admission Plugin                            │
│     ┌──────────────────────────────────────────────────────┐  │
│     │ Load ResourceQuota for namespace: team-a-prod        │  │
│     │ quota.hard.requests.cpu = 8000m                      │  │
│     │ quota.status.used.requests.cpu = 7600m               │  │
│     │                                                       │  │
│     │ Can we fit 500m more?                                 │  │
│     │ 7600m + 500m = 8100m > 8000m  → REJECT (403)        │  │
│     │                                                       │  │
│     │ If 7000m + 500m = 7500m ≤ 8000m → ALLOW              │  │
│     │ Atomically update: quota.status.used = 7500m         │  │
│     └──────────────────────────────────────────────────────┘  │
│         │                                                      │
│  3. Write Pod to etcd                                          │
└────────────────────────────────────────────────────────────────┘

PART 2: STATUS RECONCILIATION (async, in kube-controller-manager)
─────────────────────────────────────────────────────────────────
ResourceQuota controller periodically recalculates actual usage
and updates quota.status.used to match reality.
This catches any drift from the synchronous tracking.
```

### 6.4 ResourceQuota and QoS Scope Selectors

```yaml
# Quotas can target specific QoS classes or priority classes
apiVersion: v1
kind: ResourceQuota
metadata:
  name: guaranteed-quota
  namespace: production
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
  # Only apply this quota to Guaranteed QoS pods
  scopes:
    - BestEffort      # or: NotBestEffort, Terminating, NotTerminating
  # OR use scopeSelector for priority classes:
  scopeSelector:
    matchExpressions:
      - operator: In
        scopeName: PriorityClass
        values:
          - high-priority
          - critical
```

### 6.5 Important ResourceQuota Behavior Rules

```
RULE 1: If a ResourceQuota with compute resources exists in a namespace,
        EVERY Pod must specify resource requests/limits.
        Pods without resources will be REJECTED.
        Solution: Use LimitRange to inject defaults.

RULE 2: ResourceQuota admission uses optimistic concurrency.
        Two simultaneous Pod creations both check against the SAME quota.
        The API server handles conflicts with retry logic.

RULE 3: ResourceQuota status may briefly lag behind reality.
        The status is reconciled by the controller asynchronously.
        The admission plugin maintains real-time accuracy.

RULE 4: Terminating pods still count against quota until fully deleted.
        Plan for pods in Terminating state during rolling updates.
```

---

## 7. Lab: Setting ResourceQuota on Namespaces {#7-lab-resourcequota}

### 7.1 Create Namespace with Comprehensive Quota

```bash
kubectl create namespace quota-lab

# Apply a production-grade ResourceQuota
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-lab-quota
  namespace: quota-lab
  labels:
    tier: team-quota
    managed-by: platform-team
spec:
  hard:
    # ── COMPUTE RESOURCES ─────────────────────────────────────────
    # Total CPU requests across all pods (scheduler uses requests)
    requests.cpu: "4"
    # Total CPU limits across all pods
    limits.cpu: "8"
    # Total memory requests across all pods
    requests.memory: 4Gi
    # Total memory limits across all pods
    limits.memory: 8Gi

    # ── STORAGE RESOURCES ─────────────────────────────────────────
    requests.storage: 50Gi
    persistentvolumeclaims: "10"

    # ── OBJECT COUNT QUOTAS ────────────────────────────────────────
    pods: "20"
    services: "10"
    services.loadbalancers: "2"
    services.nodeports: "3"
    configmaps: "20"
    secrets: "20"
    replicationcontrollers: "5"
EOF

# Verify quota creation
kubectl describe resourcequota quota-lab-quota -n quota-lab
# Name:                   quota-lab-quota
# Namespace:              quota-lab
# Resource                Used  Hard
# --------                ----  ----
# configmaps              0     20
# limits.cpu              0     8
# limits.memory           0     8Gi
# ...
```

### 7.2 Test Quota Enforcement — Observe Usage

```bash
# Create a deployment that consumes quota
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment
  namespace: quota-lab
spec:
  replicas: 3
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
        - name: app
          image: nginx:1.25
          resources:
            requests:
              cpu: 500m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 512Mi
EOF

kubectl rollout status deployment/test-deployment -n quota-lab

# Observe quota consumption
kubectl describe resourcequota quota-lab-quota -n quota-lab
# requests.cpu:    1500m / 4    ← 3 pods × 500m
# limits.cpu:      3 / 8        ← 3 pods × 1 core
# requests.memory: 768Mi / 4Gi  ← 3 pods × 256Mi
# pods:            3 / 20
```

### 7.3 Demonstrate Quota Rejection

```bash
# Try to create a deployment that would exceed CPU quota
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quota-buster
  namespace: quota-lab
spec:
  replicas: 10    # Would need 10 × 500m = 5 CPU requests; quota is 4
  selector:
    matchLabels:
      app: quota-buster
  template:
    metadata:
      labels:
        app: quota-buster
    spec:
      containers:
        - name: app
          image: nginx:1.25
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: "1"
              memory: 1Gi
EOF

# Check what happened
kubectl get deployment quota-buster -n quota-lab
kubectl describe replicaset -n quota-lab -l app=quota-buster | \
  grep -A 5 "Events:"
# Warning  FailedCreate  5s  replicaset-controller
# Error creating: pods "quota-buster-xxxxx" is forbidden:
# exceeded quota: quota-lab-quota, requested: requests.cpu=500m,
# used: requests.cpu=1500m, limited: requests.cpu=4000m
# (Only 3 of 10 can fit because 3×500m=1500m already used, 4000m total)

# Check final state: some pods created (that fit within quota)
kubectl get pods -n quota-lab -l app=quota-buster

# Cleanup
kubectl delete deployments test-deployment quota-buster -n quota-lab
```

### 7.4 Pod Without Resources Fails When Quota Exists

```bash
# Try creating a pod without resource specs when quota exists
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-resources-pod
  namespace: quota-lab
spec:
  containers:
    - name: app
      image: nginx:1.25
      # No resources section!
EOF

# This FAILS with:
# Error from server (Forbidden): error when creating "STDIN":
# pods "no-resources-pod" is forbidden: failed quota:
# quota-lab-quota: must specify limits.cpu for: app;
#                  must specify limits.memory for: app;
#                  must specify requests.cpu for: app;
#                  must specify requests.memory for: app

# Fix 1: Add resources explicitly
# Fix 2: Add LimitRange to inject defaults (covered in next section)

kubectl delete namespace quota-lab
```

---

## 8. Understanding LimitRanges in Kubernetes {#8-understanding-limitranges}

### 8.1 What LimitRange Controls

While ResourceQuota controls **aggregate** namespace limits, LimitRange controls **per-object** constraints. It answers:
- What is the minimum CPU a container can request?
- What is the maximum memory a pod can use in total?
- What are the default resource values if a container doesn't specify any?
- How large can a PersistentVolumeClaim be?

### 8.2 LimitRange Types and Fields

| Type | Applies To | Controls |
|---|---|---|
| `Container` | Individual containers | CPU/memory min, max, default, defaultRequest, maxLimitRequestRatio |
| `Pod` | Sum of all containers in a pod | CPU/memory min, max |
| `PersistentVolumeClaim` | Storage requests | Storage min, max |

```yaml
# Complete LimitRange field reference
spec:
  limits:
    - type: Container
      # DEFAULT: Injected when no limits specified
      default:
        cpu: "200m"      # Default CPU LIMIT (applied if container has no limits)
        memory: "256Mi"  # Default memory LIMIT

      # DEFAULT REQUEST: Injected when no requests specified
      defaultRequest:
        cpu: "100m"      # Default CPU REQUEST
        memory: "128Mi"  # Default memory REQUEST

      # MINIMUM: Container cannot request LESS than this
      min:
        cpu: "50m"
        memory: "64Mi"

      # MAXIMUM: Container cannot have limit MORE than this
      max:
        cpu: "4"
        memory: "4Gi"

      # MAX RATIO: limits.cpu / requests.cpu cannot exceed this value
      # e.g., 4 means: if request=100m, limit cannot exceed 400m
      maxLimitRequestRatio:
        cpu: "4"
        memory: "2"
```

### 8.3 How LimitRange and ResourceQuota Work Together

```
SCENARIO: Namespace with BOTH LimitRange and ResourceQuota

User creates Pod with NO resource specs:

STEP 1 — LimitRanger Admission Plugin (runs FIRST):
  Detects no resources on container
  Injects from LimitRange:
    container.resources.requests.cpu    = 100m  (defaultRequest)
    container.resources.limits.cpu      = 200m  (default)
    container.resources.requests.memory = 128Mi (defaultRequest)
    container.resources.limits.memory   = 256Mi (default)
  Validates injected values against min/max constraints

STEP 2 — ResourceQuota Admission Plugin (runs SECOND):
  Pod now has resource specs (injected by LimitRanger)
  Checks: current_used + 100m CPU request <= quota.hard.requests.cpu
  If within quota: ALLOW (update quota.status.used)
  If over quota: REJECT (403)

KEY INSIGHT: LimitRange enables ResourceQuota for pods without explicit resources.
             Without LimitRange, pods with no resources would be REJECTED
             if quota includes compute resources.
```

### 8.4 LimitRange for Pods and Storage

```yaml
spec:
  limits:
    # Per-pod: sum of all containers cannot exceed this
    - type: Pod
      max:
        cpu: "4"        # All containers in pod total <= 4 CPU cores
        memory: "4Gi"   # All containers total <= 4Gi RAM

    # Per-PVC: storage request constraints
    - type: PersistentVolumeClaim
      min:
        storage: "100Mi"  # PVC must request at least 100Mi
      max:
        storage: "50Gi"   # PVC cannot request more than 50Gi
```

### 8.5 LimitRange Admission Behavior Details

```
INJECTION happens when:
  - Container has NO resources section at all

VALIDATION happens when:
  - Container has explicit resources
  - Validates against min/max/ratio constraints

ORDER MATTERS:
  - LimitRange default injection applies to the whole container spec
  - If SOME fields are specified, only MISSING fields get defaults
  - Example: If cpu.requests specified but memory.requests missing:
    → memory.requests gets injected from defaultRequest
    → cpu.requests stays as specified by user
```

---

## 9. Lab: Setting LimitRanges on Namespaces {#9-lab-limitranges}

### 9.1 Create Namespace with Comprehensive LimitRange

```bash
kubectl create namespace limitrange-lab

cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: lr-lab-limits
  namespace: limitrange-lab
spec:
  limits:
    # ── PER-CONTAINER LIMITS ─────────────────────────────────────────
    - type: Container
      # Injected when container has no limits
      default:
        cpu: "200m"
        memory: "256Mi"
      # Injected when container has no requests
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      # Container cannot go below these
      min:
        cpu: "50m"
        memory: "64Mi"
      # Container cannot exceed these
      max:
        cpu: "2"
        memory: "2Gi"
      # limits.cpu / requests.cpu <= 4
      maxLimitRequestRatio:
        cpu: "4"
        memory: "2"

    # ── PER-POD LIMITS (sum of all containers) ───────────────────────
    - type: Pod
      max:
        cpu: "4"
        memory: "4Gi"

    # ── STORAGE LIMITS ────────────────────────────────────────────────
    - type: PersistentVolumeClaim
      min:
        storage: "100Mi"
      max:
        storage: "50Gi"
EOF

kubectl describe limitrange lr-lab-limits -n limitrange-lab
```

### 9.2 Demonstrate Default Injection

```bash
# Create pod WITHOUT any resource specifications
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: no-resources-pod
  namespace: limitrange-lab
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      # Intentionally NO resources section!
EOF

# Check what was injected by LimitRange
kubectl get pod no-resources-pod -n limitrange-lab \
  -o jsonpath='{.spec.containers[0].resources}' | jq
# Output:
# {
#   "limits": {
#     "cpu": "200m",     ← from LimitRange.default
#     "memory": "256Mi"  ← from LimitRange.default
#   },
#   "requests": {
#     "cpu": "100m",     ← from LimitRange.defaultRequest
#     "memory": "128Mi"  ← from LimitRange.defaultRequest
#   }
# }

echo "LimitRange successfully injected defaults!"
kubectl delete pod no-resources-pod -n limitrange-lab
```

### 9.3 Demonstrate LimitRange Enforcement

```bash
# Attempt 1: Container exceeding max CPU
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: too-much-cpu
  namespace: limitrange-lab
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: "3"     # EXCEEDS max of 2!
          memory: 256Mi
        limits:
          cpu: "4"     # EXCEEDS max of 2!
          memory: 512Mi
EOF
# Error: maximum cpu usage per Container is 2, but limit is 4.
#        maximum cpu usage per Container is 2, but request is 3.

# Attempt 2: Bad limit/request ratio
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: bad-ratio
  namespace: limitrange-lab
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: "100m"
          memory: 128Mi
        limits:
          cpu: "500m"    # 500/100 = 5 > maxLimitRequestRatio of 4!
          memory: 512Mi  # 512/128 = 4 > maxLimitRequestRatio of 2!
EOF
# Error: max limit to request ratio for cpu is 4, but provided ratio is 5.
#        max limit to request ratio for memory is 2, but provided ratio is 4.

echo "LimitRange correctly rejected invalid resource specs!"
```

### 9.4 Combined ResourceQuota + LimitRange (Production Pattern)

```bash
kubectl create namespace full-governance

# Apply both together — the standard production pattern
cat << 'EOF' | kubectl apply -f -
---
# LimitRange FIRST: provides defaults so pods without resources work
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: full-governance
spec:
  limits:
    - type: Container
      default:
        cpu: "200m"
        memory: "256Mi"
      defaultRequest:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "2"
        memory: "2Gi"
---
# ResourceQuota SECOND: enforces namespace budget
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: full-governance
spec:
  hard:
    requests.cpu: "4"
    limits.cpu: "8"
    requests.memory: 4Gi
    limits.memory: 8Gi
    pods: "20"
    services: "10"
    secrets: "20"
    configmaps: "20"
EOF

# Now pods without resources WORK (LimitRange injects defaults)
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: governed-pod
  namespace: full-governance
spec:
  containers:
    - name: app
      image: nginx:1.25
      # No resources — LimitRange injects defaults
      # ResourceQuota tracks the injected values
EOF

# Verify governance is working
kubectl describe resourcequota namespace-quota -n full-governance
# requests.cpu: 100m / 4   ← default injected by LimitRange

kubectl get pod governed-pod -n full-governance \
  -o jsonpath='{.spec.containers[0].resources}' | jq
# Shows: cpu=100m request, cpu=200m limit (injected by LimitRange)

kubectl delete namespace full-governance
```

---

## 10. Controlling Communication Using Network Policies {#10-network-policies}

### 10.1 The Default Kubernetes Network Model

By default, Kubernetes has **no network isolation**. Every pod can reach every other pod in any namespace using their Pod IP address. This is the "flat network" model — convenient for development but dangerous in production.

NetworkPolicy provides **Layer 3/4 firewall rules** at the pod level.

### 10.2 CNI Plugin Prerequisite

```
CRITICAL: NetworkPolicy objects are stored in etcd by the API server.
They do NOT do anything by themselves.

ENFORCEMENT is done by the CNI (Container Network Interface) plugin:
  Calico    → iptables or eBPF rules per policy
  Cilium    → eBPF programs (most performant)
  Weave Net → iptables rules
  Flannel   → Does NOT support NetworkPolicy!
              (Flannel users need Calico or Cilium for enforcement)
  Canal     → Flannel + Calico = supports NetworkPolicy

If your CNI doesn't support NetworkPolicy:
  → Policies are created and stored in etcd
  → NO enforcement happens
  → Traffic flows freely regardless of policies
  → VERIFY: kubectl get pods -n kube-system | grep -E "calico|cilium|weave"
```

### 10.3 NetworkPolicy Key Concepts

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-policy
  namespace: production
spec:
  # Which pods this policy APPLIES TO (the subjects)
  podSelector:
    matchLabels:
      app: api-server

  # Which directions to control
  policyTypes:
    - Ingress   # Control incoming traffic TO selected pods
    - Egress    # Control outgoing traffic FROM selected pods

  # INGRESS RULES: who can send traffic TO the selected pods
  ingress:
    - from:
        - podSelector:              # Pods with matching labels (same namespace)
            matchLabels:
              app: frontend
        - namespaceSelector:        # All pods in matching namespaces
            matchLabels:
              environment: production
        - ipBlock:                  # External IP ranges
            cidr: 10.0.0.0/8
            except:
              - 10.0.1.0/24
      ports:
        - protocol: TCP
          port: 8080

  # EGRESS RULES: where selected pods can SEND traffic
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    - to: []      # Empty means any destination
      ports:
        - protocol: UDP
          port: 53  # Allow DNS resolution
```

### 10.4 The Critical AND vs OR Logic

```
SAME `from` item = AND (must match both):
  ingress:
    - from:
        - namespaceSelector:      ← AND
            matchLabels:
              env: production
          podSelector:            ← SAME item (AND with above)
            matchLabels:
              app: frontend
  Means: pods with app=frontend IN namespaces with env=production

DIFFERENT `from` items = OR (match either):
  ingress:
    - from:
        - namespaceSelector:      ← Item 1
            matchLabels:
              env: production
        - podSelector:            ← Item 2 (OR with Item 1)
            matchLabels:
              app: frontend
  Means: ANY pod in production namespaces OR any frontend pod anywhere

THIS IS A VERY COMMON SECURITY MISTAKE!
Wrong AND/OR logic can accidentally open or block traffic.
```

### 10.5 Default Deny Patterns

```yaml
# Pattern 1: Default Deny ALL ingress (most common starting point)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}    # {} = applies to ALL pods in namespace
  policyTypes:
    - Ingress
  # No ingress rules = deny all ingress
---
# Pattern 2: Default Deny ALL (both directions)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  # No rules = deny all ingress AND egress
---
# Pattern 3: Allow same-namespace, deny cross-namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector: {}   # Any pod in SAME namespace (empty = all)
  egress:
    - to:
        - podSelector: {}   # Any pod in SAME namespace
    - to: []                # Allow DNS (required!)
      ports:
        - protocol: UDP
          port: 53
```

### 10.6 NetworkPolicy Rules Summary Table

| Scenario | Policy Configuration | Result |
|---|---|---|
| No policy selects a pod | N/A | Pod fully open (all traffic allowed) |
| Policy exists, no ingress rules | `policyTypes: [Ingress]` | All ingress BLOCKED |
| Policy exists, no egress rules | `policyTypes: [Egress]` | All egress BLOCKED |
| `ingress: [{from: []}]` | From = empty list | Allow from ALL (any IP) |
| `ingress: []` | Empty rules list | BLOCK all ingress |
| Multiple policies match same pod | N/A | Rules are UNIONED (OR) |

---

## 11. Lab: Limiting Pod Communication Using Network Policies {#11-lab-network-policies}

### 11.1 Setup — Create Lab Environment

```bash
# Create lab namespaces
kubectl create namespace netpol-app
kubectl create namespace netpol-external

# Label namespaces (needed for namespaceSelector in policies)
kubectl label namespace netpol-app \
  environment=lab \
  team=platform \
  kubernetes.io/metadata.name=netpol-app

kubectl label namespace netpol-external \
  environment=external \
  kubernetes.io/metadata.name=netpol-external

# Create application tier pods
cat << 'EOF' | kubectl apply -f -
---
apiVersion: v1
kind: Pod
metadata:
  name: frontend
  namespace: netpol-app
  labels:
    app: frontend
    tier: web
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: api-server
  namespace: netpol-app
  labels:
    app: api
    tier: backend
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: database
  namespace: netpol-app
  labels:
    app: database
    tier: data
spec:
  containers:
    - name: postgres
      image: postgres:14-alpine
      env:
        - name: POSTGRES_PASSWORD
          value: "testpass"
      ports:
        - containerPort: 5432
---
# Client from a different namespace (simulates cross-ns access)
apiVersion: v1
kind: Pod
metadata:
  name: external-client
  namespace: netpol-external
  labels:
    app: external-client
spec:
  containers:
    - name: curl
      image: curlimages/curl:7.87.0
      command: ["sleep", "3600"]
EOF

kubectl wait pod --all -n netpol-app --for=condition=Ready --timeout=90s
kubectl wait pod --all -n netpol-external --for=condition=Ready --timeout=60s

# Capture pod IPs for testing
API_IP=$(kubectl get pod api-server -n netpol-app \
  -o jsonpath='{.status.podIP}')
DB_IP=$(kubectl get pod database -n netpol-app \
  -o jsonpath='{.status.podIP}')
FRONTEND_IP=$(kubectl get pod frontend -n netpol-app \
  -o jsonpath='{.status.podIP}')

echo "API_IP=$API_IP | DB_IP=$DB_IP | FRONTEND_IP=$FRONTEND_IP"
```

### 11.2 Baseline Test — Verify Open Communication

```bash
echo "=== BEFORE NETWORK POLICIES (all open) ==="

# Frontend → API (should work)
kubectl exec frontend -n netpol-app -- \
  curl -s --connect-timeout 3 http://$API_IP:80 > /dev/null \
  && echo "✓ Frontend → API: CONNECTED" \
  || echo "✗ Frontend → API: BLOCKED"

# Frontend → Database (security risk — should PREVENT this)
kubectl exec frontend -n netpol-app -- \
  sh -c "nc -z -w3 $DB_IP 5432 2>&1" \
  && echo "✓ Frontend → Database: CONNECTED (SECURITY RISK!)" \
  || echo "✗ Frontend → Database: BLOCKED"

# External → API
kubectl exec external-client -n netpol-external -- \
  curl -s --connect-timeout 3 http://$API_IP:80 > /dev/null \
  && echo "✓ External → API: CONNECTED" \
  || echo "✗ External → API: BLOCKED"
```

### 11.3 Apply Default-Deny Policy

```bash
# Apply default-deny-all to netpol-app namespace
cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: netpol-app
spec:
  podSelector: {}    # All pods in namespace
  policyTypes:
    - Ingress
    - Egress
  # No rules = deny everything
EOF

echo ""
echo "=== AFTER DEFAULT-DENY (all blocked) ==="

# All of these should NOW FAIL/TIMEOUT
kubectl exec frontend -n netpol-app -- \
  curl -s --connect-timeout 3 http://$API_IP:80 > /dev/null \
  && echo "✓ Frontend → API: CONNECTED" \
  || echo "✗ Frontend → API: BLOCKED (expected)"

kubectl exec external-client -n netpol-external -- \
  curl -s --connect-timeout 3 http://$API_IP:80 > /dev/null \
  && echo "✓ External → API: CONNECTED" \
  || echo "✗ External → API: BLOCKED (expected)"
```

### 11.4 Allow Frontend → API Communication

```bash
# Policy on API pod (INGRESS): allow frontend to reach it
cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-api-ingress
  namespace: netpol-app
spec:
  podSelector:
    matchLabels:
      app: api       # Policy applies TO api pods
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend    # Allow FROM frontend pods
      ports:
        - protocol: TCP
          port: 80
EOF

# Policy on Frontend pod (EGRESS): allow it to reach api
cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-egress-to-api
  namespace: netpol-app
spec:
  podSelector:
    matchLabels:
      app: frontend    # Policy applies TO frontend pods
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: api    # Allow TO api pods
      ports:
        - protocol: TCP
          port: 80
    # CRITICAL: Allow DNS resolution without this, curl fails!
    - to: []
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
EOF

echo ""
echo "=== AFTER FRONTEND→API POLICY ==="

# Frontend → API: should work now
kubectl exec frontend -n netpol-app -- \
  curl -s --connect-timeout 5 http://$API_IP:80 > /dev/null \
  && echo "✓ Frontend → API: CONNECTED (as expected)" \
  || echo "✗ Frontend → API: BLOCKED (unexpected)"

# Frontend → Database: should still be BLOCKED
kubectl exec frontend -n netpol-app -- \
  sh -c "nc -z -w3 $DB_IP 5432 2>&1" \
  && echo "✓ Frontend → Database: CONNECTED (SECURITY RISK!)" \
  || echo "✗ Frontend → Database: BLOCKED (as expected — security working!)"
```

### 11.5 Allow API → Database Communication

```bash
# Policy on Database pod (INGRESS): allow api to reach it
cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db-ingress
  namespace: netpol-app
spec:
  podSelector:
    matchLabels:
      app: database    # Policy applies TO database pods
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: api    # Allow ONLY from api pods
      ports:
        - protocol: TCP
          port: 5432
---
# Policy on API pod (EGRESS): allow it to reach database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-egress-to-db
  namespace: netpol-app
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
    # Allow DNS
    - to: []
      ports:
        - protocol: UDP
          port: 53
EOF

echo ""
echo "=== AFTER API→DB POLICY ==="

# API → Database: should work
kubectl exec api-server -n netpol-app -- \
  sh -c "nc -z -w3 $DB_IP 5432 2>&1" \
  && echo "✓ API → Database: CONNECTED (as expected)" \
  || echo "✗ API → Database: BLOCKED (unexpected)"

# Frontend → Database: still blocked (no policy allows this)
kubectl exec frontend -n netpol-app -- \
  sh -c "nc -z -w3 $DB_IP 5432 2>&1" \
  && echo "✓ Frontend → Database: CONNECTED (SECURITY RISK!)" \
  || echo "✗ Frontend → Database: BLOCKED (as expected)"
```

### 11.6 Cross-Namespace Communication

```bash
# Allow external-client namespace to reach frontend in netpol-app
cat << 'EOF' | kubectl apply -f -
---
# Policy on frontend (INGRESS): allow from external namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-ns-to-frontend
  namespace: netpol-app
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Ingress
  ingress:
    - from:
        # AND condition: namespace with environment=external label
        # AND pod with app=external-client label
        - namespaceSelector:
            matchLabels:
              environment: external
          podSelector:
            matchLabels:
              app: external-client
      ports:
        - protocol: TCP
          port: 80
---
# Policy on external-client (EGRESS): allow to netpol-app namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-egress-to-app-ns
  namespace: netpol-external
spec:
  podSelector:
    matchLabels:
      app: external-client
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              environment: lab
          podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 80
    # Allow DNS
    - to: []
      ports:
        - protocol: UDP
          port: 53
EOF

echo ""
echo "=== AFTER CROSS-NAMESPACE POLICY ==="

# External → Frontend: should work now
kubectl exec external-client -n netpol-external -- \
  curl -s --connect-timeout 5 http://$FRONTEND_IP:80 > /dev/null \
  && echo "✓ External → Frontend: CONNECTED (as expected)" \
  || echo "✗ External → Frontend: BLOCKED (unexpected)"

# External → API: still blocked (no policy allows this)
kubectl exec external-client -n netpol-external -- \
  curl -s --connect-timeout 3 http://$API_IP:80 > /dev/null \
  && echo "✓ External → API: CONNECTED (SECURITY RISK!)" \
  || echo "✗ External → API: BLOCKED (as expected)"

# Cleanup
kubectl delete namespaces netpol-app netpol-external
```

### 11.7 Allow Monitoring Namespace to Scrape All Pods

```bash
# Create monitoring namespace
kubectl create namespace monitoring
kubectl label namespace monitoring purpose=monitoring

# Policy: Allow monitoring namespace to reach metrics ports on all pods
cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-scrape
  namespace: production    # Apply to production namespace
spec:
  podSelector: {}           # ALL pods in production
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              purpose: monitoring
      ports:
        - protocol: TCP
          port: 9090    # Prometheus metrics
        - protocol: TCP
          port: 8080    # Application metrics endpoint
EOF
```

---

## 12. Built-in Controllers Deep Dive {#12-built-in-controllers}

### 12.1 Namespace Controller

Runs inside `kube-controller-manager`. Manages the complete lifecycle of Namespace objects.

**Reconciliation behavior:**
```
Namespace ADDED  → Register child resource watchers; start tracking
Namespace DELETED (kubectl delete namespace X):
  1. API server: set namespace.status.phase = Terminating
  2. Namespace controller detects Terminating namespace
  3. Delete ALL namespaced resources in cascade:
     pods → replicasets → deployments → services → endpoints
     → configmaps → secrets → serviceaccounts → roles → rolebindings
     → pvcs → networkpolicies → resourcequotas → limitranges
  4. When all resources deleted → remove "kubernetes" finalizer
  5. etcd deletes namespace object
```

```bash
# Monitor namespace controller activity
kubectl get namespace my-ns -o yaml | grep -A 3 "status:"
# status:
#   phase: Terminating ← Controller is cleaning up

# Why namespaces get stuck:
# - Custom resources from deleted CRDs
# - Resources with custom finalizers
kubectl get namespace stuck-ns \
  -o jsonpath='{.spec.finalizers}'
```

### 12.2 ResourceQuota Controller

Tracks aggregate namespace resource usage and keeps `ResourceQuota.status.used` synchronized with reality.

```bash
# Controller reconciles on:
# - Any Pod/Service/PVC create/delete in the namespace
# - ResourceQuota object changes
# - Periodic resync (every 5 minutes)

# Monitor controller via metrics
curl -sk https://127.0.0.1:10257/metrics | \
  grep 'workqueue_depth{name="resourcequota_controller"}' | \
  grep -v "^#"
```

### 12.3 ReplicaSet Controller

The RS controller respects ResourceQuota. When it attempts to create pods that exceed quota, the API server admission plugin rejects the pod creation. The RS controller logs this as a `FailedCreate` event.

```bash
kubectl describe replicaset <rs-name> -n <namespace> | \
  grep -A 3 "FailedCreate"
# Warning  FailedCreate  5s  replicaset-controller
# Error creating: pods "xxx" is forbidden: exceeded quota
```

### 12.4 Deployment Controller

Deployment controller creates and manages ReplicaSets. Each namespace is managed independently — Deployments in `team-a-prod` and `team-b-prod` with the same name are completely separate objects with separate reconciliation.

```bash
# Deployment controller interaction with namespaces:
# - Each Deployment creates RS in same namespace
# - RS creates Pods in same namespace (quota checked)
# - Rolling updates respect namespace quota boundaries
kubectl get deployments --all-namespaces -o wide
```

### 12.5 Node Controller

Manages node health with direct impact on namespaces:
- Adds `not-ready:NoExecute` taint on unhealthy nodes → triggers pod eviction
- Evicted pods rescheduled elsewhere → must fit within namespace quota
- Node controller doesn't track quotas; quota enforcement is at API server admission

```bash
# Timeline when node fails:
# T+0:    Heartbeat stops
# T+40s:  node-monitor-grace-period → Ready=Unknown
# T+5m:   pod-eviction-timeout → NoExecute taint → pods evicted
# T+5m+:  Evicted pods rescheduled → must fit within quota on remaining nodes
```

### 12.6 Service Controller

Creates cloud load balancers for `LoadBalancer` type Services. ResourceQuota can limit `services.loadbalancers` count, preventing teams from creating excessive cloud infrastructure.

```bash
# Check Service controller behavior with quota
kubectl get service -n production --field-selector=spec.type=LoadBalancer
# If LoadBalancer quota exceeded: Service created but External IP never assigned
```

### 12.7 Job Controller

Job pods must fit within namespace quota. A Job that creates pods exceeding quota will have a `FailedCreate` event. The Job controller retries with exponential backoff.

```yaml
# Job with explicit resources (required when quota exists)
spec:
  template:
    spec:
      containers:
        - name: processor
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
```

### 12.8 StatefulSet Controller

StatefulSet pods in namespaces with compute quota MUST have resources. The LimitRange `defaultRequest` injection ensures this is automatic. StatefulSet rolling updates also validate that each new pod fits within quota before scaling the old pod down.

### 12.9 DaemonSet Controller

DaemonSet pods consume namespace quota when running in user namespaces. System DaemonSets in `kube-system` consume that namespace's quota (usually unrestricted). DaemonSets should be accounted for in namespace quota planning.

### 12.10 Garbage Collector

Uses `ownerReferences` to cascade deletions. During namespace deletion, the Namespace controller triggers deletion of top-level resources; the GC cascades to owned resources (RS deletes Pods, Deployment deletes RS, etc.). Both work together to ensure complete namespace cleanup.

### 12.11 PersistentVolume Controller

Binds PVCs to PVs. PVCs are namespace-scoped and counted by ResourceQuota (`persistentvolumeclaims` and `requests.storage`). PVs themselves are cluster-scoped and not namespace-governed. LimitRange controls the per-PVC storage size.

---

## 13. Internal Working Concepts — Informers, Work Queues & Reconciliation {#13-internal-concepts}

### 13.1 The Informer Pattern

Informers are the mechanism that allows controllers to receive resource change events without polling the API server on every cycle.

```
┌───────────────────────────────────────────────────────────────────────┐
│                    INFORMER ARCHITECTURE                              │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  Reflector                                                      │ │
│  │  Initial LIST:   GET /api/v1/pods (all pods, all namespaces)    │ │
│  │  Ongoing WATCH:  GET /api/v1/pods?watch=true&rv=XXXXX          │ │
│  │  Receives:       ADDED/MODIFIED/DELETED events as JSON stream   │ │
│  └──────────────────────────────┬──────────────────────────────────┘ │
│                                 │ Delta events                       │
│                                 ▼                                    │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  Local Cache (Indexer/Lister)                                   │ │
│  │  Thread-safe in-memory store of all watched objects             │ │
│  │  Controllers READ FROM HERE (not from API server per request)  │ │
│  │  Indexed by namespace, labels, etc. for fast lookups           │ │
│  └───────────────────────────┬─────────────────────────────────────┘ │
│                              │                                       │
│              OnAdd/OnUpdate/OnDelete event handlers                  │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  Work Queue (rate-limited, deduplicated)                        │ │
│  │  Keys: "namespace/resource-name"                                │ │
│  │  100 events for same object → 1 queue item (dedup)             │ │
│  │  Failed items: exponential backoff (5ms → 10ms → ... → 1000s)  │ │
│  └───────────────────────────┬─────────────────────────────────────┘ │
│                              │                                       │
│             ┌────────────────┴─────────────────┐                    │
│             ▼                                  ▼                    │
│       Worker 1 (goroutine)           Worker N (goroutine)           │
│       reconcile("ns/obj")            reconcile("ns/obj2")           │
│       → Read from local cache                                       │
│       → Make API calls if needed                                    │
│       → Update .status via API server                               │
└───────────────────────────────────────────────────────────────────────┘
```

### 13.2 ResourceQuota Informers in Detail

The ResourceQuota controller runs **three separate informers**:

```go
// Pseudocode representing the controller's informer setup
type ResourceQuotaController struct {
  // Tracks which quotas exist per namespace
  quotaInformer cache.SharedIndexInformer
  
  // Tracks pods (for CPU/memory usage calculation)
  podInformer cache.SharedIndexInformer
  
  // Tracks all other quota-countable objects
  // (services, secrets, configmaps, PVCs, etc.)
  objectInformers []cache.SharedIndexInformer
}

// On any resource change: enqueue the namespace for recalculation
func (c *ResourceQuotaController) enqueueNamespace(obj interface{}) {
    namespace := obj.(*v1.Pod).Namespace
    c.workQueue.Add(namespace)
    // The work queue deduplicates: 1000 pod events in same ns → 1 reconcile
}
```

### 13.3 NetworkPolicy Informers (CNI Plugin Side)

The CNI plugin (e.g., Calico, Cilium) on each node runs its own informers:

```
CNI Plugin per Node:
  ┌─────────────────────────────────────────────────────────────┐
  │ NetworkPolicy Informer → Programs iptables/eBPF when        │
  │                           policies change                   │
  │                                                             │
  │ Pod Informer           → Recalculates which policies apply  │
  │                           when pod IPs are assigned         │
  │                                                             │
  │ Namespace Informer     → Re-evaluates namespace selectors   │
  │                           when namespace labels change      │
  └─────────────────────────────────────────────────────────────┘
  
Result: iptables/eBPF rules updated within seconds of policy change
```

### 13.4 Work Queue Properties

```bash
# Key properties that ensure correctness and efficiency:

# 1. DEDUPLICATION
# 1000 Pod events in "production" namespace → 1 quota recalculation
# This prevents thundering herd from cascading pod creates/deletes

# 2. ORDERED (per key)
# A single key is processed sequentially (no concurrent reconciles for same key)
# Different keys processed concurrently by different workers

# 3. BACKOFF
# Failed reconciles: 5ms → 10ms → 20ms → ... → 1000s max
# Prevents hammering API server on persistent errors

# 4. RATE LIMITING
# Prevents flooding API server with updates

# Monitor work queue health
curl -sk https://127.0.0.1:10257/metrics | \
  grep 'workqueue' | grep -v "^#" | \
  grep -E "namespace|resourcequota"
```

---

## 14. API Server and etcd Interaction {#14-api-server-etcd}

### 14.1 The Fundamental Rule

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  CONTROLLERS NEVER TALK DIRECTLY TO etcd.                           ║
║                                                                      ║
║  ALL reads and writes flow:                                          ║
║  Controller → kube-apiserver → etcd                                 ║
║                                                                      ║
║  kube-apiserver provides for every request:                         ║
║  1. AuthN (who is this caller?)                                     ║
║  2. AuthZ (RBAC: are they allowed?)                                 ║
║  3. Admission (LimitRange injection, ResourceQuota enforcement)      ║
║  4. Validation (is the object schema valid?)                        ║
║  5. Audit logging (record what happened)                            ║
║  6. Watch cache (efficient fan-out to all informers)                ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 14.2 etcd Key Structure for Namespace Objects

```bash
# Namespace objects stored at:
# /registry/namespaces/<name>

# ResourceQuota:
# /registry/resourcequotas/<namespace>/<name>

# LimitRange:
# /registry/limitranges/<namespace>/<name>

# NetworkPolicy:
# /registry/networkpolicies/<namespace>/<name>

# Verify via etcdctl (on control plane node with access):
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl get /registry/namespaces --prefix --keys-only \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key 2>/dev/null | head -20
```

### 14.3 ResourceQuota Atomic Admission

```
ATOMIC QUOTA TRACKING (prevents race conditions):

Pod A creation (requests 500m CPU):
  1. Read ResourceQuota from API server watch cache
  2. Check: 7000m + 500m = 7500m ≤ 8000m → Allow
  3. Write pod to etcd AND update quota.status.used atomically
     Uses optimistic concurrency: if write fails (conflict) → retry from step 1

Pod B creation (simultaneously, also requests 500m CPU):
  1. Read same ResourceQuota (sees 7000m used)
  2. Check: 7000m + 500m = 7500m ≤ 8000m → Allow
  3. Try to write: conflict! (Pod A already updated to 7500m)
  4. Retry from step 1: 7500m + 500m = 8000m ≤ 8000m → Allow
  5. Write succeeds: quota.status.used = 8000m

This ensures no two pods together can exceed quota,
even under high concurrency.
```

---

## 15. Leader Election — HA for Namespace Infrastructure {#15-leader-election}

### 15.1 Why Leader Election Is Needed

In an HA control plane with 3 nodes, three instances of `kube-controller-manager` run simultaneously. Without coordination:
- Three instances might simultaneously try to cascade-delete a namespace
- Three instances might write conflicting quota status updates
- Double-processing wastes resources and could cause data corruption

**Leader election ensures exactly ONE instance is active at any time.**

### 15.2 How Leader Election Works — The Lease Object

```bash
# The Lease object is the "lock" in distributed leader election
kubectl get lease kube-controller-manager -n kube-system -o yaml
```

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  acquireTime: "2025-01-01T10:00:00.000000Z"
  holderIdentity: "k8s-cp-1_abc123-def456"  # Which instance is leader
  leaseDurationSeconds: 15                    # How long lease is valid
  leaseTransitions: 3                         # Leadership change count
  renewTime: "2025-01-01T14:30:00.123000Z"   # Last heartbeat time
```

```
LEADER ELECTION FLOW:
─────────────────────────────────────────────────────────────────────
Three instances start:
  k8s-cp-1, k8s-cp-2, k8s-cp-3

Race to acquire Lease:
  k8s-cp-1 wins → becomes leader (holderIdentity = k8s-cp-1)
  k8s-cp-2, k8s-cp-3 → standby (retry every 2 seconds)

Leader renews lease:
  Every renewDeadline (10s): update renewTime in Lease
  If renewal fails 3 times: step down, another instance wins

Failover (k8s-cp-1 dies):
  T+0:  k8s-cp-1 stops renewing
  T+15s: leaseDuration expires
  T+15-17s: k8s-cp-2 wins Lease (after retry period)
  T+17s: k8s-cp-2 becomes new leader

During gap (~15-17 seconds):
  → Existing Pods/Services/NetworkPolicies continue working
  → Namespace cascade deletions PAUSED
  → Quota status updates PAUSED (admission enforcement continues!)
  → After recovery: new leader re-syncs and resumes
```

### 15.3 Important Leader Election Flags

| Flag | Default | Component | Description |
|---|---|---|---|
| `--leader-elect` | `true` | Both | Enable leader election |
| `--leader-elect-lease-duration` | `15s` | Both | Lease validity window |
| `--leader-elect-renew-deadline` | `10s` | Both | Must renew before this |
| `--leader-elect-retry-period` | `2s` | Both | Standby polling interval |
| `--leader-elect-resource-lock` | `leases` | Both | Lock resource type |
| `--leader-elect-resource-namespace` | `kube-system` | Both | Lease namespace |

### 15.4 HA Best Practices

```bash
# Verify 3 controller-manager replicas (odd number for quorum)
kubectl get pods -n kube-system \
  -l component=kube-controller-manager

# Monitor lease stability (high transitions = control plane instability)
kubectl get lease kube-controller-manager -n kube-system \
  -o jsonpath='{.spec.leaseTransitions}'

# For large clusters: tune lease duration to reduce churn
# --leader-elect-lease-duration=30s
# --leader-elect-renew-deadline=25s
# (More stability, slower failover)
```

---

## 16. Performance Tuning {#16-performance-tuning}

### 16.1 kube-controller-manager Concurrency Flags

| Flag | Default | Recommendation | Impact |
|---|---|---|---|
| `--concurrent-namespace-syncs` | 10 | 10-20 | Parallel namespace reconciliations |
| `--concurrent-resource-quota-syncs` | 5 | 10-20 for busy clusters | Parallel quota recalculations |
| `--concurrent-endpoint-syncs` | 5 | 10-20 | Endpoint updates |
| `--kube-api-qps` | 20 | 50-100 | API server requests/sec |
| `--kube-api-burst` | 30 | 100-200 | API request burst |

### 16.2 API Server Watch Cache for Quota Objects

```yaml
# In kube-apiserver manifest — tune watch cache sizes
# --watch-cache-sizes=resourcequotas#200,limitranges#200,networkpolicies#500
# Increases cache size for frequently-accessed namespace governance objects
# Default cache: 100 versions per resource type
```

### 16.3 NetworkPolicy Performance by CNI

| CNI | Performance | Network Policy Engine | Scale |
|---|---|---|---|
| **Cilium (eBPF)** | Best | Kernel-level eBPF | 100k+ nodes |
| **Calico (eBPF mode)** | Excellent | Kernel eBPF | 10k+ nodes |
| **Calico (iptables)** | Good | iptables chains | 1k-5k nodes |
| **Weave** | Moderate | iptables | <500 nodes |
| **Flannel** | Fast (but no NetPol) | None — uses bridge | Any size |

```bash
# Check current kube-proxy mode (relevant for Service routing)
kubectl get configmap kube-proxy -n kube-system \
  -o jsonpath='{.data.config\.conf}' | grep mode

# For Cilium: check eBPF mode
kubectl exec -n kube-system ds/cilium -- \
  cilium status | grep "KubeProxy replacement"
```

### 16.4 LimitRange Performance

```bash
# LimitRange admission is O(1) per request — minimal overhead
# Best practice: max 2-3 LimitRange objects per namespace
# Too many LimitRange objects = multiple validation passes per request

# Check current LimitRanges per namespace
kubectl get limitranges --all-namespaces | awk '{print $1}' | sort | uniq -c
```

### 16.5 Resource Limits for Controller Manager

```yaml
# In /etc/kubernetes/manifests/kube-controller-manager.yaml
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 2000m     # Allow burst during namespace cascade deletions
    memory: 2Gi    # Higher for clusters with many namespaces and quotas
```

---

## 17. Security Hardening Practices {#17-security-hardening}

### 17.1 RBAC Per-Namespace

```yaml
# Namespace admin role (for team leads)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: namespace-admin
  namespace: team-a-production
rules:
  - apiGroups: ["apps", "extensions"]
    resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
    verbs: ["*"]
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/exec", "services", "configmaps", "serviceaccounts"]
    verbs: ["*"]
  # READ-ONLY on quota (cannot modify namespace budget)
  - apiGroups: [""]
    resources: ["resourcequotas", "limitranges"]
    verbs: ["get", "list", "watch"]
  # READ-ONLY on network policies (platform team manages these)
  - apiGroups: ["networking.k8s.io"]
    resources: ["networkpolicies"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-admin-binding
  namespace: team-a-production
subjects:
  - kind: Group
    name: "team-a-engineers"   # Maps to OIDC group
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: namespace-admin
  apiGroup: rbac.authorization.k8s.io
---
# Platform team ClusterRole: manage quotas and network policies cluster-wide
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-governance-manager
rules:
  - apiGroups: [""]
    resources: ["namespaces", "resourcequotas", "limitranges"]
    verbs: ["*"]
  - apiGroups: ["networking.k8s.io"]
    resources: ["networkpolicies"]
    verbs: ["*"]
```

### 17.2 ServiceAccount Security Per Namespace

```yaml
# Disable default ServiceAccount auto-mount for all namespaces
# Prevents applications from accidentally having API server access
apiVersion: v1
kind: ServiceAccount
metadata:
  name: default
  namespace: production
automountServiceAccountToken: false   # Disable by default

---
# Create dedicated SA per application (least privilege)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-server-sa
  namespace: production
automountServiceAccountToken: false   # Enable only in pod spec when needed

---
# Grant only specific permissions to the SA
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: api-server-role
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
    resourceNames: ["api-config"]   # Only specific named resources
```

### 17.3 TLS for All Internal Communication

```bash
# Verify API server uses TLS
kubectl get pod kube-apiserver-$(hostname) -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | \
  tr ' ' '\n' | grep -E "tls|cert"
# Should show: --tls-cert-file, --tls-private-key-file, --client-ca-file

# Verify minimum TLS version
kubectl get pod kube-apiserver-$(hostname) -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | \
  tr ' ' '\n' | grep tls-min-version
# Should show: --tls-min-version=VersionTLS12
```

### 17.4 Enforcing Default-Deny via Admission Webhook (Kyverno)

```yaml
# Automatically apply default-deny NetworkPolicy to every new namespace
# This requires Kyverno or OPA Gatekeeper installed
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-default-deny-network-policy
spec:
  rules:
    - name: add-default-deny
      match:
        resources:
          kinds:
            - Namespace
      generate:
        kind: NetworkPolicy
        name: default-deny-all
        namespace: "{{request.object.metadata.name}}"
        synchronize: true   # Recreate if deleted
        data:
          spec:
            podSelector: {}
            policyTypes:
              - Ingress
              - Egress
```

### 17.5 Pod Security Standards Per Namespace

```bash
# Apply Pod Security Standards to namespace (v1.25+)
# This controls what pods can do, not just resource consumption

kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# Levels:
# privileged  → No restrictions (system namespaces)
# baseline    → Prevents known privilege escalation
# restricted  → Hardened; requires non-root, no host namespaces, etc.

# Test a non-compliant pod
kubectl run test-pod -n production --image=nginx --privileged
# Warning: would violate PodSecurity "restricted:latest"
# (depends on policy type: warn/audit/enforce)
```

---

## 18. Monitoring and Observability {#18-monitoring}

### 18.1 Key Prometheus Metrics

| Metric | Type | Description | Alert Threshold |
|---|---|---|---|
| `kube_resourcequota_hard` | Gauge | Hard quota limits per resource | Dashboard |
| `kube_resourcequota_used` | Gauge | Current quota usage | >80% = warning |
| `kube_namespace_status_phase` | Gauge | Namespace phase (Active=1) | Terminating >10min |
| `kube_limitrange_item_limits` | Gauge | LimitRange constraints | Dashboard |
| `apiserver_admission_duration_seconds{name="ResourceQuota"}` | Histogram | Quota admission latency | p99 >100ms |
| `kube_pod_container_resource_requests` | Gauge | Per-pod resource requests | For capacity planning |
| `kube_networkpolicy_created_total` | Counter | Total NetworkPolicies | Growth tracking |

### 18.2 Quota Utilization Prometheus Queries

```promql
# CPU quota utilization percentage per namespace
(kube_resourcequota_used{resource="requests.cpu"}
  / kube_resourcequota_hard{resource="requests.cpu"}) * 100

# Memory quota utilization per namespace
(kube_resourcequota_used{resource="requests.memory"}
  / kube_resourcequota_hard{resource="requests.memory"}) * 100

# Pod count utilization per namespace
(kube_resourcequota_used{resource="pods"}
  / kube_resourcequota_hard{resource="pods"}) * 100

# Namespaces approaching CPU quota limit
kube_resourcequota_used{resource="requests.cpu"}
  / kube_resourcequota_hard{resource="requests.cpu"} > 0.8

# Total pod count by namespace (for chargeback)
sum by (namespace) (kube_pod_info)

# Namespaces with no NetworkPolicy (security gap)
count by (namespace) (kube_pod_info) unless on(namespace)
  (count by (namespace) (kube_networkpolicy_labels))
```

### 18.3 Alert Rules

```yaml
groups:
  - name: kubernetes-namespace-governance
    rules:
      # High quota utilization
      - alert: KubeNamespaceCPUQuotaHigh
        expr: |
          kube_resourcequota_used{resource="requests.cpu"}
          / kube_resourcequota_hard{resource="requests.cpu"} > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Namespace {{ $labels.namespace }} CPU quota > 85%"

      # Namespace stuck terminating
      - alert: KubeNamespaceTerminatingTooLong
        expr: |
          kube_namespace_status_phase{phase="Terminating"} == 1
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Namespace {{ $labels.namespace }} stuck in Terminating"

      # Quota admission latency spike
      - alert: KubeQuotaAdmissionLatencyHigh
        expr: |
          histogram_quantile(0.99,
            rate(apiserver_admission_controller_admission_duration_seconds_bucket{
              name="ResourceQuota"
            }[5m])
          ) > 0.1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "ResourceQuota admission p99 latency > 100ms"
```

### 18.4 Monitoring Commands

```bash
# View quota usage across all namespaces
kubectl get resourcequota --all-namespaces \
  -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,CPU_REQ:.status.used.requests\.cpu,CPU_LIMIT:.spec.hard.requests\.cpu"

# Find namespaces with no quota (ungoverned)
kubectl get namespaces -o jsonpath='{.items[*].metadata.name}' | \
  tr ' ' '\n' | while read ns; do
    count=$(kubectl get resourcequota -n $ns \
      --no-headers 2>/dev/null | wc -l)
    [ "$count" -eq 0 ] && echo "NO QUOTA: $ns"
  done

# Find namespaces with no NetworkPolicy
kubectl get namespaces -o jsonpath='{.items[*].metadata.name}' | \
  tr ' ' '\n' | while read ns; do
    count=$(kubectl get networkpolicies -n $ns \
      --no-headers 2>/dev/null | wc -l)
    [ "$count" -eq 0 ] && echo "NO NETPOL: $ns"
  done
```

---

## 19. Troubleshooting Namespace, Quota, and Network Issues {#19-troubleshooting}

### 19.1 Pods Not Created — Quota Exceeded

```bash
# STEP 1: Identify quota rejection
kubectl describe replicaset <rs-name> -n <namespace>
# Warning  FailedCreate  5s  replicaset-controller
# Error creating: pods "xxx" is forbidden: exceeded quota

# STEP 2: Check current quota usage
kubectl describe resourcequota -n <namespace>
# Resource       Used    Hard
# requests.cpu   7800m   8000m  ← Nearly full!

# STEP 3: Find what's consuming quota
kubectl get pods -n <namespace> \
  -o custom-columns="NAME:.metadata.name,CPU_REQ:.spec.containers[0].resources.requests.cpu"

# STEP 4: Fix options
# Option A: Reduce resource requests on existing deployments
kubectl set resources deployment/my-app \
  --requests=cpu=100m,memory=128Mi \
  -n <namespace>

# Option B: Scale down non-critical deployments
kubectl scale deployment/batch-job --replicas=1 -n <namespace>

# Option C: Request quota increase (platform team increases quota)
kubectl patch resourcequota namespace-quota -n <namespace> \
  --type merge \
  -p '{"spec":{"hard":{"requests.cpu":"12"}}}'
```

### 19.2 Pods Not Created — LimitRange Violation

```bash
# Check LimitRange constraints
kubectl describe limitrange -n <namespace>

# Get specific error
kubectl get events -n <namespace> \
  --field-selector reason=FailedCreate \
  --sort-by='.lastTimestamp'
# maximum cpu usage per Container is 2, but limit is 4

# Fix: Update the deployment's resource spec
kubectl patch deployment <name> -n <namespace> \
  --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/resources/limits/cpu","value":"2"}]'
```

### 19.3 Deployment Stuck — Quota or Node Issues

```bash
# Check overall deployment status
kubectl rollout status deployment/<n> -n <namespace> --timeout=60s

# Check why pods are pending
kubectl get pods -n <namespace> --field-selector=status.phase=Pending
kubectl describe pod <pending-pod> -n <namespace>
# Events section shows: FailedScheduling, OOMKilled, etc.

# Check if it's quota vs node resources
kubectl describe resourcequota -n <namespace>   # Quota issue?
kubectl top nodes                                 # Node resources?
kubectl describe nodes | grep -A 8 "Allocated resources:"

# Check if LimitRange is blocking the deployment
kubectl get limitrange -n <namespace> -o yaml | grep max -A 5
```

### 19.4 Node NotReady — Impact on Namespaced Workloads

```bash
# Identify NotReady nodes
kubectl get nodes | grep NotReady

# Find pods on NotReady node
kubectl get pods --all-namespaces \
  --field-selector spec.nodeName=<notready-node>

# Check if replacement pods are trying to schedule
kubectl get pods --all-namespaces --field-selector=status.phase=Pending

# Check if quota is blocking replacement pods
kubectl describe resourcequota --all-namespaces | \
  grep -A 5 "Namespace:"

# Cordon and drain the node
kubectl cordon <notready-node>
kubectl drain <notready-node> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --timeout=300s
kubectl uncordon <notready-node>  # After fix
```

### 19.5 NetworkPolicy Blocking Expected Traffic

```bash
# STEP 1: Confirm it's a NetworkPolicy issue
kubectl exec pod-a -n ns-a -- \
  curl --connect-timeout 3 http://$POD_B_IP:8080
# curl: (28) Connection timed out → likely NetworkPolicy issue

# STEP 2: List policies affecting the target pod
kubectl get networkpolicies -n ns-b
kubectl describe networkpolicy <policy-name> -n ns-b

# STEP 3: Check namespace labels (needed for namespaceSelector)
kubectl get namespace ns-a --show-labels
# Compare with namespaceSelector matchLabels in the policy

# STEP 4: Verify CNI supports NetworkPolicy
kubectl get pods -n kube-system | grep -E "calico|cilium|weave"
# If only flannel: NetworkPolicy NOT enforced!

# STEP 5: Add the missing allow rule
cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-pod-a-to-pod-b
  namespace: ns-b      # Policy on RECEIVING end
spec:
  podSelector:
    matchLabels:
      app: pod-b
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ns-a  # Use this standard label!
          podSelector:
            matchLabels:
              app: pod-a
      ports:
        - port: 8080
EOF

# STEP 6: Also ensure egress is allowed from pod-a
cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-pod-a-egress
  namespace: ns-a       # Policy on SENDING end
spec:
  podSelector:
    matchLabels:
      app: pod-a
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ns-b
          podSelector:
            matchLabels:
              app: pod-b
      ports:
        - port: 8080
    - to: []
      ports:
        - protocol: UDP
          port: 53   # Always allow DNS!
EOF
```

### 19.6 Namespace Stuck in Terminating

```bash
# Find what's blocking namespace deletion
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -I{} sh -c "kubectl get {} -n stuck-ns 2>/dev/null | \
    grep -v 'No resources found' | head -1 && echo 'Found in: {}'"

# Find resources with finalizers in the namespace
kubectl get pods -n stuck-ns -o json | \
  jq '.items[] | select(.metadata.finalizers != null) |
      {name: .metadata.name, finalizers: .metadata.finalizers}'

# Check namespace finalizers
kubectl get namespace stuck-ns \
  -o jsonpath='{.spec.finalizers}'
# ["kubernetes"] = namespace controller hasn't finished

# Remove all pod finalizers in the namespace
kubectl get pods -n stuck-ns -o name | \
  xargs -I{} kubectl patch {} -n stuck-ns \
  -p '{"metadata":{"finalizers":[]}}' \
  --type=merge

# If namespace is truly stuck (LAST RESORT — data loss risk):
kubectl get namespace stuck-ns -o json | \
  jq 'del(.spec.finalizers)' | \
  kubectl replace --raw \
    /api/v1/namespaces/stuck-ns/finalize \
    -f -
```

---

## 20. Disaster Recovery Concepts {#20-disaster-recovery}

### 20.1 Stateless Controllers

```
kube-controller-manager (including Namespace and ResourceQuota controllers):
  IS COMPLETELY STATELESS — all state lives in etcd

If controller-manager crashes or leader changes:
  
  Namespace deletions IN PROGRESS: PAUSED
    → After recovery: controller re-lists Terminating namespaces
    → Resumes cascade deletion automatically
    → All partial deletions are idempotent (delete already-deleted = no-op)
  
  ResourceQuota status: may DRIFT during outage
    → Admission enforcement NEVER stops (it's in the API server)
    → Quota status updates paused during controller outage
    → After recovery: full resync recalculates actual usage
    → Any new pods created during outage have correct quota checked by admission
  
  NetworkPolicy enforcement: UNAFFECTED
    → CNI plugins run on nodes independently
    → iptables/eBPF rules stay in place until policy changes
    → No controller needed to maintain enforcement
```

### 20.2 etcd Backup — What It Contains

```bash
# etcd backup contains ALL namespace-related objects:
# - Namespace definitions (including labels for NetworkPolicy)
# - ResourceQuota objects (spec and status)
# - LimitRange objects
# - NetworkPolicy objects
# - RBAC (roles, bindings) per namespace
# - All namespaced resources (pods, deployments, services, etc.)

# Backup etcd (on control plane node)
ETCDCTL="etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key"

kubectl exec -n kube-system etcd-$(hostname) -- \
  $ETCDCTL snapshot save /tmp/etcd-snapshot-$(date +%Y%m%d).db

# Verify backup
kubectl exec -n kube-system etcd-$(hostname) -- \
  $ETCDCTL snapshot status /tmp/etcd-snapshot-$(date +%Y%m%d).db \
  --write-out=table
```

### 20.3 GitOps as Disaster Recovery

```bash
# Store ALL namespace governance config in Git:
# cluster-config/
# ├── namespaces/
# │   ├── team-a-production/
# │   │   ├── namespace.yaml
# │   │   ├── resourcequota.yaml
# │   │   ├── limitrange.yaml
# │   │   ├── networkpolicies/
# │   │   │   ├── default-deny-all.yaml
# │   │   │   ├── allow-frontend-to-api.yaml
# │   │   │   └── allow-api-to-db.yaml
# │   │   └── rbac/
# │   │       ├── role.yaml
# │   │       └── rolebinding.yaml

# Recovery from cluster failure:
git clone https://github.com/company/cluster-config.git
kubectl apply -f cluster-config/ --recursive
# All namespaces, quotas, limitranges, network policies restored!

# CNI plugin picks up NetworkPolicy objects via watch
# → iptables/eBPF rules programmed within seconds
```

---

## 21. Comparison — kube-apiserver vs kube-scheduler vs kube-controller-manager {#21-comparison}

| Dimension | kube-apiserver | kube-scheduler | kube-controller-manager |
|---|---|---|---|
| **Namespace role** | Stores all namespace objects; validates quota/limits via admission | Uses namespace for pod placement context | Namespace controller manages lifecycle |
| **Quota role** | Stores ResourceQuota; **ENFORCES** via admission plugin | N/A | **TRACKS** status.used via reconciliation |
| **LimitRange role** | Stores LimitRange; **INJECTS** defaults via admission | Reads injected values for scheduling decisions | N/A |
| **NetworkPolicy role** | Stores NetworkPolicy objects; serves to CNI watch streams | N/A | N/A |
| **etcd access** | YES (direct, exclusive) | NO (via API server informers) | NO (via API server informers) |
| **HA model** | Active-Active (all serve) | Active-Passive (leader) | Active-Passive (leader) |
| **Failure impact (namespace)** | All operations fail; no new resources | Pods not scheduled | Namespace GC paused; quota status stale |
| **NetworkPolicy enforcement** | N/A | N/A | N/A (done by CNI on nodes) |
| **Port** | 6443 | 10259 | 10257 |
| **Recovery time** | Immediate (LB failover) | ~30s (leader election) | ~30s (leader election) |

---

## 22. ASCII Architecture Diagram {#22-ascii-diagram}

```
╔════════════════════════════════════════════════════════════════════════════════════╗
║    KUBERNETES NAMESPACES, QUOTAS, LIMITRANGES & NETWORK POLICIES                  ║
╠════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                    ║
║  ADMINISTRATOR / PLATFORM TEAM                                                     ║
║  kubectl create namespace team-a-prod                                              ║
║  kubectl apply -f resourcequota.yaml limitrange.yaml networkpolicy.yaml           ║
║           │                                                                        ║
║           │ HTTPS :6443                                                            ║
║           ▼                                                                        ║
║  ┌──────────────────────────────────────────────────────────────────────────────┐  ║
║  │                     kube-apiserver :6443                                     │  ║
║  │                                                                              │  ║
║  │  ADMISSION PIPELINE (synchronous, per-request):                             │  ║
║  │  ┌─────────────────────────────────────────────────────────────────────┐   │  ║
║  │  │  1. LimitRanger (FIRST)                                             │   │  ║
║  │  │     → Inject default CPU/memory if container has none              │   │  ║
║  │  │     → Validate against min/max/ratio constraints                   │   │  ║
║  │  │     → REJECT if outside bounds                                      │   │  ║
║  │  │                                                                     │   │  ║
║  │  │  2. ResourceQuota (SECOND — after LimitRange injects defaults)      │   │  ║
║  │  │     → Check: current_used + new_request ≤ quota.hard               │   │  ║
║  │  │     → If over quota: REJECT (403 Forbidden)                        │   │  ║
║  │  │     → If within quota: ALLOW + atomically update status.used        │   │  ║
║  │  │                                                                     │   │  ║
║  │  │  3. PodSecurity Admission (namespace-label-based)                  │   │  ║
║  │  │     → Enforce restricted/baseline/privileged per namespace         │   │  ║
║  │  └─────────────────────────────────────────────────────────────────────┘   │  ║
║  │                                                                              │  ║
║  │  STORES in etcd:   /registry/namespaces/<n>                                 │  ║
║  │                    /registry/resourcequotas/<ns>/<n>                        │  ║
║  │                    /registry/limitranges/<ns>/<n>                           │  ║
║  │                    /registry/networkpolicies/<ns>/<n>                       │  ║
║  └─────────────────────────────────┬────────────────────────────────────────────┘  ║
║                                    │ Watch events to controllers and CNI           ║
║       ┌────────────────────────────┼────────────────────────────────────┐          ║
║       │                            │                                    │          ║
║  ┌────▼────────────────┐  ┌────────▼──────────────────────┐  ┌─────────▼──────┐   ║
║  │     etcd :2379       │  │  kube-controller-manager      │  │  CNI per Node  │   ║
║  │                      │  │  :10257 (Leader Elected)      │  │ (Calico/Cilium)│   ║
║  │  /registry/          │  │                               │  │                │   ║
║  │  ├── namespaces      │  │  Namespace Controller:        │  │  NetworkPolicy │   ║
║  │  ├── resourcequotas  │  │  → Cascade delete on termination│  │  Watch:        │   ║
║  │  ├── limitranges     │  │  → Remove kubernetes finalizer│  │  → List/Watch  │   ║
║  │  └── networkpolicies │  │                               │  │    policies    │   ║
║  │                      │  │  ResourceQuota Controller:    │  │  → Program     │   ║
║  │  Only kube-apiserver │  │  → Reconcile status.used      │  │    iptables/   │   ║
║  │  accesses etcd       │  │  → Full resync every 5 min    │  │    eBPF rules  │   ║
║  │  directly            │  │                               │  │                │   ║
║  │                      │  │  NEVER talk to etcd directly! │  │  Enforces on   │   ║
║  └──────────────────────┘  └───────────────────────────────┘  │  each node     │   ║
║                                                                └────────────────┘   ║
║                                                                                    ║
║  NAMESPACE ISOLATION MODEL:                                                        ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐   ║
║  │  namespace: team-a-production                                               │   ║
║  │                                                                             │   ║
║  │  ResourceQuota: requests.cpu=4, limits.cpu=8, requests.memory=4Gi          │   ║
║  │  LimitRange: defaultRequest=100m/128Mi, default=200m/256Mi, max=2/2Gi      │   ║
║  │  RBAC: team-a-engineers = Role:admin   |  others = no-access               │   ║
║  │  NetworkPolicy: default-deny-all + selective allow rules                   │   ║
║  │                                                                             │   ║
║  │  ┌──────────┐  ─allow(80)──►  ┌────────────┐  ─allow(5432)►  ┌──────────┐  │   ║
║  │  │ frontend │                 │ api-server  │                 │ database │  │   ║
║  │  │ pod      │  ◄──deny(all)── │ pod         │  ◄──deny(all)── │ pod      │  │   ║
║  │  └──────────┘                 └────────────┘                  └──────────┘  │   ║
║  │  LimitRange injects: cpu=100m req / 200m limit if not specified             │   ║
║  │  ResourceQuota tracks: total namespace CPU/memory usage                     │   ║
║  └─────────────────────────────────────────────────────────────────────────────┘   ║
╚════════════════════════════════════════════════════════════════════════════════════╝
```

---

## 23. Real-World Production Use Cases {#23-production-use-cases}

### 23.1 Enterprise Multi-Team Platform (50+ Teams)

```yaml
# Platform pattern: Namespace per team × environment
# Each namespace automatically gets governance via GitOps + Kyverno

# template: namespaces/team-template.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: "{{ .team }}-{{ .environment }}"
  labels:
    team: "{{ .team }}"
    environment: "{{ .environment }}"
    cost-center: "{{ .cost_center }}"
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
  namespace: "{{ .team }}-{{ .environment }}"
spec:
  hard:
    requests.cpu: "{{ .cpu_requests }}"
    limits.cpu: "{{ .cpu_limits }}"
    requests.memory: "{{ .memory_requests }}"
    limits.memory: "{{ .memory_limits }}"
    pods: "{{ .max_pods }}"
    services: "{{ .max_services }}"
    services.loadbalancers: "{{ .max_loadbalancers }}"
```

### 23.2 PCI-DSS Compliant Payment Processing

```yaml
# Payment namespace: maximum isolation for cardholder data

# Strict NetworkPolicy — only specific flows allowed
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-isolation
  namespace: payment-processing
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Only from API gateway namespace
    - from:
        - namespaceSelector:
            matchLabels:
              role: api-gateway
      ports:
        - port: 8443
  egress:
    # Only to payment processor external IP
    - to:
        - ipBlock:
            cidr: 203.0.113.10/32
      ports:
        - port: 443
    # Allow DNS
    - to: []
      ports:
        - protocol: UDP
          port: 53
```

### 23.3 Development Cluster with Per-Developer Namespaces

```yaml
# Each developer gets their own sandbox namespace
# Small quota prevents resource hogging by any one dev

apiVersion: v1
kind: ResourceQuota
metadata:
  name: developer-sandbox-quota
  namespace: "dev-{{ .developer_name }}"
spec:
  hard:
    requests.cpu: "2"
    limits.cpu: "4"
    requests.memory: 4Gi
    limits.memory: 8Gi
    pods: "10"
    services: "5"
    services.loadbalancers: "0"    # No cloud LBs for dev!
    services.nodeports: "0"
```

---

## 24. Best Practices for Production Environments {#24-best-practices}

### 24.1 Namespace Design

- **Never use `default` namespace for production** — always use purpose-named, governed namespaces
- **Add labels from day one** — team, environment, cost-center; these enable monitoring and NetworkPolicy
- **Use GitOps for namespace lifecycle** — namespace + quota + limitrange + networkpolicy all in Git
- **Annotate with operational metadata** — contact info, Slack channel, created-by, purpose
- **Automate namespace provisioning** — use templates (Helm, Kustomize) or Kyverno generators

### 24.2 ResourceQuota Best Practices

- **Deploy LimitRange BEFORE ResourceQuota** — ensures pods without resources can pass quota admission
- **Set both compute and object count limits** — CPU/memory alone doesn't prevent LoadBalancer sprawl
- **Start conservative, increase as needed** — easier to increase budget than explain wasted resources
- **Alert at 80% utilization** — leave headroom for rolling update surge
- **Include storage quotas** — unlimited PVC creation can exhaust storage backend

### 24.3 LimitRange Best Practices

- **Always set defaultRequest** — ensures all pods have scheduler-usable resource requests
- **Set maxLimitRequestRatio** — prevents burstable containers from starving others
- **Include PVC limits** — prevent runaway storage requests
- **Keep max low enough to prevent OOM node conditions** — max container should be < node allocatable / min replicas

### 24.4 NetworkPolicy Best Practices

- **Default-deny in every namespace** — applies to all new namespaces via automation
- **Always allow DNS (port 53)** — the most commonly forgotten rule; breaks everything if missing
- **Use AND logic (same `from` item) for cross-namespace** — most secure option
- **Use namespace `kubernetes.io/metadata.name` label** — automatically added, unique, reliable
- **Test policies in staging** — same policies as production; catch issues early
- **Document every policy** — use annotations to explain purpose and allowed traffic flows

---

## 25. Common Mistakes and Pitfalls {#25-mistakes-pitfalls}

### 25.1 ResourceQuota Without LimitRange — Pod Rejection

**Mistake:** Creating ResourceQuota (with compute resources) without LimitRange defaults.  
**Impact:** Every pod without explicit resources is rejected: "must specify requests.cpu"  
**Fix:** Deploy LimitRange with `defaultRequest` and `default` in the same apply as ResourceQuota.

---

### 25.2 Forgetting DNS in Default-Deny NetworkPolicy

**Mistake:** Default-deny-all egress without allowing DNS (port 53 UDP/TCP).  
**Impact:** All DNS lookups fail → service discovery broken → application cannot connect to anything by name.  
**Fix:** ALWAYS add DNS egress rule:
```yaml
egress:
  - to: []
    ports:
      - protocol: UDP
        port: 53
      - protocol: TCP
        port: 53
```

---

### 25.3 Flannel Without NetworkPolicy Support

**Mistake:** Applying NetworkPolicy on a Flannel-only cluster without adding Calico/Cilium.  
**Impact:** Policies stored in etcd, appear to be applied, but NO enforcement occurs. All traffic flows freely despite policies showing as "applied".  
**Fix:** Verify: `kubectl get pods -n kube-system | grep -E "calico|cilium|weave"`. Add Calico CNI overlay or migrate to Cilium.

---

### 25.4 AND vs OR Confusion in NetworkPolicy from/to

**Mistake:**
```yaml
# INTENDED: Allow from pods in "production" namespace with app=frontend
from:
  - namespaceSelector:
      matchLabels:
        env: production
  - podSelector:              ← SEPARATE item = OR, not AND!
      matchLabels:
        app: frontend
# ACTUAL EFFECT: Allow from ANY pod in production ns
#                OR any pod anywhere with app=frontend label
```
**Fix:** Use same `from` item (AND):
```yaml
from:
  - namespaceSelector:
      matchLabels:
        env: production
    podSelector:               ← SAME item = AND
      matchLabels:
        app: frontend
```

---

### 25.5 Deleting a Namespace With Active Traffic

**Mistake:** `kubectl delete namespace production` while the application is serving traffic.  
**Impact:** ALL resources immediately scheduled for cascade deletion. Service outage.  
**Fix:** Before deleting any namespace: (1) Scale all deployments to 0, (2) Wait for pods to terminate, (3) Verify no traffic, (4) Delete namespace.

---

### 25.6 Missing Namespace Labels for NetworkPolicy

**Mistake:** NetworkPolicy uses `namespaceSelector.matchLabels: {environment: production}` but the namespace doesn't have that label.  
**Impact:** Policy allows/blocks wrong traffic. Traffic that should flow is blocked; traffic that should be blocked flows.  
**Fix:** Use `kubernetes.io/metadata.name: <ns-name>` label which is automatically added and reliable:
```yaml
namespaceSelector:
  matchLabels:
    kubernetes.io/metadata.name: team-a-production
```

---

### 25.7 ResourceQuota Counting Terminating Pods

**Mistake:** Not accounting for pods in Terminating state during rolling updates.  
**Impact:** During rolling update: old pods terminating + new pods starting = temporarily double the pod count → quota exceeded → rollout stalls.  
**Fix:** Set quota high enough to allow surge: `pods: "desired_replicas * 1.5"`. Or use `maxSurge: 0` in rolling update strategy.

---

## 26. Interview Questions — Beginner to Advanced {#26-interview-questions}

### Beginner Level

**Q1: What are Kubernetes Namespaces and why are they used?**

**A:** Namespaces are logical partitions within a Kubernetes cluster that create virtual isolated environments. They're used for:
1. **Name scoping** — same resource names can exist in different namespaces
2. **Access control** — RBAC applies per-namespace, so teams only access their namespace
3. **Resource governance** — ResourceQuota limits total consumption per namespace
4. **Cost tracking** — namespace labels enable per-team billing/chargeback
5. **Policy scoping** — LimitRange and NetworkPolicy apply per-namespace

The four system namespaces are: `default`, `kube-system`, `kube-public`, and `kube-node-lease`.

---

**Q2: What is the difference between ResourceQuota and LimitRange?**

**A:** These are complementary but distinct:

- **ResourceQuota** enforces **aggregate/total** limits for a namespace. Example: "All pods combined can use at most 8 CPU cores." It's the team's total budget.
- **LimitRange** enforces **per-object** constraints. Example: "Each individual container can use at most 2 CPU cores" or "If a container doesn't specify resources, inject 100m CPU as default."

They work together: LimitRange injects defaults so containers pass ResourceQuota admission (which requires resource specs to calculate usage). Deploy LimitRange BEFORE ResourceQuota.

---

**Q3: What happens when you apply a default-deny NetworkPolicy to a namespace?**

**A:** Once any NetworkPolicy selects a pod (including empty `podSelector: {}`), all traffic NOT explicitly allowed is denied. A `default-deny-all` policy with `podSelector: {}` selects ALL pods in the namespace, causing:
- All ingress traffic blocked to all pods
- All egress traffic blocked from all pods
- Pods cannot resolve DNS (also blocked!)

After applying, you must add explicit allow policies for each legitimate traffic flow, including a DNS egress allow rule (port 53 UDP/TCP).

**Important**: This only works if your CNI plugin supports NetworkPolicy (Calico, Cilium, etc.). Flannel does NOT enforce NetworkPolicy without Calico overlay.

---

### Intermediate Level

**Q4: Explain the two-part quota system in Kubernetes — admission enforcement vs controller reconciliation.**

**A:** Kubernetes handles ResourceQuota with two independent systems:

**Admission Enforcement** (in kube-apiserver, synchronous):
- Runs inline for every pod/service/PVC creation request
- Checks: `current_used + new_request ≤ quota.hard`
- Uses optimistic concurrency for atomic updates
- If over limit: 403 Forbidden immediately
- This is the real enforcement — it never misses a request

**ResourceQuota Controller** (in kube-controller-manager, async):
- Periodically re-calculates actual usage from all pods/services/etc.
- Updates `quota.status.used` to match reality
- Runs every 5 minutes plus on resource changes
- This reconciles any drift between status and reality

The separation exists because admission must be synchronous (block requests), while status tracking can be eventually consistent. Admission provides accuracy; the controller provides observability and drift correction.

---

**Q5: What are the critical differences between `from: [{podSelector: {}, namespaceSelector: {}}]` and `from: [{podSelector: {}}, {namespaceSelector: {}}]` in a NetworkPolicy?**

**A:** This is one of the most dangerous gotchas in NetworkPolicy:

**Same item (AND logic)**:
```yaml
from:
  - podSelector: {}         # AND
    namespaceSelector: {}   # same item = AND
# Means: pods matching BOTH selectors (all pods in all namespaces)
```

**Separate items (OR logic)**:
```yaml
from:
  - podSelector: {}         # OR
  - namespaceSelector: {}   # separate item = OR
# Means: any pod in same namespace (podSelector: {}) 
#        OR any pod anywhere (namespaceSelector: {})
# EFFECTIVELY: allow from everywhere!
```

For cross-namespace access with pod restrictions, use the AND form (same item) to ensure you're selecting pods with specific labels FROM a specific namespace, not broadly.

---

### Advanced Level

**Q6: A namespace has ResourceQuota with `requests.cpu: 8` configured, and the current usage is at 7.8 cores. A Deployment scales from 5 to 10 replicas simultaneously. What happens, and why?**

**A:** The scaling will PARTIALLY succeed. Here's the detailed flow:

1. Deployment controller creates a new ReplicaSet (or updates existing), requesting 5 more pods
2. ReplicaSet controller tries to create 5 pods simultaneously
3. For each pod creation, the ResourceQuota admission plugin:
   - Reads current `quota.status.used` (7.8 cores with some requests value)
   - Calculates if new pod fits: 7.8 + pod_cpu_request ≤ 8
4. The API server uses **optimistic concurrency** — all 5 pod requests race simultaneously
5. Some pods succeed (those processed before quota fills), others get 403 Forbidden
6. ReplicaSet controller receives `FailedCreate` events for rejected pods
7. RS retries failed creations with exponential backoff

**Result**: Some pods are created (filling the remaining ~0.2 core), others stay in `FailedCreate` state. The RS shows `3/10` or similar. The Deployment appears partially scaled.

**Fix**: Increase the quota, or scale in steps with sufficient quota headroom. Also ensure the LimitRange defaults match your pod requests to accurately predict quota consumption.

---

**Q7: Explain how NetworkPolicy enforcement propagates through the cluster when a new NetworkPolicy is created. What happens step-by-step?**

**A:** The enforcement chain is:

1. `kubectl apply -f networkpolicy.yaml` → PATCH/POST to kube-apiserver (port 6443)
2. API server validates (is it syntactically correct?), stores in etcd at `/registry/networkpolicies/<ns>/<name>`
3. **API server watch cache** notified → sends MODIFIED event to all watchers
4. **CNI plugin** (Calico/Cilium/etc.) running on each node has an informer watching NetworkPolicy objects
5. Each node's CNI controller receives the `ADDED` watch event for the new policy
6. CNI controller reconciles: What pods on this node are selected by `podSelector`?
7. For each matching pod, CNI calculates new iptables/eBPF rules:
   - Which source IPs are in `ingress.from.podSelector`? (look up those pods' IPs)
   - Which destination IPs match `egress.to.podSelector`?
8. iptables rules (or eBPF programs) updated on each node
9. Traffic enforcement begins — typically within 1-5 seconds of the kubectl apply

**Important nuance**: If a pod is newly created AFTER the policy exists, the CNI controller reacts to the pod's `ADDED` event and immediately programs rules for it. If namespace labels change (affecting `namespaceSelector`), the CNI controller reacts to the namespace `MODIFIED` event and re-evaluates all relevant policies.

---

## 27. Cheat Sheet — Commands, Flags & Manifests {#27-cheat-sheet}

### 27.1 Namespace Commands

```bash
# Create
kubectl create namespace <name>
kubectl apply -f namespace.yaml

# Read
kubectl get namespaces
kubectl get ns -l team=platform         # Filter by label
kubectl describe namespace <name>
kubectl get namespace <name> -o yaml

# Update (add labels/annotations)
kubectl label namespace <name> environment=production
kubectl annotate namespace <name> contact="team@company.com"

# Delete (CASCADES!)
kubectl delete namespace <name>

# Context management
kubectl config set-context --current --namespace=<name>
kubectl config view --minify | grep namespace
kubens <name>          # requires kubectx package

# List all resources in a namespace
kubectl get all -n <name>
```

### 27.2 ResourceQuota Commands

```bash
# Create
kubectl create quota <name> \
  --hard=cpu=4,memory=8Gi,pods=20 \
  -n <namespace>
kubectl apply -f resourcequota.yaml

# View
kubectl get resourcequota -n <namespace>
kubectl describe resourcequota <name> -n <namespace>
# Shows: Resource, Used, Hard columns

# Update
kubectl patch resourcequota <name> -n <namespace> \
  --type merge \
  -p '{"spec":{"hard":{"requests.cpu":"8"}}}'

# Monitor utilization
kubectl get resourcequota --all-namespaces \
  -o custom-columns="NS:.metadata.namespace,\
  NAME:.metadata.name,\
  CPU_USED:.status.used.requests\.cpu,\
  CPU_HARD:.spec.hard.requests\.cpu"
```

### 27.3 LimitRange Commands

```bash
# Create
kubectl apply -f limitrange.yaml

# View
kubectl get limitranges -n <namespace>
kubectl describe limitrange <name> -n <namespace>

# Verify injection on pod
kubectl get pod <name> -n <ns> \
  -o jsonpath='{.spec.containers[0].resources}' | jq

# Check if pod was auto-injected with defaults
kubectl get pod <name> -o yaml | grep -A 10 "resources:"
```

### 27.4 NetworkPolicy Commands

```bash
# Create
kubectl apply -f networkpolicy.yaml

# View
kubectl get networkpolicies -n <namespace>
kubectl describe networkpolicy <name> -n <namespace>

# Test connectivity
kubectl exec <pod-a> -n <ns-a> -- \
  curl --connect-timeout 3 http://<pod-b-ip>:<port>
kubectl exec <pod-a> -n <ns-a> -- \
  nc -zv <pod-b-ip> <port>

# Debug: Check what policies apply to a pod
kubectl get networkpolicies -n <ns> -o yaml | \
  grep -B5 -A10 "app: <pod-label>"

# Find namespaces without default-deny
kubectl get ns -o jsonpath='{.items[*].metadata.name}' | \
  tr ' ' '\n' | while read ns; do
    count=$(kubectl get netpol -n $ns --no-headers 2>/dev/null | \
      grep "default-deny" | wc -l)
    [ "$count" -eq 0 ] && echo "NO DEFAULT-DENY: $ns"
  done
```

### 27.5 Quick Reference Templates

```yaml
# ── DEFAULT-DENY + DNS ALLOW ────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  egress:
    - to: []
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53

# ── ALLOW APP-TO-APP ────────────────────────────────────────────────────
# Ingress policy (on receiving pod)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-from-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080

# ── CROSS-NAMESPACE (AND logic) ─────────────────────────────────────────
ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: team-a
        podSelector:         # SAME item = AND
          matchLabels:
            app: frontend
```

### 27.6 Troubleshooting Quick Reference

```bash
# Quota rejection events
kubectl describe rs <rs-name> -n <ns> | grep FailedCreate
kubectl get events -n <ns> --field-selector reason=FailedCreate

# LimitRange violations
kubectl get events -n <ns> | grep "outside"

# Namespace stuck terminating
kubectl get namespace stuck-ns \
  -o jsonpath='{.spec.finalizers}'
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -I{} kubectl get {} -n stuck-ns 2>/dev/null

# NetworkPolicy connectivity test
kubectl exec pod-a -n ns-a -- curl -sv \
  --connect-timeout 3 http://$POD_B_IP:80 2>&1 | \
  grep -E "Connected|timeout|refused|error"

# Verify CNI supports NetworkPolicy
kubectl get pods -n kube-system | \
  grep -E "calico|cilium|weave|canal"
```

---

## 28. Key Takeaways & Summary {#28-key-takeaways}

### The Four-Layer Governance Model

```
COMPLETE NAMESPACE GOVERNANCE (all four layers required):
─────────────────────────────────────────────────────────────────────

LAYER 1 — ACCESS CONTROL (Who can do what):
  Namespace + RBAC → team-a has admin in team-a-prod; zero access elsewhere

LAYER 2 — RESOURCE BUDGET (How much can be consumed):
  ResourceQuota → team-a-prod: max 8 CPU, 16Gi memory, 50 pods, 2 LBs

LAYER 3 — OBJECT SIZING (How each container is sized):
  LimitRange → each container: default=100m/128Mi; max=2CPU/2Gi

LAYER 4 — NETWORK ISOLATION (Who can communicate with whom):
  NetworkPolicy → default-deny-all + explicit allow rules per flow

DEPLOYMENT ORDER: LimitRange → ResourceQuota → Namespace → NetworkPolicy
REASON: LimitRange must exist before ResourceQuota (injects defaults)
─────────────────────────────────────────────────────────────────────
```

### Critical Concepts Summary

| Concept | Key Rule |
|---|---|
| **ResourceQuota admission** | Synchronous; enforced at every request; admission NEVER stops even during controller outage |
| **ResourceQuota controller** | Asynchronous; reconciles `status.used`; runs every 5 min |
| **LimitRange injection** | Runs BEFORE quota admission; injects defaults into pods missing resources |
| **NetworkPolicy enforcement** | Done by CNI on each node; API server only stores the objects |
| **Controller access to etcd** | NEVER direct — always via kube-apiserver |
| **Default network model** | OPEN (all pods can reach all pods); NetworkPolicy restricts this |
| **Leader election** | One active controller-manager at a time; ~15-30s failover |
| **Namespace deletion** | Cascades to ALL resources; check for finalizers if stuck |
| **DNS in NetworkPolicy** | ALWAYS allow port 53 egress or service discovery breaks |

### The 10 Production Rules

1. **Never use `default` namespace for production** — always named, governed namespaces
2. **Deploy LimitRange before or with ResourceQuota** — pods without resources fail without it
3. **Default-deny-all NetworkPolicy in every namespace** — enforce via Kyverno/automation
4. **Always allow DNS egress (port 53)** — forgetting this breaks everything
5. **AND vs OR in NetworkPolicy**: same `from` item = AND; separate items = OR
6. **Use `kubernetes.io/metadata.name` label** — reliable, auto-added, unique namespace identifier
7. **Controllers never access etcd directly** — all operations via kube-apiserver
8. **ResourceQuota admission is synchronous** — quotas enforced even when controller-manager is down
9. **Verify CNI supports NetworkPolicy** — Flannel without Calico = zero enforcement
10. **Store all namespace config in Git** — namespaces, quotas, limitranges, networkpolicies are infrastructure as code

---

> **This guide covers Kubernetes v1.29+ namespace management. Consult the official documentation at https://kubernetes.io/docs/concepts/policy/ and https://kubernetes.io/docs/concepts/services-networking/network-policies/ for the most current information.**

---

*End of: Kubernetes Namespaces, Resource Quotas, LimitRanges & Network Policies — Complete Production-Grade Guide*
