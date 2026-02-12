# The Complete Guide to kube-scheduler in Kubernetes
## From Absolute Beginner to Production Expert

## 1. What Is kube-scheduler & Why Kubernetes Needs It

### 1.1 Core Identity

The **kube-scheduler** is the control plane component responsible for assigning newly created pods to nodes. It doesn't run the pods — that's the kubelet's job. The scheduler only makes the placement decision.

| **Attribute**       | **Value**                                    |
|---------------------|----------------------------------------------|
| **Binary**          | `kube-scheduler`                             |
| **Default Port**    | 10259 (metrics/health)                       |
| **Primary Role**    | Watch unscheduled pods, assign to nodes      |
| **Decision Input**  | Node resources, affinities, taints, policies |
| **Decision Output** | Updates `pod.spec.nodeName`                  |
| **Runs As**         | Static pod on control plane nodes            |

### 1.2 The Scheduling Problem

When you create a pod with `kubectl apply -f pod.yaml`, the API server stores it in etcd with `spec.nodeName` empty (unscheduled). The scheduler's job:

1. **Watch** for pods with empty `nodeName`
2. **Filter** nodes that can run the pod (predicates/filters)
3. **Score** remaining nodes (priorities/scoring)
4. **Bind** pod to the highest-scoring node (write `nodeName`)

```
┌─────────────────────────────────────────────────────────┐
│  User creates Pod                                       │
│    kubectl apply -f pod.yaml                            │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │   kube-apiserver     │
          │  stores pod in etcd  │
          │  nodeName: ""        │ ← unscheduled
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │   kube-scheduler     │
          │  watches unscheduled │
          │  picks best node     │
          │  binds pod → node    │
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │   kube-apiserver     │
          │  updates pod:        │
          │  nodeName: "node-3"  │ ← scheduled!
          └──────────┬───────────┘
                     │
                     ▼
          ┌──────────────────────┐
          │   kubelet (node-3)   │
          │  sees pod assigned   │
          │  runs containers     │
          └──────────────────────┘
```

### 1.3 Why Separate the Scheduler?

**Single Responsibility:** The scheduler only decides placement. It doesn't manage pod lifecycle, networking, or storage — that's kubelet, CNI, CSI. This separation enables:

- **Custom schedulers** for special workloads (GPU, ML, batch)
- **Scheduler extenders** to add custom logic
- **Multi-scheduler** clusters (different pods use different schedulers)

### 1.4 Key Concepts

- **Unscheduled Pod:** `spec.nodeName == ""` or `nil`
- **Scheduled Pod:** `spec.nodeName == "worker-node-5"`
- **Predicate/Filter:** Pass/fail — can this node run the pod?
- **Priority/Score:** 0-100 — how good is this node for the pod?
- **Binding:** Writing the `nodeName` to the pod object

---

## 2. How the Scheduler Works — The Scheduling Cycle

### 2.1 The Scheduling Loop (Simplified)

```go
// Pseudo-code
for {
    pod := waitForUnscheduledPod()
    
    // Phase 1: Filtering (Predicates)
    feasibleNodes := filterNodes(allNodes, pod)
    if len(feasibleNodes) == 0 {
        markPodUnschedulable(pod)
        continue
    }
    
    // Phase 2: Scoring (Priorities)
    scores := scoreNodes(feasibleNodes, pod)
    bestNode := pickHighestScore(scores)
    
    // Phase 3: Binding
    bindPodToNode(pod, bestNode)
}
```

### 2.2 Detailed Scheduling Cycle

#### Step 1: Watch for Unscheduled Pods

The scheduler uses an **Informer** (from client-go) to watch pods where `spec.nodeName` is empty.

```yaml
# Example unscheduled pod in etcd:
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
spec:
  nodeName: ""    # ← Scheduler will fill this
  containers:
  - name: nginx
    image: nginx:1.21
```

#### Step 2: Filtering Phase (Predicates)

For each node, run ~15 predicate checks. All must pass.

| **Predicate**                    | **Checks**                                              |
|----------------------------------|---------------------------------------------------------|
| `PodFitsResources`               | Node has enough CPU/memory?                             |
| `PodFitsHostPorts`               | Host port not already in use?                           |
| `MatchNodeSelector`              | Pod's `nodeSelector` matches node labels?               |
| `NoVolumeZoneConflict`           | PV's zone matches node's zone?                          |
| `NoDiskConflict`                 | Requested volumes not already mounted on node?          |
| `PodToleratesNodeTaints`         | Pod tolerates node's taints?                            |
| `CheckNodeCondition`             | Node is `Ready`, not under memory/disk pressure?        |
| `CheckVolumeBinding`             | Can PVCs bind to available PVs?                         |
| `MatchInterPodAffinity`          | Pod affinity/anti-affinity rules satisfied?             |

**Example Failure:**

```
Pod requests:
  cpu: 4 cores
  memory: 16Gi

Node has:
  cpu: 8 cores total, 7 cores allocated
  memory: 32Gi total, 20Gi allocated

Available:
  cpu: 1 core  ← INSUFFICIENT
  memory: 12Gi

Result: Node FILTERED OUT (PodFitsResources fails)
```

#### Step 3: Scoring Phase (Priorities)

Each remaining node gets scored 0-100 by multiple priority functions. Scores are weighted and summed.

| **Priority Function**              | **Favors**                                         | **Weight** |
|------------------------------------|----------------------------------------------------|------------|
| `LeastRequestedPriority`           | Nodes with more available resources                | 1          |
| `BalancedResourceAllocation`       | Nodes where CPU and memory usage are balanced      | 1          |
| `SelectorSpreadPriority`           | Spread pods across nodes (service anti-affinity)   | 1          |
| `NodeAffinityPriority`             | Nodes matching `preferredDuringScheduling` rules   | 1          |
| `InterPodAffinityPriority`         | Nodes satisfying pod affinity preferences          | 1          |
| `TaintTolerationPriority`          | Nodes with fewer untolerated taints                | 1          |
| `ImageLocalityPriority`            | Nodes that already have the image cached           | 1          |

**Example Scoring:**

```
Pod requests: cpu=2, memory=4Gi

Node A:
  Available: cpu=6/8, memory=16/32Gi
  LeastRequestedPriority: (6/8)*10 + (16/32)*10 = 12.5
  BalancedResourceAllocation: score based on (cpu% - mem%) = high
  Total: 85/100

Node B:
  Available: cpu=2/8, memory=4/32Gi
  LeastRequestedPriority: (2/8)*10 + (4/32)*10 = 3.75
  BalancedResourceAllocation: score = medium
  Total: 45/100

Winner: Node A (score=85 > 45)
```

#### Step 4: Binding

The scheduler updates the pod's `spec.nodeName` via the API server.

```bash
# Scheduler calls:
# PATCH /api/v1/namespaces/default/pods/nginx/binding
# Body: {"target": {"name": "node-a", "kind": "Node"}}
```

The API server:
1. Validates the binding
2. Writes `nodeName: "node-a"` to etcd
3. Notifies kubelet on node-a via watch stream
4. Kubelet pulls image and starts container

---

## 3. Predicates (Filtering) — Feasible Nodes

### 3.1 What Are Predicates?

**Predicates** are boolean checks that determine if a node **can** run a pod. If any predicate fails, the node is excluded from consideration.

### 3.2 Key Predicates Deep Dive

#### PodFitsResources

Checks if node has enough CPU, memory, ephemeral-storage, and extended resources (GPU).

```yaml
# Pod requests:
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: "2"
        memory: "4Gi"
      limits:
        cpu: "4"
        memory: "8Gi"

# Scheduler checks:
# Node allocatable: cpu=8, memory=32Gi
# Already allocated: cpu=6, memory=24Gi
# Available: cpu=2, memory=8Gi
# Can fit? YES (requests: 2 cpu, 4Gi < available)
```

**Important:** Scheduler uses **requests**, not **limits**, for scheduling decisions.

#### PodToleratesNodeTaints

Nodes can have **taints** that repel pods unless the pod has a matching **toleration**.

```yaml
# Node with taint:
apiVersion: v1
kind: Node
metadata:
  name: gpu-node-1
spec:
  taints:
  - key: nvidia.com/gpu
    value: "true"
    effect: NoSchedule

# Pod WITHOUT toleration: FAILS predicate
# Pod WITH toleration: PASSES predicate
spec:
  tolerations:
  - key: nvidia.com/gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
```

#### MatchNodeSelector

Simple label-based node selection.

```yaml
# Pod with nodeSelector:
spec:
  nodeSelector:
    disktype: ssd

# Only nodes with label disktype=ssd pass this predicate
```

#### CheckNodeCondition

Ensures node is healthy.

```bash
kubectl describe node worker-1
# Conditions:
#   Ready: True
#   MemoryPressure: False
#   DiskPressure: False
#   PIDPressure: False

# If MemoryPressure=True → node FILTERED OUT
```

#### VolumeBinding

Checks if PersistentVolumeClaims can bind to available PersistentVolumes in the node's zone/region.

```yaml
# PVC requests:
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 100Gi

# Available PV must:
# 1. Have ≥100Gi capacity
# 2. Match accessMode
# 3. Be in same zone as node (for zonal storage)
```

### 3.3 Custom Predicates (Advanced)

You can add custom predicates via:
- **Scheduler Extenders** (webhook to external service)
- **Custom Scheduler** (write your own in Go using scheduler framework)

---

## 4. Priorities (Scoring) — Best Node Selection

### 4.1 What Are Priorities?

**Priorities** rank nodes that passed filtering. Each priority function scores nodes 0-100. Weighted sum determines the winner.

### 4.2 Key Priority Functions

#### LeastRequestedPriority

**Goal:** Spread pods across nodes with most available resources.

**Formula:**
```
score = ((nodeCapacity - nodeRequested) / nodeCapacity) * 10
```

**Example:**
```
Node A: cpu=8, requested=2 → (8-2)/8 * 10 = 7.5
Node B: cpu=8, requested=6 → (8-6)/8 * 10 = 2.5
Node A scores higher (more available)
```

#### BalancedResourceAllocation

**Goal:** Balance CPU and memory utilization (avoid skew).

**Formula:**
```
score = 10 - abs(cpuFraction - memoryFraction) * 10
```

**Example:**
```
Node A: cpu=50% used, memory=50% used → abs(0.5-0.5)*10=0 → score=10
Node B: cpu=80% used, memory=20% used → abs(0.8-0.2)*10=6 → score=4
Node A scores higher (balanced)
```

#### NodeAffinityPriority

**Goal:** Favor nodes matching `preferredDuringScheduling` affinity rules.

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: [us-west-2a]

# Nodes in us-west-2a get +100 to their score
```

#### InterPodAffinityPriority

**Goal:** Co-locate pods that should be near each other (e.g., web + cache).

```yaml
spec:
  affinity:
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 50
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: redis
          topologyKey: kubernetes.io/hostname

# Nodes running redis pods get +50 score
```

---

## 5. Scheduler & etcd Interaction (via API Server)

### 5.1 Scheduler NEVER Talks to etcd Directly

**Critical Architecture Rule:** Like all Kubernetes components except the API server, the scheduler only talks to the kube-apiserver over HTTPS. The API server is the sole gateway to etcd.

```
┌──────────────────┐
│  kube-scheduler  │
└────────┬─────────┘
         │ HTTPS (REST API / Watch streams)
         ▼
┌──────────────────┐
│ kube-apiserver   │
└────────┬─────────┘
         │ gRPC (TLS)
         ▼
┌──────────────────┐
│      etcd        │
└──────────────────┘
```

### 5.2 What the Scheduler Reads (via API Server)

1. **Pods** — watches for `spec.nodeName == ""`
2. **Nodes** — caches all nodes with their capacity, conditions, taints
3. **PersistentVolumes / PersistentVolumeClaims** — for volume binding checks
4. **Services / ReplicaSets** — for pod affinity/anti-affinity calculations
5. **ResourceQuotas / LimitRanges** — to respect namespace quotas

### 5.3 What the Scheduler Writes

1. **Pod bindings** — updates `pod.spec.nodeName`
2. **Events** — writes scheduling events for debugging

```bash
# Events the scheduler creates:
kubectl describe pod my-pod
# Events:
#   Normal   Scheduled  10s  default-scheduler  Successfully assigned default/my-pod to worker-2
#   Warning  FailedScheduling  15s  default-scheduler  0/3 nodes available: insufficient cpu
```

### 5.4 Authentication to API Server

The scheduler uses a kubeconfig with a client certificate:

```yaml
# /etc/kubernetes/scheduler.conf
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/pki/ca.crt
    server: https://lb:6443
  name: kubernetes
users:
- name: system:kube-scheduler
  user:
    client-certificate: /etc/kubernetes/pki/scheduler.crt
    client-key: /etc/kubernetes/pki/scheduler.key
```

---

## 6. Node Affinity & Anti-Affinity

### 6.1 What Is Node Affinity?

**Node Affinity** lets you constrain which nodes a pod can be scheduled on, based on node labels. It's more expressive than `nodeSelector`.

### 6.2 requiredDuringScheduling (Hard Constraint)

Pod **will NOT** be scheduled if no node matches.

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values: [ssd, nvme]

# Translation: "Pod MUST land on a node with disktype=ssd OR disktype=nvme"
# If no such node exists → pod stays Pending
```

**Operators:**
- `In` — label value must be in list
- `NotIn` — label value must NOT be in list
- `Exists` — label key must exist (value doesn't matter)
- `DoesNotExist` — label key must NOT exist
- `Gt` — label value > number (for numeric labels)
- `Lt` — label value < number

### 6.3 preferredDuringScheduling (Soft Constraint)

Pod **prefers** nodes matching the rule, but can land elsewhere if needed.

```yaml
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values: [us-west-2a]
      - weight: 50
        preference:
          matchExpressions:
          - key: instance-type
            operator: In
            values: [m5.xlarge]

# Translation: "Strongly prefer us-west-2a (weight=100), somewhat prefer m5.xlarge (weight=50)"
# Scheduler adds these weights to node scores
```

### 6.4 Real-World Example: Multi-Zone Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 6
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: failure-domain.beta.kubernetes.io/zone
                operator: In
                values: [us-west-2a, us-west-2b, us-west-2c]

# Ensures all 6 replicas land in one of these three zones
# Protects against single-zone failure
```

---

## 7. Pod Affinity & Anti-Affinity

### 7.1 What Is Pod Affinity?

**Pod Affinity** co-locates pods (e.g., put web pods near cache pods).  
**Pod Anti-Affinity** separates pods (e.g., spread replicas across nodes).

### 7.2 Pod Affinity (Co-Location)

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: redis
        topologyKey: kubernetes.io/hostname

# Translation: "This pod MUST be on the SAME node as a pod with app=redis"
# If no node has a redis pod → this pod stays Pending
```

**topologyKey** defines the "closeness":
- `kubernetes.io/hostname` → same physical node
- `topology.kubernetes.io/zone` → same availability zone
- `topology.kubernetes.io/region` → same region

### 7.3 Pod Anti-Affinity (Separation)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: web
            topologyKey: kubernetes.io/hostname

# Translation: "No two pods with app=web can be on the same node"
# Guarantees 3 replicas land on 3 different nodes
# If cluster has only 2 nodes → 3rd pod stays Pending
```

### 7.4 Preferred Anti-Affinity (Soft Separation)

```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web
          topologyKey: kubernetes.io/hostname

# Translation: "Try to avoid putting two web pods on same node, but it's OK if unavoidable"
# Useful when you have 10 replicas but only 3 nodes
```

### 7.5 Real-World Example: Database HA

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  replicas: 3
  template:
    spec:
      affinity:
        # ANTI-AFFINITY: Spread across nodes
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: postgres
            topologyKey: kubernetes.io/hostname
        # NODE AFFINITY: Only on SSD nodes
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values: [ssd]

# Result: 3 PostgreSQL replicas, each on a different node, all on SSD
```

---

## 8. Taints, Tolerations & Node Selectors

### 8.1 Taints & Tolerations

**Taints** repel pods from nodes.  
**Tolerations** allow pods to ignore taints.

#### Taint a Node

```bash
kubectl taint nodes node-1 key=value:NoSchedule
```

**Effect Options:**
- `NoSchedule` — new pods won't schedule (existing pods stay)
- `PreferNoSchedule` — avoid scheduling if possible (soft)
- `NoExecute` — new pods won't schedule + evict existing pods without toleration

#### Tolerate the Taint

```yaml
spec:
  tolerations:
  - key: key
    operator: Equal
    value: value
    effect: NoSchedule
```

**Operators:**
- `Equal` — key and value must match
- `Exists` — key must exist (value ignored)

#### Example: Dedicated GPU Nodes

```bash
# Taint all GPU nodes
kubectl taint nodes gpu-node-1 nvidia.com/gpu=true:NoSchedule
kubectl taint nodes gpu-node-2 nvidia.com/gpu=true:NoSchedule

# Only GPU pods can schedule here:
spec:
  tolerations:
  - key: nvidia.com/gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  nodeSelector:
    accelerator: nvidia-tesla-v100
```

### 8.2 Node Selectors (Simple Label Matching)

```yaml
spec:
  nodeSelector:
    disktype: ssd
    region: us-west

# Pod will ONLY schedule on nodes with BOTH labels
```

**When to use:**
- **nodeSelector** — simple, single-label matching
- **nodeAffinity** — complex logic (In, NotIn, Exists, numeric comparisons)

### 8.3 Built-In Taints

Kubernetes automatically taints nodes in certain conditions:

| **Taint**                              | **Meaning**                                   |
|----------------------------------------|-----------------------------------------------|
| `node.kubernetes.io/not-ready`         | Node is NotReady (kubelet down)               |
| `node.kubernetes.io/unreachable`       | Node is unreachable                           |
| `node.kubernetes.io/memory-pressure`   | Node has memory pressure                      |
| `node.kubernetes.io/disk-pressure`     | Node has disk pressure                        |
| `node.kubernetes.io/network-unavailable` | Node network is unavailable                 |
| `node.kubernetes.io/unschedulable`     | Node is marked unschedulable (cordon)         |

Pods can tolerate these to handle special cases (e.g., DaemonSets tolerate `not-ready` to run on all nodes).

---

## 9. Resource Requests, Limits & QoS Classes

### 9.1 Requests vs Limits

**Requests:**
- What the scheduler uses for placement decisions
- Guaranteed minimum resources
- Sum of all pod requests on a node ≤ node allocatable

**Limits:**
- Maximum resources a container can use
- Enforced by cgroup
- Container killed if it exceeds memory limit (OOM)
- CPU is throttled if it exceeds CPU limit

```yaml
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: "500m"     # 0.5 cores
        memory: "256Mi"
      limits:
        cpu: "1"        # 1 core max
        memory: "512Mi" # killed if exceeded
```

### 9.2 How Scheduler Uses Requests

```
Node capacity: cpu=4, memory=16Gi
Already scheduled pods request: cpu=2.5, memory=10Gi
Available: cpu=1.5, memory=6Gi

New pod requests: cpu=1, memory=4Gi
Can fit? YES
  cpu: 1 < 1.5 ✓
  memory: 4Gi < 6Gi ✓

New pod requests: cpu=2, memory=8Gi
Can fit? NO
  cpu: 2 > 1.5 ✗ → PodFitsResources predicate fails
```

### 9.3 QoS Classes

Kubernetes assigns a **Quality of Service** class based on requests/limits:

| **QoS Class**   | **Criteria**                                             | **Eviction Priority** |
|-----------------|----------------------------------------------------------|-----------------------|
| **Guaranteed**  | requests == limits for all containers (cpu & memory)     | Last (most protected) |
| **Burstable**   | At least one container has request OR limit (not equal)  | Middle                |
| **BestEffort**  | No requests or limits set                                | First (evicted first) |

**Example:**

```yaml
# Guaranteed QoS
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: "1"
        memory: "1Gi"
      limits:
        cpu: "1"        # same as request
        memory: "1Gi"   # same as request

# Burstable QoS
spec:
  containers:
  - name: app
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "2"        # higher than request
        memory: "2Gi"

# BestEffort QoS
spec:
  containers:
  - name: app
    # No resources specified
```

**When Node Runs Out of Memory:**
1. BestEffort pods evicted first
2. Burstable pods evicted next (those using most memory over request)
3. Guaranteed pods evicted last (only if system critical)

---

## 10. Topology Spread Constraints

### 10.1 What Is Topology Spread?

**Topology Spread Constraints** evenly distribute pods across failure domains (zones, nodes, racks) to improve availability.

### 10.2 Basic Example: Spread Across Nodes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 6
  template:
    metadata:
      labels:
        app: web
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: web

# Translation: "Spread 6 web pods across nodes"
# maxSkew=1 means: max difference between any two nodes is 1 pod
#
# Good distribution (3 nodes):
#   node-1: 2 pods
#   node-2: 2 pods
#   node-3: 2 pods  ✓ maxSkew satisfied (2-2=0 < 1)
#
# Bad distribution (would be rejected):
#   node-1: 4 pods
#   node-2: 1 pod
#   node-3: 1 pod  ✗ maxSkew violated (4-1=3 > 1)
```

### 10.3 Spread Across Zones

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web

# If 9 replicas across 3 zones:
#   us-west-2a: 3 pods
#   us-west-2b: 3 pods
#   us-west-2c: 3 pods
```

### 10.4 whenUnsatisfiable Options

- **DoNotSchedule** (hard) — pod stays Pending if constraint can't be satisfied
- **ScheduleAnyway** (soft) — try to satisfy, but schedule anyway if impossible

### 10.5 Multiple Constraints

```yaml
spec:
  topologySpreadConstraints:
  # Spread across zones (higher priority)
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web
  # Spread across nodes within each zone (lower priority)
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: web
```

---

## 11. Custom Schedulers & Scheduler Extenders

### 11.1 Why Custom Schedulers?

Default scheduler is general-purpose. Custom schedulers handle special cases:
- **GPU scheduling** (advanced placement based on GPU memory, model)
- **Batch workloads** (pack jobs tightly to minimize nodes)
- **Cost optimization** (prefer spot instances)
- **Compliance** (data must stay in specific regions)

### 11.2 Running Multiple Schedulers

```yaml
# Deploy custom scheduler as a pod
apiVersion: v1
kind: Pod
metadata:
  name: my-custom-scheduler
  namespace: kube-system
spec:
  containers:
  - name: scheduler
    image: my-custom-scheduler:1.0
    command:
    - /usr/local/bin/kube-scheduler
    - --config=/etc/kubernetes/my-scheduler-config.yaml
```

**Use custom scheduler for specific pods:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-job
spec:
  schedulerName: my-custom-scheduler  # ← not "default-scheduler"
  containers:
  - name: app
    image: ml-training:latest
```

### 11.3 Scheduler Extenders (Webhooks)

Extend default scheduler with HTTP callbacks.

```yaml
# Scheduler config with extender
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
extenders:
- urlPrefix: "http://my-extender-service:8888"
  filterVerb: "filter"
  prioritizeVerb: "prioritize"
  weight: 5
  enableHTTPS: false
```

**Extender receives:**
```json
POST /filter
{
  "nodes": [...],
  "pod": {...}
}
```

**Extender responds:**
```json
{
  "nodes": ["node-a", "node-c"],  // filtered list
  "failedNodes": {
    "node-b": "GPU not available"
  }
}
```

### 11.4 Scheduler Framework (Plugins)

Modern approach (v1.19+): write plugins in Go.

**Plugin points:**
- **PreFilter** — runs before filtering
- **Filter** — add custom predicate
- **PostFilter** — runs if no nodes passed filtering
- **PreScore** — runs before scoring
- **Score** — add custom priority function
- **Reserve** — reserve resources before binding
- **Permit** — approve or deny final decision
- **PreBind / Bind / PostBind** — custom binding logic

---

## 12. Scheduler Performance & Tuning

### 12.1 Scheduler Throughput

Default scheduler handles ~1000 pods/second in large clusters. Bottlenecks:
- **API server latency** (network, etcd)
- **Node count** (more nodes = more scoring work)
- **Affinity complexity** (inter-pod affinity is expensive)

### 12.2 Critical Flags

```bash
--kube-api-qps=50              # API requests/sec (increase for large clusters)
--kube-api-burst=100           # Burst allowance
--percentageOfNodesToScore=50  # Only score top 50% of nodes (performance vs accuracy trade-off)
```

**percentageOfNodesToScore Explanation:**

In a 1000-node cluster:
- Default scheduler filters all 1000 nodes
- Scores top 500 (50%)
- Picks best from those 500

**Trade-off:**
- Lower percentage → faster scheduling, possibly suboptimal placement
- Higher percentage → slower scheduling, better placement
- 100% → always optimal (but slow for huge clusters)

### 12.3 Resource Sizing

```yaml
# Production scheduler pod resources:
resources:
  requests:
    cpu: 200m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 2Gi

# For clusters with 500+ nodes:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 2000m
    memory: 4Gi
```

### 12.4 Leader Election

Like controller-manager, multiple scheduler instances can run with leader election:

```bash
--leader-elect=true
--leader-elect-lease-duration=15s
--leader-elect-renew-deadline=10s
```

Only the leader actively schedules. Others standby for failover.

---

## 13. Scheduler Security & RBAC

### 13.1 Scheduler Service Account Permissions

The scheduler needs these RBAC permissions:

```yaml
# ClusterRole: system:kube-scheduler
rules:
# Watch unscheduled pods
- apiGroups: [""]
  resources: [pods]
  verbs: [get, list, watch]

# Update pod bindings
- apiGroups: [""]
  resources: [pods/binding]
  verbs: [create]

# Read nodes
- apiGroups: [""]
  resources: [nodes]
  verbs: [get, list, watch]

# Read PVs/PVCs for volume binding
- apiGroups: [""]
  resources: [persistentvolumes, persistentvolumeclaims]
  verbs: [get, list, watch]

# Create events
- apiGroups: [""]
  resources: [events]
  verbs: [create, patch, update]

# Leader election (Lease objects)
- apiGroups: [coordination.k8s.io]
  resources: [leases]
  verbs: [get, create, update]
```

### 13.2 Secure Flags

```bash
--authentication-kubeconfig=/etc/kubernetes/scheduler.conf
--authorization-kubeconfig=/etc/kubernetes/scheduler.conf
--kubeconfig=/etc/kubernetes/scheduler.conf

# TLS
--tls-cert-file=/etc/kubernetes/pki/scheduler.crt
--tls-private-key-file=/etc/kubernetes/pki/scheduler.key

# Secure metrics
--bind-address=127.0.0.1  # or use TLS + auth for external scraping
```

---

## 14. Monitoring & Observability

### 14.1 Metrics Endpoint

Exposed on port 10259 (HTTPS, requires auth).

```bash
curl -k --cert /etc/kubernetes/pki/client.crt \
  --key /etc/kubernetes/pki/client.key \
  https://localhost:10259/metrics
```

### 14.2 Critical Metrics

| **Metric**                                  | **What It Tells You**                                      |
|---------------------------------------------|------------------------------------------------------------|
| `scheduler_pending_pods`                    | Pods waiting to be scheduled (should be near 0)            |
| `scheduler_schedule_attempts_total`         | Total scheduling attempts (by result: success, error, unschedulable) |
| `scheduler_scheduling_duration_seconds`     | How long scheduling takes (p50, p95, p99)                  |
| `scheduler_e2e_scheduling_duration_seconds` | End-to-end latency from pod creation to binding            |
| `scheduler_framework_extension_point_duration_seconds` | Time spent in each plugin (filter, score, bind) |
| `rest_client_requests_total`                | API server requests (watch for rate limiting)              |

### 14.3 Prometheus Alerts

```yaml
groups:
- name: kube-scheduler
  rules:
  - alert: SchedulerHighPendingPods
    expr: scheduler_pending_pods > 100
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High number of pending pods"

  - alert: SchedulerSlowScheduling
    expr: histogram_quantile(0.99, rate(scheduler_scheduling_duration_seconds_bucket[5m])) > 5
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Scheduler p99 latency > 5s"

  - alert: SchedulerUnschedulablePods
    expr: increase(scheduler_schedule_attempts_total{result="unschedulable"}[5m]) > 50
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Many pods unschedulable (resource constraints?)"
```

### 14.4 Events for Debugging

Scheduler writes events to pods:

```bash
kubectl describe pod my-pod
# Events:
#   Warning  FailedScheduling  1m (x12 over 3m)  default-scheduler  0/3 nodes available: 
#                                                                    1 node had taint {node.kubernetes.io/not-ready: }, 
#                                                                    2 Insufficient cpu
```

---

## 15. Common Scheduler Issues & Troubleshooting

### 15.1 Issue: "0/N nodes available: Insufficient cpu"

**Symptom:** Pod stays Pending forever.

**Diagnosis:**

```bash
kubectl describe pod my-pod
# Events: Insufficient cpu

kubectl describe nodes
# Check Allocatable vs Allocated
```

**Causes:**
1. **Over-requested resources** — pod asks for more than any single node has
2. **Fragmented resources** — total cluster has capacity, but no single node does

**Solutions:**
- Reduce pod's `requests.cpu`
- Add more nodes
- Scale down other workloads
- Use **cluster autoscaler** to add nodes automatically

---

### 15.2 Issue: "0/N nodes available: node(s) had taint..."

**Symptom:** Pod stays Pending with taint message.

**Diagnosis:**

```bash
kubectl describe pod my-pod
# Events: node(s) had taint {key=value:NoSchedule} that the pod didn't tolerate

kubectl describe nodes | grep Taints
```

**Solution:**

```yaml
# Add toleration to pod:
spec:
  tolerations:
  - key: key
    operator: Equal
    value: value
    effect: NoSchedule
```

---

### 15.3 Issue: "0/N nodes available: pod affinity/anti-affinity"

**Symptom:** Pod stays Pending due to affinity rules.

**Diagnosis:**

```bash
kubectl describe pod my-pod
# Events: pod didn't match inter-pod affinity rules
```

**Common Causes:**
1. **requiredDuringScheduling** anti-affinity can't be satisfied (not enough nodes)
2. **podAffinity** targeting pods that don't exist

**Solutions:**
- Change `required` to `preferred`
- Increase `maxSkew` in topology spread constraints
- Add more nodes
- Check if target pods (for podAffinity) actually exist

---

### 15.4 Issue: "0/N nodes available: node selector doesn't match"

```bash
kubectl describe pod my-pod
# Events: node(s) didn't match node selector

kubectl get nodes --show-labels
# Check if any node has the required labels
```

**Solution:**

```bash
# Add missing label to node:
kubectl label nodes worker-1 disktype=ssd
```

---

### 15.5 Issue: Slow Scheduling (High Latency)

**Symptom:** Pods take 10+ seconds to schedule.

**Diagnosis:**

```bash
# Check scheduler metrics
curl -k https://localhost:10259/metrics | grep scheduling_duration_seconds

# Look for slow plugins
curl -k https://localhost:10259/metrics | grep framework_extension_point_duration
```

**Common Causes:**
1. **Complex pod affinity** (scheduler must evaluate all pod pairs)
2. **Large cluster** (1000+ nodes, scoring is expensive)
3. **API server latency** (network or etcd slow)

**Solutions:**
- Reduce affinity complexity
- Increase `--percentageOfNodesToScore` (trade accuracy for speed)
- Use `preferredDuringScheduling` instead of `requiredDuringScheduling`
- Scale API server / etcd

---

### 15.6 Issue: Pod Scheduled to Wrong Node

**Symptom:** Pod lands on node without expected labels.

**Diagnosis:**

```bash
kubectl get pod my-pod -o yaml | grep nodeName
kubectl describe node <nodeName> | grep Labels
```

**Cause:** Likely using `preferredDuringScheduling` (soft constraint) instead of `requiredDuringScheduling` (hard constraint).

**Solution:** Change to `requiredDuringScheduling` if placement is critical.

---

## 16. Scheduler Topics in CKA Exam

### 16.1 Common Tasks

- **Assign pod to specific node** (nodeName, nodeSelector, affinity)
- **Troubleshoot unschedulable pod** (describe pod, check events)
- **Configure taints and tolerations**
- **Set resource requests/limits**
- **Understand scheduling decisions** (why pod landed on node X not Y)

### 16.2 Essential Commands

```bash
# Manually assign pod to node (bypass scheduler)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeName: worker-2  # Directly specify node
  containers:
  - name: nginx
    image: nginx
EOF

# Use nodeSelector
kubectl run nginx --image=nginx --dry-run=client -o yaml | \
  kubectl patch --local -f - -p '{"spec":{"nodeSelector":{"disktype":"ssd"}}}' --dry-run=client -o yaml | \
  kubectl apply -f -

# Taint node
kubectl taint nodes node-1 key=value:NoSchedule

# Remove taint
kubectl taint nodes node-1 key=value:NoSchedule-

# Cordon node (prevent scheduling)
kubectl cordon node-1

# Uncordon
kubectl uncordon node-1

# Drain node (evict pods + cordon)
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# Check pod scheduling events
kubectl describe pod <pod-name>

# Check node capacity vs allocated
kubectl describe node <node-name>
```

### 16.3 CKA Exam Tips

1. **Read events first:** `kubectl describe pod` shows why it's not scheduling
2. **nodeSelector vs affinity:** For simple "must be on node with label X", use `nodeSelector` (faster to type)
3. **Taints:** Remember to remove taint with `-` suffix: `key:NoSchedule-`
4. **Manual scheduling:** Setting `spec.nodeName` bypasses scheduler entirely (useful for testing)
5. **Cordon vs Drain:**
   - Cordon: just prevents new pods
   - Drain: evicts existing + cordons (use for node maintenance)

---

## 17. Real-World Production Use Cases

### 17.1 Multi-Tier Application with Zone Spread

**Scenario:** 3-tier app (web, api, db) across 3 availability zones.

```yaml
# Web tier: spread across zones, prefer nodes with less load
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 9
  template:
    metadata:
      labels:
        tier: web
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            tier: web
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  tier: web
              topologyKey: kubernetes.io/hostname
```

### 17.2 GPU Workloads

**Scenario:** ML training jobs need specific GPU models.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-training
spec:
  # Only on GPU nodes
  nodeSelector:
    accelerator: nvidia-tesla-v100
  # Tolerate GPU taint
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule
  # Request GPU
  containers:
  - name: trainer
    image: tensorflow/tensorflow:latest-gpu
    resources:
      limits:
        nvidia.com/gpu: 2  # 2 GPUs
```

**Node setup:**

```bash
kubectl label nodes gpu-node-1 accelerator=nvidia-tesla-v100
kubectl taint nodes gpu-node-1 nvidia.com/gpu=true:NoSchedule
```

### 17.3 Cost Optimization: Spot Instance Affinity

**Scenario:** Batch jobs prefer spot instances; critical services avoid them.

```yaml
# Batch job: prefer spot
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: node.kubernetes.io/instance-type
                operator: In
                values: [spot]

# Critical service: avoid spot
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node.kubernetes.io/instance-type
                operator: NotIn
                values: [spot]
```

### 17.4 Database with Local SSD and Zone Affinity

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cassandra
spec:
  replicas: 3
  template:
    spec:
      # Anti-affinity: separate nodes
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: cassandra
            topologyKey: kubernetes.io/hostname
        # Node affinity: SSD only
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values: [ssd]
      # Tolerate NoSchedule for dedicated DB nodes
      tolerations:
      - key: dedicated
        operator: Equal
        value: database
        effect: NoSchedule
```

---

## 18. Production Best Practices

### 18.1 Always Set Resource Requests

**Why:** Scheduler can't make good decisions without knowing pod needs.

```yaml
# ✗ BAD: No requests
spec:
  containers:
  - name: app
    image: myapp

# ✓ GOOD: Explicit requests
spec:
  containers:
  - name: app
    image: myapp
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "2"
        memory: "2Gi"
```

### 18.2 Use Pod Disruption Budgets

Prevents scheduler/drain from evicting too many pods during node maintenance.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2  # At least 2 replicas must be running
  selector:
    matchLabels:
      app: web
```

### 18.3 Spread Critical Workloads

Use topology spread or anti-affinity for HA.

```yaml
# Anti-affinity for critical services
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: critical-api
        topologyKey: kubernetes.io/hostname
```

### 18.4 Use Preferred, Not Required, When Possible

**Preferred** constraints are more resilient — pod can schedule even if constraint can't be satisfied.

```yaml
# ✗ RISKY: If no SSD nodes available, pod stuck Pending
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
    - matchExpressions:
      - key: disktype
        operator: In
        values: [ssd]

# ✓ SAFER: Prefers SSD but can use HDD if needed
nodeAffinity:
  preferredDuringSchedulingIgnoredDuringExecution:
  - weight: 100
    preference:
      matchExpressions:
      - key: disktype
        operator: In
        values: [ssd]
```

### 18.5 Monitor Scheduler Health

```yaml
# Prometheus alerts
- alert: SchedulerDown
  expr: up{job="kube-scheduler"} == 0
  for: 5m

- alert: HighPendingPods
  expr: scheduler_pending_pods > 50
  for: 10m

- alert: SchedulerLatencyHigh
  expr: histogram_quantile(0.99, rate(scheduler_e2e_scheduling_duration_seconds_bucket[5m])) > 10
  for: 5m
```

### 18.6 Enable Leader Election

Run multiple scheduler instances for HA.

```bash
--leader-elect=true
--leader-elect-lease-duration=15s
--leader-elect-renew-deadline=10s
```

---

## 19. Common Mistakes & Pitfalls

### 19.1 Mistake: No Resource Requests

**Impact:** Scheduler has no information, defaults to 0. Node gets overcommitted.

**Example:**
```
Node: 8 CPU cores
Pod A (no requests): scheduled
Pod B (no requests): scheduled
Pod C (no requests): scheduled
...
All 10 pods scheduled to same node (oops!)
Runtime: All pods compete for CPU, performance degrades
```

**Fix:** Always set requests.

---

### 19.2 Mistake: Using `requiredDuringScheduling` Anti-Affinity Without Enough Nodes

```yaml
# 5 replicas, anti-affinity: must be on different nodes
spec:
  replicas: 5
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: web
        topologyKey: kubernetes.io/hostname

# Cluster has only 3 nodes
# Result: 3 pods scheduled, 2 stuck Pending FOREVER
```

**Fix:** Use `preferredDuringScheduling` or ensure cluster has enough nodes.

---

### 19.3 Mistake: Forgetting Tolerations for Tainted Nodes

```bash
kubectl taint nodes node-1 key=value:NoSchedule

# Pod without toleration → can't schedule on node-1
# If node-1 is the only node with enough resources → pod stuck Pending
```

**Fix:** Add toleration or remove taint.

---

### 19.4 Mistake: Setting Requests > Limits

```yaml
# ✗ INVALID: Kubernetes rejects this
resources:
  requests:
    cpu: "2"
    memory: "4Gi"
  limits:
    cpu: "1"       # limit < request!
    memory: "2Gi"

# Error: "Limit must be >= Request"
```

---

### 19.5 Mistake: Ignoring QoS Classes

**Scenario:** Critical database pod has `BestEffort` QoS because no requests/limits set.

**Impact:** During memory pressure, database pod evicted first (before batch jobs).

**Fix:** Set requests=limits for `Guaranteed` QoS on critical pods.

---

## 20. Interview Questions & Answers

### 20.1 Beginner: What does the kube-scheduler do?

**A:** The scheduler watches for newly created pods that have no node assigned (`spec.nodeName` is empty). It evaluates all nodes in the cluster, filters out infeasible nodes (insufficient resources, taints, etc.), scores the remaining nodes, and binds the pod to the highest-scoring node by updating `spec.nodeName`.

---

### 20.2 Beginner: What's the difference between nodeSelector and nodeAffinity?

**A:**
- **nodeSelector** is simple key-value matching. Pod must land on node with exact label match.
- **nodeAffinity** is more expressive: supports `In`, `NotIn`, `Exists`, `DoesNotExist` operators, numeric comparisons, and both `required` (hard) and `preferred` (soft) constraints.

**Example:**
```yaml
# nodeSelector (simple)
nodeSelector:
  disktype: ssd

# nodeAffinity (expressive)
nodeAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    nodeSelectorTerms:
    - matchExpressions:
      - key: disktype
        operator: In
        values: [ssd, nvme]
```

---

### 20.3 Intermediate: Explain the difference between pod affinity and pod anti-affinity.

**A:**
- **Pod Affinity** co-locates pods (schedule near each other). Example: web pods prefer to be on same nodes as cache pods.
- **Pod Anti-Affinity** separates pods (schedule away from each other). Example: spread replicas across nodes for high availability.

Both use `topologyKey` to define "closeness" (same node, same zone, same region).

---

### 20.4 Intermediate: What happens if a pod has resource requests but no limits?

**A:**
- **Scheduling:** Scheduler uses requests to decide if node can fit the pod.
- **QoS Class:** Pod is `Burstable`.
- **Runtime:** Pod can use more resources than requested (up to node capacity), but has no ceiling. If node runs out of memory, `Burstable` pods may be evicted.

---

### 20.5 Advanced: How does the scheduler handle a pod with both required node affinity and required pod anti-affinity?

**A:** Both constraints must be satisfied simultaneously.

1. **Filter Phase:** Nodes are filtered by both predicates. A node passes only if:
   - It matches the node affinity rule, AND
   - The pod anti-affinity rule is satisfied (no conflicting pods on that node/topology)

2. **If no nodes pass both:** Pod stays `Pending` (assuming `requiredDuringScheduling`).

3. **Example:**
```yaml
nodeAffinity:
  required: zone=us-west-2a  # Filters to nodes in us-west-2a

podAntiAffinity:
  required: no other app=web pods on same node  # Further filters

# Only nodes in us-west-2a WITHOUT existing web pods pass
```

---

### 20.6 Advanced: How would you debug why a pod is not scheduling?

**A:** Step-by-step diagnosis:

1. **Check pod status:**
   ```bash
   kubectl get pod <pod> -o wide
   # Status: Pending
   ```

2. **Read events:**
   ```bash
   kubectl describe pod <pod>
   # Events section shows scheduler's rejection reasons:
   #   - "Insufficient cpu" → resource issue
   #   - "node(s) had taint..." → toleration missing
   #   - "didn't match node selector" → label mismatch
   ```

3. **Check nodes:**
   ```bash
   kubectl describe nodes
   # Look at Allocatable vs Allocated resources
   # Check Taints, Labels, Conditions
   ```

4. **Check scheduler logs:**
   ```bash
   kubectl logs -n kube-system kube-scheduler-master-1
   # Look for errors or "Predicate failed" messages
   ```

5. **Verify scheduler is running:**
   ```bash
   kubectl get pods -n kube-system | grep scheduler
   ```

---

### 20.7 Advanced: Explain topology spread constraints vs pod anti-affinity.

**A:**

| **Aspect**                  | **Pod Anti-Affinity**                        | **Topology Spread Constraints**               |
|-----------------------------|----------------------------------------------|-----------------------------------------------|
| **Goal**                    | Keep pods apart (any distribution)           | Evenly distribute pods (balanced distribution)|
| **Control**                 | Binary: same topology or not                 | Fine-grained: `maxSkew` controls imbalance    |
| **Use Case**                | HA (spread replicas across nodes/zones)      | Even load (balanced distribution)             |

**Example:**

```yaml
# Anti-affinity: "Don't put two web pods on same node" (but all could land on node-1 and node-2, none on node-3)

# Topology spread: "Distribute 6 web pods evenly across nodes with maxSkew=1"
# Result: 2 pods per node (balanced)
```

---

## 21. Hands-On Examples & Mini Projects

### 21.1 Lab 1: Observe Scheduling Decision

**Goal:** See why scheduler picked node X over node Y.

```bash
# Step 1: Enable scheduler verbose logging
kubectl edit cm -n kube-system kube-scheduler
# Add: --v=4 to logging flags

# Step 2: Create pod
kubectl run nginx --image=nginx

# Step 3: Check logs
kubectl logs -n kube-system kube-scheduler-master-1 | grep nginx
# Look for scoring output:
# "node-1 score: 85"
# "node-2 score: 92"  ← Winner
# "node-3 score: 78"

# Step 4: Verify
kubectl get pod nginx -o wide
# NODE: node-2
```

---

### 21.2 Lab 2: Test Node Affinity

```bash
# Step 1: Label nodes
kubectl label nodes node-1 disktype=ssd
kubectl label nodes node-2 disktype=hdd

# Step 2: Create pod with affinity
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
spec:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values: [ssd]
  containers:
  - name: nginx
    image: nginx
EOF

# Step 3: Verify
kubectl get pod ssd-pod -o wide
# Should be on node-1 (disktype=ssd)

# Step 4: Test rejection
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nvme-pod
spec:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values: [nvme]  # No node has this
  containers:
  - name: nginx
    image: nginx
EOF

kubectl describe pod nvme-pod
# Events: "0/2 nodes available: node(s) didn't match node selector"
```

---

### 21.3 Lab 3: Pod Anti-Affinity

```bash
# Create deployment with anti-affinity
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: web
            topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx
EOF

# Verify spread
kubectl get pods -o wide
# Should see 3 pods on 3 different nodes

# Test: Scale to 5 replicas in 3-node cluster
kubectl scale deployment web --replicas=5

kubectl get pods
# 3 Running, 2 Pending (not enough nodes to satisfy anti-affinity)
```

---

### 21.4 Lab 4: Taint & Toleration

```bash
# Step 1: Taint a node
kubectl taint nodes node-1 dedicated=gpu:NoSchedule

# Step 2: Try scheduling without toleration
kubectl run cpu-job --image=busybox --command -- sleep 3600

kubectl get pod cpu-job -o wide
# Should NOT be on node-1

# Step 3: Schedule with toleration
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: gpu-job
spec:
  tolerations:
  - key: dedicated
    operator: Equal
    value: gpu
    effect: NoSchedule
  containers:
  - name: gpu-app
    image: nvidia/cuda:11.0-base
    command: [sleep, "3600"]
EOF

kubectl get pod gpu-job -o wide
# CAN schedule on node-1

# Step 4: Cleanup
kubectl taint nodes node-1 dedicated=gpu:NoSchedule-
```

---

### 21.5 Lab 5: Resource Requests Impact

```bash
# Step 1: Check node capacity
kubectl describe node node-1 | grep -A 5 Allocatable
# cpu: 4
# memory: 16Gi

# Step 2: Create pod requesting 5 CPUs (more than node has)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: huge-pod
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "5"
EOF

# Step 3: Check status
kubectl get pod huge-pod
# STATUS: Pending

kubectl describe pod huge-pod
# Events: "0/N nodes available: Insufficient cpu"

# Step 4: Fix
kubectl delete pod huge-pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: huge-pod
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        cpu: "2"  # Reduced
EOF

kubectl get pod huge-pod
# STATUS: Running
```

---

## 22. Summary Cheat Sheet

### 22.1 The Scheduling Cycle

```
1. Watch for pods with nodeName == ""
2. Filter nodes (predicates): PodFitsResources, Taints, Affinity, etc.
3. Score nodes (priorities): LeastRequested, BalancedAllocation, Affinity weights
4. Bind pod to highest-scoring node (update nodeName)
```

---

### 22.2 Key Concepts Quick Reference

| **Concept**            | **Quick Definition**                                                                 |
|------------------------|--------------------------------------------------------------------------------------|
| **Predicate/Filter**   | Pass/fail check: can this node run the pod?                                         |
| **Priority/Score**     | 0-100 score: how good is this node for the pod?                                      |
| **nodeSelector**       | Simple label matching (key=value)                                                    |
| **nodeAffinity**       | Advanced label matching (In, NotIn, Exists, required/preferred)                     |
| **podAffinity**        | Co-locate pods (schedule near each other)                                           |
| **podAntiAffinity**    | Separate pods (schedule away from each other)                                       |
| **Taint**              | Repels pods from node                                                                |
| **Toleration**         | Allows pod to ignore taint                                                           |
| **Requests**           | Guaranteed minimum resources (scheduler uses this)                                   |
| **Limits**             | Maximum resources (runtime enforcement)                                              |
| **QoS Classes**        | Guaranteed (req==lim), Burstable (has req/lim), BestEffort (no req/lim)             |
| **Topology Spread**    | Evenly distribute pods across zones/nodes (maxSkew controls balance)                |

---

### 22.3 Critical Scheduler Flags

```bash
--leader-elect=true                # Run multiple instances with leader election
--percentageOfNodesToScore=50      # Only score top 50% for performance
--kube-api-qps=50                  # API request rate
--v=4                              # Verbose logging (debug only)
```

---

### 22.4 Troubleshooting Quick Guide

| **Symptom**                             | **First Check**                                           |
|-----------------------------------------|-----------------------------------------------------------|
| Pod Pending: "Insufficient cpu/memory"  | `kubectl describe nodes` → check Allocatable vs Allocated|
| Pod Pending: "node(s) had taint..."     | `kubectl describe nodes | grep Taints` → add toleration  |
| Pod Pending: "didn't match node selector" | `kubectl get nodes --show-labels` → verify labels       |
| Pod Pending: "pod affinity/anti-affinity" | Check if enough nodes to satisfy constraints            |
| Slow scheduling (10+ sec)               | Check scheduler metrics → reduce affinity complexity      |

---

### 22.5 Common Commands

```bash
# Manually assign pod to node (bypass scheduler)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeName: worker-2  # Direct assignment
  containers:
  - name: nginx
    image: nginx
EOF

# Use nodeSelector
kubectl run nginx --image=nginx --overrides='{"spec":{"nodeSelector":{"disktype":"ssd"}}}'

# Taint node
kubectl taint nodes node-1 key=value:NoSchedule

# Remove taint
kubectl taint nodes node-1 key:NoSchedule-

# Cordon (prevent scheduling, don't evict)
kubectl cordon node-1

# Drain (evict + cordon)
kubectl drain node-1 --ignore-daemonsets

# Check pod events
kubectl describe pod <pod>

# Check scheduler logs
kubectl logs -n kube-system kube-scheduler-<node>

# Check node allocatable
kubectl describe node <node> | grep -A 10 Allocatable
```

---

### 22.6 Best Practices Summary

✅ **Always set resource requests** (scheduler needs this)  
✅ **Use `preferredDuringScheduling` for resilience** (can schedule even if constraint fails)  
✅ **Set PodDisruptionBudgets** for critical services  
✅ **Spread critical workloads** with anti-affinity or topology spread  
✅ **Monitor scheduler metrics** (pending pods, latency, unschedulable rate)  
✅ **Run multiple scheduler instances** with leader election  
✅ **Use taints for dedicated nodes** (GPU, storage, etc.)  
✅ **Set requests == limits** for `Guaranteed` QoS on critical pods  

---

### 22.7 Affinity/Anti-Affinity Decision Tree

```
Need to place pod on specific node type?
├─ Simple label match → nodeSelector
└─ Complex logic (In/NotIn/Exists) → nodeAffinity

Need pods near each other?
└─ Use podAffinity (co-location)

Need pods far apart?
└─ Use podAntiAffinity (separation)

Need even distribution?
└─ Use topologySpreadConstraints (balanced spread)

Must satisfy constraint?
├─ Yes → requiredDuringScheduling
└─ No → preferredDuringScheduling
```

---

## Closing Summary

You've mastered the **kube-scheduler** from first principles to production expertise. The scheduler is the decision-maker that turns declarative pod specs into actual running containers by intelligently placing them on the best-fit nodes.

**Key Takeaways:**

1. **Scheduling is a two-phase process:** Filter (predicates) → Score (priorities)
2. **Scheduler never talks to etcd** — only to API server
3. **Requests are for scheduling, limits are for runtime** — scheduler ignores limits
4. **Affinity/anti-affinity** give fine-grained control over placement
5. **QoS classes** determine eviction priority during resource pressure
6. **Topology spread** ensures even distribution for high availability
7. **Custom schedulers** and **extenders** handle special workloads

**Next Steps:**
- Practice CKA labs on killer.sh focusing on scheduling scenarios
- Set up a multi-node cluster and experiment with affinity rules
- Monitor scheduler metrics in production
- Explore custom scheduler plugins for advanced use cases

The scheduler is where Kubernetes resource management comes to life. Master it, and you control the placement intelligence of your entire cluster! 🎯
