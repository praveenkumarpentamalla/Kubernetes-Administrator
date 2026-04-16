## Table of Contents

1. [Introduction — Why Deployments Are the Core Workload Primitive](#1-introduction--why-deployments-are-the-core-workload-primitive)
2. [Core Identity Table — Deployment Components](#2-core-identity-table--deployment-components)
3. [Understanding Deployments — Architecture Deep Dive](#3-understanding-deployments--architecture-deep-dive)
4. [Features of ReplicaSet and Deployment](#4-features-of-replicaset-and-deployment)
5. [The Controller Pattern — Watch → Compare → Act → Loop](#5-the-controller-pattern--watch--compare--act--loop)
6. [Lab: Creating a Basic Deployment YAML](#6-lab-creating-a-basic-deployment-yaml)
7. [Using Deployments with Affinity](#7-using-deployments-with-affinity)
8. [Lab: Creating a Deployment with Affinity](#8-lab-creating-a-deployment-with-affinity)
9. [Lab: Creating a Deployment with Anti-Pod Affinity](#9-lab-creating-a-deployment-with-anti-pod-affinity)
10. [Services in Kubernetes — ClusterIP, NodePort, LoadBalancer](#10-services-in-kubernetes--clusterip-nodeport-loadbalancer)
11. [Lab: Creating a Deployment with Service](#11-lab-creating-a-deployment-with-service)
12. [Understanding Manual Scaling of Pods](#12-understanding-manual-scaling-of-pods)
13. [Lab: Scaling Deployment Manually](#13-lab-scaling-deployment-manually)
14. [Understanding Horizontal Pod Autoscaler (HPA)](#14-understanding-horizontal-pod-autoscaler-hpa)
15. [Lab: Creating HPA and Attaching It to a Deployment](#15-lab-creating-hpa-and-attaching-it-to-a-deployment)
16. [Understanding Different Deployment Strategies](#16-understanding-different-deployment-strategies)
17. [Lab: Upgrading Application Using Rolling Update Strategy](#17-lab-upgrading-application-using-rolling-update-strategy)
18. [Built-in Controllers Deep Dive](#18-built-in-controllers-deep-dive)
19. [Internal Working Concepts — Informers, Work Queues & Reconciliation](#19-internal-working-concepts--informers-work-queues--reconciliation)
20. [API Server and etcd Interaction](#20-api-server-and-etcd-interaction)
21. [Leader Election — HA for Deployment Infrastructure](#21-leader-election--ha-for-deployment-infrastructure)
22. [Performance Tuning for Deployments](#22-performance-tuning-for-deployments)
23. [Security Hardening Practices](#23-security-hardening-practices)
24. [Monitoring and Observability](#24-monitoring-and-observability)
25. [Troubleshooting Deployments — Real kubectl Commands](#25-troubleshooting-deployments--real-kubectl-commands)
26. [Lab: Performing Troubleshooting of Common Deployment Scenarios](#26-lab-performing-troubleshooting-of-common-deployment-scenarios)
27. [Disaster Recovery Concepts](#27-disaster-recovery-concepts)
28. [Comparison — kube-apiserver vs kube-scheduler vs kube-controller-manager](#28-comparison--kube-apiserver-vs-kube-scheduler-vs-kube-controller-manager)
29. [ASCII Architecture Diagram](#29-ascii-architecture-diagram)
30. [Real-World Production Use Cases](#30-real-world-production-use-cases)
31. [Best Practices for Production Environments](#31-best-practices-for-production-environments)
32. [Common Mistakes and Pitfalls](#32-common-mistakes-and-pitfalls)
33. [Interview Questions — Beginner to Advanced](#33-interview-questions--beginner-to-advanced)
34. [Cheat Sheet — Commands, Flags & Manifests](#34-cheat-sheet--commands-flags--manifests)
35. [Key Takeaways & Summary](#35-key-takeaways--summary)

---

## 1. Introduction — Why Deployments Are the Core Workload Primitive

The **Deployment** is the most important and most frequently used workload resource in Kubernetes. It is the standard way to run stateless applications at scale, manage rolling updates, and handle rollbacks. Understanding Deployments thoroughly — their internals, their relationship with ReplicaSets, their scaling mechanisms, their update strategies — is the foundation of Kubernetes operations.

Before Deployments, teams had to manually manage Replication Controllers and update Pods in-place, creating complex and error-prone update procedures. Deployments introduced a **declarative model** for application lifecycle: you describe the desired state, and Kubernetes continuously works to achieve and maintain it.

### Why Deployments Are Essential

| Without Deployments | With Deployments |
|---|---|
| Manual Pod creation and management | Declarative desired state management |
| No self-healing if Pod crashes | Automatic replacement of failed Pods |
| Manual image updates with downtime | Zero-downtime rolling updates |
| No rollback mechanism | One-command rollback to any revision |
| Manual scaling | One-command manual or automatic (HPA) scaling |
| No update history | Full rollout history with change causes |
| Pods tied to specific nodes | Flexible scheduling across the cluster |

### What This Guide Covers

This guide builds a complete mental model of Kubernetes Deployments: from the YAML manifest and ReplicaSet internals, through scheduling with affinity rules, Service exposure, scaling (manual and automatic), deployment strategies, and comprehensive troubleshooting — all grounded in production-grade examples.

---

## 2. Core Identity Table — Deployment Components

| Component | Kind / Binary | API Group | Port | Role |
|---|---|---|---|---|
| **Deployment** | `deployment` | `apps/v1` | N/A | Manages ReplicaSets; defines desired state for stateless apps |
| **ReplicaSet** | `replicaset` | `apps/v1` | N/A | Ensures N Pod replicas are running; owned by Deployment |
| **Pod** | `pod` | `core/v1` | N/A | Smallest deployable unit; owned by ReplicaSet |
| **Service (ClusterIP)** | `service` | `core/v1` | Configurable | Stable internal IP and DNS for Pod discovery |
| **Service (NodePort)** | `service` | `core/v1` | 30000-32767 | External access via node IP and port |
| **Service (LoadBalancer)** | `service` | `core/v1` | Configurable | External LB via cloud provider |
| **Endpoints/EndpointSlice** | `endpointslice` | `discovery.k8s.io/v1` | N/A | Tracks ready Pod IPs backing a Service |
| **HPA** | `horizontalpodautoscaler` | `autoscaling/v2` | N/A | Automatically scales Deployment replicas based on metrics |
| **kube-controller-manager** | Binary | N/A | 10257 | Runs Deployment, ReplicaSet, HPA controllers |
| **kube-scheduler** | Binary | N/A | 10259 | Assigns Pods to Nodes |
| **kube-apiserver** | Binary | N/A | 6443 | REST gateway; stores all Deployment state in etcd |
| **metrics-server** | Deployment | N/A | 443 | Provides CPU/memory metrics consumed by HPA |
| **etcd** | Binary | N/A | 2379 | Stores all Deployment specs, status, and history |

---

## 3. Understanding Deployments — Architecture Deep Dive

### 3.1 The Deployment Object Hierarchy

```
Deployment (desired state declaration)
    │
    ├── Creates and manages ReplicaSets
    │       │
    │       └── Creates and manages Pods
    │               │
    │               └── Runs Containers (via kubelet + CRI)
    │
    ├── Old ReplicaSet (scaling down during update)
    └── New ReplicaSet (scaling up during update)
```

### 3.2 What a Deployment Controls

A Deployment manages **ReplicaSets** — it never directly manages Pods. The Deployment controller:
- Creates a new ReplicaSet when the Pod template changes
- Scales the old RS down and new RS up (rolling update)
- Preserves rollout history (configurable number of old RS to keep)
- Updates `.status` with current replica counts

```
                 ┌─────────────────────────────────────────┐
                 │            DEPLOYMENT                    │
                 │  spec.replicas: 3                       │
                 │  spec.template: (Pod spec)              │
                 │  spec.strategy: RollingUpdate           │
                 └───────────────────┬─────────────────────┘
                                     │ owns
                  ┌──────────────────┼──────────────────┐
                  │                  │                   │
         ┌────────▼───────┐  ┌───────▼───────┐  (history)
         │ ReplicaSet v2  │  │ ReplicaSet v1  │
         │ replicas: 3    │  │ replicas: 0    │
         └────────┬───────┘  └───────────────┘
                  │ owns
         ┌────────┼────────┐
         │        │        │
    ┌────▼──┐ ┌───▼───┐ ┌──▼────┐
    │ Pod 1 │ │ Pod 2 │ │ Pod 3 │
    └───────┘ └───────┘ └───────┘
```

### 3.3 Deployment vs Other Workload Types

| Workload | Use Case | Pods Ordered | Stable Storage | Stable Network ID |
|---|---|---|---|---|
| **Deployment** | Stateless apps (web, API) | No | No | No |
| **StatefulSet** | Stateful apps (DB, MQ) | Yes (0,1,2...) | Yes (per-Pod PVC) | Yes (`pod-0.svc`) |
| **DaemonSet** | Per-node agents | No | Optional | No |
| **Job** | Batch tasks (run to completion) | No | No | No |
| **CronJob** | Scheduled batch tasks | No | No | No |

---

## 4. Features of ReplicaSet and Deployment

### 4.1 ReplicaSet Features

| Feature | Description |
|---|---|
| **Pod replica count** | Maintains exact N copies of Pods at all times |
| **Self-healing** | Replaces crashed or deleted Pods immediately |
| **Label selector** | Uses matchLabels/matchExpressions to identify owned Pods |
| **Pod template** | Template for creating new Pods when replicas are needed |
| **Scaling** | `spec.replicas` can be changed to scale up/down |
| **ownerReference** | Created Pods have RS as owner (enables GC cascade) |
| **Pod adoption** | Can "adopt" orphaned Pods matching its selector |

### 4.2 Deployment Features (Everything RS Has + More)

| Feature | Description |
|---|---|
| **All ReplicaSet features** | Inherits RS's self-healing, scaling, label matching |
| **Rolling updates** | Zero-downtime updates with configurable pace |
| **Rollback** | `kubectl rollout undo` to any historical revision |
| **Rollout history** | `.spec.revisionHistoryLimit` controls how many RS to keep |
| **Update strategies** | RollingUpdate (default) and Recreate |
| **Pause/Resume** | Pause a rollout mid-way for canary testing |
| **Progress deadline** | `.spec.progressDeadlineSeconds` fails the rollout if not done |
| **Change-cause annotation** | Records why a rollout was triggered |

### 4.3 ReplicaSet vs Deployment — When to Use Each

```
USE ReplicaSet directly:
  → Rare. Only when you need fine-grained control over Pod set
  → Custom controllers that manage their own update logic
  → When you don't need update history or rollback

USE Deployment (almost always in practice):
  → All stateless application workloads
  → When you need rolling updates and rollbacks
  → When you want change history tracking
  → CI/CD pipelines deploying new image versions
```

---

## 5. The Controller Pattern — Watch → Compare → Act → Loop

The Deployment controller (running inside kube-controller-manager) follows the fundamental Kubernetes controller pattern.

### 5.1 The Reconciliation Loop

```
┌─────────────────────────────────────────────────────────────────────┐
│              DEPLOYMENT CONTROLLER RECONCILIATION LOOP              │
│                                                                     │
│  WATCH:                                                             │
│  ─────                                                              │
│  Informer watches Deployment objects via API server watch stream    │
│  Informer watches ReplicaSet objects (to detect RS changes)         │
│  Informer watches Pods (to detect Pod status changes)               │
│                                                                     │
│  On event: ADDED, MODIFIED, DELETED → key placed in work queue     │
│                                                                     │
│  COMPARE:                                                           │
│  ────────                                                           │
│  Worker goroutine picks key "default/my-deployment"                 │
│  Reads Deployment.spec.replicas (desired: 3)                        │
│  Reads current ReplicaSet Pod count (current: 2)                   │
│  Computes delta: need 1 more Pod                                    │
│                                                                     │
│  ACT:                                                               │
│  ────                                                               │
│  API call: PATCH ReplicaSet.spec.replicas = 3                      │
│  ReplicaSet controller detects change → creates 1 new Pod          │
│  Deployment.status updated: readyReplicas=2, updatedReplicas=3     │
│                                                                     │
│  LOOP:                                                              │
│  ─────                                                              │
│  workQueue.Forget(key) on success                                   │
│  workQueue.AddRateLimited(key) on error (exponential backoff)       │
│  Return to WATCH                                                    │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 What Triggers Reconciliation

```bash
# These events trigger the Deployment controller to reconcile:
# 1. Deployment created/updated/deleted
# 2. ReplicaSet associated with this Deployment changes
# 3. Pod with matching labels changes (crash, restart, ready state)
# 4. Periodic resync (every ~10-30 minutes regardless of events)

# Observe reconciliation happening
kubectl get replicasets --all-namespaces -w &
kubectl set image deployment/my-app container=nginx:1.25
# Watch new RS appear and scale up while old RS scales down
kill %1
```

---

## 6. Lab: Creating a Basic Deployment YAML

### 6.1 Minimal Deployment

```yaml
# File: minimal-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
```

```bash
kubectl apply -f minimal-deployment.yaml
kubectl get deployment nginx-deployment
kubectl get replicasets -l app=nginx
kubectl get pods -l app=nginx
kubectl delete -f minimal-deployment.yaml
```

### 6.2 Production-Grade Deployment YAML

```yaml
# File: production-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  namespace: production
  labels:
    app: api-server
    tier: backend
    version: "2.1.0"
    managed-by: helm
  annotations:
    kubernetes.io/change-cause: "v2.1.0: Added Redis caching layer, improved connection pooling"
    deployment.kubernetes.io/contact: "platform-team@company.com"

spec:
  # ── REPLICA COUNT ────────────────────────────────────────────────────
  replicas: 3
  
  # ── REVISION HISTORY (how many old ReplicaSets to keep for rollback) ─
  revisionHistoryLimit: 5    # Default is 10; reduce to save etcd space
  
  # ── PROGRESS DEADLINE ────────────────────────────────────────────────
  progressDeadlineSeconds: 600    # Fail rollout if not done in 10 min

  # ── POD SELECTOR ─────────────────────────────────────────────────────
  selector:
    matchLabels:
      app: api-server    # Must match template labels exactly

  # ── UPDATE STRATEGY ──────────────────────────────────────────────────
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Create 1 extra pod during update
      maxUnavailable: 0    # Never go below desired count (zero-downtime)

  # ── POD TEMPLATE ─────────────────────────────────────────────────────
  template:
    metadata:
      labels:
        app: api-server      # Must match selector.matchLabels
        version: "2.1.0"
        tier: backend
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"

    spec:
      # Service account (dedicated per app)
      serviceAccountName: api-server-sa
      automountServiceAccountToken: false

      # Termination grace period (must be > max request timeout)
      terminationGracePeriodSeconds: 60

      # Init containers (run before app containers)
      initContainers:
        - name: init-check-db
          image: busybox:1.36
          command: ['sh', '-c',
            'until nc -z postgres-svc 5432; do echo waiting for db; sleep 2; done']
          resources:
            requests:
              cpu: 10m
              memory: 16Mi

      # Main application containers
      containers:
        - name: api
          image: my-company/api-server:2.1.0
          imagePullPolicy: IfNotPresent

          # ── PORTS ─────────────────────────────────────────────────────
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
            - name: metrics
              containerPort: 9090
              protocol: TCP

          # ── ENVIRONMENT ───────────────────────────────────────────────
          env:
            - name: APP_ENV
              value: "production"
            - name: LOG_LEVEL
              value: "warn"
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: api-server-config
                  key: DB_HOST
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: api-server-secrets
                  key: db_password
            # Downward API: inject pod metadata into container
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace

          # ── RESOURCE MANAGEMENT ───────────────────────────────────────
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi

          # ── HEALTH PROBES ─────────────────────────────────────────────
          startupProbe:
            httpGet:
              path: /health/startup
              port: http
            failureThreshold: 30
            periodSeconds: 10

          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5

          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            periodSeconds: 5
            failureThreshold: 3
            successThreshold: 1

          # ── SECURITY ──────────────────────────────────────────────────
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1000
            capabilities:
              drop:
                - ALL

          # ── LIFECYCLE ─────────────────────────────────────────────────
          lifecycle:
            preStop:
              exec:
                # Drain in-flight requests before SIGTERM
                command: ["/bin/sh", "-c", "sleep 5"]

          # ── VOLUME MOUNTS ─────────────────────────────────────────────
          volumeMounts:
            - name: tmp-dir
              mountPath: /tmp
            - name: app-config
              mountPath: /etc/config
              readOnly: true

      # ── VOLUMES ───────────────────────────────────────────────────────
      volumes:
        - name: tmp-dir
          emptyDir: {}
        - name: app-config
          configMap:
            name: api-server-config

      # ── IMAGE PULL SECRETS ────────────────────────────────────────────
      imagePullSecrets:
        - name: registry-credentials

      # ── NODE SELECTION ────────────────────────────────────────────────
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: api-server
```

```bash
# Apply and verify
kubectl apply -f production-deployment.yaml

# Watch the rollout
kubectl rollout status deployment/api-server -n production

# Verify all components
kubectl get deployment api-server -n production
kubectl get replicasets -l app=api-server -n production
kubectl get pods -l app=api-server -n production -o wide
kubectl describe deployment api-server -n production
```

---

## 7. Using Deployments with Affinity

### 7.1 Why Affinity Matters for Deployments

Affinity rules control WHERE your Deployment's Pods land in the cluster. Without them, Pods are distributed by the scheduler's default algorithm (resource-based scoring). With affinity rules, you can:
- Ensure replicas spread across availability zones (HA)
- Co-locate Pods with their data (performance)
- Separate Pods from each other (fault isolation)
- Keep Pods away from certain node types (compliance)

### 7.2 Affinity Types in Deployments

| Type | Level | Behavior | Use Case |
|---|---|---|---|
| `nodeAffinity.required` | Node | Hard rule: pod stuck Pending if no match | Hardware-specific workloads |
| `nodeAffinity.preferred` | Node | Soft: tries to match, falls back to any | Cost optimization |
| `podAffinity.required` | Pod→Pod | Hard: co-locate on same node/zone | Latency-sensitive pairs |
| `podAffinity.preferred` | Pod→Pod | Soft: prefer co-location | Cache locality |
| `podAntiAffinity.required` | Pod↔Pod | Hard: force separation | HA across nodes |
| `podAntiAffinity.preferred` | Pod↔Pod | Soft: prefer separation | Zone spread |
| `topologySpreadConstraints` | Pod spread | Even distribution | Modern HA spreading |

---

## 8. Lab: Creating a Deployment with Affinity

### 8.1 Node Affinity — Target Specific Node Types

```bash
# Setup: Label your nodes
kubectl label node k8s-worker-1 hardware/disk=ssd topology.kubernetes.io/zone=us-east-1a
kubectl label node k8s-worker-2 hardware/disk=ssd topology.kubernetes.io/zone=us-east-1b
kubectl label node k8s-worker-3 hardware/disk=hdd topology.kubernetes.io/zone=us-east-1c

# Verify labels
kubectl get nodes --show-labels | grep disk
```

```bash
# Deploy with required node affinity (SSD nodes only)
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ssd-required-deployment
  labels:
    app: ssd-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ssd-app
  template:
    metadata:
      labels:
        app: ssd-app
    spec:
      affinity:
        nodeAffinity:
          # HARD: Must land on SSD nodes
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: hardware/disk
                    operator: In
                    values:
                      - ssd
          # SOFT: Prefer east zone
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 80
              preference:
                matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values:
                      - us-east-1a
                      - us-east-1b
      containers:
        - name: app
          image: nginx:1.25
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
EOF

kubectl get pods -l app=ssd-app -o wide
# All pods on k8s-worker-1 or k8s-worker-2 (SSD nodes)

kubectl delete deployment ssd-required-deployment
```

### 8.2 Pod Affinity — Co-locate App with Cache

```bash
# First, deploy the Redis cache
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis-cache
      tier: cache
  template:
    metadata:
      labels:
        app: redis-cache
        tier: cache
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
EOF

# Deploy API that prefers to be on same node as Redis cache
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-colocated
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-colocated
  template:
    metadata:
      labels:
        app: api-colocated
    spec:
      affinity:
        podAffinity:
          # PREFER: same node as redis-cache pods
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    tier: cache
                topologyKey: kubernetes.io/hostname
      containers:
        - name: api
          image: nginx:1.25
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
EOF

# Verify co-location
kubectl get pods -o wide | grep -E "redis|api-colocated"

kubectl delete deployment redis-cache api-colocated
```

---

## 9. Lab: Creating a Deployment with Anti-Pod Affinity

### 9.1 Hard Anti-Affinity — Force HA Spread

```bash
# Deploy payment service with REQUIRED zone separation
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  labels:
    app: payment-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
        criticality: high
    spec:
      affinity:
        podAntiAffinity:
          # HARD: No two payment-service pods on same node
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: payment-service
              topologyKey: kubernetes.io/hostname
              # ^ Change to topology.kubernetes.io/zone for zone separation
      containers:
        - name: payment-api
          image: nginx:1.25
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
EOF

# Verify: each pod on a different node
kubectl get pods -l app=payment-service -o wide
# NAME                          NODE
# payment-service-xxx-abc       k8s-worker-1
# payment-service-xxx-def       k8s-worker-2
# payment-service-xxx-xyz       k8s-worker-3

kubectl delete deployment payment-service
```

### 9.2 topologySpreadConstraints — Modern Alternative to Anti-Affinity

```bash
# For spreading use cases, topologySpreadConstraints is preferred
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-web-service
spec:
  replicas: 6
  selector:
    matchLabels:
      app: ha-web
  template:
    metadata:
      labels:
        app: ha-web
    spec:
      topologySpreadConstraints:
        # HARD: Max 1 skew between zones (even zone distribution)
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: ha-web
        # SOFT: Max 2 skew between nodes
        - maxSkew: 2
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: ha-web
      containers:
        - name: web
          image: nginx:1.25
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
EOF

kubectl get pods -l app=ha-web -o wide
# Even distribution: 2 pods per zone, spread across nodes

kubectl delete deployment ha-web-service
```

---

## 10. Services in Kubernetes — ClusterIP, NodePort, LoadBalancer

### 10.1 Why Services Are Required

Pods are ephemeral. They get new IP addresses on every restart, and they can be scheduled to any node. Without Services, consuming applications would need to track Pod IPs — an impossible task.

A **Service** provides:
- A stable virtual IP (ClusterIP) that never changes
- A stable DNS name (`<service>.<namespace>.svc.cluster.local`)
- Automatic load balancing across all healthy, ready Pods
- Automatic endpoint management as Pods come and go

### 10.2 Service Types Deep Dive

```
┌───────────────────────────────────────────────────────────────────────┐
│                    SERVICE TYPE HIERARCHY                             │
│                                                                       │
│  ClusterIP (base)                                                     │
│  → Virtual IP accessible only within cluster                         │
│  → DNS: service.namespace.svc.cluster.local                          │
│  → Port: configurable                                                 │
│                                                                       │
│  NodePort (extends ClusterIP)                                         │
│  → Everything ClusterIP provides PLUS                                 │
│  → Static port on EVERY node IP                                      │
│  → Port range: 30000-32767                                           │
│  → Access: NodeIP:NodePort → ClusterIP:Port → PodIP:TargetPort       │
│                                                                       │
│  LoadBalancer (extends NodePort)                                      │
│  → Everything NodePort provides PLUS                                  │
│  → External cloud load balancer provisioned automatically            │
│  → External IP assigned by cloud provider                            │
│  → Access: ExternalIP:Port → NodePort → ClusterIP → Pod             │
│                                                                       │
│  ExternalName (DNS alias, no proxying)                               │
│  → Maps service to external DNS name                                 │
│  → No IP allocation, no port mapping                                 │
└───────────────────────────────────────────────────────────────────────┘
```

### 10.3 Service Type Comparison Table

| Feature | ClusterIP | NodePort | LoadBalancer | ExternalName |
|---|---|---|---|---|
| **Internal cluster access** | ✅ | ✅ | ✅ | ✅ (via DNS) |
| **External access** | ❌ | ✅ (via NodeIP) | ✅ (via LB IP) | ✅ (DNS redirect) |
| **Automatic IP assigned** | Virtual IP | Virtual IP | Virtual IP + External IP | No IP |
| **Port exposure** | ClusterIP port | NodePort + ClusterIP | LB port + NodePort | N/A |
| **Cloud provider required** | ❌ | ❌ | ✅ | ❌ |
| **Production for external** | ❌ (internal only) | Dev/on-prem | ✅ Cloud production | Integration |
| **Use case** | Internal microservices | Dev, on-prem | Cloud production | External DB alias |

### 10.4 Service and Endpoints Relationship

```bash
# When you create a Service, Kubernetes auto-creates Endpoints
# Endpoints = list of Pod IPs that are Ready and match the selector

kubectl get service my-api
# NAME      TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
# my-api    ClusterIP   10.96.50.1    <none>        80/TCP    5m

kubectl get endpoints my-api
# NAME      ENDPOINTS                              AGE
# my-api    10.244.1.5:8080,10.244.2.3:8080,...   5m

# EndpointSlices (modern replacement for Endpoints, default in 1.21+)
kubectl get endpointslices -l kubernetes.io/service-name=my-api
```

### 10.5 How kube-proxy Implements Services

```
Service ClusterIP (10.96.50.1:80) → iptables/IPVS rules on every node
                                     → DNAT to random Pod IP

iptables (default mode):
  KUBE-SERVICES chain → KUBE-SVC-XXXXX chain → KUBE-SEP-XXXXX → DNAT

IPVS mode (better performance at scale):
  ipvsadm -L -n → virtual server table
  Hash table lookup O(1) vs iptables O(n)
```

---

## 11. Lab: Creating a Deployment with Service

### 11.1 Complete Application Stack

```bash
# ── STEP 1: Create Namespace ─────────────────────────────────────────────
kubectl create namespace app-demo

# ── STEP 2: Create ConfigMap ─────────────────────────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
  namespace: app-demo
data:
  ENVIRONMENT: "demo"
  MAX_CONNECTIONS: "50"
EOF

# ── STEP 3: Create Deployment ────────────────────────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: app-demo
  labels:
    app: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: webapp
        version: "1.0"
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - name: http
              containerPort: 80
          envFrom:
            - configMapRef:
                name: webapp-config
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 5
EOF

# ── STEP 4: Create ClusterIP Service ─────────────────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: webapp-clusterip
  namespace: app-demo
  labels:
    app: webapp
spec:
  type: ClusterIP
  selector:
    app: webapp      # Routes to pods with this label
  ports:
    - name: http
      port: 80           # Service port (what clients connect to)
      targetPort: http   # Container port name (from deployment)
      protocol: TCP
EOF

# ── STEP 5: Create NodePort Service ──────────────────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: webapp-nodeport
  namespace: app-demo
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
    - name: http
      port: 80
      targetPort: http
      nodePort: 31080    # External port (omit for auto-assignment)
      protocol: TCP
EOF

# ── STEP 6: Create LoadBalancer Service (cloud clusters) ─────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: webapp-loadbalancer
  namespace: app-demo
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"  # AWS-specific
spec:
  type: LoadBalancer
  selector:
    app: webapp
  ports:
    - name: http
      port: 80
      targetPort: http
  loadBalancerSourceRanges:
    - "10.0.0.0/8"        # Restrict to private IP range
EOF

# ── VERIFICATION ──────────────────────────────────────────────────────────
kubectl get all -n app-demo
kubectl get endpoints -n app-demo

# Test ClusterIP via port-forward
kubectl port-forward service/webapp-clusterip 8080:80 -n app-demo &
curl http://localhost:8080
kill %1

# Test NodePort
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
curl http://${NODE_IP}:31080

# Describe services to see endpoints
kubectl describe service webapp-clusterip -n app-demo
kubectl describe service webapp-nodeport -n app-demo
```

### 11.2 Service-to-Deployment Mapping Verification

```bash
# Verify Service correctly routes to Deployment pods
# The Service selector must match Deployment pod template labels

# Check what pods a Service selects
kubectl get pods -n app-demo \
  -l app=webapp \
  -o custom-columns="NAME:.metadata.name,IP:.status.podIP,READY:.status.conditions[?(@.type=='Ready')].status"

# Verify endpoints match pod IPs
kubectl get endpoints webapp-clusterip -n app-demo
# Should show the same IPs as the pods above

# What happens to endpoints when pods fail readinessProbe?
# → Failing pod removed from endpoints
# → Traffic stops going to that pod until it passes readiness check again

# Cleanup
kubectl delete namespace app-demo
```

---

## 12. Understanding Manual Scaling of Pods

### 12.1 How Scaling Works Internally

When you scale a Deployment, the following happens:
1. `kubectl scale` sends PATCH request to API server → updates `spec.replicas`
2. Deployment controller detects change via informer
3. Deployment controller updates the active ReplicaSet's `spec.replicas`
4. ReplicaSet controller detects change → creates/deletes Pods
5. Scheduler assigns new Pods to nodes
6. kubelet starts containers on assigned nodes
7. Pods become Ready → added to Service Endpoints

```
Scale UP (replicas 3 → 5):
  kubectl scale → API server → etcd stores replicas=5
  Deployment controller: desired=5, current=3, create 2 Pods
  ReplicaSet: creates 2 new Pod objects
  Scheduler: assigns Pods to nodes
  kubelet: starts containers
  Time to ready: typically 10-60 seconds

Scale DOWN (replicas 5 → 3):
  kubectl scale → API server → etcd stores replicas=3
  Deployment controller: desired=3, current=5, delete 2 Pods
  ReplicaSet: marks 2 oldest Pods for deletion
  Pods: graceful termination (terminationGracePeriodSeconds)
  Service Endpoints: updated immediately when Pods stop being ready
```

### 12.2 Scale Zero — The Cost Optimization Pattern

```bash
# Scale to zero during off-hours (saves cost)
kubectl scale deployment my-app --replicas=0

# Scale back when needed
kubectl scale deployment my-app --replicas=3

# Or use KEDA (Kubernetes Event-Driven Autoscaler) for automatic scale-to-zero
```

---

## 13. Lab: Scaling Deployment Manually

### 13.1 Basic Scaling Operations

```bash
# Create initial deployment
kubectl create deployment scale-demo \
  --image=nginx:1.25 \
  --replicas=2

kubectl get deployment scale-demo
# READY: 2/2

# ── SCALE UP ─────────────────────────────────────────────────────────────
kubectl scale deployment scale-demo --replicas=5

# Watch the scale-up
kubectl get pods -l app=scale-demo -w &
sleep 30
kill %1

kubectl get deployment scale-demo
# READY: 5/5

# ── SCALE DOWN ───────────────────────────────────────────────────────────
kubectl scale deployment scale-demo --replicas=2

kubectl rollout status deployment scale-demo
kubectl get deployment scale-demo

# ── SCALE TO ZERO ────────────────────────────────────────────────────────
kubectl scale deployment scale-demo --replicas=0
kubectl get pods -l app=scale-demo   # No pods
kubectl get deployment scale-demo
# READY: 0/0

# ── SCALE BACK UP ─────────────────────────────────────────────────────────
kubectl scale deployment scale-demo --replicas=3
kubectl get pods -l app=scale-demo -w
```

### 13.2 Scaling with Resource Verification

```bash
# Verify node capacity before scaling
kubectl top nodes
kubectl describe nodes | grep -A 5 "Allocated resources:"

# Scale with awareness of resource constraints
kubectl scale deployment scale-demo --replicas=10

# Check if all pods scheduled (some may be Pending due to resource constraints)
kubectl get pods -l app=scale-demo --field-selector=status.phase=Pending
kubectl describe pod <pending-pod-name> | grep -A 5 "Events:"

# Scale back to reasonable number
kubectl scale deployment scale-demo --replicas=3
```

### 13.3 Scaling Deployment via YAML Patch

```bash
# Patch the deployment directly
kubectl patch deployment scale-demo \
  -p '{"spec":{"replicas":4}}'

# Using kubectl edit (opens in $EDITOR)
kubectl edit deployment scale-demo
# Change spec.replicas: N

# Using apply with updated file
kubectl get deployment scale-demo -o yaml > scale-demo.yaml
# Edit replicas field in scale-demo.yaml
kubectl apply -f scale-demo.yaml

# Cleanup
kubectl delete deployment scale-demo
```

---

## 14. Understanding Horizontal Pod Autoscaler (HPA)

### 14.1 How HPA Works

The HPA controller runs in kube-controller-manager and periodically adjusts the replica count of a Deployment (or RS, StatefulSet) based on observed metrics.

```
HPA CONTROL LOOP (runs every 15 seconds by default):
─────────────────────────────────────────────────────────────────────────
1. Collect metrics from metrics-server (or custom metrics adapter)
   CPU:     metrics.k8s.io/v1beta1/pods/{name}/cpu
   Memory:  metrics.k8s.io/v1beta1/pods/{name}/memory

2. Calculate desired replicas:
   desiredReplicas = ceil(currentReplicas * (currentMetric / targetMetric))

   Example: 
   currentReplicas = 3
   currentCPU = 150m (50% of 300m request across 3 pods)
   target     = 50% CPU utilization
   → desiredReplicas = ceil(3 * (150/150)) = 3 (no change)

   If load increases:
   currentCPU = 225m (75% utilization)
   → desiredReplicas = ceil(3 * (225/150)) = 5 (scale up)

3. Apply min/max constraints:
   desiredReplicas = max(minReplicas, min(maxReplicas, desiredReplicas))

4. Scale if desiredReplicas != currentReplicas
   (with cooldown periods: scaleUp=0s default, scaleDown=5min)
```

### 14.2 HPA Scaling Behavior Configuration

```yaml
# HPA v2 behavior configuration
behavior:
  scaleUp:
    stabilizationWindowSeconds: 0     # Scale up immediately
    policies:
      - type: Pods
        value: 4                       # Add up to 4 pods per period
        periodSeconds: 60
      - type: Percent
        value: 100                     # Or double every 60s (whichever is faster)
        periodSeconds: 60
    selectPolicy: Max                  # Use the policy that allows most change
  scaleDown:
    stabilizationWindowSeconds: 300   # Wait 5 minutes before scaling down
    policies:
      - type: Pods
        value: 1                       # Remove only 1 pod per period
        periodSeconds: 60             # Conservative scale-down
    selectPolicy: Min                  # Use the policy that allows least change
```

### 14.3 HPA with Custom and External Metrics

```yaml
# HPA with multiple metrics sources
spec:
  metrics:
    # CPU utilization (most common)
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50

    # Memory utilization
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 400Mi

    # Custom metric from Prometheus adapter
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"

    # External metric (e.g., queue depth)
    - type: External
      external:
        metric:
          name: sqs_queue_depth
          selector:
            matchLabels:
              queue: payment-processor
        target:
          type: AverageValue
          averageValue: "30"
```

---

## 15. Lab: Creating HPA and Attaching It to a Deployment

### 15.1 Setup — Install Metrics Server

```bash
# Install metrics-server (required for CPU/memory based HPA)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# For development environments with self-signed certs:
kubectl patch deployment metrics-server \
  -n kube-system \
  --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

# Verify metrics-server is working
kubectl top nodes
kubectl top pods --all-namespaces
```

### 15.2 Create HPA Target Deployment

```bash
# Create a deployment with proper resource requests (REQUIRED for HPA)
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-demo
  labels:
    app: hpa-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hpa-demo
  template:
    metadata:
      labels:
        app: hpa-demo
    spec:
      containers:
        - name: web
          image: nginx:1.25
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 200m       # HPA uses requests as the denominator for % calculation
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
EOF

kubectl rollout status deployment/hpa-demo
```

### 15.3 Create HPA with kubectl

```bash
# Method 1: Imperative command
kubectl autoscale deployment hpa-demo \
  --min=2 \
  --max=10 \
  --cpu-percent=50

# View the HPA
kubectl get hpa hpa-demo
# NAME       REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
# hpa-demo   Deployment/hpa-demo   5%/50%    2         10        2          30s
```

### 15.4 Create HPA with YAML (Recommended)

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hpa-demo-v2
spec:
  # Target the Deployment
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-demo

  # Replica bounds
  minReplicas: 2
  maxReplicas: 10

  # Scaling metrics
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50   # Scale when avg CPU > 50% of requests

    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70   # Scale when avg memory > 70%

  # Scaling behavior tuning
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Pods
          value: 3
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300     # 5-minute cooldown before scaling down
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
EOF

kubectl describe hpa hpa-demo-v2
```

### 15.5 Load Test to Trigger HPA

```bash
# Create a load generator pod
kubectl run load-gen \
  --image=busybox:1.36 \
  --restart=Never \
  -- sh -c "while true; do \
    wget -q -O- http://$(kubectl get svc hpa-demo-svc -o jsonpath='{.spec.clusterIP}') > /dev/null; \
  done"

# Monitor HPA scaling in real time
kubectl get hpa hpa-demo-v2 -w &
kubectl get deployment hpa-demo -w &

# Watch for scale up (takes ~15-30 seconds to react)
sleep 60
kubectl top pods -l app=hpa-demo

# Stop load test
kubectl delete pod load-gen --ignore-not-found

# Watch scale down (5-minute cooldown)
kubectl get hpa hpa-demo-v2 -w

# Cleanup
kubectl delete deployment hpa-demo
kubectl delete hpa hpa-demo hpa-demo-v2
```

---

## 16. Understanding Different Deployment Strategies

### 16.1 Strategy Overview

| Strategy | Downtime | Speed | Risk | Use Case |
|---|---|---|---|---|
| **RollingUpdate** | Zero | Controlled | Low-Medium | Standard production updates |
| **Recreate** | Yes (brief) | Fast | High | Dev environments, version incompatibility |
| **Blue-Green** (manual) | Zero | Fast | Low | Critical services needing instant cutover |
| **Canary** (manual/progressive) | Zero | Slow | Very Low | High-risk changes, gradual traffic shift |

### 16.2 RollingUpdate Strategy Deep Dive

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1          # Max extra pods during update
    maxUnavailable: 0    # Min pods always available = desired - maxUnavailable

# With replicas: 3, maxSurge: 1, maxUnavailable: 0:
#
# Initial state:   [v1] [v1] [v1]    = 3 running
# Step 1:          [v1] [v1] [v1] [v2] = 4 (surge 1 new pod)
# Step 2:          [v1] [v1] [v2]    = 3 (remove 1 old pod when v2 is ready)
# Step 3:          [v1] [v1] [v2] [v2] = 4 (surge 1 more new pod)
# Step 4:          [v1] [v2] [v2]    = 3 (remove 1 old pod)
# Step 5:          [v1] [v2] [v2] [v2] = 4 (surge 1 more new pod)
# Final:           [v2] [v2] [v2]    = 3 (remove last v1 pod)
```

### 16.3 Recreate Strategy

```yaml
strategy:
  type: Recreate
  # All existing pods terminated BEFORE new pods are created
  # → Brief downtime during the gap
  # → Use for: migrations requiring schema changes, version incompatibility
```

### 16.4 Blue-Green Deployment (Manual Implementation)

```bash
# Blue-Green using Service selector switching

# Step 1: Blue (current production) is running
kubectl create deployment blue-app --image=nginx:1.24 --replicas=3
kubectl label pods -l app=blue-app color=blue

# Step 2: Create Service pointing to blue
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: production-svc
spec:
  selector:
    color: blue    # Points to blue
  ports:
    - port: 80
EOF

# Step 3: Deploy Green (new version)
kubectl create deployment green-app --image=nginx:1.25 --replicas=3
kubectl label pods -l app=green-app color=green

# Step 4: Test green (via separate service)
# ... run tests against green ...

# Step 5: Switch traffic to green (instantaneous)
kubectl patch service production-svc \
  -p '{"spec":{"selector":{"color":"green"}}}'

# Step 6: Clean up blue after verification
kubectl delete deployment blue-app
```

### 16.5 Canary Deployment

```bash
# Canary: Route 10% of traffic to new version
# Achieved by controlling replica ratio

# 9 replicas of stable version
kubectl create deployment stable --image=nginx:1.24 --replicas=9
kubectl label pods -l app=stable track=stable

# 1 replica of canary (10% of total)
kubectl create deployment canary --image=nginx:1.25 --replicas=1
kubectl label pods -l app=canary track=canary

# One Service selecting BOTH
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: canary-svc
spec:
  selector:
    # Note: we're using app=stable OR app=canary
    # This doesn't work with one label — use labels on pods
    # Better: use track=stable or track=canary for routing
    # Or: use Ingress with weighted routing
    # Or: use service mesh (Istio/Linkerd) for precise traffic splitting
  ports:
    - port: 80
EOF

# For precise canary: use Argo Rollouts or Flagger
# kubectl argo rollouts set image my-rollout nginx=nginx:1.25
```

---

## 17. Lab: Upgrading Application Using Rolling Update Strategy

### 17.1 Setup Initial Deployment

```bash
# Create initial deployment
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-update-demo
  annotations:
    kubernetes.io/change-cause: "Initial deployment - nginx 1.24"
spec:
  replicas: 4
  revisionHistoryLimit: 5
  progressDeadlineSeconds: 300
  selector:
    matchLabels:
      app: rolling-demo
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0    # Zero downtime
  template:
    metadata:
      labels:
        app: rolling-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.24    # Starting version
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
EOF

kubectl rollout status deployment/rolling-update-demo
kubectl rollout history deployment/rolling-update-demo
```

### 17.2 Perform Rolling Update

```bash
# ── METHOD 1: Set image directly ─────────────────────────────────────────
kubectl set image deployment/rolling-update-demo \
  nginx=nginx:1.25 \
  --record    # Deprecated but still used; records command as change-cause

# ── METHOD 2: Patch the deployment ────────────────────────────────────────
kubectl patch deployment rolling-update-demo \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.25"}]}}}}'

# ── METHOD 3: Apply updated YAML (preferred in production) ───────────────
# Update the YAML file's image tag and annotation, then:
# kubectl apply -f updated-deployment.yaml

# Monitor the rollout in real time
kubectl rollout status deployment/rolling-update-demo --timeout=5m

# Watch pods transitioning
kubectl get pods -l app=rolling-demo -w &

# See ReplicaSet changes (old RS scales down, new RS scales up)
kubectl get replicasets -l app=rolling-demo -w
kill %1

# View updated rollout history
kubectl rollout history deployment/rolling-update-demo
# REVISION  CHANGE-CAUSE
# 1         Initial deployment - nginx 1.24
# 2         kubectl set image ...
```

### 17.3 Rollback

```bash
# View rollout history with details
kubectl rollout history deployment/rolling-update-demo --revision=1
kubectl rollout history deployment/rolling-update-demo --revision=2

# Rollback to previous version
kubectl rollout undo deployment/rolling-update-demo

# Rollback to specific revision
kubectl rollout undo deployment/rolling-update-demo --to-revision=1

# Verify rollback
kubectl rollout status deployment/rolling-update-demo
kubectl get pods -l app=rolling-demo -o jsonpath='{range .items[*]}{.spec.containers[0].image}{"\n"}{end}'
# Should show nginx:1.24 after rollback to revision 1
```

### 17.4 Pause and Resume Rollout (Canary Gate)

```bash
# Start a rollout
kubectl set image deployment/rolling-update-demo nginx=nginx:1.25

# Pause it immediately (only some pods updated — acts as canary)
kubectl rollout pause deployment/rolling-update-demo

# Check current state: some pods running new version, some old
kubectl get pods -l app=rolling-demo -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.containers[0].image}{"\n"}{end}'

# Run smoke tests, monitor metrics...
# ...

# If tests pass: resume
kubectl rollout resume deployment/rolling-update-demo

# If tests fail: rollback
kubectl rollout undo deployment/rolling-update-demo

# Cleanup
kubectl delete deployment rolling-update-demo
```

---

## 18. Built-in Controllers Deep Dive

### 18.1 Deployment Controller

The Deployment controller watches Deployments and ReplicaSets, managing the RS lifecycle for versioned updates.

**Reconciliation:**
1. New Deployment → Create ReplicaSet with same Pod template
2. Pod template changes → Create new RS; scale up new, scale down old
3. Scale change only → Modify existing RS replica count
4. Deployment deleted → Delete all RS (cascade)

### 18.2 ReplicaSet Controller

The RS controller ensures the correct number of Pods are running.

```
desired  = replicaset.spec.replicas
current  = count(pods matching selector, not terminating)
if current < desired → create new Pods
if current > desired → delete excess Pods (oldest first, then by phase)
```

### 18.3 Node Controller

The Node controller manages Node health and influences Deployment scheduling by:
- Adding `not-ready:NoSchedule` taint when node becomes unhealthy (no new pods)
- Adding `not-ready:NoExecute` taint → evicts existing pods
- Pod evictions trigger Deployment controller → creates replacement pods on healthy nodes

```bash
# Timeline when node fails:
# T+0:    Node heartbeat stops
# T+40s:  node-monitor-grace-period expires → Ready=Unknown
# T+5m:   pod-eviction-timeout → NoExecute taint → pods evicted
# T+5m+:  Deployment controller creates replacement pods elsewhere
```

### 18.4 Service Controller

The Service controller (inside kube-controller-manager) manages cloud load balancers for `LoadBalancer` type Services. For Deployments:
- Creates external LB when Service type=LoadBalancer
- Updates LB when backend pods change
- Deletes LB when Service is deleted

### 18.5 Namespace Controller

Manages Namespace lifecycle. When a Namespace is deleted, the Namespace controller cascades deletion to all resources — including Deployments, ReplicaSets, and Pods.

### 18.6 Horizontal Pod Autoscaler Controller

```
HPA RECONCILIATION (every 15 seconds):
  Read deployment.status.readyReplicas
  Fetch metrics from metrics-server
  Calculate desired replicas
  Compare with current replicas
  If different: PATCH deployment.spec.replicas
  Deployment controller reacts → creates/deletes pods via RS
```

### 18.7 EndpointSlice Controller

Watches Services and Pods, maintaining the EndpointSlice objects that kube-proxy uses for routing:
- Pod Ready → Added to EndpointSlice
- Pod NotReady or Terminating → Removed from EndpointSlice
- kube-proxy watches EndpointSlice → updates iptables/IPVS rules

### 18.8 Job Controller

Manages Pod lifecycle for batch workloads. Unlike Deployments, Jobs track completions:
- Creates pods up to `spec.parallelism`
- Retries failed pods up to `spec.backoffLimit`
- Marks Job Complete when `spec.completions` pods succeed

### 18.9 StatefulSet Controller

Creates Pods with stable identities for stateful applications:
- Ordered creation/deletion (pod-0 before pod-1)
- Each Pod gets its own PVC
- Stable DNS: `pod-0.service.namespace.svc.cluster.local`

### 18.10 Garbage Collector

Uses `ownerReferences` to clean up orphaned objects:
- Deployment deleted → GC deletes ReplicaSets (cascade)
- ReplicaSet deleted → GC deletes Pods
- Supports: Foreground (parent waits for children), Background (parent deleted immediately), Orphan (children not deleted)

### 18.11 PersistentVolume Controller

Manages PV binding to PVCs. Relevant for Deployments when:
- Deployment uses PVCs for shared volumes
- Multiple pods in deployment share a `ReadWriteMany` PV
- Static vs dynamic provisioning

---

## 19. Internal Working Concepts — Informers, Work Queues & Reconciliation

### 19.1 The Informer Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                    INFORMER INTERNALS FOR DEPLOYMENTS                │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Reflector (Deployment Informer)                            │    │
│  │  Initial LIST: GET /apis/apps/v1/deployments                │    │
│  │  Ongoing WATCH: GET /apis/apps/v1/deployments?watch=true    │    │
│  └──────────────────────────────┬──────────────────────────────┘    │
│                                 │ Events: ADDED/MODIFIED/DELETED    │
│                                 ▼                                    │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Local Cache (Indexer/Lister)                               │    │
│  │  Thread-safe in-memory store                                │    │
│  │  Controllers READ FROM HERE (not API server)               │    │
│  └────────────────────────────┬────────────────────────────────┘    │
│                               │                                      │
│                 OnAdd/OnUpdate/OnDelete handlers                      │
│                               │                                      │
│                               ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │  Work Queue (rate-limited, deduplicated)                    │    │
│  │  Keys: "default/my-deployment"                             │    │
│  │  Multiple events → single item (dedup)                      │    │
│  │  Backoff on error: 5ms → 10ms → 20ms → ... → 1000s         │    │
│  └────────────────────────────┬────────────────────────────────┘    │
│                               │                                      │
│               ┌───────────────┼──────────────────┐                  │
│               ▼               ▼                  ▼                  │
│        Worker 1          Worker 2         Worker N                  │
│         (5 by default for Deployment controller)                    │
│               │                                                      │
│               ▼                                                      │
│         reconcile("default/my-deployment")                          │
│         → Read from local cache                                      │
│         → Make API calls to create/update RS/Pods                   │
└──────────────────────────────────────────────────────────────────────┘
```

### 19.2 Work Queue Guarantees

```bash
# Key properties of work queues:
# 1. DEDUPLICATION: 100 events for same deployment → 1 reconcile
# 2. ORDERING: No specific order guarantee
# 3. BACKOFF: Failed reconciliations retry with exponential backoff
# 4. PARALLELISM: Multiple workers process different deployments concurrently
# 5. RATE LIMITING: Prevents thundering herd after bulk changes

# Monitor work queue depth
curl -sk https://127.0.0.1:10257/metrics | \
  grep 'workqueue_depth{name="deployment"}' | grep -v "^#"

# High queue depth → controller struggling to keep up
# Check: controller manager CPU limits, --concurrent-deployment-syncs flag
```

---

## 20. API Server and etcd Interaction

### 20.1 The Golden Rule

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  CONTROLLERS NEVER ACCESS ETCD DIRECTLY.                             ║
║                                                                      ║
║  ALL operations flow:                                                ║
║  Deployment Controller → kube-apiserver → etcd                      ║
║                                                                      ║
║  The API server provides for Deployments:                           ║
║  1. AuthN/AuthZ: RBAC controls who can create/update Deployments    ║
║  2. Validation: Ensures Deployment spec is well-formed              ║
║  3. Defaulting: Fills in default values (revisionHistoryLimit=10)   ║
║  4. Admission: PodSecurity, resource quota checks                   ║
║  5. Audit: Logs every Deployment create/update/delete               ║
║  6. Watch cache: Efficient fan-out to all controllers + kubelets    ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 20.2 Deployment etcd Key Structure

```bash
# Deployments stored at:
# /registry/deployments/<namespace>/<name>

# ReplicaSets:
# /registry/replicasets/<namespace>/<name>

# Pods:
# /registry/pods/<namespace>/<name>

# Verify (from control plane with etcdctl):
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl get /registry/deployments --prefix --keys-only \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### 20.3 API Server Request Flow for Deployment Update

```bash
# What happens when you run:
# kubectl set image deployment/my-app nginx=nginx:1.25

# 1. kubectl builds PATCH request
# PATCH /apis/apps/v1/namespaces/default/deployments/my-app

# 2. API server:
#    - Authenticates kubectl
#    - Authorizes via RBAC
#    - Validates the patch
#    - Applies defaulting
#    - Writes to etcd

# 3. API server watch cache notified:
#    - Deployment controller informer receives MODIFIED event
#    - Event queued in work queue

# 4. Deployment controller worker:
#    - Reads deployment from cache
#    - Detects template change (new image)
#    - Creates new ReplicaSet via API server
#    - Scales new RS up, old RS down

# Observe the API calls (-v=9 for full HTTP):
kubectl set image deployment/my-app nginx=nginx:1.25 -v=6 2>&1 | grep PATCH
```

---

## 21. Leader Election — HA for Deployment Infrastructure

### 21.1 Deployment Controller Leader Election

```bash
# Multiple kube-controller-manager instances compete
# Only ONE actively reconciles Deployments at a time
# Others are warm standbys

kubectl get lease kube-controller-manager -n kube-system -o yaml
# spec:
#   holderIdentity: "k8s-cp-1_abc123"   ← Active leader
#   renewTime: "2025-01-01T14:30:00Z"   ← Last heartbeat
#   leaseDurationSeconds: 15
#   leaseTransitions: 2

# During leader failover (15-30 seconds):
# - Existing pods continue running (no controller needed)
# - New rollouts pause
# - HPA scaling pauses
# - After leader election: all queued reconciliations resume
```

### 21.2 Scheduler Leader Election

```bash
# The scheduler also uses leader election
kubectl get lease kube-scheduler -n kube-system -o yaml

# During scheduler failover:
# - Existing pods keep running (scheduler only places NEW pods)
# - Pending pods wait until new leader elected (~30 seconds)
# - HPA-created pods queue up but don't get placed until scheduler recovers
```

### 21.3 Leader Election Flags

| Flag | Default | Component | Description |
|---|---|---|---|
| `--leader-elect` | `true` | Both | Enable leader election |
| `--leader-elect-lease-duration` | `15s` | Both | Lease validity |
| `--leader-elect-renew-deadline` | `10s` | Both | Must renew before this |
| `--leader-elect-retry-period` | `2s` | Both | Standby check interval |

### 21.4 HA Best Practices

```bash
# 1. Run 3 control plane nodes (odd number for quorum)
kubectl get pods -n kube-system | grep -E "controller-manager|scheduler"
# Should see 3 pods of each

# 2. Monitor lease transitions (high count = instability)
kubectl get lease -n kube-system | grep -E "controller-manager|scheduler"

# 3. Use load-balanced API server VIP for all clients
kubectl config view | grep server
# Should point to VIP, not specific control-plane node

# 4. For large clusters, tune lease duration
# --leader-elect-lease-duration=30s (more stable, slower failover)
# --leader-elect-renew-deadline=25s
```

---

## 22. Performance Tuning for Deployments

### 22.1 kube-controller-manager Flags

| Flag | Default | Production Recommendation |
|---|---|---|
| `--concurrent-deployment-syncs` | 5 | 10-20 for clusters with frequent deployments |
| `--concurrent-replicaset-syncs` | 5 | 10-20 |
| `--concurrent-endpoint-syncs` | 5 | 10-20 |
| `--concurrent-endpointslice-syncs` | 5 | 10-20 |
| `--deployment-controller-sync-period` | 30s | 30s (default is fine) |
| `--kube-api-qps` | 20 | 50-100 for large clusters |
| `--kube-api-burst` | 30 | 100-200 for large clusters |

### 22.2 Deployment Spec Optimization

```yaml
# Production performance settings
spec:
  revisionHistoryLimit: 3     # Default 10; reduce to save etcd space
  progressDeadlineSeconds: 300  # Fail fast; default 600 is often too slow

  strategy:
    rollingUpdate:
      maxSurge: "25%"         # String percentage for dynamic calculation
      maxUnavailable: "25%"   # Allows faster rollouts vs 0

  template:
    spec:
      # Terminate quickly on node drain
      terminationGracePeriodSeconds: 30  # Match your app's shutdown time

      # Faster scheduling decisions
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: ScheduleAnyway  # Soft constraint for speed
          labelSelector:
            matchLabels:
              app: my-app
```

### 22.3 HPA Performance Tuning

```yaml
# HPA controller flags (in kube-controller-manager)
# --horizontal-pod-autoscaler-sync-period=15s   (how often HPA checks)
# --horizontal-pod-autoscaler-downscale-stabilization=5m
# --horizontal-pod-autoscaler-cpu-initialization-period=5m
# --horizontal-pod-autoscaler-initial-readiness-delay=30s

# HPA behavior for production
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300   # 5 min before scaling down (prevent thrashing)
  scaleUp:
    stabilizationWindowSeconds: 30    # Scale up quickly (30 seconds)
```

### 22.4 Resource Limits for Control Plane Components

```yaml
# kube-controller-manager resource limits (in static pod manifest)
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 2000m   # High limit for burst during many simultaneous deployments
    memory: 2Gi
```

---

## 23. Security Hardening Practices

### 23.1 RBAC for Deployments

```yaml
# Minimal role for CI/CD pipeline
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-manager
  namespace: production
rules:
  # Can update existing deployments (rolling updates)
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "update", "patch"]
  # Can check rollout status
  - apiGroups: ["apps"]
    resources: ["replicasets"]
    verbs: ["get", "list"]
  # Can check pod status
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list"]
---
# Read-only role for developers
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-viewer
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
```

### 23.2 Pod Security for Deployment Pods

```yaml
# Apply Pod Security Standards to namespace
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted

# Required pod security settings (restricted profile)
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
```

### 23.3 Network Policies for Deployments

```yaml
# Default deny + explicit allow for production deployments
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-server-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - port: 5432
    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - port: 53
          protocol: UDP
```

---

## 24. Monitoring and Observability

### 24.1 Key Deployment Metrics

| Metric | Description | Alert Threshold |
|---|---|---|
| `kube_deployment_status_replicas_unavailable` | Unavailable replicas | > 0 for > 5 min |
| `kube_deployment_status_replicas_ready` | Ready replicas | < spec.replicas for > 5 min |
| `kube_deployment_spec_replicas` | Desired replicas | Used as denominator |
| `kube_deployment_status_observed_generation` | Controller has seen latest version | != spec.generation |
| `kube_pod_container_status_restarts_total` | Container restarts | Rate > 1/hour |
| `kube_pod_status_phase{phase="Failed"}` | Failed pods | > 0 |
| `kube_replicaset_status_replicas_ready` | Ready RS replicas | Used during rollouts |
| `kube_horizontalpodautoscaler_status_current_replicas` | HPA current replicas | For scaling visibility |
| `container_cpu_cfs_throttled_seconds_total` | CPU throttling | Rate > 10% |
| `container_memory_working_set_bytes` | Memory usage | > 80% of limit |

### 24.2 Prometheus Alert Rules

```yaml
groups:
  - name: kubernetes-deployments
    rules:
      # Deployment has unavailable replicas
      - alert: KubeDeploymentReplicasMismatch
        expr: |
          kube_deployment_spec_replicas != 
          kube_deployment_status_replicas_available
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} has unavailable replicas"

      # Rollout is stalled
      - alert: KubeDeploymentRolloutStuck
        expr: |
          kube_deployment_status_condition{condition="Progressing",status="false"} == 1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} rollout is stalled"

      # High restart rate (CrashLoopBackOff)
      - alert: KubePodCrashLooping
        expr: |
          rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15 > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"

      # HPA at maximum replicas
      - alert: KubeHPAMaxReplicas
        expr: |
          kube_horizontalpodautoscaler_status_current_replicas >= 
          kube_horizontalpodautoscaler_spec_max_replicas
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "HPA {{ $labels.namespace }}/{{ $labels.hpa }} is at maximum replicas"
```

### 24.3 Metrics Endpoints

```bash
# kube-controller-manager metrics (includes Deployment controller)
curl -sk https://127.0.0.1:10257/metrics | grep 'deployment\|replicaset\|workqueue'

# kube-scheduler metrics
curl -sk https://127.0.0.1:10259/metrics | grep 'scheduler_pending_pods'

# kube-proxy metrics (for Service routing)
curl -s http://127.0.0.1:10249/metrics | grep 'sync_proxy_rules'
```

---

## 25. Troubleshooting Deployments — Real kubectl Commands

### 25.1 Pods Not Created

```bash
# ── STEP 1: Overall status ────────────────────────────────────────────────
kubectl get deployment <n> -n <namespace>
# Check: READY column shows desired/available

kubectl get replicasets -l app=<label> -n <namespace>
# Check: Multiple RS? Are they at correct replica counts?

kubectl get pods -l app=<label> -n <namespace>
# Check: STATUS column (Pending, CrashLoopBackOff, ImagePullBackOff, etc.)

# ── STEP 2: Get events ────────────────────────────────────────────────────
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -20
kubectl describe deployment <n> -n <namespace> | tail -30

# ── STEP 3: Describe failing pod ─────────────────────────────────────────
kubectl describe pod <failing-pod> -n <namespace>
# Key sections: Events, Conditions, Init Containers

# ── COMMON CAUSES AND FIXES ───────────────────────────────────────────────

# Insufficient resources (Pending state)
# Error: 0/3 nodes available: 3 Insufficient cpu
kubectl top nodes
kubectl describe nodes | grep -A 5 "Allocated resources:"
# Fix: Scale up cluster or reduce pod resource requests

# Image pull failure
# Error: Back-off pulling image "registry/image:tag"
kubectl describe pod <n> | grep -A 5 "Events:"
# Fix: Check image name, registry credentials, network
kubectl create secret docker-registry regcred \
  --docker-server=<server> \
  --docker-username=<user> \
  --docker-password=<token>

# ConfigMap/Secret not found
# Error: secret "db-credentials" not found
kubectl get secrets -n <namespace>
kubectl get configmaps -n <namespace>
# Fix: Create the missing objects first

# Node affinity unsatisfiable
# Error: 0/3 nodes available: 3 node(s) didn't match node selector
kubectl get nodes --show-labels
kubectl get deployment <n> -o yaml | grep -A 10 "affinity\|nodeSelector"
# Fix: Check label existence on nodes; adjust affinity rules
```

### 25.2 Deployment Stuck / Rolling Update Frozen

```bash
# Check rollout status
kubectl rollout status deployment/<n> --timeout=30s
# "Waiting for rollout to finish: X of Y updated replicas are available"

# Check what happened
kubectl describe deployment <n> | grep -A 5 "Conditions:"
# Look for: ProgressDeadlineExceeded, ReplicaFailure

# View current ReplicaSets
kubectl get replicasets -l app=<label> -o wide
# Both old and new RS should show replicas

# Check new pods
kubectl get pods -l app=<label>
# Look for: Pending, CrashLoopBackOff, ImagePullBackOff, Error

# New pod details
kubectl describe pod <new-pod-name>
kubectl logs <new-pod-name> --previous  # If crashed

# Check readiness probe is passing
kubectl get pod <n> -o yaml | grep readiness -A 10
# Is the endpoint /health/ready actually accessible?
kubectl exec <pod> -- curl -f http://localhost:8080/health/ready

# ROLLBACK if update is bad
kubectl rollout undo deployment/<n>
kubectl rollout status deployment/<n>
```

### 25.3 Node NotReady — Impact on Deployments

```bash
# Identify NotReady nodes
kubectl get nodes | grep NotReady

# See what's running on problem node
kubectl get pods --all-namespaces \
  --field-selector spec.nodeName=<node-name>

# Check if replacement pods are being created
kubectl get pods -l app=<label> -o wide

# Manually cordon + drain for maintenance
kubectl cordon <node-name>
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data

# After fix: uncordon
kubectl uncordon <node-name>
```

---

## 26. Lab: Performing Troubleshooting of Common Deployment Scenarios

### 26.1 Scenario 1 — Wrong Image Tag

```bash
# Create a deployment with a bad image
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-image
spec:
  replicas: 2
  selector:
    matchLabels:
      app: broken-image
  template:
    metadata:
      labels:
        app: broken-image
    spec:
      containers:
        - name: app
          image: nginx:99.99.99-does-not-exist    # Bad tag
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
EOF

# Diagnose
kubectl get pods -l app=broken-image
# STATUS: ImagePullBackOff or ErrImagePull

kubectl describe pod $(kubectl get pods -l app=broken-image -o name | head -1)
# Events:
#   Warning  Failed  5s  kubelet  Failed to pull image "nginx:99.99.99-does-not-exist"

# Fix: Update to valid image tag
kubectl set image deployment/broken-image app=nginx:1.25

kubectl rollout status deployment/broken-image
kubectl get pods -l app=broken-image
# STATUS: Running

kubectl delete deployment broken-image
```

### 26.2 Scenario 2 — Insufficient Resources

```bash
# Create a deployment requesting too many resources
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-hog
spec:
  replicas: 3
  selector:
    matchLabels:
      app: resource-hog
  template:
    metadata:
      labels:
        app: resource-hog
    spec:
      containers:
        - name: app
          image: nginx:1.25
          resources:
            requests:
              cpu: "50"        # Requesting 50 CPU cores!
              memory: "100Gi"  # Requesting 100 GiB RAM!
EOF

# Diagnose
kubectl get pods -l app=resource-hog
# STATUS: Pending

kubectl describe pod $(kubectl get pods -l app=resource-hog -o name | head -1)
# Events:
#   Warning  FailedScheduling  0/3 nodes are available:
#   3 Insufficient cpu, 3 Insufficient memory

# Check actual node capacity
kubectl describe nodes | grep -A 8 "Allocated resources:"

# Fix: Adjust resource requests to realistic values
kubectl patch deployment resource-hog \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","resources":{"requests":{"cpu":"100m","memory":"128Mi"},"limits":{"cpu":"200m","memory":"256Mi"}}}]}}}}'

kubectl rollout status deployment/resource-hog
kubectl delete deployment resource-hog
```

### 26.3 Scenario 3 — Failed Readiness Probe

```bash
# Create deployment with wrong readiness probe path
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bad-probe
spec:
  replicas: 2
  selector:
    matchLabels:
      app: bad-probe
  template:
    metadata:
      labels:
        app: bad-probe
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          readinessProbe:
            httpGet:
              path: /this-path-does-not-exist    # Wrong path!
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
EOF

# Diagnose
kubectl get pods -l app=bad-probe
# STATUS: Running but READY: 0/1
# The pod runs but fails readiness → never added to Service endpoints!

kubectl describe pod $(kubectl get pods -l app=bad-probe -o name | head -1)
# Events:
#   Warning  Unhealthy  readiness probe failed: HTTP probe failed with statuscode: 404

# Test manually
kubectl exec $(kubectl get pods -l app=bad-probe -o name | head -1) \
  -- curl -s -o /dev/null -w "%{http_code}" http://localhost:80/this-path-does-not-exist
# 404

# Fix: Update readiness probe to correct path
kubectl patch deployment bad-probe \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","readinessProbe":{"httpGet":{"path":"/"}}}]}}}}'

kubectl rollout status deployment/bad-probe
kubectl get pods -l app=bad-probe
# READY: 1/1 now!

kubectl delete deployment bad-probe
```

### 26.4 Scenario 4 — OOMKilled

```bash
# Create a pod that exceeds its memory limit
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: memory-hog
spec:
  replicas: 1
  selector:
    matchLabels:
      app: memory-hog
  template:
    metadata:
      labels:
        app: memory-hog
    spec:
      containers:
        - name: stress
          image: polinux/stress:latest
          command: ["stress"]
          args: ["--vm", "1", "--vm-bytes", "256M", "--vm-hang", "1"]
          resources:
            requests:
              memory: 50Mi
            limits:
              memory: 100Mi    # Container will try to use 256M but limit is 100M
EOF

# Watch OOMKill
kubectl get pods -l app=memory-hog -w

# After OOMKill:
kubectl describe pod $(kubectl get pods -l app=memory-hog -o name)
# State: Last State:
#   Reason: OOMKilled
#   Exit Code: 137

# Check events
kubectl get events | grep -i OOM

# Fix: Increase memory limit or fix memory leak
kubectl patch deployment memory-hog \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"stress","args":["--vm","1","--vm-bytes","50M","--vm-hang","1"],"resources":{"limits":{"memory":"256Mi"}}}]}}}}'

kubectl delete deployment memory-hog
```

### 26.5 Scenario 5 — Stuck Rolling Update Due to CrashLoopBackOff

```bash
# Simulate a bad deployment update
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crashloop-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: crashloop-demo
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0    # Won't remove old pods until new ones are ready
  template:
    metadata:
      labels:
        app: crashloop-demo
    spec:
      containers:
        - name: app
          image: nginx:1.25
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
EOF

kubectl rollout status deployment/crashloop-demo

# Now update to a "broken" version
kubectl set image deployment/crashloop-demo \
  app=busybox:1.36

# BusyBox exits immediately → CrashLoopBackOff
kubectl get pods -l app=crashloop-demo
# Some old nginx pods still running (maxUnavailable: 0)
# New busybox pod in CrashLoopBackOff

# Rolling update STUCK: new pods never become ready
kubectl rollout status deployment/crashloop-demo --timeout=30s
# Waiting for rollout to finish: 1 out of 3 new replicas have been updated

# Diagnose
kubectl describe pod $(kubectl get pods -l app=crashloop-demo --field-selector=status.phase!=Running -o name | head -1)
kubectl logs $(kubectl get pods -l app=crashloop-demo --field-selector=status.phase!=Running -o name | head -1) --previous

# Rollback!
kubectl rollout undo deployment/crashloop-demo

kubectl rollout status deployment/crashloop-demo
kubectl get pods -l app=crashloop-demo
# All pods Running again with nginx:1.25

kubectl delete deployment crashloop-demo
```

---

## 27. Disaster Recovery Concepts

### 27.1 Stateless Controllers — Safe Restart Any Time

```
kube-controller-manager is COMPLETELY STATELESS.
All Deployment, ReplicaSet, and HPA state lives in etcd.

If kube-controller-manager crashes:
  → Running Pods CONTINUE running (kubelet manages them)
  → Services CONTINUE working (kube-proxy rules intact)
  → HPA STOPS scaling (no controller to read metrics)
  → New rollouts PAUSE (no controller to manage RS)

Recovery (after leader election, ~30 seconds):
  → Controller re-lists ALL Deployments, RS, Pods from API server
  → Reconciles any missed events during downtime
  → Fully operational within seconds of leader election
  → No data loss — all desired state was in etcd
```

### 27.2 etcd Backup and Deployment Recovery

```bash
# Backup etcd (includes ALL Deployment specs, RS history, Pod specs)
ETCDCTL="etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key"

kubectl exec -n kube-system etcd-$(hostname) -- \
  $ETCDCTL snapshot save /tmp/etcd-backup-$(date +%Y%m%d-%H%M%S).db

# After cluster restore from etcd backup:
# - All Deployment specs restored
# - All rollout history preserved (revisionHistoryLimit RS objects)
# - HPA configurations restored
# - Service configurations restored

# GitOps as DR: Store all Deployments in Git
# Recovery: kubectl apply -f deployments/
# → Creates all deployments; controllers reconcile; pods start running
```

### 27.3 Deployment-Level DR Practices

```bash
# PodDisruptionBudget: protect Deployments during node drain/upgrade
cat << 'EOF' | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
  namespace: production
spec:
  minAvailable: 2    # At least 2 pods must remain available
  selector:
    matchLabels:
      app: api-server
EOF

# With PDB:
kubectl drain <node> --ignore-daemonsets
# Will wait/fail if draining would violate the PDB

# TopologySpread for inherent resilience
spec:
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: api-server
# → Pods spread across zones automatically
# → Zone failure only loses 1/3 of pods (with 3 zones)
```

---

## 28. Comparison — kube-apiserver vs kube-scheduler vs kube-controller-manager

| Dimension | kube-apiserver | kube-scheduler | kube-controller-manager |
|---|---|---|---|
| **Core Function** | REST gateway; sole etcd accessor | Assigns Pods to Nodes | Runs all reconciliation loops |
| **Deployment role** | Validates/stores Deployment YAML | Assigns Deployment Pods to Nodes | Manages RS lifecycle; drives rolling updates |
| **HA Model** | Active-Active (multiple serve) | Active-Passive (leader election) | Active-Passive (leader election) |
| **etcd Access** | YES (direct) | NO (via API server) | NO (via API server) |
| **Port** | 6443 | 10259 | 10257 |
| **Failure impact** | Cluster management halts | New Pods not scheduled | Rolling updates/scaling paused |
| **Recovery time** | Immediate (behind LB) | ~30s (leader election) | ~30s (leader election) |
| **HPA integration** | Stores HPA specs | N/A | Runs HPA controller |
| **Service integration** | Stores Service specs | N/A | EndpointSlice controller |
| **Audit logging** | Every API call logged | N/A | N/A |

---

## 29. ASCII Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║               KUBERNETES DEPLOYMENTS — COMPLETE ARCHITECTURE                         ║
╠══════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                      ║
║  DEVELOPER / CI-CD                                                                   ║
║  kubectl apply -f deployment.yaml                                                    ║
║  kubectl scale deployment/my-app --replicas=5                                        ║
║  kubectl set image deployment/my-app nginx=nginx:1.25                                ║
║           │                                                                          ║
║           │ HTTPS :6443                                                              ║
║           ▼                                                                          ║
║  ┌──────────────────────────────────────────────────────────────────────────────┐    ║
║  │                         CONTROL PLANE                                        │    ║
║  │                                                                              │    ║
║  │  ┌────────────────────────────────────────────────────────────────────────┐  │    ║
║  │  │                     kube-apiserver :6443                               │  │    ║
║  │  │  AuthN → AuthZ → Admission → Validation → Audit → Watch Cache         │  │    ║
║  │  │  Stores: Deployments, ReplicaSets, Pods, Services, HPA                │  │    ║
║  │  └───────────────────────────────────┬────────────────────────────────────┘  │    ║
║  │                                      │                                       │    ║
║  │         ┌────────────────────────────▼───────────────────────────────┐       │    ║
║  │         │                    etcd :2379                              │       │    ║
║  │         │  /registry/deployments/   /registry/replicasets/           │       │    ║
║  │         │  /registry/pods/          /registry/services/              │       │    ║
║  │         │  /registry/horizontalpodautoscalers/                       │       │    ║
║  │         └────────────────────────────────────────────────────────────┘       │    ║
║  │                                                                              │    ║
║  │  ┌─────────────────────────────────────────────────────────────────────┐    │    ║
║  │  │          kube-controller-manager :10257 (Leader-Elected)            │    │    ║
║  │  │  ┌──────────────────┐  ┌───────────────┐  ┌───────────────────┐   │    │    ║
║  │  │  │Deployment Ctrl   │  │ ReplicaSet Ctrl│  │  HPA Controller  │   │    │    ║
║  │  │  │ Watch→Compare    │  │ Watch→Compare  │  │ Watch→Metrics    │   │    │    ║
║  │  │  │ Act: manage RS   │  │ Act: manage Pod│  │ Act: scale Deploy│   │    │    ║
║  │  │  └──────────────────┘  └───────────────┘  └───────────────────┘   │    │    ║
║  │  │  ┌──────────────────────────────────────────────────────────────┐  │    │    ║
║  │  │  │ EndpointSlice Ctrl │ Node Ctrl │ Service Ctrl │ GC Ctrl      │  │    │    ║
║  │  │  └──────────────────────────────────────────────────────────────┘  │    │    ║
║  │  │  All controllers → API server (NEVER → etcd directly)              │    │    ║
║  │  └─────────────────────────────────────────────────────────────────────┘    │    ║
║  │                                                                              │    ║
║  │  ┌─────────────────────────────────────────────────────────────────────┐    │    ║
║  │  │          kube-scheduler :10259 (Leader-Elected)                     │    │    ║
║  │  │  Watch unscheduled pods → Filter nodes → Score nodes → Bind        │    │    ║
║  │  │  Applies: nodeAffinity, podAffinity, taints, topologySpread        │    │    ║
║  │  └─────────────────────────────────────────────────────────────────────┘    │    ║
║  │                                                                              │    ║
║  │  ┌─────────────────────────────────────────────────────────────────────┐    │    ║
║  │  │  metrics-server :443                                                │    │    ║
║  │  │  Scrapes kubelet cadvisor → HPA reads CPU/memory metrics           │    │    ║
║  │  └─────────────────────────────────────────────────────────────────────┘    │    ║
║  └──────────────────────────────────────────────────────────────────────────────┘    ║
║                                │ Pod spec with nodeName assigned                     ║
║              ┌─────────────────┼───────────────────────┐                            ║
║              │                 │                       │                            ║
║  ┌───────────▼──────────┐  ┌───▼──────────────────┐  ┌▼──────────────────────┐    ║
║  │  WORKER NODE 1       │  │  WORKER NODE 2       │  │  WORKER NODE 3         │    ║
║  │                      │  │                      │  │                        │    ║
║  │  kubelet :10250      │  │  kubelet :10250      │  │  kubelet :10250        │    ║
║  │  kube-proxy (IPVS)   │  │  kube-proxy (IPVS)   │  │  kube-proxy (IPVS)    │    ║
║  │                      │  │                      │  │                        │    ║
║  │  ┌────────────────┐  │  │  ┌────────────────┐  │  │  ┌────────────────┐   │    ║
║  │  │ Pod (v2 - new) │  │  │  │ Pod (v1 - old) │  │  │  │ Pod (v2 - new) │   │    ║
║  │  │ nginx:1.25     │  │  │  │ nginx:1.24     │  │  │  │ nginx:1.25     │   │    ║
║  │  │ READY: true    │  │  │  │ Terminating    │  │  │  │ READY: true    │   │    ║
║  │  └────────────────┘  │  │  └────────────────┘  │  │  └────────────────┘   │    ║
║  │                      │  │                      │  │                        │    ║
║  │  ┌────────────────┐  │  │  ┌────────────────┐  │  │  ┌────────────────┐   │    ║
║  │  │Service routing │  │  │  │Service routing │  │  │  │Service routing │   │    ║
║  │  │ClusterIP:80 → │  │  │  │ClusterIP:80 → │  │  │  │ClusterIP:80 → │   │    ║
║  │  │PodIP:8080      │  │  │  │PodIP:8080      │  │  │  │PodIP:8080      │   │    ║
║  │  └────────────────┘  │  │  └────────────────┘  │  │  └────────────────┘   │    ║
║  └──────────────────────┘  └──────────────────────┘  └────────────────────────┘    ║
║                                                                                      ║
║  DEPLOYMENT HIERARCHY:                HPA FEEDBACK LOOP:                            ║
║  Deployment → ReplicaSet → Pods       metrics-server → HPA → Deployment.replicas    ║
║                                                                                      ║
║  SERVICE TYPES:                                                                      ║
║  ClusterIP: Internal only │ NodePort: Node IP access │ LB: Cloud external           ║
╚══════════════════════════════════════════════════════════════════════════════════════╝
```

---

## 30. Real-World Production Use Cases

### 30.1 Zero-Downtime Release Pipeline

```yaml
# GitOps-based release: bump image tag → rollout starts automatically

# CI/CD pipeline updates the image tag:
# image: registry.company.com/api:${COMMIT_SHA}

# Deployment configured for zero-downtime:
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%         # Create extra pods first
    maxUnavailable: 0     # Never reduce capacity

# readinessProbe ensures traffic only goes to healthy new pods
readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5

# preStop hook drains existing connections
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]

terminationGracePeriodSeconds: 60
```

### 30.2 Auto-Scaling for Variable Traffic

```yaml
# E-commerce platform with daily traffic spikes
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: checkout-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: checkout-service
  minReplicas: 3       # Minimum for HA
  maxReplicas: 50      # Maximum for peak load
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60    # React quickly to traffic spike
    scaleDown:
      stabilizationWindowSeconds: 300   # Slow down scale-down to avoid thrashing
```

### 30.3 Multi-Region Active-Active

```yaml
# Production deployment across 3 AZs with zone spreading
spec:
  replicas: 9    # 3 per zone
  template:
    spec:
      topologySpreadConstraints:
        - maxSkew: 0           # Perfectly even
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: payment-service
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node.kubernetes.io/instance-type
                    operator: NotIn
                    values:
                      - spot    # Don't use spot instances for payment service
```

---

## 31. Best Practices for Production Environments

### 31.1 Deployment Spec

- **Always set resource requests AND limits** — requests drive scheduling; limits prevent node exhaustion
- **Configure readiness probes carefully** — wrong probe path keeps pods out of service rotation forever
- **Set appropriate terminationGracePeriodSeconds** — must be greater than max request time + preStop sleep
- **Use `revisionHistoryLimit: 3-5`** — default 10 wastes etcd space; 3 is sufficient for rollback
- **Set `progressDeadlineSeconds`** — fail fast (300s) rather than silently stuck (default 600s)
- **Always specify `maxUnavailable: 0` for production** — guarantees zero-downtime with proper readiness probes

### 31.2 Scaling

- **Set resource requests before enabling HPA** — HPA CPU% is meaningless without a request value
- **Configure HPA scaleDown stabilization** — default 5 minutes prevents thrashing
- **Set reasonable minReplicas** — 1 means downtime if the single pod crashes; use 2+ for HA
- **Test HPA behavior** before enabling in production — generate load and verify scaling works as expected

### 31.3 Services

- **Use named ports** in both Deployment and Service — `targetPort: http` is more maintainable than `targetPort: 80`
- **Set `sessionAffinity: ClientIP`** only when your app truly requires sticky sessions
- **Use LoadBalancer source ranges** to restrict external access: `loadBalancerSourceRanges`
- **Consider Ingress over LoadBalancer** — Ingress reduces cloud LB count; supports path-based routing and TLS

### 31.4 Operational

- **Annotate rollouts with `kubernetes.io/change-cause`** — makes rollout history useful
- **Create PodDisruptionBudgets** — protects Deployments during node maintenance
- **Use topologySpreadConstraints** for HA across zones — more robust than pod anti-affinity
- **Test disaster recovery** regularly — verify rollbacks work; practice drain procedures

---

## 32. Common Mistakes and Pitfalls

### 32.1 Selector/Template Label Mismatch

**Mistake:**
```yaml
selector:
  matchLabels:
    app: my-api
template:
  metadata:
    labels:
      app: my-service   # DIFFERENT from selector!
```
**Impact:** Deployment controller creates pods but can't find them → infinite pod creation loop.  
**Fix:** `selector.matchLabels` must be a subset of `template.metadata.labels`.

---

### 32.2 No readinessProbe with maxUnavailable: 0

**Mistake:** Setting `maxUnavailable: 0` but not configuring a readinessProbe.  
**Impact:** Kubernetes marks new pods as Ready as soon as they START (not when they're actually ready) → traffic sent to pods before they can handle it.  
**Fix:** Always configure a proper readinessProbe that tests actual application health.

---

### 32.3 HPA Without Resource Requests

**Mistake:** Creating an HPA targeting CPU utilization, but pods have no `resources.requests.cpu`.  
**Impact:** HPA reports `<unknown>/50%` for CPU — cannot calculate; HPA doesn't scale.  
**Fix:** Always set `resources.requests.cpu` on pods managed by HPA.

---

### 32.4 Forgetting terminationGracePeriod

**Mistake:** Default 30s termination grace period, but app handles requests that take up to 60s.  
**Impact:** Pod killed mid-request → request errors for users during rolling updates.  
**Fix:** Set `terminationGracePeriodSeconds: 90` (request max time + 30s buffer). Add preStop hook to drain gracefully.

---

### 32.5 Rolling Update with 0 maxSurge AND 0 maxUnavailable

**Mistake:**
```yaml
rollingUpdate:
  maxSurge: 0
  maxUnavailable: 0
```
**Impact:** Update stalls immediately — can't create new pods (maxSurge=0) but can't remove old ones (maxUnavailable=0).  
**Fix:** At least one of maxSurge or maxUnavailable must be non-zero.

---

### 32.6 Using imagePullPolicy: Always Without Tag

**Mistake:** `image: nginx:latest` with `imagePullPolicy: Always`.  
**Impact:** Different nodes may run different "latest" versions; `latest` tag changes unpredictably.  
**Fix:** Always use specific image tags (`nginx:1.25.3`). Use `imagePullPolicy: IfNotPresent` for performance.

---

### 32.7 Service Selector Doesn't Match Pod Labels

**Mistake:** Service selector: `app: my-api`, but Pod has label: `app: myapi` (missing hyphen).  
**Impact:** Service has no endpoints; all traffic goes nowhere.  
**Fix:** Verify with `kubectl get endpoints <service-name>` — empty endpoints = selector mismatch.

---

## 33. Interview Questions — Beginner to Advanced

### Beginner Level

**Q1: What is the difference between a Deployment and a ReplicaSet?**

**A:** A ReplicaSet ensures a specified number of Pod replicas are running at all times. It provides self-healing but has no concept of versioning or update management.

A Deployment manages ReplicaSets. When you update a Deployment's Pod template, the Deployment controller creates a new ReplicaSet with the new template and manages the transition (scaling new RS up, old RS down). The Deployment maintains history of old ReplicaSets for rollback capability.

In production, you almost never create ReplicaSets directly — you create Deployments and let the Deployment controller manage the ReplicaSets.

---

**Q2: What is the difference between ClusterIP, NodePort, and LoadBalancer Service types?**

**A:**
- **ClusterIP**: Creates a stable internal IP address accessible only within the cluster. DNS name: `service.namespace.svc.cluster.local`. Use for internal microservice communication.
- **NodePort**: Extends ClusterIP by opening a static port (30000-32767) on every node's IP. External clients access via `NodeIP:NodePort`. Use for development or on-premises clusters without cloud LBs.
- **LoadBalancer**: Extends NodePort by provisioning an external cloud load balancer with a public IP. Use for production external access on cloud platforms (AWS, GCP, Azure).

---

**Q3: What is a readinessProbe and why is it important for Deployments?**

**A:** A readinessProbe periodically checks if a container is ready to accept traffic. Only containers with passing readiness probes are included in Service Endpoints.

This is critical for Deployments because:
1. **Rolling updates**: New pods are only switched into production traffic after their readiness probe passes, ensuring zero-downtime updates
2. **Startup handling**: Slow-starting apps (JVM warm-up, DB migration) don't receive traffic until ready
3. **Graceful degradation**: If a pod becomes overloaded or starts failing health checks, it's automatically removed from routing

Without a readiness probe, traffic is sent to pods that might not be ready yet → user-visible errors during deployments.

---

### Intermediate Level

**Q4: How does HPA calculate the desired number of replicas, and what conditions must be met for it to work?**

**A:** HPA uses this formula:
```
desiredReplicas = ceil(currentReplicas × (currentMetricValue / targetMetricValue))
```

For CPU utilization (the most common):
- `currentMetricValue` = average CPU usage across all pods (in millicores)
- `targetMetricValue` = target CPU value based on the percentage target and requests

For this to work, **three conditions must be met**:
1. The metrics-server must be deployed and healthy
2. Pods MUST have `resources.requests.cpu` set — HPA calculates CPU% relative to requests
3. The HPA must be attached to the correct Deployment/RS

HPA respects `minReplicas` and `maxReplicas` boundaries after calculating the desired count. Scale-down has a stabilization window (default 5 minutes) to prevent thrashing.

---

**Q5: Explain the difference between `maxSurge` and `maxUnavailable` in a rolling update strategy.**

**A:**

- **maxSurge**: The maximum number of **extra** pods that can exist above the desired count during an update. With `replicas: 3, maxSurge: 1`, you can have up to 4 pods total during the rollout. This is how new pods are created before old ones are deleted.

- **maxUnavailable**: The maximum number of pods that can be **unavailable** (not Ready) during an update. With `replicas: 3, maxUnavailable: 0`, you always have at least 3 Ready pods. This ensures zero-downtime updates.

Common production configurations:
- `maxSurge: 1, maxUnavailable: 0`: Zero-downtime, one-at-a-time (default safe choice)
- `maxSurge: 25%, maxUnavailable: 25%`: Faster rollout, accepts brief capacity reduction
- `maxSurge: 0, maxUnavailable: 1`: No extra capacity needed; one pod offline at a time

---

**Q6: Describe what happens when you run `kubectl rollout undo deployment/my-app`.**

**A:**
1. kubectl sends PATCH to the API server to update the Deployment's revision annotation
2. Deployment controller reads the previous ReplicaSet (the one that was scaled to 0 during the last rollout)
3. Controller sets that previous RS's `spec.replicas` back to the desired count
4. Controller sets the current RS's `spec.replicas` to 0
5. This triggers the RS controllers to create new (old version) pods and terminate new (current version) pods
6. The rollout proceeds with the same rolling update settings as forward updates
7. The Deployment's `status.currentRevision` is decremented

You can see available revisions with `kubectl rollout history deployment/my-app` and roll back to any specific revision with `--to-revision=N`.

---

### Advanced Level

**Q7: Explain how the Deployment controller handles the transition from old ReplicaSet to new ReplicaSet during a rolling update, and what could cause the update to stall.**

**A:** The Deployment controller implements rolling updates through a continuous reconciliation loop:

1. Pod template change detected → controller creates new RS with new template
2. Loop: while newRS.readyReplicas < desired:
   - If maxSurge allows: increment newRS.replicas by 1
   - Wait for new pod to become Ready (passes readiness probe)
   - Once Ready: decrement oldRS.replicas by 1 (if maxUnavailable allows)
3. Continues until newRS has all replicas, oldRS has 0

**Causes of rollout stalling:**
- **readinessProbe never passes**: New pods start but never become Ready → update waits forever → `ProgressDeadlineExceeded` condition after `spec.progressDeadlineSeconds`
- **OOMKilled**: New pods exceed memory limits → CrashLoopBackOff → never Ready
- **ImagePullBackOff**: Wrong image tag → pods can't start
- **Resource constraints**: Not enough cluster capacity for the surge pods → Pending
- **Init container failure**: Init container crashes → app container never starts
- **maxUnavailable: 0 AND maxSurge: 0**: Impossible state — stalls immediately (configuration error)

Diagnosis: `kubectl rollout status --timeout=5m` fails; `kubectl describe deployment` shows condition `Progressing=false`. Check new pod events: `kubectl describe pod <new-pod>`.

---

**Q8: How would you design a deployment strategy for a database schema migration that requires both old and new application versions to run simultaneously during the update?**

**A:** This is the classic "expand and contract" migration pattern:

**Step 1 (Expand)**: Migrate the database to support BOTH old and new schema. Apply backwards-compatible changes: add new columns/tables, keep old columns.

**Step 2 (Deploy new version)**: Deploy new app version that uses the new schema. Since the DB still has old schema support, old app pods still work. Rolling update proceeds safely with BOTH versions running.

**Step 3 (Verify)**: Confirm new version works correctly with the new schema.

**Step 4 (Contract)**: Once all pods are running new version, clean up old schema elements (remove old columns, etc.).

For Kubernetes deployment:
- Use `maxSurge: 1, maxUnavailable: 0` for the update
- Disable readiness probe (or use minimal probe) during schema migration
- Use PreJob (initContainer or Job) for the schema migration before the new Deployment is applied
- Implement graceful shutdown so old pods drain properly before terminating

The key insight: database migrations must be backwards-compatible if you want zero-downtime deployments. "Break the schema then deploy" → outage.

---

## 34. Cheat Sheet — Commands, Flags & Manifests

### 34.1 Deployment Management

```bash
# ── CREATE ────────────────────────────────────────────────────────────────
kubectl create deployment <n> --image=<image> --replicas=N
kubectl apply -f deployment.yaml
kubectl create deployment <n> --image=<image> --dry-run=client -o yaml > dep.yaml

# ── READ ──────────────────────────────────────────────────────────────────
kubectl get deployments [-n namespace] [-o wide] [-o yaml]
kubectl describe deployment <n>
kubectl get replicasets -l app=<label>
kubectl get pods -l app=<label> -o wide

# ── UPDATE ────────────────────────────────────────────────────────────────
kubectl set image deployment/<n> container=image:tag
kubectl patch deployment <n> -p '{"spec":{"replicas":5}}'
kubectl edit deployment <n>
kubectl apply -f updated.yaml

# ── ROLLOUT MANAGEMENT ────────────────────────────────────────────────────
kubectl rollout status deployment/<n> [--timeout=5m]
kubectl rollout history deployment/<n>
kubectl rollout history deployment/<n> --revision=N
kubectl rollout undo deployment/<n>
kubectl rollout undo deployment/<n> --to-revision=N
kubectl rollout pause deployment/<n>
kubectl rollout resume deployment/<n>
kubectl rollout restart deployment/<n>

# ── SCALING ───────────────────────────────────────────────────────────────
kubectl scale deployment <n> --replicas=N
kubectl autoscale deployment <n> --min=2 --max=10 --cpu-percent=50

# ── DELETE ────────────────────────────────────────────────────────────────
kubectl delete deployment <n>
kubectl delete -f deployment.yaml
```

### 34.2 Service Management

```bash
kubectl expose deployment <n> --type=ClusterIP --port=80 --target-port=8080
kubectl expose deployment <n> --type=NodePort --port=80 --node-port=31080
kubectl expose deployment <n> --type=LoadBalancer --port=80

kubectl get services [-n namespace]
kubectl describe service <n>
kubectl get endpoints <service-name>
kubectl get endpointslices -l kubernetes.io/service-name=<svc>

# Port-forward for testing
kubectl port-forward service/<n> 8080:80
kubectl port-forward deployment/<n> 8080:80
```

### 34.3 HPA Management

```bash
kubectl autoscale deployment <n> --min=2 --max=10 --cpu-percent=50
kubectl get hpa
kubectl describe hpa <n>
kubectl get hpa <n> -w    # Watch scaling events
kubectl delete hpa <n>

# Check metrics
kubectl top pods -l app=<label>
kubectl top nodes
```

### 34.4 Key Deployment Flags/Spec Reference

```yaml
spec:
  replicas: 3                              # Desired pod count
  revisionHistoryLimit: 5                  # RS history (default: 10)
  progressDeadlineSeconds: 300             # Rollout timeout (default: 600)
  paused: false                            # Pause rollout

  selector:
    matchLabels:
      app: my-app                          # Must match template labels

  strategy:
    type: RollingUpdate                    # or Recreate
    rollingUpdate:
      maxSurge: 1                          # Extra pods (int or %)
      maxUnavailable: 0                    # Missing pods allowed (int or %)

  template:
    spec:
      terminationGracePeriodSeconds: 60    # Default: 30s
      containers:
        - name: app
          image: registry/app:v1.0
          imagePullPolicy: IfNotPresent    # Always, Never, IfNotPresent
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
```

### 34.5 Troubleshooting Commands

```bash
# Deployment status
kubectl rollout status deployment/<n>
kubectl describe deployment <n> | tail -20
kubectl get events -n <ns> --sort-by='.lastTimestamp' | tail -20

# Pod debugging
kubectl describe pod <n>
kubectl logs <pod> --tail=100
kubectl logs <pod> --previous    # After crash
kubectl exec -it <pod> -- sh

# Resource checking
kubectl top nodes
kubectl top pods -l app=<label>
kubectl describe nodes | grep -A 5 "Allocated resources:"

# Service debugging
kubectl get endpoints <svc-name>
kubectl describe service <svc-name>
```

---

## 35. Key Takeaways & Summary

### The Deployment Mental Model

```
HIERARCHY:
  Deployment  →  manages →  ReplicaSets  →  manages →  Pods
  (desired state)           (replica count)             (running containers)

THE CONTROLLER LOOP:
  Watch (informer) → Compare (desired vs actual) → Act (API calls) → Loop
  All controllers talk to API server ONLY (never etcd directly)

ROLLING UPDATE:
  New RS scales up ←→ Old RS scales down
  Only when new pod is Ready does old pod get removed
  readinessProbe is the gate between "started" and "ready"

SERVICES:
  Label selector → stable ClusterIP → DNS → load balancing across pods
  Endpoints auto-updated as pods start/stop being Ready

SCALING:
  Manual: kubectl scale → updates spec.replicas → RS controller reacts
  HPA:    metrics-server → HPA calculates → updates spec.replicas → RS reacts
  Both: eventually reach the scheduler → pods placed on nodes by scheduler
```

### The 10 Rules of Production Deployments

1. **Always set resource requests and limits** — scheduling and QoS depend on them
2. **Configure readiness probes properly** — they gate production traffic during rollouts
3. **Use `maxUnavailable: 0` for zero-downtime** — combined with readiness probes
4. **Set `revisionHistoryLimit: 3-5`** — default 10 wastes etcd space needlessly
5. **Annotate rollouts with change-cause** — history is useless without descriptions
6. **Match Service selector to Pod labels exactly** — mismatch = empty endpoints = all traffic dropped
7. **Set `terminationGracePeriodSeconds` > max request time** — prevent mid-request kills
8. **Controllers never touch etcd directly** — all writes go through the API server
9. **Test HPA with actual load** before production — verify metrics and scaling behavior
10. **Use topologySpreadConstraints for HA** — spread pods across zones for fault tolerance

---

> **This guide covers Kubernetes v1.29+ Deployments. Always consult the official Kubernetes documentation at https://kubernetes.io/docs/concepts/workloads/controllers/deployment/ for the most current information.**

---

*End of: Kubernetes Deployments — Complete Production-Grade Guide*
