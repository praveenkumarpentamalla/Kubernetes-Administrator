## Table of Contents

1. [Introduction & Why Deployments Matter](#1-introduction)
2. [Core Identity Table](#2-core-identity)
3. [The Controller Pattern: Watch → Compare → Act → Loop](#3-controller-pattern)
4. [Deployment Deep Dive](#4-deployment-deep-dive)
5. [ReplicaSet Controller](#5-replicaset-controller)
6. [Node Controller](#6-node-controller)
7. [Service Controller](#7-service-controller)
8. [Namespace Controller](#8-namespace-controller)
9. [Job Controller](#9-job-controller)
10. [StatefulSet Controller](#10-statefulset-controller)
11. [DaemonSet Controller](#11-daemonset-controller)
12. [Garbage Collector Controller](#12-garbage-collector-controller)
13. [PersistentVolume Controller](#13-persistentvolume-controller)
14. [Internal Working Concepts: Reconciliation, Informers, Work Queues](#14-internal-working-concepts)
15. [Leader Election](#15-leader-election)
16. [Interaction with API Server and etcd](#16-api-server-and-etcd)
17. [Performance Tuning](#17-performance-tuning)
18. [Security Hardening](#18-security-hardening)
19. [Monitoring & Observability](#19-monitoring-and-observability)
20. [Troubleshooting with kubectl](#20-troubleshooting)
21. [Disaster Recovery](#21-disaster-recovery)
22. [Interview Questions & Answers](#22-interview-questions)
23. [Real-World Production Use Cases](#23-real-world-production-use-cases)
24. [Best Practices for Production](#24-best-practices)
25. [Common Mistakes and Pitfalls](#25-common-mistakes)
26. [Hands-On Labs](#26-hands-on-labs)
27. [Comparison: Deployment vs kube-apiserver vs kube-scheduler](#27-comparisons)
28. [ASCII Architecture Diagram](#28-architecture-diagram)
29. [Cheat Sheet](#29-cheat-sheet)
30. [Key Takeaways & Summary](#30-key-takeaways)

---

## 1. Introduction

### What is a Kubernetes Deployment?

A **Deployment** is the most widely used workload resource in Kubernetes. It provides a declarative way to manage the lifecycle of stateless applications — describing not just *what* should run, but *how* to roll it out, *how* to update it, and *how* to recover when something goes wrong.

At its core, a Deployment manages **ReplicaSets**, which in turn manage **Pods**. This three-tier hierarchy gives you:

- **Desired state management**: Declare `replicas: 5` and Kubernetes ensures 5 Pods always run
- **Rolling updates**: Replace old Pods gradually with zero downtime
- **Rollback**: Instantly revert to any previous revision
- **Revision history**: A full audit trail of every change
- **Pause and resume**: Stage a rollout and inspect before completing

### The Deployment Hierarchy

```
Deployment
    │
    ├── ReplicaSet (revision 3 — current, replicas=5)
    │       ├── Pod-1
    │       ├── Pod-2
    │       ├── Pod-3
    │       ├── Pod-4
    │       └── Pod-5
    │
    ├── ReplicaSet (revision 2 — idle, replicas=0)
    └── ReplicaSet (revision 1 — idle, replicas=0)
```

### Why Deployments Are the Cornerstone of Kubernetes

| Concern | Without Deployment | With Deployment |
|---|---|---|
| App crash | Manual restart | Auto-healed via ReplicaSet |
| New version rollout | Downtime required | Zero-downtime rolling update |
| Bad release | Manual Pod recreation | Instant rollback in one command |
| Scale-up | Manual Pod creation | `kubectl scale` or HPA |
| Audit / compliance | No change history | Full revision history |
| Progressive delivery | Manual canary management | Controlled via maxSurge/maxUnavailable |

### Position in the Kubernetes Architecture

Deployments are first-class citizens of the `apps/v1` API group. They are managed by the **Deployment Controller** inside `kube-controller-manager`. This controller watches Deployment objects and manages the lifecycle of ReplicaSets in response to changes — new image versions, replica count changes, or spec updates.

---

## 2. Core Identity

### Deployment Resource Core Identity

| Field | Value |
|---|---|
| **API Group** | `apps/v1` |
| **Kind** | `Deployment` |
| **Managed By** | `kube-controller-manager` (deployment-controller) |
| **Controller Binary** | `/usr/local/bin/kube-controller-manager` |
| **Default Metrics Port** | `10257` (HTTPS) |
| **Health Endpoint** | `https://<host>:10257/healthz` |
| **Controller Name** | `deployment-controller` |
| **Manages** | ReplicaSets → Pods |
| **Namespace Scoped** | Yes |
| **Update Strategies** | `RollingUpdate`, `Recreate` |
| **Selector Immutability** | Selector is immutable after creation |
| **Revision History** | Controlled by `revisionHistoryLimit` |
| **Rollback Support** | Yes, via `kubectl rollout undo` |
| **etcd Interaction** | Indirect — via kube-apiserver only |
| **Leader Election Required** | Yes (in HA setups) |
| **Key Status Fields** | `replicas`, `updatedReplicas`, `readyReplicas`, `availableReplicas` |
| **Sub-resources** | `/scale`, `/status` |

### kube-controller-manager Core Identity

| Field | Value |
|---|---|
| **Binary** | `kube-controller-manager` |
| **Secure Port** | `10257` |
| **Insecure Port** | `10252` (deprecated, disable with `--port=0`) |
| **Config File** | `/etc/kubernetes/controller-manager.conf` |
| **Leader Election Mechanism** | Kubernetes `Lease` objects |
| **Lease Namespace** | `kube-system` |
| **Lease Name** | `kube-controller-manager` |
| **Service Account** | `system:kube-controller-manager` |
| **TLS Cert** | `/etc/kubernetes/pki/controller-manager.crt` |

---

## 3. The Controller Pattern: Watch → Compare → Act → Loop

The controller pattern is the philosophical and mechanical foundation of all Kubernetes automation. Every controller — including the Deployment controller — follows the same four-step continuous loop.

```
┌────────────────────────────────────────────────────────────┐
│                   THE CONTROLLER LOOP                       │
│                                                            │
│    WATCH ──► COMPARE ──► ACT ──► LOOP (back to WATCH)      │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Step 1: WATCH

The controller sets up an **Informer** (explained in Section 14) that maintains a persistent watch connection to the kube-apiserver. It subscribes to change events for relevant resource types.

For the Deployment controller, it watches:
- `Deployment` objects (desired state)
- `ReplicaSet` objects (intermediate state)
- `Pod` objects (actual state)

### Step 2: COMPARE

The controller reads the **desired state** from the spec and compares it to the **actual state** observed from the cluster:

```
Desired:  replicas=5, image=nginx:1.25
Actual:   replicas=3, image=nginx:1.24
Delta:    Scale up to 5, update image → trigger rolling update
```

### Step 3: ACT

The controller takes the minimum corrective action to close the gap:
- Create a new ReplicaSet for the updated Pod template
- Scale down old ReplicaSet gradually
- Scale up new ReplicaSet gradually (respecting `maxSurge` and `maxUnavailable`)

### Step 4: LOOP

After acting, the controller re-enters the watch state and continues monitoring. The loop is infinite and runs as long as the controller process is alive.

### Concrete Example: Rolling Update Timeline

```
t=0:  Deployment spec updated — image nginx:1.24 → nginx:1.25
t=0:  Controller detects change via Informer event
t=0:  New ReplicaSet created (RS-v2) with nginx:1.25
t=1:  RS-v2 scaled to 1 Pod (maxSurge=1)
t=2:  Pod in RS-v2 becomes Ready
t=3:  RS-v1 scaled down by 1 (maxUnavailable=1)
t=4:  RS-v2 scaled to 2... (cycle repeats)
t=N:  RS-v2 at 5 replicas (full capacity)
t=N:  RS-v1 scaled to 0 (retained for rollback)
```

---

## 4. Deployment Deep Dive

### 4.1 The Full Anatomy of a Production Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
  labels:
    app: my-app
    tier: backend
    team: platform
  annotations:
    kubernetes.io/change-cause: "v2.1.0 - Fix memory leak in cache layer"
spec:
  # ── Replica Management ──────────────────────────────────────────
  replicas: 5
  revisionHistoryLimit: 10          # Keep last 10 ReplicaSets for rollback
  progressDeadlineSeconds: 600      # Fail if no progress for 10 minutes

  # ── Pod Selection ───────────────────────────────────────────────
  selector:                         # IMMUTABLE after creation
    matchLabels:
      app: my-app

  # ── Update Strategy ─────────────────────────────────────────────
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1             # At most 1 Pod unavailable at a time
      maxSurge: 1                   # At most 1 extra Pod above desired count

  # ── Minimum Availability ────────────────────────────────────────
  minReadySeconds: 30               # Pod must be ready 30s before counted

  # ── Pod Template ────────────────────────────────────────────────
  template:
    metadata:
      labels:
        app: my-app                 # Must match selector
        version: v2.1.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
    spec:
      # Graceful termination
      terminationGracePeriodSeconds: 60

      # Topology spread across zones
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: my-app

      # Anti-affinity: no two Pods on same node
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              topologyKey: kubernetes.io/hostname
              labelSelector:
                matchLabels:
                  app: my-app

      # Service Account
      serviceAccountName: my-app-sa
      automountServiceAccountToken: false

      # Init Container
      initContainers:
      - name: db-migration-check
        image: my-app:v2.1.0
        command: ["./wait-for-db.sh"]
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: host

      containers:
      - name: my-app
        image: my-app:v2.1.0
        imagePullPolicy: IfNotPresent

        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        - name: metrics
          containerPort: 9090
          protocol: TCP

        # Environment variables
        env:
        - name: ENV
          value: "production"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        envFrom:
        - configMapRef:
            name: my-app-config

        # Resource management
        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "512Mi"

        # Health checks
        startupProbe:
          httpGet:
            path: /startup
            port: 8080
          failureThreshold: 30
          periodSeconds: 10

        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          successThreshold: 1
          failureThreshold: 3

        livenessProbe:
          httpGet:
            path: /live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3

        # Volume mounts
        volumeMounts:
        - name: config
          mountPath: /etc/my-app
          readOnly: true
        - name: tmp
          mountPath: /tmp

        # Security context
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop: ["ALL"]

      volumes:
      - name: config
        configMap:
          name: my-app-config
      - name: tmp
        emptyDir: {}

      # Image pull secret for private registry
      imagePullSecrets:
      - name: registry-credentials
```

### 4.2 Update Strategies Explained

#### RollingUpdate (Default)

```
Before:  [v1][v1][v1][v1][v1]   (5 pods, all v1)

Step 1:  [v1][v1][v1][v1][ ↓ ]  maxUnavailable=1: kill 1 v1
         [v1][v1][v1][v1][v2]   maxSurge=1: create 1 v2

Step 2:  [v1][v1][v1][ ↓ ][v2]
         [v1][v1][v1][v2][v2]

Step N:  [v2][v2][v2][v2][v2]   All updated, zero downtime
```

#### Recreate

```
Before:  [v1][v1][v1][v1][v1]
         ↓ (all killed simultaneously)
During:  [ ][ ][ ][ ][ ]        ← DOWNTIME
         ↓ (all created)
After:   [v2][v2][v2][v2][v2]
```

| Strategy | Zero Downtime | Speed | Use Case |
|---|---|---|---|
| `RollingUpdate` | ✅ Yes | Gradual | Standard stateless services |
| `Recreate` | ❌ No | Instant | Single-instance DBs, config requiring restart |

### 4.3 maxUnavailable and maxSurge Deep Dive

Both values can be **absolute numbers** or **percentages**:

```yaml
strategy:
  rollingUpdate:
    maxUnavailable: 25%    # At most 25% of desired Pods unavailable
    maxSurge: 25%          # At most 25% extra Pods above desired
```

**With replicas: 10, maxUnavailable: 25%, maxSurge: 25%:**

```
Min pods during update:  10 - 25% = 8  (floor)
Max pods during update:  10 + 25% = 12 (ceiling)
```

**Special Values:**

| maxUnavailable | maxSurge | Behavior |
|---|---|---|
| `0` | `1` | Create new before deleting old (safest — always full capacity) |
| `1` | `0` | Delete old before creating new (resource-constrained environments) |
| `25%` | `25%` | Standard rolling (Kubernetes default) |
| `100%` | `0` | Recreate-like (all old Pods deleted first) |
| `0` | `100%` | Blue-green-like (all new Pods created first) |

### 4.4 progressDeadlineSeconds

```yaml
spec:
  progressDeadlineSeconds: 600
```

If a rollout makes no progress for this duration, the Deployment's `Progressing` condition is set to `False` with reason `ProgressDeadlineExceeded`. The Deployment controller will **not** automatically roll back — you must take action.

```bash
# Check for stalled deployment
kubectl rollout status deployment/my-app
# Output: error: deployment "my-app" exceeded its progress deadline

# Check condition
kubectl get deployment my-app -o jsonpath='{.status.conditions}'
```

### 4.5 minReadySeconds

```yaml
spec:
  minReadySeconds: 30
```

A newly created Pod is considered **available** only after it has been Ready for at least `minReadySeconds`. Without this, a Pod that passes readiness briefly then crashes would still be counted, allowing the rollout to proceed with a broken version.

### 4.6 revisionHistoryLimit

```yaml
spec:
  revisionHistoryLimit: 10
```

Controls how many old ReplicaSets are retained after updates. Old RSes are scaled to 0 but kept for rollback. Setting this to 0 disables rollback capability but frees etcd space.

**Practical guidance:**

| Environment | Recommended Value | Reason |
|---|---|---|
| Production | 10 | Maximum rollback flexibility |
| Staging | 5 | Balance history and resources |
| Development | 2 | Minimal etcd overhead |
| CI/CD heavy | 3-5 | High churn — limit old RS accumulation |

### 4.7 Deployment Conditions

A Deployment maintains a set of conditions in its status:

| Condition Type | Status | Reason | Meaning |
|---|---|---|---|
| `Progressing` | `True` | `NewReplicaSetCreated` | Rollout started |
| `Progressing` | `True` | `FoundNewReplicaSet` | RS exists, scaling |
| `Progressing` | `True` | `ReplicaSetUpdated` | RS was scaled |
| `Progressing` | `False` | `ProgressDeadlineExceeded` | Rollout stalled |
| `Available` | `True` | `MinimumReplicasAvailable` | Deployment is healthy |
| `Available` | `False` | `MinimumReplicasUnavailable` | Not enough ready Pods |
| `ReplicaFailure` | `True` | `FailedCreate` | Pod creation failing |

```bash
# Check all conditions
kubectl get deployment my-app -o jsonpath='{.status.conditions}' | python3 -m json.tool
```

---

## 5. ReplicaSet Controller

### 5.1 Purpose and Role

A **ReplicaSet** is the immediate owner of Pods. The Deployment controller manages ReplicaSets; the ReplicaSet controller manages Pods. This separation of concerns is deliberate — ReplicaSets are the mechanism for maintaining a stable set of replica Pods, while Deployments layer update and rollback logic on top.

### 5.2 How Deployment and ReplicaSet Interact

```
Deployment Controller observes:
  - spec.template changed (new image, env vars, etc.)
    → Create new ReplicaSet with updated template hash
    → Gradually scale new RS up, old RS down

  - spec.replicas changed
    → Update current RS replicas directly (no new RS needed)

  - spec.strategy changed
    → Adjust rolling update behavior going forward
```

### 5.3 Pod Template Hash

Every ReplicaSet created by a Deployment gets a unique `pod-template-hash` label computed from the Pod template contents:

```yaml
# ReplicaSet metadata (auto-generated)
metadata:
  name: my-app-7f8b6d9c4    # Deployment name + hash
  labels:
    app: my-app
    pod-template-hash: 7f8b6d9c4   # Hash of Pod template spec
```

This hash:
- Uniquely identifies the RS for a given Pod template
- Prevents the Deployment from accidentally adopting RSes it didn't create
- Is stable — same template always produces same hash

```bash
# View all ReplicaSets for a Deployment
kubectl get rs -l app=my-app

# View the pod-template-hash
kubectl get rs -l app=my-app -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.pod-template-hash}{"\n"}{end}'
```

### 5.4 ReplicaSet Lifecycle During Rollout

```
Initial state: RS-abc (v1, replicas=5)

Update triggered:
1. RS-def created (v2, replicas=0)
2. RS-def scaled to 1  →  RS-abc scaled to 4
3. RS-def scaled to 2  →  RS-abc scaled to 3
4. RS-def scaled to 3  →  RS-abc scaled to 2
5. RS-def scaled to 4  →  RS-abc scaled to 1
6. RS-def scaled to 5  →  RS-abc scaled to 0

Post-rollout:
  RS-def: replicas=5 (active)
  RS-abc: replicas=0 (retained for rollback, revisionHistoryLimit)
```

---

## 6. Node Controller

### 6.1 Purpose

The **Node Controller** monitors node health and manages node lifecycle. It doesn't directly manage Deployments, but its actions — evicting Pods from unhealthy nodes — trigger Deployment and ReplicaSet controllers to reschedule Pods.

### 6.2 Responsibilities

| Responsibility | Mechanism | Impact on Deployments |
|---|---|---|
| Monitor node heartbeats | `NodeStatus` updates from kubelet | Detects unhealthy nodes |
| Mark nodes `NotReady` | After `node-monitor-grace-period` (40s) | Triggers Pod eviction |
| Apply taints | `node.kubernetes.io/not-ready:NoExecute` | Pods with no toleration evicted |
| Evict Pods | After `pod-eviction-timeout` (5m) | Deployment RS creates replacements |
| CIDR assignment | Assigns Pod IP ranges to new nodes | Enables Pod networking |

### 6.3 Eviction Flow Leading to Deployment Reconciliation

```
Node stops sending heartbeats
        │
        ▼ 40 seconds (node-monitor-grace-period)
Node marked NotReady
Taint applied: node.kubernetes.io/not-ready:NoExecute
        │
        ▼ 300 seconds (pod-eviction-timeout) for pods with toleration
Pods on node → phase: Unknown / Terminating
        │
        ▼
ReplicaSet controller sees pod count < desired
        │
        ▼
New Pods scheduled on healthy nodes
        │
        ▼
Deployment returns to healthy state
```

### 6.4 Zone-Aware Eviction

```bash
# When a zone goes down, eviction rate is reduced to preserve availability
--node-eviction-rate=0.1               # 1 node per 10 seconds (normal)
--secondary-node-eviction-rate=0.01    # Much slower when zone is unhealthy
--unhealthy-zone-threshold=0.55        # >55% nodes down = zone unhealthy
--large-cluster-size-threshold=50      # Cluster size for rate adjustment
```

---

## 7. Service Controller

### 7.1 Purpose

The **Service Controller** manages cloud-provider load balancers for Services of type `LoadBalancer`. It is part of the cloud-controller-manager in modern clusters, but was historically in kube-controller-manager.

### 7.2 Relationship to Deployments

Services don't own Pods — they **select** them by labels. This means a Deployment can be updated (creating new Pods with new labels like `version: v2`) while the Service continues routing to all matching Pods during the transition.

```yaml
# Service selects all Pods with app=my-app regardless of version
spec:
  selector:
    app: my-app       # Matches both v1 and v2 Pods during rolling update
```

### 7.3 Service Types

| Type | Description | Deployment Use Case |
|---|---|---|
| `ClusterIP` | Internal cluster IP | Internal microservice traffic |
| `NodePort` | Node IP + static port | Direct node access (dev/testing) |
| `LoadBalancer` | Cloud LB with external IP | Production ingress |
| `ExternalName` | DNS CNAME alias | Migrate from external to internal |
| `Headless` | No ClusterIP (`ClusterIP: None`) | StatefulSets, direct Pod DNS |

### 7.4 Endpoint Slices and Rolling Updates

When Pods are replaced during a rolling update, the EndpointSlice controller automatically adds/removes Pod IPs from the Endpoint/EndpointSlice objects. This ensures the Service always routes only to Ready Pods.

```bash
# Watch endpoint changes during rolling update
kubectl get endpointslices -l kubernetes.io/service-name=my-app -w
```

---

## 8. Namespace Controller

### 8.1 Purpose

The **Namespace Controller** handles namespace termination — the cascade deletion of all resources when a namespace is deleted.

### 8.2 Impact on Deployments

When a namespace is deleted:
1. Namespace phase → `Terminating`
2. Namespace controller deletes all objects: Deployments → ReplicaSets → Pods → Services → ConfigMaps → Secrets → PVCs
3. Finalizers are removed
4. Namespace removed from etcd

```bash
# Namespace stuck in Terminating
kubectl get ns production -o json | \
  jq '.spec.finalizers = []' | \
  kubectl replace --raw "/api/v1/namespaces/production/finalize" -f -
```

### 8.3 Resource Quotas and Deployments

Namespaces support `ResourceQuota` objects that limit total resources. If a Deployment tries to scale beyond quota, new Pods will fail with `FailedCreate` reason.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "100"
    count/deployments.apps: "20"
```

---

## 9. Job Controller

### 9.1 Purpose

The **Job Controller** ensures a specified number of Pods successfully complete. Unlike Deployments, Jobs run Pods to **completion** rather than keeping them running.

### 9.2 Deployment vs Job

| Feature | Deployment | Job |
|---|---|---|
| Pod lifecycle | Runs forever | Runs to completion |
| Restart policy | `Always` | `OnFailure` or `Never` |
| Success metric | N running Pods | N successful completions |
| Use case | Long-running services | Batch jobs, migrations, one-time tasks |
| Scaling | Replica count | `completions` × `parallelism` |
| After success | Pods keep running | Pods terminated (retained for logs) |

### 9.3 Job YAML

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration-v2
  namespace: production
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3
  activeDeadlineSeconds: 600
  ttlSecondsAfterFinished: 3600   # Auto-delete job 1h after completion
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: migration
        image: my-app:v2.1.0
        command: ["./migrate.sh", "--target=v2"]
        env:
        - name: DB_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
```

### 9.4 Using Jobs Alongside Deployments

A common pattern is to run a Job (for DB migration) before scaling up a new Deployment version:

```bash
# 1. Run migration Job
kubectl apply -f migration-job.yaml
kubectl wait --for=condition=complete job/db-migration-v2 --timeout=300s

# 2. Only then update Deployment
kubectl set image deployment/my-app my-app=my-app:v2.1.0

# 3. Monitor rollout
kubectl rollout status deployment/my-app
```

---

## 10. StatefulSet Controller

### 10.1 Purpose

The **StatefulSet Controller** manages stateful applications requiring stable identity, stable storage, and ordered operations.

### 10.2 Deployment vs StatefulSet

| Feature | Deployment | StatefulSet |
|---|---|---|
| **Pod Names** | Random (`my-app-abc12`) | Stable (`my-app-0`, `my-app-1`) |
| **Storage** | Shared PVC or ephemeral | Dedicated PVC per Pod |
| **DNS** | Random | Stable: `my-app-0.my-app.ns.svc` |
| **Scale Up Order** | Parallel | Sequential: 0→1→2 |
| **Scale Down Order** | Random | Reverse: 2→1→0 |
| **Rolling Update** | Via Deployment RS management | Ordered, reverse: 2→1→0 |
| **Use Case** | Stateless services | Databases, Kafka, ZooKeeper |
| **Suitable for** | HTTP APIs, microservices | Postgres, MySQL, Redis, Elasticsearch |

### 10.3 StatefulSet YAML Example

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: "postgres"
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
        image: postgres:15.4
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: postgres-secret
              key: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2000m"
            memory: "4Gi"
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0   # Pods with ordinal >= partition are updated
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "fast-nvme"
      resources:
        requests:
          storage: 500Gi
```

---

## 11. DaemonSet Controller

### 11.1 Purpose

The **DaemonSet Controller** ensures exactly one Pod runs on every node (or a subset). When new nodes join the cluster, the DaemonSet controller automatically schedules Pods on them.

### 11.2 Deployment vs DaemonSet

| Feature | Deployment | DaemonSet |
|---|---|---|
| **Scheduling** | Any available node | Every node (or matching nodes) |
| **Replica count** | Explicit `replicas` field | Implicit = number of matching nodes |
| **Scale to zero** | Yes | No (always one per node) |
| **Node addition** | No automatic scheduling | Auto-schedules on new nodes |
| **Use case** | Application workloads | Node agents, monitoring, networking |

### 11.3 Common DaemonSet Use Cases

| Category | Examples |
|---|---|
| Log collection | Fluentd, Filebeat, Promtail, Fluent Bit |
| Monitoring | Prometheus Node Exporter, Datadog Agent, Dynatrace |
| Networking | Calico, Cilium, Flannel, Weave |
| Storage | Ceph CSI, Portworx agent |
| Security | Falco, Aqua enforcer, Prisma Cloud |

### 11.4 DaemonSet YAML

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  selector:
    matchLabels:
      app: fluent-bit
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      serviceAccountName: fluent-bit
      containers:
      - name: fluent-bit
        image: fluent/fluent-bit:2.2
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
        volumeMounts:
        - name: varlog
          mountPath: /var/log
          readOnly: true
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

---

## 12. Garbage Collector Controller

### 12.1 Purpose

The **Garbage Collector (GC) Controller** deletes objects that have no more owners, using the `ownerReferences` chain established when objects are created.

### 12.2 Ownership in the Deployment Chain

```
Deployment
   ownerRef → (none — top level)
      │
      ▼
   ReplicaSet
      ownerRef → Deployment UID
         │
         ▼
      Pod
         ownerRef → ReplicaSet UID
```

When a Deployment is deleted, the GC walks down this chain and deletes owned ReplicaSets, which triggers deletion of owned Pods.

### 12.3 Deletion Propagation Modes

| Mode | Command | Behavior |
|---|---|---|
| `Background` (default) | `kubectl delete deployment my-app` | Deployment deleted immediately; GC cleans up RS and Pods async |
| `Foreground` | `kubectl delete deployment my-app --cascade=foreground` | Deployment marked "terminating"; children deleted first; then deployment |
| `Orphan` | `kubectl delete deployment my-app --cascade=orphan` | Deployment deleted; RS and Pods kept running (unmanaged) |

```bash
# Orphan pods (keep them running after deleting deployment)
kubectl delete deployment my-app --cascade=orphan

# Verify pods are now orphaned
kubectl get pods -l app=my-app -o jsonpath='{.items[*].metadata.ownerReferences}'
```

---

## 13. PersistentVolume Controller

### 13.1 Purpose

The **PersistentVolume (PV) Controller** manages the binding between PersistentVolumeClaims (PVCs) and PersistentVolumes (PVs). While Deployments typically use ephemeral storage, there are valid use cases where stateless apps need persistent storage (e.g., shared config files, uploaded assets).

### 13.2 Deployment with PVC

```yaml
# Static PVC for a Deployment (shared across all replicas — ReadWriteMany required)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-assets
  namespace: production
spec:
  accessModes: ["ReadWriteMany"]    # RWX needed for multi-Pod access
  storageClassName: "efs-sc"        # AWS EFS supports RWX
  resources:
    requests:
      storage: 100Gi
---
# Deployment using the PVC
spec:
  template:
    spec:
      volumes:
      - name: assets
        persistentVolumeClaim:
          claimName: shared-assets
      containers:
      - name: my-app
        volumeMounts:
        - name: assets
          mountPath: /app/assets
```

### 13.3 PV/PVC Lifecycle

```
PVC Created (status: Pending)
        │
        ▼
PV Controller: find matching PV (or dynamic provision via StorageClass)
        │
        ▼
Bind PVC → PV (both status: Bound)
        │
        ▼
Pod mounts PVC successfully
        │
        ▼
Pod deleted (Deployment rolling update)
        │
        ▼
PVC remains Bound (data preserved)
        │
        ▼
New Pod (next revision) mounts same PVC
```

### 13.4 Storage Access Modes

| Mode | Abbreviation | Multiple Pods | Use with Deployment |
|---|---|---|---|
| `ReadWriteOnce` | RWO | No (single node) | Only if replicas=1 |
| `ReadOnlyMany` | ROX | Yes (read-only) | Read-only shared data |
| `ReadWriteMany` | RWX | Yes (read-write) | Shared writable storage (NFS/EFS) |
| `ReadWriteOncePod` | RWOP | No (single pod) | Strictly exclusive access |

---

## 14. Internal Working Concepts

### 14.1 The Reconciliation Loop

The reconciliation loop is the mechanism that gives Kubernetes its **self-healing** property. For the Deployment controller, it runs as a set of goroutines (Go's lightweight threads) processing items from a work queue.

```go
// Simplified Deployment controller reconciliation
func (dc *DeploymentController) syncDeployment(key string) error {
    namespace, name := parseKey(key)

    // 1. Fetch current deployment from local cache (no API call)
    deployment, err := dc.dLister.Deployments(namespace).Get(name)
    if err != nil { return err }

    // 2. Get all ReplicaSets owned by this Deployment
    rsList, err := dc.getReplicaSetsForDeployment(deployment)
    if err != nil { return err }

    // 3. Get all Pods owned by these ReplicaSets
    podMap, err := dc.getPodMapForDeployment(deployment, rsList)
    if err != nil { return err }

    // 4. Sync based on strategy
    switch deployment.Spec.Strategy.Type {
    case apps.RecreateDeploymentStrategyType:
        return dc.rolloutRecreate(deployment, rsList, podMap)
    case apps.RollingUpdateDeploymentStrategyType:
        return dc.rolloutRolling(deployment, rsList, podMap)
    }
    return nil
}
```

**Key properties of the reconciliation loop:**
- **Level-triggered** (not edge-triggered): It reconciles based on current state, not just change events. Even if events are missed, the next reconciliation cycle corrects any drift.
- **Idempotent**: Running the loop multiple times with the same state produces the same result — no duplicate Pods, no double-deletions.
- **Eventually consistent**: Convergence is guaranteed but not instantaneous.

### 14.2 The Informer Pattern

Informers are the efficient state-synchronization mechanism that powers all Kubernetes controllers. They solve the problem of controllers needing to know the current state of cluster objects without polling the API server constantly.

#### Informer Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    INFORMER INTERNALS                        │
│                                                             │
│  kube-apiserver                                             │
│       │                                                     │
│       │ (1) Initial LIST of all objects                     │
│       │ (2) Persistent WATCH for delta updates              │
│       ▼                                                     │
│  ┌──────────┐    ┌─────────────────────┐                   │
│  │ Reflector│───►│  Delta FIFO Queue   │                   │
│  │          │    │                     │                   │
│  │ - LIST   │    │  [Added][Modified]  │                   │
│  │ - WATCH  │    │  [Deleted][Sync]    │                   │
│  └──────────┘    └──────────┬──────────┘                   │
│                             │                               │
│                  ┌──────────▼──────────┐                   │
│                  │    Store/Indexer    │◄─── Controller     │
│                  │  (in-memory cache) │     reads here     │
│                  └──────────┬──────────┘     (no API call) │
│                             │                               │
│                  ┌──────────▼──────────┐                   │
│                  │  Event Handlers     │                   │
│                  │                     │                   │
│                  │  OnAdd(obj)         │                   │
│                  │  OnUpdate(old, new) │                   │
│                  │  OnDelete(obj)      │                   │
│                  └──────────┬──────────┘                   │
│                             │                               │
│                  ┌──────────▼──────────┐                   │
│                  │    Work Queue       │                   │
│                  │  (deduplicated,     │                   │
│                  │   rate-limited)     │                   │
│                  └──────────┬──────────┘                   │
│                             │                               │
│                  ┌──────────▼──────────┐                   │
│                  │  Worker Goroutines  │                   │
│                  │  reconcile(key)     │                   │
│                  └─────────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

#### Shared Informer Factory

Multiple controllers share informers to avoid duplicate watch connections to the API server:

```go
// Both Deployment and ReplicaSet controllers share the same Pod informer
sharedFactory := informers.NewSharedInformerFactory(kubeClient, resyncPeriod)

podInformer := sharedFactory.Core().V1().Pods()
rsInformer  := sharedFactory.Apps().V1().ReplicaSets()
depInformer := sharedFactory.Apps().V1().Deployments()
```

**Efficiency gains:**

| Without Shared Informers | With Shared Informers |
|---|---|
| Each controller: 1 watch per resource | 1 watch per resource total |
| N controllers × M resources = N×M watches | M watches (regardless of controllers) |
| API server fan-out overhead | Minimal API server load |
| Duplicate event processing | Single event, multiple handlers |

#### Informer Resync

The informer periodically replays all cached objects through the `OnUpdate` event handler, even if nothing changed. This catches any missed events due to watch connection drops:

```
Default resync: 12h (--min-resync-period=12h)
Actual resync:  base + random jitter (prevents thundering herd)
```

### 14.3 Work Queues

Work queues decouple event detection (informer handlers) from processing (reconciliation). They provide three critical properties:

#### Deduplication

```
Event 1: Pod-A deleted → enqueue("production/my-app")
Event 2: Pod-B deleted → enqueue("production/my-app")  ← DUPLICATE, ignored
Event 3: Pod-C deleted → enqueue("production/my-app")  ← DUPLICATE, ignored

Worker processes "production/my-app" once
  → Reconcile: 3 Pods missing → Create 3 Pods in one pass
```

Without deduplication, each pod deletion would trigger a separate reconciliation, each creating 1 Pod — resulting in race conditions and potential over-creation.

#### Rate Limiting with Exponential Backoff

```
Attempt 1 fails → retry in 5ms
Attempt 2 fails → retry in 10ms
Attempt 3 fails → retry in 20ms
Attempt 4 fails → retry in 40ms
...
Max retry delay: 1000s (configurable)
```

#### Work Queue Implementation

```go
// Rate limited queue used by all production controllers
queue := workqueue.NewRateLimitingQueue(
    workqueue.NewItemExponentialFailureRateLimiter(
        5*time.Millisecond,   // base delay
        1000*time.Second,     // max delay
    ),
)

// Event handler adds to queue
func onDeploymentUpdate(old, new interface{}) {
    key, _ := cache.MetaNamespaceKeyFunc(new)
    queue.Add(key)   // Deduplicated
}

// Worker processes from queue
func runWorker() {
    for {
        key, quit := queue.Get()
        if quit { return }

        err := reconcile(key.(string))
        if err != nil {
            queue.AddRateLimited(key)  // Retry with backoff
        } else {
            queue.Forget(key)          // Reset backoff counter
        }
        queue.Done(key)
    }
}
```

---

## 15. Leader Election

### 15.1 Why Leader Election is Needed

In a production HA Kubernetes cluster, `kube-controller-manager` runs as multiple instances (typically 3, one per control plane node). Without coordination:

```
Instance A: "My-app Deployment has 3/5 Pods — I'll create 2 more"
Instance B: "My-app Deployment has 3/5 Pods — I'll create 2 more"
Result:     6 Pods created instead of 2 → Deployment over-replicated

Instance A: "Pod count is 5 — scale down by 1"
Instance B: "Pod count is 5 — scale down by 1"
Result:     2 Pods deleted instead of 1 → Deployment under-replicated
```

Leader election ensures **only one instance** runs the controller loops at any time.

### 15.2 How Leader Election Works (Lease Object)

Kubernetes uses a `coordination.k8s.io/v1` **Lease** object as a distributed lock:

```yaml
# Auto-managed Lease object in kube-system
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  holderIdentity: "master-1_a1b2c3d4-e5f6-7890-abcd-ef1234567890"
  leaseDurationSeconds: 15
  renewTime: "2024-03-15T14:30:05.000000Z"
  acquireTime: "2024-03-15T14:00:00.000000Z"
  leaseTransitions: 3
```

#### Election Sequence

```
Step 1: All 3 instances start simultaneously
        Master-1, Master-2, Master-3 all try to CREATE the Lease

Step 2: API server handles concurrent creates atomically
        Master-1 wins (first to succeed)
        Master-2 and Master-3 get Conflict error → enter STANDBY mode

Step 3: Master-1 (leader) starts all controller loops
        Master-1 renews Lease every ~5s (leaseDuration/3)

Step 4: Master-2 and Master-3 watch the Lease
        They check renewal time: if (now - renewTime) > leaseDuration → leader may be dead

Step 5: Master-1 crashes
        Lease not renewed for 15 seconds
        Master-2 and Master-3 both try to UPDATE Lease with their identity

Step 6: Master-2 wins the UPDATE (atomic compare-and-swap)
        Master-2 becomes new leader, starts controller loops
        Master-3 returns to STANDBY

Total failover time: up to leaseDuration (15s) + processing delay
```

### 15.3 Important Leader Election Flags

```bash
# Enable/disable leader election (always true in production HA)
--leader-elect=true

# How long a lease is valid before standby instances try to acquire
--leader-elect-lease-duration=15s

# How long the leader has to renew before it considers itself lost
# Must be < lease-duration
--leader-elect-renew-deadline=10s

# How often standby instances attempt to acquire the lease
# Must be < renew-deadline
--leader-elect-retry-period=2s

# Resource type used as the lock
--leader-elect-resource-lock=leases     # Recommended (leases/endpoints/configmaps)

# Resource name and namespace
--leader-elect-resource-name=kube-controller-manager
--leader-elect-resource-namespace=kube-system
```

### 15.4 Timing Relationships and Constraints

```
CRITICAL CONSTRAINT: leaseDuration > renewDeadline > retryPeriod

Safe example:
  leaseDuration:  15s   ← Standbys wait 15s before attempting takeover
  renewDeadline:  10s   ← Leader must renew within 10s or step down
  retryPeriod:    2s    ← Standbys retry acquisition every 2s

DANGER ZONE: If renewDeadline >= leaseDuration
  → Leader could lose lease while still running
  → Two active leaders simultaneously (split-brain)
  → Race conditions in reconciliation
```

### 15.5 HA Best Practices

| Practice | Detail | Command/Config |
|---|---|---|
| **3 controller-manager instances** | One per control plane node | kubeadm HA topology |
| **Spread across AZs** | Use pod anti-affinity | `podAntiAffinity.requiredDuringScheduling` |
| **Monitor leader status** | Alert on `leader_election_master_status == 0` | Prometheus alert |
| **Watch Lease transitions** | High `leaseTransitions` indicates instability | `kubectl get lease ... -w` |
| **Consistent timing flags** | Verify all instances use same flags | `kubectl get pod ... -o yaml` |
| **Use Leases (not endpoints)** | Leases have less API impact | `--leader-elect-resource-lock=leases` |

```bash
# Check current leader
kubectl get lease kube-controller-manager -n kube-system \
  -o jsonpath='{.spec.holderIdentity}{"\n"}'

# Watch leader transitions
kubectl get lease kube-controller-manager -n kube-system -w

# Count leadership changes
kubectl get lease kube-controller-manager -n kube-system \
  -o jsonpath='{.spec.leaseTransitions}'
```

---

## 16. Interaction with API Server and etcd

### 16.1 The Golden Rule

```
╔═══════════════════════════════════════════════════════════════╗
║                                                               ║
║   CONTROLLERS NEVER COMMUNICATE DIRECTLY WITH etcd           ║
║                                                               ║
║   ALL cluster state changes flow through kube-apiserver       ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

This is not just a convention — it's enforced by architecture. Controllers don't have etcd credentials. They only have a kubeconfig pointing to the kube-apiserver.

### 16.2 Why This Architecture Exists

| Concern | Direct etcd Access | Via kube-apiserver |
|---|---|---|
| **Authentication** | None | Full TLS + bearer token auth |
| **Authorization** | None | RBAC enforced for every operation |
| **Admission Control** | Bypassed | All mutation + validation webhooks applied |
| **Object Validation** | None | Schema validation against OpenAPI spec |
| **Audit Logging** | None | Complete audit trail for all operations |
| **Rate Limiting** | None | API priority and fairness |
| **Object Versioning** | Manual | Automatic resource version management |
| **Watch Semantics** | Complex | Simplified list-watch API |

### 16.3 Complete Request Flow: Deployment Controller Creating a Pod

```
1. ┌─ Deployment Controller ─────────────────────────────────┐
   │  Reconcile: Need to create a new Pod for RS-v2           │
   │  pod := buildPod(rs.Spec.Template)                      │
   │  _, err = kubeClient.CoreV1().Pods(ns).Create(ctx, pod) │
   └─────────────────────────────────────────────────────────┘
                          │
                          │ HTTPS POST /api/v1/namespaces/production/pods
                          │ Authorization: Bearer <service-account-token>
                          ▼
2. ┌─ kube-apiserver ─────────────────────────────────────────┐
   │                                                          │
   │  a) Authentication: Verify service account JWT token     │
   │  b) Authorization:  RBAC check                           │
   │     system:controller:deployment-controller              │
   │     can CREATE pods in namespace "production"?  → YES    │
   │                                                          │
   │  c) Admission Control:                                   │
   │     → MutatingAdmissionWebhook (add labels, inject env) │
   │     → PodSecurityAdmission (check security policy)      │
   │     → ValidatingAdmissionWebhook (custom validators)    │
   │                                                          │
   │  d) Validation: Is Pod spec valid per OpenAPI schema?    │
   │                                                          │
   │  e) Persistence: Write Pod to etcd                       │
   │     Key: /registry/pods/production/<pod-name>            │
   │                                                          │
   │  f) Return: 201 Created + Pod object with UID/version    │
   └─────────────────────────────────────────────────────────┘
                          │
                          │ 201 Created {pod object}
                          ▼
3. ┌─ Deployment Controller ─────────────────────────────────┐
   │  Pod created successfully                                │
   │  Update ReplicaSet status via PATCH                      │
   └─────────────────────────────────────────────────────────┘
                          │
4. ┌─ kube-scheduler ─────────────────────────────────────────┐
   │  Watches for Pods with spec.nodeName == ""               │
   │  Selects best node using filtering + scoring             │
   │  Binds Pod to Node via:                                  │
   │  POST /api/v1/namespaces/production/pods/<name>/binding  │
   └─────────────────────────────────────────────────────────┘
                          │
5. ┌─ kubelet (on selected node) ─────────────────────────────┐
   │  Watches for Pods assigned to its node                   │
   │  Pulls image, creates container, starts Pod              │
   │  Updates Pod status: phase → Running                     │
   └─────────────────────────────────────────────────────────┘
```

### 16.4 Read from Cache, Write to API — The Critical Pattern

```go
// ✅ CORRECT: Read from local informer cache — ZERO API calls
deployment, err := dc.dLister.Deployments(namespace).Get(name)
pods, err := dc.podLister.Pods(namespace).List(selector)

// ✅ CORRECT: Write through API server — HTTPS call
_, err = dc.client.AppsV1().ReplicaSets(namespace).Create(ctx, rs, opts)
_, err = dc.client.CoreV1().Pods(namespace).Delete(ctx, name, opts)

// ❌ NEVER: Direct etcd access (not possible by design)
etcdClient.Get("/registry/deployments/production/my-app")
```

### 16.5 Resource Version and Optimistic Concurrency

Every Kubernetes object has a `resourceVersion` field. When controllers update objects, they include the current `resourceVersion` — if another writer has modified the object since the last read, the API server returns a `409 Conflict`, and the controller must re-read and retry.

```bash
# View resource version
kubectl get deployment my-app -o jsonpath='{.metadata.resourceVersion}'

# This prevents lost updates — a controller working with stale data
# cannot accidentally overwrite a more recent change
```

---

## 17. Performance Tuning

### 17.1 Key Deployment-Related Flags

```bash
# Number of concurrent Deployment reconciliation workers
--concurrent-deployment-syncs=5       # Default: 5
# Increase for large clusters with many Deployments

# Number of concurrent ReplicaSet reconciliation workers
--concurrent-replicaset-syncs=5       # Default: 5

# Number of concurrent endpoint syncs (affects Service updates during rollout)
--concurrent-endpoint-syncs=5         # Default: 5

# API server request rate limits
--kube-api-qps=20                     # Default: 20 requests/sec
--kube-api-burst=30                   # Default: 30 burst capacity

# Terminated Pod garbage collection threshold
--terminated-pod-gc-threshold=12500   # Default: 12500

# Minimum resync period for informers
--min-resync-period=12h               # Default: 12h
```

### 17.2 Concurrency Tuning by Cluster Size

| Cluster Size | Deployments | `--concurrent-deployment-syncs` | `--kube-api-qps` | `--kube-api-burst` |
|---|---|---|---|---|
| Small (<100 nodes, <500 pods) | <100 | 5 (default) | 20 (default) | 30 (default) |
| Medium (100-500 nodes) | 100-500 | 10 | 50 | 75 |
| Large (500-2000 nodes) | 500-2000 | 20 | 100 | 150 |
| Very Large (>2000 nodes) | >2000 | 50 | 200 | 300 |

> ⚠️ **Warning**: Increasing concurrency increases API server load proportionally. Always scale `--kube-api-qps` and `--kube-api-burst` together with sync concurrency.

### 17.3 Resource Recommendations for kube-controller-manager

```yaml
# Static Pod spec resource section
resources:
  requests:
    cpu: "200m"
    memory: "512Mi"
  limits:
    cpu: "2000m"
    memory: "2Gi"
```

**Cluster-size based recommendations:**

| Cluster Size | CPU Request | Memory Request | CPU Limit | Memory Limit |
|---|---|---|---|---|
| Small | 100m | 256Mi | 500m | 512Mi |
| Medium | 200m | 512Mi | 1000m | 1Gi |
| Large | 500m | 1Gi | 2000m | 2Gi |
| Very Large | 1000m | 2Gi | 4000m | 4Gi |

### 17.4 Rolling Update Performance Considerations

```yaml
# For maximum update speed (risk: higher disruption)
strategy:
  rollingUpdate:
    maxUnavailable: 50%
    maxSurge: 50%

# For maximum availability (slower update)
strategy:
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1          # Only one new Pod created at a time

# For resource-constrained environments (no extra pods)
strategy:
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 0
```

### 17.5 Profiling the Controller Manager

```bash
# Enable pprof (default: enabled, disable in production)
--profiling=false    # Production recommendation

# If profiling enabled, access CPU profile
kubectl -n kube-system port-forward \
  $(kubectl -n kube-system get pods -l component=kube-controller-manager -o name | head -1) \
  10257:10257 &

# CPU profile (30 second capture)
curl -sk https://localhost:10257/debug/pprof/profile?seconds=30 > cpu.prof
go tool pprof -http=:8080 cpu.prof

# Memory profile
curl -sk https://localhost:10257/debug/pprof/heap > heap.prof
go tool pprof -http=:8081 heap.prof
```

### 17.6 Deployment Rollout Performance Tuning

```bash
# For large-scale deployments, tune these Deployment spec fields:

# Increase batch size during rollout
spec:
  strategy:
    rollingUpdate:
      maxUnavailable: 20%   # Update 20% of pods at a time
      maxSurge: 20%         # Allow 20% extra pods during rollout

# Reduce readiness wait to speed up rollouts
spec:
  minReadySeconds: 0        # Don't wait after readiness (risky)
  # vs
  minReadySeconds: 30       # Wait 30s for stability (safer)

# Set aggressive progress deadline for faster failure detection
spec:
  progressDeadlineSeconds: 120   # Fail after 2 min of no progress
```

---

## 18. Security Hardening

### 18.1 RBAC for the Deployment Controller

The Deployment controller uses a dedicated service account with precisely scoped permissions:

```yaml
# Deployment Controller ClusterRole (simplified)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:controller:deployment-controller
rules:
# Manage Deployments
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments/status"]
  verbs: ["update", "patch"]
- apiGroups: ["apps"]
  resources: ["deployments/finalizers"]
  verbs: ["update"]

# Manage ReplicaSets
- apiGroups: ["apps"]
  resources: ["replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

# Read Pods
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]

# Create Events
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch", "update"]
---
# Bind to deployment-controller service account
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:controller:deployment-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:controller:deployment-controller
subjects:
- kind: ServiceAccount
  name: deployment-controller
  namespace: kube-system
```

### 18.2 TLS Configuration

```bash
# All communication is TLS-encrypted
--tls-cert-file=/etc/kubernetes/pki/controller-manager.crt
--tls-private-key-file=/etc/kubernetes/pki/controller-manager.key
--client-ca-file=/etc/kubernetes/pki/ca.crt

# Require client certificates for metrics/health endpoint access
--requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
--requestheader-allowed-names=front-proxy-client

# Disable plaintext HTTP port completely
--port=0              # Disables --address endpoint (insecure, deprecated)
--secure-port=10257   # Only serve HTTPS
```

### 18.3 Service Account Security

```bash
# Each controller uses its own service account (not the default)
--use-service-account-credentials=true

# Service account token signing key
--service-account-private-key-file=/etc/kubernetes/pki/sa.key

# Root CA for verifying service accounts
--root-ca-file=/etc/kubernetes/pki/ca.crt
```

### 18.4 Pod Security for Application Deployments

```yaml
# Apply Pod security best practices in Deployment spec
spec:
  template:
    spec:
      # Run as non-root user
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault

      containers:
      - name: my-app
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE  # Only if binding to port <1024
```

### 18.5 RBAC for Application Deployments

```yaml
# Minimal ServiceAccount for application Pods
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
automountServiceAccountToken: false   # Don't auto-mount API token
---
# Only grant what the application actually needs
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-app-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["my-app-config"]    # Only specific ConfigMap
  verbs: ["get", "watch"]
```

### 18.6 Security Hardening Checklist

| Category | Item | Priority |
|---|---|---|
| **TLS** | Disable insecure port (`--port=0`) | 🔴 Critical |
| **TLS** | Use strong cipher suites | 🟡 Important |
| **RBAC** | `--use-service-account-credentials=true` | 🔴 Critical |
| **RBAC** | Review and scope ClusterRoles | 🟡 Important |
| **Profiling** | `--profiling=false` in production | 🟡 Important |
| **Pods** | `readOnlyRootFilesystem: true` | 🟡 Important |
| **Pods** | `allowPrivilegeEscalation: false` | 🔴 Critical |
| **Pods** | `runAsNonRoot: true` | 🔴 Critical |
| **Pods** | Drop ALL capabilities | 🟡 Important |
| **Pods** | `automountServiceAccountToken: false` | 🟡 Important |
| **Network** | Define NetworkPolicies | 🟡 Important |
| **Admission** | Enable Pod Security Admission | 🔴 Critical |
| **Secrets** | Never use env vars for secrets (use secretKeyRef) | 🟡 Important |
| **Images** | Use specific digest instead of tag | 🟢 Recommended |
| **Images** | Scan images in CI pipeline | 🟢 Recommended |

---

## 19. Monitoring & Observability

### 19.1 Metrics Endpoint

```bash
# kube-controller-manager exposes Prometheus metrics at:
https://<control-plane>:10257/metrics

# Access via kubectl port-forward
kubectl -n kube-system port-forward \
  $(kubectl -n kube-system get pods -l component=kube-controller-manager -o name | head -1) \
  10257:10257

curl -sk https://localhost:10257/metrics | grep deployment
```

### 19.2 Key Prometheus Metrics

#### Deployment-Specific Metrics

| Metric | Type | Description |
|---|---|---|
| `kube_deployment_status_replicas` | Gauge | Current replica count |
| `kube_deployment_status_replicas_available` | Gauge | Available replicas |
| `kube_deployment_status_replicas_ready` | Gauge | Ready replicas |
| `kube_deployment_status_replicas_updated` | Gauge | Updated replicas (during rollout) |
| `kube_deployment_status_observed_generation` | Gauge | Last observed generation |
| `kube_deployment_spec_replicas` | Gauge | Desired replica count |
| `kube_deployment_spec_paused` | Gauge | `1` if deployment is paused |
| `kube_deployment_status_condition` | Gauge | Deployment condition status |

#### Work Queue Metrics

| Metric | Type | Description |
|---|---|---|
| `workqueue_adds_total{name="deployment"}` | Counter | Total items added to deployment work queue |
| `workqueue_depth{name="deployment"}` | Gauge | Current depth of deployment work queue |
| `workqueue_queue_duration_seconds{name="deployment"}` | Histogram | Time item waits in queue |
| `workqueue_work_duration_seconds{name="deployment"}` | Histogram | Reconciliation processing time |
| `workqueue_retries_total{name="deployment"}` | Counter | Total retries (indicates errors) |
| `workqueue_longest_running_processor_seconds` | Gauge | Longest running worker |

#### Leader Election Metrics

| Metric | Type | Description |
|---|---|---|
| `leader_election_master_status{name="kube-controller-manager"}` | Gauge | `1` = this instance is leader |

#### API Server Client Metrics

| Metric | Type | Description |
|---|---|---|
| `rest_client_requests_total` | Counter | Total HTTP requests to API server |
| `rest_client_request_duration_seconds` | Histogram | API request latency |
| `rest_client_rate_limiter_duration_seconds` | Histogram | Rate limiter wait time |

### 19.3 Prometheus Alerting Rules

```yaml
groups:
- name: kubernetes-deployments
  rules:

  # Deployment not available
  - alert: DeploymentReplicasMismatch
    expr: |
      kube_deployment_spec_replicas != kube_deployment_status_replicas_available
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Deployment {{ $labels.deployment }} has mismatched replicas"
      description: "Desired: {{ $value }} but available replicas differ"

  # Rollout stuck
  - alert: DeploymentRolloutStuck
    expr: |
      kube_deployment_status_condition{condition="Progressing",status="false"} == 1
    for: 0m
    labels:
      severity: critical
    annotations:
      summary: "Deployment {{ $labels.deployment }} rollout is stuck"

  # No available replicas
  - alert: DeploymentNoAvailableReplicas
    expr: |
      kube_deployment_status_replicas_available == 0
        and
      kube_deployment_spec_replicas > 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Deployment {{ $labels.deployment }} has zero available replicas"

  # Controller manager leader election alert
  - alert: KubeControllerManagerNotLeader
    expr: leader_election_master_status{job="kube-controller-manager"} == 0
    for: 5m
    annotations:
      summary: "kube-controller-manager instance is not the leader"

  # High work queue depth
  - alert: ControllerWorkQueueDepthHigh
    expr: workqueue_depth{name="deployment"} > 100
    for: 5m
    annotations:
      summary: "Deployment controller work queue is backed up"

  # High reconciliation errors
  - alert: ControllerReconcileErrors
    expr: rate(workqueue_retries_total{name="deployment"}[5m]) > 1
    for: 5m
    annotations:
      summary: "Deployment controller has high retry rate"
```

### 19.4 Useful Grafana PromQL Queries

```promql
# Deployment availability ratio
kube_deployment_status_replicas_available / kube_deployment_spec_replicas

# Deployments in progress (during rollout)
sum by (deployment, namespace) (
  kube_deployment_status_replicas_updated != kube_deployment_spec_replicas
)

# Controller-manager API request rate
rate(rest_client_requests_total{job="kube-controller-manager"}[5m])

# P99 reconciliation latency for deployments
histogram_quantile(0.99,
  rate(workqueue_work_duration_seconds_bucket{name="deployment"}[5m])
)

# Work queue processing rate
rate(workqueue_adds_total{name="deployment"}[5m])

# Leader election status across all instances
leader_election_master_status{job="kube-controller-manager"}
```

### 19.5 Deployment Monitoring from kubectl

```bash
# Real-time rollout status
kubectl rollout status deployment/my-app -n production --timeout=5m

# Watch replica status during rollout
kubectl get deployment my-app -n production -w

# Events for a Deployment
kubectl describe deployment my-app -n production | grep -A 20 "Events:"

# Detailed Pod status
kubectl get pods -l app=my-app -n production -o wide

# Previous container logs (after restart)
kubectl logs <pod-name> -c my-app --previous -n production
```

---

## 20. Troubleshooting with kubectl

### 20.1 Pods Not Being Created

```bash
# STEP 1: Check Deployment status
kubectl get deployment my-app -n production
# Look for: READY, UP-TO-DATE, AVAILABLE columns

# STEP 2: Describe for events
kubectl describe deployment my-app -n production
# Focus on Events section — look for:
# - FailedCreate: Quota exceeded, image pull errors
# - ProgressDeadlineExceeded: Rollout stalled

# STEP 3: Check ReplicaSet
kubectl get rs -l app=my-app -n production
kubectl describe rs <rs-name> -n production

# STEP 4: Check for resource quota issues
kubectl describe resourcequota -n production
kubectl get events -n production --field-selector reason=FailedCreate

# STEP 5: Check node capacity
kubectl describe nodes | grep -A 10 "Allocated resources"
kubectl top nodes

# STEP 6: Check if image is pullable
kubectl run debug-pull --image=my-app:v2.1.0 --restart=Never -n production
kubectl describe pod debug-pull -n production | grep -A 5 "Events:"

# STEP 7: Check controller-manager logs
kubectl logs -n kube-system -l component=kube-controller-manager \
  --tail=200 | grep -i "deployment\|error\|fail"

# STEP 8: Check admission webhooks
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
# Failing webhooks can block Pod creation

# STEP 9: Check PodSecurityAdmission
kubectl get events -n production | grep -i "security\|policy\|privilege"
```

### 20.2 Deployment Stuck in Rolling Update

```bash
# STEP 1: Check rollout status and identify which condition failed
kubectl rollout status deployment/my-app -n production
kubectl get deployment my-app -n production -o jsonpath='{.status.conditions}' \
  | python3 -m json.tool

# STEP 2: Identify old vs new ReplicaSet
kubectl get rs -l app=my-app -n production \
  -o custom-columns="NAME:.metadata.name,DESIRED:.spec.replicas,READY:.status.readyReplicas,IMAGE:.spec.template.spec.containers[0].image"

# STEP 3: Check new Pod status
kubectl get pods -l app=my-app -n production --sort-by=.metadata.creationTimestamp
kubectl describe pod <newest-pod> -n production

# STEP 4: Check container logs
kubectl logs -l app=my-app -n production --tail=50

# STEP 5: Check readiness probe
kubectl describe pod <pod-name> -n production | grep -A 10 "Readiness:"
# If readiness failing, check probe endpoint
kubectl exec -it <pod-name> -n production -- curl -v localhost:8080/ready

# STEP 6: Check CrashLoopBackOff
kubectl get pods -l app=my-app -n production | grep CrashLoopBackOff
kubectl logs <crashing-pod> -n production --previous

# STEP 7: Check for OOMKilled
kubectl describe pod <pod-name> -n production | grep -i "OOM\|killed\|reason"

# STEP 8: Check progressDeadlineSeconds
kubectl get deployment my-app -n production \
  -o jsonpath='{.spec.progressDeadlineSeconds}'
# If exceeded, ProgressDeadlineExceeded condition will be False

# STEP 9: Manual rollback if needed
kubectl rollout undo deployment/my-app -n production
kubectl rollout status deployment/my-app -n production

# STEP 10: Pause rollout to investigate
kubectl rollout pause deployment/my-app -n production
# ... investigate ...
kubectl rollout resume deployment/my-app -n production
```

### 20.3 Node NotReady

```bash
# STEP 1: Identify NotReady nodes
kubectl get nodes
kubectl get nodes --field-selector status.conditions[?(@.type=="Ready")].status=False

# STEP 2: Describe node for conditions and events
kubectl describe node <node-name>
# Check: Conditions, Capacity, Allocated Resources, Events

# STEP 3: Identify specific condition
kubectl get node <node-name> -o jsonpath='{.status.conditions}' \
  | python3 -m json.tool
# Check: MemoryPressure, DiskPressure, PIDPressure, NetworkUnavailable

# STEP 4: Check kubelet on the node
ssh <node-name>
systemctl status kubelet
journalctl -u kubelet --since "10 minutes ago" --no-pager

# STEP 5: Check container runtime
systemctl status containerd  # or docker
crictl ps                     # List running containers

# STEP 6: Check node resources
free -h          # Memory
df -h            # Disk
top              # CPU/Process

# STEP 7: Check network connectivity
ping <api-server-ip>
curl -k https://<api-server-ip>:6443/healthz

# STEP 8: Impact on Deployments — check orphaned Pods
kubectl get pods -n production --field-selector spec.nodeName=<node-name>
kubectl get pods -n production | grep -v Running

# STEP 9: Cordon the node (prevent new scheduling)
kubectl cordon <node-name>

# STEP 10: Drain for maintenance
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --grace-period=60 \
  --timeout=300s

# STEP 11: Uncordon after fix
kubectl uncordon <node-name>
```

### 20.4 Common Error Messages and Solutions

| Error | Cause | Solution |
|---|---|---|
| `FailedCreate: exceeded quota` | ResourceQuota limit hit | Increase quota or reduce replicas |
| `FailedCreate: no nodes available` | All nodes unschedulable | Add nodes, check taints/tolerations |
| `Back-off pulling image` | Image not found or auth failure | Check image name, `imagePullSecrets` |
| `Readiness probe failed` | App not ready | Check probe path, timeout, app startup time |
| `OOMKilled` | Container exceeded memory limit | Increase memory limit |
| `CrashLoopBackOff` | Container keeps crashing | Check logs, startup command |
| `Pending: Insufficient cpu` | Not enough CPU on any node | Reduce requests or scale cluster |
| `ProgressDeadlineExceeded` | Rollout stalled | Fix underlying issue, then rollback |
| `ImagePullBackOff` | Can't pull image | Verify registry access, credentials |

---

## 21. Disaster Recovery

### 21.1 Stateless Nature of Controller Manager

The `kube-controller-manager` stores **no persistent state**. Its entire operational state is:
- Informer cache (rebuilt from API server on restart)
- Work queue (rebuilt from informer events on restart)
- Leader election lease (in etcd, via API server)

```
What the controller-manager needs to recover:
  ✅ API server must be accessible
  ✅ etcd must be healthy (via API server)
  ✅ Service account tokens must be valid
  ❌ No local state to restore
  ❌ No disk backups needed for controller-manager itself
```

### 21.2 Recovery Sequence After Control Plane Failure

```
Phase 1: etcd Restored
  └─ etcd cluster back online from backup
  └─ All Deployment/RS/Pod objects restored to last snapshot state

Phase 2: kube-apiserver Restarts
  └─ Reads state from etcd
  └─ Serving API requests
  └─ Watch connections accepted

Phase 3: kube-controller-manager Restarts
  └─ Leader election: one instance wins
  └─ All informers run initial LIST to rebuild cache
  └─ Cache populated with current state from API server
  └─ Reconciliation loops start

Phase 4: Full Reconciliation
  └─ Deployment controller checks all Deployments
  └─ Mismatches detected:
     - Missing Pods → created
     - Orphaned RSes → cleaned up
     - Node health rechecked
  └─ Cluster converges to desired state

Phase 5: Application Recovery
  └─ kube-scheduler reschedules unbound Pods
  └─ kubelet on each node recreates missing containers
  └─ Services start routing to healthy Pods
```

### 21.3 etcd Backup Strategy

Since all desired state (Deployments, ReplicaSets, Pods, ConfigMaps, Secrets) lives in etcd, etcd backup is the **most critical disaster recovery task**:

```bash
# Automated etcd backup script
#!/bin/bash
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_DIR="/backup/etcd"
SNAPSHOT="${BACKUP_DIR}/etcd-snapshot-${TIMESTAMP}.db"

mkdir -p ${BACKUP_DIR}

ETCDCTL_API=3 etcdctl snapshot save ${SNAPSHOT} \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/peer.crt \
  --key=/etc/kubernetes/pki/etcd/peer.key

# Verify backup integrity
ETCDCTL_API=3 etcdctl snapshot status ${SNAPSHOT} --write-out=table

# Upload to S3
aws s3 cp ${SNAPSHOT} s3://my-cluster-backups/etcd/${TIMESTAMP}/

# Retain last 7 days
find ${BACKUP_DIR} -name "*.db" -mtime +7 -delete

echo "Backup complete: ${SNAPSHOT}"
```

```yaml
# CronJob for automated etcd backup
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"    # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          tolerations:
          - key: node-role.kubernetes.io/control-plane
            operator: Exists
            effect: NoSchedule
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          containers:
          - name: etcd-backup
            image: bitnami/etcd:3.5
            command: ["/scripts/backup.sh"]
            volumeMounts:
            - name: etcd-certs
              mountPath: /etc/kubernetes/pki/etcd
              readOnly: true
            - name: backup
              mountPath: /backup
          volumes:
          - name: etcd-certs
            hostPath:
              path: /etc/kubernetes/pki/etcd
          - name: backup
            hostPath:
              path: /var/backup/etcd
          restartPolicy: OnFailure
```

### 21.4 Deployment-Specific Disaster Recovery

```bash
# Export all Deployments for external backup
kubectl get deployments --all-namespaces -o yaml > deployments-backup.yaml

# Export with namespace context
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  kubectl get deployments -n $ns -o yaml > "deployments-${ns}.yaml" 2>/dev/null
done

# Restore a specific Deployment
kubectl apply -f deployments-production.yaml

# Force rollout after etcd restore (trigger reconciliation)
kubectl rollout restart deployment/my-app -n production
```

### 21.5 DR Best Practices

| Practice | Implementation | Frequency |
|---|---|---|
| etcd snapshots | CronJob every 6h | Every 6 hours |
| Off-cluster backup | Upload to S3/GCS/Azure Blob | With every snapshot |
| Backup retention | 7 daily, 4 weekly, 12 monthly | Automated lifecycle policy |
| Restore drills | Spin up test cluster, restore etcd | Monthly |
| Manifest backups | GitOps repo (Flux/ArgoCD) | On every change |
| Runbook documentation | DR procedures documented | Review quarterly |
| RTO/RPO definition | Define and test targets | Annually |

---

## 22. Interview Questions & Answers

### Beginner Level

**Q1: What is the difference between a Pod, ReplicaSet, and Deployment?**

> A **Pod** is the smallest deployable unit — one or more containers running together. A **ReplicaSet** ensures a specified number of identical Pods are always running — if one dies, a new one is created. A **Deployment** manages ReplicaSets and adds update and rollback capabilities. In practice: you always use Deployments for stateless apps; you rarely create ReplicaSets or Pods directly.

**Q2: What is a rolling update in Kubernetes?**

> A rolling update gradually replaces old Pods with new ones, maintaining availability throughout the transition. The Deployment controller creates a new ReplicaSet with the updated Pod template, then incrementally scales it up while scaling the old ReplicaSet down. The rate is controlled by `maxUnavailable` (how many old Pods can be removed at a time) and `maxSurge` (how many extra Pods can exist above the desired count). The result is zero-downtime upgrades.

**Q3: How do you roll back a Deployment?**

> ```bash
> kubectl rollout undo deployment/my-app
> ```
> This switches the active ReplicaSet back to the previous revision. For a specific revision: `kubectl rollout undo deployment/my-app --to-revision=3`. The history is maintained by old ReplicaSets (scaled to 0) retained up to `revisionHistoryLimit`.

**Q4: What does `kubectl rollout status` tell you?**

> It watches and reports the progress of a Deployment rollout in real time. It exits 0 when the rollout completes successfully (all desired Pods updated and available), exits non-zero if the rollout is stuck or fails. It's useful in CI/CD pipelines to gate deployments: `kubectl rollout status deployment/my-app --timeout=5m || kubectl rollout undo deployment/my-app`.

---

### Intermediate Level

**Q5: Explain `maxUnavailable: 0` and `maxSurge: 1`. When would you use this?**

> With `maxUnavailable: 0`, no existing Pods are ever terminated before a replacement is Ready. With `maxSurge: 1`, only one new Pod is created at a time. This combination is the **safest rolling update strategy**: the cluster always has the full desired Pod count available to serve traffic, at the cost of temporarily using one extra Pod's worth of resources. Use this when: zero-downtime is absolutely required, you have tight resource constraints (only 1 extra Pod), and you can afford a slower rollout.

**Q6: What is the purpose of `minReadySeconds` in a Deployment?**

> Without `minReadySeconds`, a Pod that briefly passes its readiness probe before crashing is counted as "available", causing the rollout to proceed with potentially broken Pods. Setting `minReadySeconds: 30` means a Pod must remain Ready for 30 continuous seconds before it's counted as available. This adds stability — transient readiness failures don't mistakenly advance the rollout. The tradeoff is a slower rollout (each Pod adds `minReadySeconds` to the total time).

**Q7: What happens when `progressDeadlineSeconds` is exceeded?**

> The Deployment's `Progressing` condition is set to `False` with reason `ProgressDeadlineExceeded`. The **controller does NOT automatically roll back** — this is intentional. The operator must investigate and decide: fix the issue and let the rollout continue, or manually rollback with `kubectl rollout undo`. This avoids automatic rollbacks on legitimate slow rollouts (like a large cluster rolling update). You can detect this in CI/CD: `kubectl rollout status` returns non-zero when the deadline is exceeded.

**Q8: How does the Deployment controller know which ReplicaSet to update vs. create during a rolling update?**

> The Deployment controller uses the **pod-template-hash** label — a hash of the Pod template spec — to identify ReplicaSets. If the new template matches an existing RS (same hash), it scales that RS up. If the template is different (new image, new env var, etc.), it creates a new RS. This means if you revert a Deployment to a previously used image, the controller finds the existing RS for that template and scales it back up — no new RS created.

**Q9: What is the reconciliation loop and why is it eventually consistent?**

> The reconciliation loop continuously compares desired state (Deployment spec) with actual state (running Pods/ReplicaSets) and takes corrective action. It's "eventually consistent" because: corrections require API calls that take time, Pods take time to start/stop, informer events have latency, and work queues process items with rate limiting. The system is never "instantly" in sync but always converging toward the desired state. This is a deliberate design choice — overly aggressive immediate reconciliation could cascade failures.

---

### Advanced Level

**Q10: How does the Deployment controller prevent creating duplicate ReplicaSets when multiple controllers compete in a non-HA scenario where leader election fails?**

> The Deployment controller uses **optimistic concurrency control** via the `resourceVersion` field on Kubernetes objects. When it tries to create a new ReplicaSet, it includes the current `resourceVersion` of the Deployment in its write operations. If another controller instance has already modified the Deployment (incrementing `resourceVersion`), the API server returns `409 Conflict`. The controller re-reads the Deployment, discovers the RS was already created, and takes no further action. This prevents duplicate RSes even if leader election briefly allows two active leaders.

**Q11: Explain how informers prevent API server overload in large clusters with hundreds of Deployments.**

> Each controller uses a **SharedInformerFactory** that creates one watch connection per resource type, shared across all controller instances and all objects of that type. So even with 500 Deployments, there is still only ONE watch stream from the controller-manager to the API server for Deployment objects. All 500 Deployment objects' states are delivered through this single stream and cached locally. Controllers read from the local cache (zero API calls) and only make API calls for writes. Without informers, each controller would need to poll all 500 Deployments every few seconds — O(N) API calls per second.

**Q12: What are the implications of setting `revisionHistoryLimit: 0`?**

> Setting it to 0 means old ReplicaSets are deleted immediately after a rollout completes, freeing etcd space and reducing clutter. The implications: (1) **no rollback capability** — there are no old RSes to scale back up; (2) **no rollout history** — `kubectl rollout history` shows nothing; (3) if a bad release is deployed, you must create a new Deployment update (not rollback). This is acceptable only for CI/CD environments with frequent deploys where rollback is handled by the pipeline, or where etcd size is a primary constraint.

**Q13: How does the Deployment controller handle a Deployment with `paused: true`?**

> When `spec.paused: true`, the Deployment controller skips all reconciliation for that Deployment. No new RSes are created, no scaling occurs, no rollouts progress. The Deployment enters a "frozen" state. The `Progressing` condition remains `True` but the rollout halts. This is used to stage updates: `kubectl rollout pause deployment/my-app` → make multiple changes (image, env, resources) → `kubectl rollout resume deployment/my-app` → a single rollout incorporates all changes. Without pause, each change would trigger a separate rollout.

**Q14: Walk me through what happens internally when you run `kubectl set image deployment/my-app app=nginx:1.25`.**

> 1. kubectl sends `PATCH /apis/apps/v1/namespaces/production/deployments/my-app` to API server, updating `spec.template.spec.containers[0].image` to `nginx:1.25` and incrementing `metadata.generation`.
> 2. API server validates, runs admission webhooks, persists the update to etcd, increments `resourceVersion`.
> 3. API server delivers a MODIFIED event to all watchers — including the Deployment controller's informer.
> 4. Informer updates local cache, calls `OnUpdate` event handler.
> 5. Event handler adds `production/my-app` key to the work queue.
> 6. A worker goroutine dequeues the key and calls `syncDeployment`.
> 7. Controller computes new `pod-template-hash` for `nginx:1.25` template — different from current hash.
> 8. No existing RS matches the new hash → create new RS `my-app-<new-hash>`.
> 9. Controller begins rolling update: scale new RS up, old RS down per `maxSurge`/`maxUnavailable`.
> 10. kube-scheduler binds new Pods to nodes; kubelet starts `nginx:1.25` containers.
> 11. As new Pods become Ready, the old RS is scaled further down.
> 12. Rollout completes when new RS has all replicas; old RS scaled to 0.
> 13. Deployment `status.observedGeneration` updated to match `metadata.generation`.

---

## 23. Real-World Production Use Cases

### 23.1 Zero-Downtime Deployment with Pre/Post Hooks

```yaml
# Using lifecycle hooks for graceful shutdown
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0     # Never reduce capacity
      maxSurge: 1
  template:
    spec:
      terminationGracePeriodSeconds: 90
      containers:
      - name: my-app
        lifecycle:
          preStop:
            exec:
              # Signal app to stop accepting new requests
              # but finish in-flight requests
              command: ["/bin/sh", "-c", "sleep 15"]
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          # Remove from Service BEFORE termination
          # Combined with preStop sleep = graceful drain
```

### 23.2 Blue-Green Deployment

```bash
# Create Green deployment (new version)
kubectl apply -f deployment-green.yaml

# Verify green is healthy
kubectl rollout status deployment/my-app-green -n production
kubectl get deployment my-app-green -o jsonpath='{.status.readyReplicas}'

# Atomic traffic switch (update Service selector)
kubectl patch service my-app -n production \
  -p '{"spec":{"selector":{"slot":"green"}}}'

# Monitor for errors (wait 10 minutes)
sleep 600
kubectl logs -l slot=green,app=my-app -n production --tail=50

# If healthy: scale down blue
kubectl scale deployment my-app-blue -n production --replicas=0

# If unhealthy: instant rollback
kubectl patch service my-app -n production \
  -p '{"spec":{"selector":{"slot":"blue"}}}'
```

### 23.3 Canary Deployment

```bash
# Stable: 9 replicas serving 90% traffic
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: my-app
      track: stable
  template:
    metadata:
      labels:
        app: my-app
        track: stable
    spec:
      containers:
      - name: app
        image: my-app:v1.0
EOF

# Canary: 1 replica serving 10% traffic
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      track: canary
  template:
    metadata:
      labels:
        app: my-app
        track: canary
    spec:
      containers:
      - name: app
        image: my-app:v2.0
EOF

# Service routes to ALL pods (both stable and canary)
# because it only selects on app: my-app
```

### 23.4 CI/CD Pipeline Integration

```bash
#!/bin/bash
# Production deployment script

set -euo pipefail

APP="my-app"
NAMESPACE="production"
IMAGE="registry.company.com/my-app:${CI_COMMIT_SHA}"
TIMEOUT="10m"

echo "=== Deploying ${IMAGE} to ${NAMESPACE} ==="

# 1. Run pre-deployment checks
kubectl get deployment ${APP} -n ${NAMESPACE}

# 2. Update image
kubectl set image deployment/${APP} ${APP}=${IMAGE} -n ${NAMESPACE}

# 3. Annotate with change cause
kubectl annotate deployment/${APP} -n ${NAMESPACE} \
  kubernetes.io/change-cause="CI build ${CI_PIPELINE_ID}: ${CI_COMMIT_MESSAGE}" \
  --overwrite

# 4. Wait for rollout
if ! kubectl rollout status deployment/${APP} -n ${NAMESPACE} --timeout=${TIMEOUT}; then
  echo "=== Rollout FAILED — initiating rollback ==="
  kubectl rollout undo deployment/${APP} -n ${NAMESPACE}
  kubectl rollout status deployment/${APP} -n ${NAMESPACE} --timeout=5m
  echo "=== Rollback complete ==="
  exit 1
fi

# 5. Verify health
READY=$(kubectl get deployment ${APP} -n ${NAMESPACE} \
  -o jsonpath='{.status.readyReplicas}')
DESIRED=$(kubectl get deployment ${APP} -n ${NAMESPACE} \
  -o jsonpath='{.spec.replicas}')

if [ "${READY}" != "${DESIRED}" ]; then
  echo "=== Health check failed: ${READY}/${DESIRED} ready ==="
  kubectl rollout undo deployment/${APP} -n ${NAMESPACE}
  exit 1
fi

echo "=== Deployment successful: ${READY}/${DESIRED} replicas ready ==="
```

### 23.5 Feature Flag Deployment Pattern

```yaml
# Deploy with feature flags via ConfigMap reference
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
      - name: my-app
        image: my-app:v2.0
        envFrom:
        - configMapRef:
            name: my-app-feature-flags
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-app-feature-flags
data:
  FEATURE_NEW_CHECKOUT: "false"
  FEATURE_AI_RECOMMENDATIONS: "true"
  FEATURE_DARK_MODE: "true"
```

```bash
# Enable feature without redeployment
kubectl patch configmap my-app-feature-flags -n production \
  -p '{"data":{"FEATURE_NEW_CHECKOUT":"true"}}'

# Restart to pick up ConfigMap change
kubectl rollout restart deployment/my-app -n production
```

---

## 24. Best Practices for Production

### 24.1 Deployment Spec Best Practices

| Category | Practice | Reason |
|---|---|---|
| **Replicas** | `replicas ≥ 2` for all production services | Eliminate single point of failure |
| **Strategy** | Use `RollingUpdate` with `maxUnavailable: 0, maxSurge: 1` for critical services | Zero-downtime guarantee |
| **Resources** | Always set `requests` AND `limits` | Prevent noisy neighbor issues, proper scheduling |
| **Probes** | All three: startup + readiness + liveness | Proper lifecycle management |
| **minReadySeconds** | Set to `30` or higher | Prevent rolling out with transiently-ready Pods |
| **Image tags** | Use specific immutable tags (not `latest`) | Reproducible, auditable deployments |
| **Image digest** | Use digest (`image@sha256:...`) for absolute immutability | Defense against tag mutation |
| **PDB** | Define `PodDisruptionBudget` for all production deployments | Protect against concurrent node drains |
| **Topology** | Use `topologySpreadConstraints` | Spread across zones for HA |
| **Anti-affinity** | Prefer `podAntiAffinity` on `hostname` | Avoid all Pods on same node |
| **History** | Set `revisionHistoryLimit: 10` | Maintain rollback capability |
| **Progress deadline** | Set `progressDeadlineSeconds: 300-600` | Fast detection of stuck rollouts |
| **Change cause** | Always annotate with `kubernetes.io/change-cause` | Audit trail for rollout history |
| **Labels** | Use consistent, meaningful label schema | Enables monitoring, routing, filtering |
| **Graceful shutdown** | Set `terminationGracePeriodSeconds` appropriately | Allow in-flight requests to complete |
| **Security context** | `readOnlyRootFilesystem`, `runAsNonRoot`, drop ALL caps | Defense in depth |

### 24.2 GitOps Best Practices

```
repository structure:
  deploy/
    base/
      deployment.yaml
      service.yaml
      kustomization.yaml
    overlays/
      staging/
        kustomization.yaml  (patch: image tag, replicas)
      production/
        kustomization.yaml  (patch: image tag, replicas, resources)
```

```bash
# ArgoCD or Flux: auto-sync from Git
# All changes go through PR → review → merge → automatic sync
# Rollback = revert Git commit

# Emergency rollback (bypass GitOps)
kubectl rollout undo deployment/my-app -n production
# Then create PR to sync Git state
```

### 24.3 Pod Disruption Budget

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
  namespace: production
spec:
  # Ensure at least 3 Pods are ALWAYS available during voluntary disruptions
  minAvailable: 3
  # OR: Allow at most 1 Pod to be unavailable at a time
  # maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
```

---

## 25. Common Mistakes and Pitfalls

### 25.1 The `latest` Tag Problem

```yaml
# ❌ WRONG: Non-deterministic — different nodes may run different images
containers:
- name: app
  image: my-app:latest
  imagePullPolicy: Always   # Re-pulls on every Pod restart

# ✅ CORRECT: Immutable, auditable
containers:
- name: app
  image: my-app:v2.1.0
  imagePullPolicy: IfNotPresent

# ✅ BEST: Content-addressed — tag mutation cannot affect you
containers:
- name: app
  image: my-app@sha256:a1b2c3d4e5f6...
```

### 25.2 Missing Readiness Probe

```yaml
# ❌ WRONG: Rolling update proceeds even if new Pods are broken
containers:
- name: app
  image: my-app:v2.0
  # No readinessProbe!
  # Deployment assumes Pod is ready as soon as container starts

# ✅ CORRECT: Rolling update waits for Pods to actually be ready
containers:
- name: app
  image: my-app:v2.0
  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
    initialDelaySeconds: 10
    periodSeconds: 5
    failureThreshold: 3
```

### 25.3 Identical Selector Conflict

```yaml
# ❌ DANGER: Two Deployments with identical selectors will fight over Pods
# deployment-a.yaml
spec:
  selector:
    matchLabels:
      app: my-app    # Conflicts with deployment-b!

# deployment-b.yaml
spec:
  selector:
    matchLabels:
      app: my-app    # Same selector — Pod chaos!

# ✅ CORRECT: Always use unique selectors per Deployment
# deployment-a.yaml
spec:
  selector:
    matchLabels:
      app: my-app
      component: frontend

# deployment-b.yaml
spec:
  selector:
    matchLabels:
      app: my-app
      component: backend
```

### 25.4 Forgetting Resource Requests

```yaml
# ❌ WRONG: No resource requests
# Pods scheduled without capacity consideration
# One bad Pod can consume all node resources
containers:
- name: app
  image: my-app:v2.0
  # No resources block

# ✅ CORRECT
containers:
- name: app
  resources:
    requests:          # Used by scheduler for placement
      cpu: "200m"
      memory: "256Mi"
    limits:            # Hard cap enforced by cgroups
      cpu: "1000m"
      memory: "512Mi"
```

### 25.5 Aggressive Liveness Probes

```yaml
# ❌ WRONG: Liveness probe that's too strict
# Causes restart loops during high load or slow startup
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 3     # Too short — app may not be ready
  periodSeconds: 3            # Checks every 3s — too frequent
  failureThreshold: 1         # Kills container on first failure!

# ✅ CORRECT: Use startup probe for slow-starting apps
# Give plenty of time to start, then monitor with liveness
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30        # 30 × 10s = 5 minutes to start
  periodSeconds: 10

livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 0      # startupProbe already handled startup
  periodSeconds: 20           # Not too frequent
  failureThreshold: 3         # 3 failures over 60s before restart
```

### 25.6 Common Pitfall Summary Table

| Pitfall | Consequence | Solution |
|---|---|---|
| `image: latest` | Non-deterministic deployments | Use specific version tags |
| No readiness probe | Traffic to unready Pods | Always configure readiness probe |
| No resource limits | Node OOM, evictions | Set requests and limits |
| No PDB | Node drain causes outage | Define PDB for all production |
| Selector overlap between Deployments | Pod ownership conflicts | Unique selectors per deployment |
| Aggressive liveness probe | Restart loops | Use startup probe, increase thresholds |
| `replicas: 1` | Single point of failure | Minimum 2 for production |
| No topology spread | All Pods on one zone | Use `topologySpreadConstraints` |
| Ignoring `progressDeadlineSeconds` | Silent failed rollouts | Set 5-10 minutes, monitor |
| `revisionHistoryLimit: 0` | No rollback capability | Set to at least 5 |
| No `change-cause` annotation | No audit trail | Annotate every rollout |
| Unconfigured `terminationGracePeriodSeconds` | In-flight request drops | Set to max request duration + buffer |
| No preStop hook | SIGTERM before load balancer drain | Add `preStop: sleep 15` |

---

## 26. Hands-On Labs

### Lab 1: Deploying and Observing a Rolling Update

```bash
# Step 1: Create initial Deployment
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lab-deploy
  annotations:
    kubernetes.io/change-cause: "Initial deployment - nginx 1.24"
spec:
  replicas: 4
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: lab-deploy
  template:
    metadata:
      labels:
        app: lab-deploy
    spec:
      containers:
      - name: nginx
        image: nginx:1.24
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 3
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
EOF

# Step 2: Watch initial rollout
kubectl rollout status deployment/lab-deploy

# Step 3: Check revision history
kubectl rollout history deployment/lab-deploy

# Step 4: In a separate terminal, watch Pods
kubectl get pods -l app=lab-deploy -w &

# Step 5: Trigger rolling update
kubectl set image deployment/lab-deploy nginx=nginx:1.25
kubectl annotate deployment/lab-deploy \
  kubernetes.io/change-cause="Upgrade to nginx 1.25" --overwrite

# Step 6: Watch the rolling update happen
kubectl rollout status deployment/lab-deploy

# Step 7: Observe the ReplicaSets
kubectl get rs -l app=lab-deploy

# Step 8: View rollout history
kubectl rollout history deployment/lab-deploy

# Step 9: Rollback
kubectl rollout undo deployment/lab-deploy
kubectl rollout status deployment/lab-deploy

# Step 10: Rollback to specific revision
kubectl rollout history deployment/lab-deploy
kubectl rollout undo deployment/lab-deploy --to-revision=1
kubectl rollout status deployment/lab-deploy

# Cleanup
kubectl delete deployment lab-deploy
```

### Lab 2: Simulating a Failed Deployment and Rollback

```bash
# Step 1: Deploy good version
kubectl create deployment fail-test --image=nginx:1.25 --replicas=3
kubectl rollout status deployment/fail-test

# Step 2: Deploy a bad version (image doesn't exist)
kubectl set image deployment/fail-test nginx=nginx:99.99.99-does-not-exist
kubectl annotate deployment/fail-test \
  kubernetes.io/change-cause="Bad deploy — wrong image" --overwrite

# Step 3: Watch the deployment fail
kubectl get pods -l app=fail-test -w &
sleep 30

# Step 4: Observe the failed Pods
kubectl get pods -l app=fail-test
kubectl describe pod $(kubectl get pods -l app=fail-test | grep -v Running | tail -1 | awk '{print $1}')

# Step 5: Rollback
kubectl rollout undo deployment/fail-test
kubectl rollout status deployment/fail-test

# Step 6: Verify recovery
kubectl get pods -l app=fail-test

# Cleanup
kubectl delete deployment fail-test
```

### Lab 3: Pause and Resume Rollout

```bash
# Step 1: Create deployment
kubectl create deployment pause-test --image=nginx:1.24 --replicas=5
kubectl rollout status deployment/pause-test

# Step 2: Pause the deployment
kubectl rollout pause deployment/pause-test

# Step 3: Make multiple changes while paused (no rollout triggered)
kubectl set image deployment/pause-test nginx=nginx:1.25
kubectl set resources deployment/pause-test \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=500m,memory=256Mi
kubectl set env deployment/pause-test NGINX_VERSION=1.25

# Step 4: Verify no rollout started (pod count unchanged)
kubectl get pods -l app=pause-test
kubectl get rs -l app=pause-test

# Step 5: Resume — all changes deployed in one rollout
kubectl rollout resume deployment/pause-test
kubectl rollout status deployment/pause-test

# Cleanup
kubectl delete deployment pause-test
```

### Lab 4: Horizontal Pod Autoscaler with Deployment

```bash
# Requires metrics-server
kubectl top nodes  # Verify metrics-server is running

# Step 1: Create deployment with resource requests
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hpa-test
  template:
    metadata:
      labels:
        app: hpa-test
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "200m"
          limits:
            cpu: "500m"
EOF

# Step 2: Expose the deployment
kubectl expose deployment hpa-test --port=80

# Step 3: Create HPA
kubectl autoscale deployment hpa-test \
  --cpu-percent=50 --min=2 --max=10

kubectl get hpa hpa-test -w &

# Step 4: Generate load
kubectl run -it load-generator --image=busybox --restart=Never -- \
  sh -c "while true; do wget -q -O- http://hpa-test; done" &

# Step 5: Watch HPA scale up the deployment
kubectl get hpa hpa-test
kubectl get deployment hpa-test -w

# Step 6: Stop load and watch scale down
kubectl delete pod load-generator

# Cleanup
kubectl delete deployment hpa-test
kubectl delete hpa hpa-test
kubectl delete service hpa-test
```

### Lab 5: Inspecting Leader Election

```bash
# Step 1: Check which controller-manager is the leader
kubectl get lease kube-controller-manager -n kube-system -o yaml

# Step 2: Extract just the leader identity
kubectl get lease kube-controller-manager -n kube-system \
  -o jsonpath='{.spec.holderIdentity}{"\n"}'

# Step 3: Check lease duration settings
kubectl get lease kube-controller-manager -n kube-system \
  -o jsonpath='{.spec.leaseDurationSeconds}{"\n"}'

# Step 4: Count leadership transitions
kubectl get lease kube-controller-manager -n kube-system \
  -o jsonpath='{.spec.leaseTransitions}{"\n"}'

# Step 5: Watch for real-time changes (useful in HA clusters)
kubectl get lease kube-controller-manager -n kube-system -w

# Step 6: Check controller-manager flags
kubectl get pod -n kube-system \
  $(kubectl get pods -n kube-system -l component=kube-controller-manager -o name | head -1 | sed 's/pod\///') \
  -o jsonpath='{.spec.containers[0].command}' | python3 -m json.tool | grep leader
```

---

## 27. Comparisons

### 27.1 Deployment vs kube-apiserver

| Aspect | Deployment (Controller) | kube-apiserver |
|---|---|---|
| **Nature** | A Kubernetes resource type + controller | Core control plane component |
| **Primary role** | Manages ReplicaSet/Pod lifecycle | REST API gateway for all cluster operations |
| **Talks to etcd** | Never (via apiserver only) | Yes — the ONLY component with etcd access |
| **Stateful** | No (in-memory cache) | Yes (via etcd) |
| **Failure impact** | No new rollouts; existing Pods continue | Cluster API unavailable; everything halts |
| **HA model** | Multiple instances via leader election | Multiple instances (all active, load-balanced) |
| **Accepts requests from** | Only API server (via informers) | All cluster components, kubectl, users |
| **Default port** | 10257 (controller-manager) | 6443 |
| **Restart behavior** | Rebuilds cache from API server | Rebuilds from etcd |
| **Update trigger** | Watches for Deployment object changes | Receives PUT/PATCH from users/controllers |

### 27.2 Deployment vs kube-scheduler

| Aspect | Deployment (Controller) | kube-scheduler |
|---|---|---|
| **Primary concern** | How many Pods? Which version? | Which node gets a Pod? |
| **Operates on** | Deployments → ReplicaSets | Unscheduled Pods |
| **Output** | Create/delete ReplicaSets and update replica counts | Set `spec.nodeName` on Pods |
| **Knows about nodes** | Only via Node controller events | Deep awareness of node capacity, taints, labels |
| **Placement logic** | Topology spread, anti-affinity (passed to scheduler) | Filtering + weighted scoring algorithms |
| **HA model** | Leader election (only one active) | Leader election (only one active) |
| **Replacement** | Custom controllers (Operator pattern) | Custom schedulers, scheduler plugins |
| **Order of operation** | Creates Pods (sets desired state) | Binds Pods to nodes (after creation) |
| **Default port** | 10257 (shared controller-manager) | 10259 |

### 27.3 Deployment vs StatefulSet vs DaemonSet

| Feature | Deployment | StatefulSet | DaemonSet |
|---|---|---|---|
| **Primary use** | Stateless services | Stateful apps | Node agents |
| **Pod naming** | Random (`abc12`) | Ordered (`pod-0`) | `<ds-name>-<node>` |
| **Update strategy** | Rolling (via RS swap) | Ordered rolling | Rolling (node-by-node) |
| **Storage** | Shared PVC (RWX) or ephemeral | Dedicated PVC per Pod | Host paths |
| **Scale down order** | Random | Reverse ordinal (N-1→0) | Follows node removal |
| **DNS** | Via Service + random | Stable headless DNS | N/A |
| **Rollback** | Via revision history (RS) | No native rollback | No native rollback |
| **Examples** | HTTP APIs, workers, caches | Postgres, Kafka, Redis | Fluentd, Prometheus node-exporter |

---

## 28. ASCII Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║                  KUBERNETES DEPLOYMENT ARCHITECTURE — FULL STACK                     ║
╠══════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                      ║
║   USER / CI-CD PIPELINE                                                              ║
║   ┌──────────────────────────────────────────────────┐                              ║
║   │  kubectl apply -f deployment.yaml                 │                              ║
║   │  kubectl set image deployment/my-app app=v2       │                              ║
║   │  kubectl rollout undo deployment/my-app           │                              ║
║   └──────────────────┬───────────────────────────────┘                              ║
║                      │ HTTPS :6443                                                   ║
║                      ▼                                                               ║
║   ╔══════════════════════════════════════════════════════════════════════╗           ║
║   ║                    CONTROL PLANE                                     ║           ║
║   ║                                                                      ║           ║
║   ║  ┌──────────────────────────────────────────────────────────────┐   ║           ║
║   ║  │                  kube-apiserver (:6443)                       │   ║           ║
║   ║  │                                                              │   ║           ║
║   ║  │  [Authn]──[Authz/RBAC]──[Admission]──[Validate]──[Persist]  │   ║           ║
║   ║  └────────────────────────┬─────────────────────────────────────┘   ║           ║
║   ║                           │ (ONLY path to etcd)                      ║           ║
║   ║                           ▼                                          ║           ║
║   ║  ┌───────────────────┐  ┌──────────────────────────────────────┐   ║           ║
║   ║  │       etcd         │  │    kube-controller-manager (:10257)   │   ║           ║
║   ║  │  (source of truth)│◄─►│                                      │   ║           ║
║   ║  │                   │  │  ┌─────────────────────────────────┐  │   ║           ║
║   ║  │  /registry/       │  │  │  Deployment Controller           │  │   ║           ║
║   ║  │  ├─deployments/   │  │  │  - Watch Deployments, RS, Pods  │  │   ║           ║
║   ║  │  ├─replicasets/   │  │  │  - Create/scale ReplicaSets     │  │   ║           ║
║   ║  │  ├─pods/          │  │  │  - Manage rolling updates       │  │   ║           ║
║   ║  │  ├─services/      │  │  │  - Track revision history       │  │   ║           ║
║   ║  │  └─...            │  │  ├─────────────────────────────────┤  │   ║           ║
║   ║  └───────────────────┘  │  │  ReplicaSet Controller           │  │   ║           ║
║   ║                          │  │  Node Controller                 │  │   ║           ║
║   ║                          │  │  Job Controller                  │  │   ║           ║
║   ║                          │  │  StatefulSet Controller          │  │   ║           ║
║   ║                          │  │  DaemonSet Controller            │  │   ║           ║
║   ║                          │  │  GC Controller                   │  │   ║           ║
║   ║                          │  └─────────────────────────────────┘  │   ║           ║
║   ║                          │                                        │   ║           ║
║   ║                          │  Leader Election: kube-system/Lease    │   ║           ║
║   ║                          │  (only ONE instance is active)         │   ║           ║
║   ║                          └──────────────────────────────────────┘   ║           ║
║   ║                                                                      ║           ║
║   ║  ┌────────────────────────────────────────────────────────────────┐  ║           ║
║   ║  │                  kube-scheduler (:10259)                       │  ║           ║
║   ║  │  - Watches unscheduled Pods                                    │  ║           ║
║   ║  │  - Filters nodes (taints, affinity, resources)                 │  ║           ║
║   ║  │  - Scores nodes (topology, spread)                             │  ║           ║
║   ║  │  - Binds Pod → Node via API server                             │  ║           ║
║   ║  └────────────────────────────────────────────────────────────────┘  ║           ║
║   ╚══════════════════════════════════════════════════════════════════════╝           ║
║                                                                                      ║
║   ╔══════════════════════════════════════════════════════════════════════╗           ║
║   ║                         WORKER NODES                                 ║           ║
║   ║                                                                      ║           ║
║   ║   Node-1 (zone-a)         Node-2 (zone-b)         Node-3 (zone-c)   ║           ║
║   ║  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  ║           ║
║   ║  │ kubelet          │  │ kubelet          │  │ kubelet          │  ║           ║
║   ║  │ kube-proxy       │  │ kube-proxy       │  │ kube-proxy       │  ║           ║
║   ║  │ containerd       │  │ containerd       │  │ containerd       │  ║           ║
║   ║  │                  │  │                  │  │                  │  ║           ║
║   ║  │ ┌────┐  ┌────┐   │  │ ┌────┐  ┌────┐  │  │ ┌────┐          │  ║           ║
║   ║  │ │Pod1│  │Pod2│   │  │ │Pod3│  │Pod4│  │  │ │Pod5│          │  ║           ║
║   ║  │ │ v2 │  │ v2 │   │  │ │ v2 │  │ v2 │  │  │ │ v2 │          │  ║           ║
║   ║  │ └────┘  └────┘   │  │ └────┘  └────┘  │  │ └────┘          │  ║           ║
║   ║  │    ReplicaSet-v2 (current, replicas=5)               │  ║           ║
║   ║  └──────────────────┘  └──────────────────┘  └──────────────────┘  ║           ║
║   ╚══════════════════════════════════════════════════════════════════════╝           ║
║                                                                                      ║
╠══════════════════════════════════════════════════════════════════════════════════════╣
║   ROLLING UPDATE FLOW (maxUnavailable=1, maxSurge=1, replicas=5)                     ║
║                                                                                      ║
║   Deployment spec changed (image v1→v2)                                              ║
║   │                                                                                  ║
║   ├─[1] New RS-v2 created (replicas=0)                                               ║
║   ├─[2] RS-v2 scaled to 1  │  RS-v1 stays at 5  (total=6, surge=1)                  ║
║   ├─[3] RS-v2 Pod Ready    │  RS-v1 scaled to 4  (total=5, stable)                  ║
║   ├─[4] RS-v2 scaled to 2  │  RS-v1 stays at 4  (total=6, surge=1)                  ║
║   ├─[5] RS-v2 Pod Ready    │  RS-v1 scaled to 3  (total=5, stable)                  ║
║   ├─... (continues)                                                                  ║
║   └─[N] RS-v2=5, RS-v1=0   (rollout complete, RS-v1 retained for rollback)          ║
╚══════════════════════════════════════════════════════════════════════════════════════╝


╔══════════════════════════════════════════════════════════════╗
║          DEPLOYMENT OBJECT OWNERSHIP CHAIN                   ║
║                                                              ║
║  Deployment my-app                                           ║
║      │ owns (via ownerRef)                                   ║
║      ├── ReplicaSet my-app-7f8b6d9c4  (current, rs=5)       ║
║      │       │ owns (via ownerRef)                           ║
║      │       ├── Pod my-app-7f8b6d9c4-abc12  (Running)      ║
║      │       ├── Pod my-app-7f8b6d9c4-def34  (Running)      ║
║      │       ├── Pod my-app-7f8b6d9c4-ghi56  (Running)      ║
║      │       ├── Pod my-app-7f8b6d9c4-jkl78  (Running)      ║
║      │       └── Pod my-app-7f8b6d9c4-mno90  (Running)      ║
║      │                                                       ║
║      ├── ReplicaSet my-app-6b4f5a3c2  (prev, rs=0)          ║
║      └── ReplicaSet my-app-5c3e8d7b1  (old,  rs=0)          ║
║                                                              ║
║  Garbage Collection: Delete Deployment → GC deletes all     ║
║  RSes and Pods in background (or foreground cascade)        ║
╚══════════════════════════════════════════════════════════════╝
```

---

## 29. Cheat Sheet

### 29.1 Core Deployment Commands

```bash
# ── CREATE ──────────────────────────────────────────────────────────────────
kubectl create deployment <name> --image=<image> --replicas=<n>
kubectl apply -f deployment.yaml
kubectl apply -f deployment.yaml --record   # Deprecated; use annotation instead

# ── READ ────────────────────────────────────────────────────────────────────
kubectl get deployment <name>
kubectl get deployment <name> -n <namespace> -o wide
kubectl get deployment <name> -o yaml
kubectl describe deployment <name>
kubectl get deployments --all-namespaces

# ── UPDATE ──────────────────────────────────────────────────────────────────
kubectl set image deployment/<name> <container>=<new-image>
kubectl set resources deployment/<name> --requests=cpu=100m,memory=256Mi --limits=cpu=500m,memory=512Mi
kubectl set env deployment/<name> KEY=VALUE
kubectl patch deployment <name> -p '{"spec":{"replicas":5}}'
kubectl edit deployment <name>

# ── SCALE ───────────────────────────────────────────────────────────────────
kubectl scale deployment <name> --replicas=<n>
kubectl autoscale deployment <name> --cpu-percent=70 --min=2 --max=20

# ── ROLLOUT ─────────────────────────────────────────────────────────────────
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout history deployment/<name> --revision=3
kubectl rollout undo deployment/<name>
kubectl rollout undo deployment/<name> --to-revision=<n>
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>
kubectl rollout restart deployment/<name>   # Triggers rolling restart

# ── DELETE ──────────────────────────────────────────────────────────────────
kubectl delete deployment <name>
kubectl delete deployment <name> --cascade=orphan      # Keep Pods
kubectl delete deployment <name> --cascade=foreground  # Wait for children

# ── ANNOTATE (change-cause) ──────────────────────────────────────────────────
kubectl annotate deployment <name> \
  kubernetes.io/change-cause="Deploy v2.1.0 — fix memory leak" --overwrite
```

### 29.2 Inspection and Debugging Commands

```bash
# Check rollout status in detail
kubectl get deployment <name> -o jsonpath='{.status}' | python3 -m json.tool

# Check conditions
kubectl get deployment <name> \
  -o jsonpath='{range .status.conditions[*]}{.type}{"\t"}{.status}{"\t"}{.reason}{"\n"}{end}'

# Get all ReplicaSets for a Deployment
kubectl get rs -l <selector-label>=<value>
kubectl get rs --selector=app=my-app

# Compare RS images (identify current vs old)
kubectl get rs -l app=my-app \
  -o custom-columns="NAME:.metadata.name,REPLICAS:.spec.replicas,IMAGE:.spec.template.spec.containers[0].image"

# Find which revision is current
kubectl get rs -l app=my-app \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.annotations.deployment\.kubernetes\.io/revision}{"\n"}{end}'

# Watch Pods during rollout
kubectl get pods -l app=my-app -w

# Get Pod IPs during rollout
kubectl get pods -l app=my-app -o wide

# Events for namespace
kubectl get events -n production --sort-by='.lastTimestamp'
kubectl get events -n production --field-selector reason=FailedCreate

# Container logs
kubectl logs -l app=my-app --tail=50
kubectl logs <pod-name> -c <container-name> --previous

# Exec into a Pod
kubectl exec -it <pod-name> -- /bin/bash

# Check Pod owner references
kubectl get pod <pod-name> \
  -o jsonpath='{.metadata.ownerReferences[*].name}'
```

### 29.3 kube-controller-manager Flags Reference

```bash
# ── CORE ────────────────────────────────────────────────────────────────────
--kubeconfig=<path>
--master=<api-server-url>
--controllers=*                     # Run all controllers
--use-service-account-credentials=true

# ── LEADER ELECTION ─────────────────────────────────────────────────────────
--leader-elect=true
--leader-elect-lease-duration=15s
--leader-elect-renew-deadline=10s
--leader-elect-retry-period=2s
--leader-elect-resource-lock=leases
--leader-elect-resource-name=kube-controller-manager
--leader-elect-resource-namespace=kube-system

# ── CONCURRENCY ─────────────────────────────────────────────────────────────
--concurrent-deployment-syncs=5
--concurrent-replicaset-syncs=5
--concurrent-endpoint-syncs=5
--concurrent-statefulset-syncs=10
--concurrent-daemonset-syncs=2
--concurrent-job-syncs=5
--concurrent-gc-syncs=20
--concurrent-namespace-syncs=10

# ── API RATE LIMITING ───────────────────────────────────────────────────────
--kube-api-qps=20
--kube-api-burst=30

# ── NODE CONTROLLER ─────────────────────────────────────────────────────────
--node-monitor-grace-period=40s
--node-monitor-period=5s
--pod-eviction-timeout=5m0s
--node-eviction-rate=0.1
--secondary-node-eviction-rate=0.01
--unhealthy-zone-threshold=0.55

# ── SECURITY ────────────────────────────────────────────────────────────────
--tls-cert-file=<path>
--tls-private-key-file=<path>
--client-ca-file=<path>
--port=0                            # Disable insecure HTTP port
--secure-port=10257
--profiling=false                   # Disable in production

# ── GARBAGE COLLECTION ──────────────────────────────────────────────────────
--terminated-pod-gc-threshold=12500

# ── LOGGING ─────────────────────────────────────────────────────────────────
--v=2                               # Verbosity (0=quiet, 9=extremely verbose)
```

### 29.4 Deployment Spec Quick Reference

```yaml
spec:
  replicas: 3                          # Desired Pod count
  revisionHistoryLimit: 10             # Old RSes to keep
  progressDeadlineSeconds: 600         # Seconds before "stuck" declaration
  minReadySeconds: 30                  # Seconds Ready before "available"
  paused: false                        # Freeze rollouts

  selector:                            # IMMUTABLE — choose carefully
    matchLabels:
      app: my-app

  strategy:
    type: RollingUpdate                # or Recreate
    rollingUpdate:
      maxUnavailable: 1                # Absolute or percentage
      maxSurge: 1                      # Absolute or percentage

  template:
    metadata:
      labels:
        app: my-app                    # Must match selector
      annotations:
        prometheus.io/scrape: "true"

    spec:
      terminationGracePeriodSeconds: 60
      serviceAccountName: my-app-sa
      automountServiceAccountToken: false

      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: my-app

      containers:
      - name: app
        image: my-app:v1.0.0           # Never use 'latest'
        imagePullPolicy: IfNotPresent

        resources:
          requests:
            cpu: "200m"
            memory: "256Mi"
          limits:
            cpu: "1000m"
            memory: "512Mi"

        startupProbe:
          httpGet: {path: /startup, port: 8080}
          failureThreshold: 30
          periodSeconds: 10

        readinessProbe:
          httpGet: {path: /ready, port: 8080}
          initialDelaySeconds: 10
          periodSeconds: 5

        livenessProbe:
          httpGet: {path: /live, port: 8080}
          initialDelaySeconds: 30
          periodSeconds: 20
          failureThreshold: 3

        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]

        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop: ["ALL"]
```

### 29.5 One-Liners for Production Operations

```bash
# Get all Deployments with their images
kubectl get deployments --all-namespaces \
  -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,IMAGE:.spec.template.spec.containers[0].image,REPLICAS:.spec.replicas"

# Find Deployments not at desired replicas
kubectl get deployments --all-namespaces \
  -o json | jq -r '.items[] | select(.spec.replicas != .status.availableReplicas) | "\(.metadata.namespace)/\(.metadata.name): desired=\(.spec.replicas) available=\(.status.availableReplicas)"'

# Quick rollout of all deployments in namespace (e.g., after secret rotation)
kubectl rollout restart deployments -n production

# Wait for all deployments in namespace to complete
kubectl get deployments -n production -o name | \
  xargs -I{} kubectl rollout status {} -n production

# Export all deployments (for GitOps reconciliation)
kubectl get deployments --all-namespaces -o yaml | \
  grep -v "resourceVersion\|uid\|selfLink\|generation\|creationTimestamp" > all-deployments.yaml

# Check which Pods have been up longest (old Pods = no rollout)
kubectl get pods -n production -l app=my-app \
  --sort-by=.metadata.creationTimestamp

# Find Pods that are not Running
kubectl get pods --all-namespaces --field-selector status.phase!=Running | \
  grep -v Completed

# Get container images across all namespaces
kubectl get pods --all-namespaces \
  -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.spec.containers[*].image}{"\n"}{end}' | sort -u
```

---

## 30. Key Takeaways & Summary

### The Five Core Principles of Deployments

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  1. DECLARATIVE DESIRED STATE                                        │
│     Declare WHAT you want. Kubernetes figures out HOW to get there. │
│                                                                     │
│  2. CONTINUOUS RECONCILIATION                                        │
│     The loop never stops. Drift from desired state is always fixed. │
│                                                                     │
│  3. ZERO-DOWNTIME UPDATES                                            │
│     Rolling updates ensure service availability during deploys.     │
│                                                                     │
│  4. INSTANT ROLLBACK                                                 │
│     Old ReplicaSets are never deleted — they're kept for rollback.  │
│                                                                     │
│  5. API-SERVER AS THE SINGLE SOURCE OF TRUTH                         │
│     All state flows through the API server. No shortcuts to etcd.   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Architecture Summary Table

| Layer | Component | Role |
|---|---|---|
| **Storage** | etcd | Source of truth for all cluster state |
| **Gateway** | kube-apiserver | Validates, persists, serves all API operations |
| **Reconciliation** | kube-controller-manager | Continuously reconciles actual to desired state |
| **Placement** | kube-scheduler | Assigns unscheduled Pods to nodes |
| **Execution** | kubelet | Creates and manages containers on nodes |
| **Desired State** | Deployment | Declares application desired state |
| **Replica Control** | ReplicaSet | Maintains exact Pod count |
| **Unit** | Pod | Runs the actual application container |

### Deployment Feature Matrix

| Feature | Deployment | RS | StatefulSet | DaemonSet |
|---|---|---|---|---|
| Zero-downtime update | ✅ | ❌ | ✅ (ordered) | ✅ |
| Rollback | ✅ | ❌ | ❌ | ❌ |
| Arbitrary replica count | ✅ | ✅ | ✅ | ❌ (=node count) |
| Stable Pod identity | ❌ | ❌ | ✅ | Partial |
| Per-Pod storage | ❌ | ❌ | ✅ | N/A |
| One-per-node guarantee | ❌ | ❌ | ❌ | ✅ |
| Pause/resume rollouts | ✅ | ❌ | ❌ | ❌ |
| HPA support | ✅ | ✅ | ✅ | ❌ |

### Production Readiness Checklist

```
□ replicas ≥ 2 (never single-replica production services)
□ Resource requests AND limits defined for all containers
□ Startup probe configured (for slow-starting apps)
□ Readiness probe configured (correctly — not too aggressive)
□ Liveness probe configured (with appropriate thresholds)
□ topologySpreadConstraints defined (zone spreading)
□ podAntiAffinity configured (node spreading)
□ PodDisruptionBudget created and tested
□ revisionHistoryLimit: 10 set
□ progressDeadlineSeconds: 300-600 set
□ minReadySeconds: 30 set
□ Specific image tags used (not 'latest')
□ preStop hook for graceful drain
□ terminationGracePeriodSeconds set appropriately
□ securityContext: readOnlyRootFilesystem, runAsNonRoot, drop ALL caps
□ automountServiceAccountToken: false (unless needed)
□ Prometheus alerts configured (replica mismatch, rollout stuck)
□ GitOps: all changes via Pull Request to Git repository
□ CI/CD: automated rollback on failed rollout status
□ PDB tested with node drain
□ Rollback tested regularly
□ etcd backups automated and restore-tested
```

### Mental Model

> Think of a Deployment as a **contract with Kubernetes**: *"I want 5 instances of version 2.1 of my app always running, and when I update to version 2.2, replace them gradually with no downtime, and keep the old version ready for me to roll back to."* The Deployment controller is the **enforcement agent** of that contract — watching 24/7, correcting any deviation, and orchestrating updates with surgical precision. Every tool in its arsenal — ReplicaSets, rolling update math, revision history — exists to fulfill that contract reliably at any scale.

---

*Guide Version: 1.0 | Kubernetes: v1.29+ | Last Updated: 2024*
*Maintained by: Platform Engineering Team*

---

> 📌 **Reference Links**:
> - [Official Kubernetes Docs: Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
> - [Deployment API Reference](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/deployment-v1/)
> - [kube-controller-manager Flags](https://kubernetes.io/docs/reference/command-line-tools-reference/kube-controller-manager/)
> - [Rolling Update Best Practices](https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/)
> - [Pod Disruption Budgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/)
