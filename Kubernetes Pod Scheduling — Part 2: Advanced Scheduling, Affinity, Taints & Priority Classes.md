## Table of Contents

1. [Introduction — Why Advanced Scheduling Matters](#1-introduction--why-advanced-scheduling-matters)
2. [Core Identity Table — Scheduling Components](#2-core-identity-table--scheduling-components)
3. [How the Kubernetes Scheduler Works — Deep Internals](#3-how-the-kubernetes-scheduler-works--deep-internals)
4. [The Controller Pattern — Watch → Compare → Act → Loop](#4-the-controller-pattern--watch--compare--act--loop)
5. [Running Pods on Specific Nodes — The Full Spectrum](#5-running-pods-on-specific-nodes--the-full-spectrum)
6. [Understanding nodeSelector — Label-Based Node Selection](#6-understanding-nodeselector--label-based-node-selection)
7. [Lab: Using nodeSelectors in Pod Scheduling](#7-lab-using-nodeselectors-in-pod-scheduling)
8. [Affinity and Anti-Affinity — Expressive Placement Rules](#8-affinity-and-anti-affinity--expressive-placement-rules)
9. [Lab: Deploying Pods with Required Node Affinity](#9-lab-deploying-pods-with-required-node-affinity)
10. [Lab: Deploying Pods with Preferred Node Affinity](#10-lab-deploying-pods-with-preferred-node-affinity)
11. [Advanced Affinity Options — Operators, Weights & Topology](#11-advanced-affinity-options--operators-weights--topology)
12. [Lab: Multiple Affinity Selection](#12-lab-multiple-affinity-selection)
13. [Lab: Using Not-Equal Operators in Affinity Selection](#13-lab-using-not-equal-operators-in-affinity-selection)
14. [Taints and Tolerations — Node-Side Repulsion Mechanism](#14-taints-and-tolerations--node-side-repulsion-mechanism)
15. [Lab: Applying Taints on Nodes](#15-lab-applying-taints-on-nodes)
16. [Lab: Scheduling Pods with Tolerations](#16-lab-scheduling-pods-with-tolerations)
17. [Understanding Pod Priority Classes](#17-understanding-pod-priority-classes)
18. [Lab: Creating Pod Priority Classes](#18-lab-creating-pod-priority-classes)
19. [Lab: Scheduling Pods with Priority Classes](#19-lab-scheduling-pods-with-priority-classes)
20. [Built-in Controllers Deep Dive](#20-built-in-controllers-deep-dive)
21. [Internal Working Concepts — Informers, Work Queues & Reconciliation](#21-internal-working-concepts--informers-work-queues--reconciliation)
22. [API Server and etcd Interaction](#22-api-server-and-etcd-interaction)
23. [Leader Election for HA Scheduling](#23-leader-election-for-ha-scheduling)
24. [Performance Tuning for the Scheduler](#24-performance-tuning-for-the-scheduler)
25. [Security Hardening Practices](#25-security-hardening-practices)
26. [Monitoring and Observability](#26-monitoring-and-observability)
27. [Troubleshooting Scheduling Issues](#27-troubleshooting-scheduling-issues)
28. [Disaster Recovery Concepts](#28-disaster-recovery-concepts)
29. [Comparison — kube-apiserver vs kube-scheduler vs kube-controller-manager](#29-comparison--kube-apiserver-vs-kube-scheduler-vs-kube-controller-manager)
30. [ASCII Architecture Diagram](#30-ascii-architecture-diagram)
31. [Real-World Production Use Cases](#31-real-world-production-use-cases)
32. [Best Practices for Production Environments](#32-best-practices-for-production-environments)
33. [Common Mistakes and Pitfalls](#33-common-mistakes-and-pitfalls)
34. [Interview Questions — Beginner to Advanced](#34-interview-questions--beginner-to-advanced)
35. [Cheat Sheet — Commands, Flags & Manifests](#35-cheat-sheet--commands-flags--manifests)
36. [Key Takeaways & Summary](#36-key-takeaways--summary)

---

## 1. Introduction — Why Advanced Scheduling Matters

In a Kubernetes cluster with dozens or hundreds of nodes, placing Pods on the right nodes is not merely an optimization — it is a **critical operational requirement**. The default scheduler places Pods efficiently based on resource availability, but production workloads demand much more:

- A machine learning training job **must** run on nodes with GPU hardware
- A database Pod **must not** share a node with another database Pod (single point of failure)
- A compliance-regulated application **must** run only in specific geographic zones
- Control plane components **must not** be disrupted by user workload scheduling
- A critical payment service **must** preempt lower-priority batch jobs when resources are scarce

Without fine-grained scheduling control, you end up with:
- Performance degradation (wrong node type for workload)
- Availability risk (all replicas on the same node or zone)
- Cost inefficiency (expensive GPU nodes used for web servers)
- Compliance violations (regulated data in wrong datacenter)
- Cascading failures (high-priority systems killed to make room for batch jobs)

This guide covers the complete toolkit for controlling Pod placement: **nodeSelector**, **Node Affinity**, **Pod Affinity and Anti-Affinity**, **Taints and Tolerations**, and **Priority Classes** — from fundamentals through production-grade configurations.

### The Scheduling Placement Toolkit

| Mechanism | Direction | Scope | Enforcement | Best For |
|---|---|---|---|---|
| `nodeName` | Pod → specific Node | Hard assignment | Mandatory | Debug, bootstrapping |
| `nodeSelector` | Pod → Node label | Simple match | Mandatory | Basic hardware selection |
| Node Affinity `required` | Pod → Node label | Expressive match | Mandatory | Hardware, zone constraints |
| Node Affinity `preferred` | Pod → Node label | Soft preference | Best-effort | Cost optimization |
| Pod Affinity `required` | Pod → Pod (same node/zone) | Co-location | Mandatory | Latency optimization |
| Pod Anti-Affinity `required` | Pod ↔ Pod (different) | Separation | Mandatory | HA spread |
| Pod Anti-Affinity `preferred` | Pod ↔ Pod (different) | Soft spread | Best-effort | Zone distribution |
| Taint + Toleration | Node repels → Pod accepts | Node exclusivity | Node-enforced | Dedicated nodes |
| Priority Class | Pod rank in preemption | Cluster-wide | Eviction order | Critical system pods |

---

## 2. Core Identity Table — Scheduling Components

| Component | Binary | Port | Protocol | Role in Advanced Scheduling |
|---|---|---|---|---|
| **kube-scheduler** | `kube-scheduler` | 10259 | HTTPS | Applies all placement rules: nodeSelector, affinity, taints |
| **kube-apiserver** | `kube-apiserver` | 6443 | HTTPS | Stores node labels, taints, Pod specs, priority classes |
| **kube-controller-manager** | `kube-controller-manager` | 10257 | HTTPS | Node controller manages taints for unhealthy nodes |
| **kubelet** | `kubelet` | 10250 | HTTPS | Reports node labels, capacity, allocatable resources |
| **etcd** | `etcd` | 2379, 2380 | HTTPS | Stores all affinity rules, taints, priority objects |
| **PriorityClass** | `priorityclass.scheduling.k8s.io` | N/A | API resource | Defines integer priority values for Pods |
| **Node** | `node` resource | N/A | API resource | Carries labels and taints evaluated by scheduler |
| **Pod** | `pod` resource | N/A | API resource | Carries nodeSelector, affinity, tolerations, priorityClassName |
| **Scheduler Profile** | Config file | N/A | YAML | Configures plugins, weights, scheduler name |
| **Lease object** | `coordination.k8s.io/v1` | N/A | API resource | Scheduler HA leader election |

---

## 3. How the Kubernetes Scheduler Works — Deep Internals

### 3.1 The Scheduling Pipeline

The kube-scheduler continuously watches for **unscheduled Pods** (Pods with `spec.nodeName == ""`). For each such Pod, it runs a multi-stage pipeline to select the optimal node.

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    SCHEDULER PIPELINE (per Pod)                           │
│                                                                            │
│  UNSCHEDULED POD (nodeName == "")                                         │
│           │                                                                │
│           ▼                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  1. SCHEDULING QUEUE SORT                                           │  │
│  │  Pods ordered by: PriorityClass (high first), then FIFO             │  │
│  │  Low-priority Pod waits → high-priority Pod jumps ahead             │  │
│  └─────────────────────────────────┬───────────────────────────────────┘  │
│                                    │                                       │
│  ┌─────────────────────────────────▼───────────────────────────────────┐  │
│  │  2. FILTER PHASE (NodeName, NodeUnschedulable, ...)                  │  │
│  │  Runs all Filter plugins against EVERY node                          │  │
│  │  Any single failure → node eliminated                                │  │
│  │                                                                      │  │
│  │  Plugins (order matters):                                            │  │
│  │  ├── NodeName            (spec.nodeName matches?)                    │  │
│  │  ├── NodeUnschedulable   (node.spec.unschedulable == false?)         │  │
│  │  ├── NodeResourcesFit    (enough CPU, memory, extended resources?)   │  │
│  │  ├── NodeSelector        (nodeSelector labels match node labels?)    │  │
│  │  ├── NodeAffinity        (affinity rules satisfied?)                 │  │
│  │  ├── TaintToleration     (node taints tolerated by pod?)             │  │
│  │  ├── InterPodAffinity    (pod affinity/anti-affinity satisfied?)     │  │
│  │  ├── VolumeBinding       (required volumes available on node?)       │  │
│  │  └── ...more plugins                                                 │  │
│  │                                                                      │  │
│  │  Result: Feasible nodes list (0 to N nodes)                         │  │
│  └─────────────────────────────────┬───────────────────────────────────┘  │
│                                    │                                       │
│                         ┌──────────▼──────────┐                           │
│                         │  0 feasible nodes?  │                           │
│                         │  → PostFilter phase │                           │
│                         │  Try preemption:    │                           │
│                         │  evict lower-pri    │                           │
│                         │  pod to make room   │                           │
│                         └──────────┬──────────┘                           │
│                                    │ ≥1 feasible nodes                    │
│  ┌─────────────────────────────────▼───────────────────────────────────┐  │
│  │  3. SCORE PHASE                                                       │  │
│  │  Each plugin assigns 0-100 to each feasible node                     │  │
│  │  Plugins run in parallel; weighted sum computed                      │  │
│  │                                                                      │  │
│  │  Score Plugins:                                                      │  │
│  │  ├── NodeResourcesFit    (LeastAllocated or MostAllocated strategy)  │  │
│  │  ├── NodeAffinity        (preferred affinity adds score)             │  │
│  │  ├── InterPodAffinity    (preferred pod affinity adds score)         │  │
│  │  ├── ImageLocality       (higher score if image cached on node)      │  │
│  │  ├── TaintToleration     (tolerated taints may reduce score)         │  │
│  │  └── ...more plugins                                                 │  │
│  │                                                                      │  │
│  │  Result: Ranked node list (highest score wins)                       │  │
│  └─────────────────────────────────┬───────────────────────────────────┘  │
│                                    │                                       │
│  ┌─────────────────────────────────▼───────────────────────────────────┐  │
│  │  4. NORMALIZE SCORES                                                  │  │
│  │  Each plugin normalizes its scores to [0, 100]                       │  │
│  │  Prevents plugins with different scales from dominating              │  │
│  └─────────────────────────────────┬───────────────────────────────────┘  │
│                                    │                                       │
│  ┌─────────────────────────────────▼───────────────────────────────────┐  │
│  │  5. SELECT NODE                                                       │  │
│  │  Highest weighted score wins                                         │  │
│  │  Tie-breaking: random selection among tied nodes                     │  │
│  └─────────────────────────────────┬───────────────────────────────────┘  │
│                                    │                                       │
│  ┌─────────────────────────────────▼───────────────────────────────────┐  │
│  │  6. RESERVE → PERMIT → BIND                                           │  │
│  │  Reserve: Optimistically mark resources claimed on node              │  │
│  │  Permit: Allow/wait/deny the binding decision                        │  │
│  │  Bind: Write spec.nodeName to API server → stored in etcd            │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 The Scheduling Framework Extension Points

```
Scheduling Cycle (runs in serial per Pod):
  Sort → PreFilter → Filter → PostFilter → PreScore → Score → NormalizeScore → Reserve → Permit

Binding Cycle (runs asynchronously):
  WaitOnPermit → PreBind → Bind → PostBind
```

### 3.3 Filter vs Score: Mandatory vs Preferred

| Stage | What It Does | Failure Behavior | Used By |
|---|---|---|---|
| **Filter** | Eliminates nodes that absolutely cannot work | Pod stays Pending | nodeSelector, required affinity, taints |
| **Score** | Ranks remaining nodes | Pod may land on suboptimal node | Preferred affinity, resource balance |

### 3.4 Observing Scheduler Decisions

```bash
# See scheduling decisions in scheduler logs
kubectl logs -n kube-system kube-scheduler-$(hostname) \
  --tail=100 | grep -E "Attempting to bind|successfully bound|scheduling cycle"

# See scheduling events for a pod
kubectl describe pod <pod-name> | grep -A 10 "Events:"
# Normal  Scheduled  5s  default-scheduler  Successfully assigned default/<pod> to <node>

# Check scheduler metrics
kubectl get --raw /metrics 2>/dev/null | \
  grep scheduler | head -20

# See pending pods in scheduling queue
kubectl get pods --all-namespaces --field-selector=status.phase=Pending
```

---

## 4. The Controller Pattern — Watch → Compare → Act → Loop

The scheduler itself is not a controller in the traditional sense, but it uses the same informer pattern. Understanding this helps you reason about scheduler behavior during node failures, label changes, and resource pressure.

### 4.1 Scheduler's Watch Loop

```
┌──────────────────────────────────────────────────────────────────────┐
│                    SCHEDULER INTERNAL LOOP                           │
│                                                                      │
│   WATCH:    Informer watches for Pods with nodeName==""              │
│             Informer watches for Node events (added, labels changed, │
│             condition changed, taint added)                          │
│                                                                      │
│   COMPARE:  For each unscheduled Pod:                                │
│             - Desired: Pod needs a node                              │
│             - Current: Pod has no node assigned                      │
│             - Compute: Which node best satisfies all constraints?    │
│                                                                      │
│   ACT:      Write spec.nodeName to API server (Bind API)             │
│             Pod moves from scheduling queue to bound state           │
│                                                                      │
│   LOOP:     Return to watching for next unscheduled Pod              │
│             Triggered by: new Pod, node label change, node join      │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.2 What Triggers Re-evaluation

The scheduler caches node state and only re-evaluates when:
- New Pod arrives in queue
- Node labels change (affects nodeSelector + affinity)
- Node taints change (affects TaintToleration filter)
- Node capacity changes (affects NodeResourcesFit filter)
- Existing pod evicted (frees resources for re-evaluation)

---

## 5. Running Pods on Specific Nodes — The Full Spectrum

### 5.1 Decision Guide — Which Mechanism to Use

```
Do you need to place a Pod on a specific node?
│
├── YES, absolute specific node (name known)
│   └── Use: spec.nodeName = "<node-name>"
│       Note: No scheduling checks, taint checks, or resource checks
│
├── YES, by node characteristics (hardware, zone, etc.)
│   │
│   ├── Simple label match only?
│   │   └── Use: spec.nodeSelector (key: value)
│   │
│   ├── Complex expressions needed (In, NotIn, Exists, etc.)?
│   │   └── Use: spec.affinity.nodeAffinity
│   │       ├── Must be satisfied?
│   │       │   └── requiredDuringSchedulingIgnoredDuringExecution
│   │       └── Prefer but not required?
│   │           └── preferredDuringSchedulingIgnoredDuringExecution
│   │
│   └── Want to spread or co-locate with OTHER pods?
│       └── Use: spec.affinity.podAffinity / podAntiAffinity
│
└── YES, and you want to dedicate nodes (block non-tolerating pods)?
    └── Use: kubectl taint node <n> key=value:Effect
        + Pod toleration: spec.tolerations
```

### 5.2 Node Labels — What's Built-In

Every Kubernetes node has well-known labels applied automatically:

```bash
# See all labels on a node
kubectl get node k8s-worker-1 --show-labels

# Well-known node labels (automatically applied):
kubectl get node k8s-worker-1 \
  -o jsonpath='{.metadata.labels}' | jq

# KEY BUILT-IN LABELS:
# kubernetes.io/hostname: "k8s-worker-1"
# kubernetes.io/arch: "amd64"
# kubernetes.io/os: "linux"
# node.kubernetes.io/instance-type: "m5.large"  (cloud)
# topology.kubernetes.io/region: "us-east-1"    (cloud)
# topology.kubernetes.io/zone: "us-east-1a"     (cloud)
# node-role.kubernetes.io/control-plane: ""     (control-plane nodes)
# node-role.kubernetes.io/worker: ""            (worker nodes if labeled)
# beta.kubernetes.io/arch: "amd64"              (deprecated, use kubernetes.io/arch)
```

### 5.3 Adding Custom Node Labels

```bash
# Add custom hardware labels
kubectl label node k8s-worker-1 \
  hardware/gpu=nvidia-tesla-t4 \
  hardware/disk=nvme-ssd \
  hardware/network=10gbe

# Add environment labels
kubectl label node k8s-worker-1 \
  environment=production \
  tier=compute \
  team=ml-platform

# Add zone labels (if not auto-applied by cloud provider)
kubectl label node k8s-worker-1 \
  topology.kubernetes.io/zone=us-east-1a \
  topology.kubernetes.io/region=us-east-1

# Add rack/data-center labels (for bare-metal)
kubectl label node k8s-worker-1 \
  failure-domain.beta.kubernetes.io/rack=rack-42 \
  datacenter=dc-east

# Verify
kubectl get nodes --show-labels
kubectl get node k8s-worker-1 -o yaml | grep -A 30 "labels:"
```

---

## 6. Understanding nodeSelector — Label-Based Node Selection

### 6.1 How nodeSelector Works

`nodeSelector` is the **simplest** way to constrain which nodes a Pod can be scheduled on. It specifies a map of key-value pairs. For a Pod to be schedulable on a node, the node must have ALL the specified labels.

**Important:** `nodeSelector` is evaluated during the **Filter phase** — if no node has all the required labels, the Pod stays `Pending` indefinitely.

```yaml
spec:
  nodeSelector:
    key1: value1    # Node MUST have this label
    key2: value2    # AND this label (AND logic)
                    # No OR logic, no expressions — use affinity for that
```

### 6.2 nodeSelector Limitations

| Limitation | Workaround |
|---|---|
| AND logic only (all labels must match) | Use nodeAffinity with multiple matchExpressions |
| No OR logic (match any of several values) | Use nodeAffinity with `In` operator |
| No Not-equal logic | Use nodeAffinity with `NotIn` operator |
| No "label exists" check | Use nodeAffinity with `Exists` operator |
| No weight/preference | Use preferredDuringScheduling nodeAffinity |
| No pod-to-pod awareness | Use podAffinity/podAntiAffinity |

### 6.3 nodeSelector YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-training-pod
spec:
  nodeSelector:
    hardware/gpu: nvidia-tesla-t4    # Only GPU nodes
    environment: production           # AND production nodes
  containers:
    - name: trainer
      image: tensorflow/tensorflow:2.13-gpu
      resources:
        limits:
          nvidia.com/gpu: "1"    # Extended resource for GPU
```

---

## 7. Lab: Using nodeSelectors in Pod Scheduling

### 7.1 Setup — Label Nodes

```bash
# ── LAB SETUP ────────────────────────────────────────────────────────────

# Check existing labels on all nodes
kubectl get nodes --show-labels

# Label worker nodes with hardware characteristics
kubectl label node k8s-worker-1 \
  disk=ssd \
  region=us-east

kubectl label node k8s-worker-2 \
  disk=hdd \
  region=us-west

# Verify labels
kubectl get nodes \
  -o custom-columns="NAME:.metadata.name,DISK:.metadata.labels.disk,REGION:.metadata.labels.region"
```

### 7.2 Create Pods with nodeSelector

```bash
# ── POD 1: Requires SSD disk ─────────────────────────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
  labels:
    app: ssd-app
spec:
  nodeSelector:
    disk: ssd    # Only land on SSD nodes
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

# Verify placement
kubectl get pod ssd-pod -o wide
# Should show k8s-worker-1 (has disk=ssd)

# ── POD 2: Requires HDD disk ─────────────────────────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hdd-pod
spec:
  nodeSelector:
    disk: hdd
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

kubectl get pod hdd-pod -o wide
# Should show k8s-worker-2 (has disk=hdd)

# ── POD 3: Impossible nodeSelector (no node has this label) ───────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: impossible-pod
spec:
  nodeSelector:
    hardware: quantum-processor   # No node has this label
  containers:
    - name: app
      image: nginx:1.25
EOF

kubectl get pod impossible-pod
# STATUS: Pending

kubectl describe pod impossible-pod | grep -A 5 "Events:"
# 0/2 nodes are available: 2 node(s) didn't match node selector.
# preemption: 0/2 nodes are available...

kubectl delete pod impossible-pod
```

### 7.3 nodeSelector with Deployment

```bash
# Deploy a stateless app to SSD-backed nodes
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cache-deployment
  labels:
    app: cache
spec:
  replicas: 3
  selector:
    matchLabels:
      app: cache
  template:
    metadata:
      labels:
        app: cache
    spec:
      nodeSelector:
        disk: ssd      # All 3 replicas must be on SSD nodes
      containers:
        - name: redis
          image: redis:7-alpine
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
EOF

# All replicas on k8s-worker-1 (only SSD node)
kubectl get pods -l app=cache -o wide

# Note: if you have only 1 SSD node, all 3 replicas land there!
# For HA spread, use topologySpreadConstraints or pod anti-affinity

# Cleanup
kubectl delete deployment cache-deployment
kubectl delete pods ssd-pod hdd-pod
```

### 7.4 Removing nodeSelector Labels — What Happens?

```bash
# Create a pod on the SSD node
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: label-test-pod
spec:
  nodeSelector:
    disk: ssd
  containers:
    - name: app
      image: nginx:1.25
EOF

kubectl get pod label-test-pod -o wide
# Running on k8s-worker-1

# Remove the label from the node
kubectl label node k8s-worker-1 disk-

# The pod CONTINUES RUNNING (schedulingIgnoredDuringExecution)
# nodeSelector does NOT enforce during execution — only during scheduling!
kubectl get pod label-test-pod -o wide
# Still Running on k8s-worker-1

# The pod will only be affected if it's rescheduled (crashes, is deleted, etc.)

# Cleanup
kubectl delete pod label-test-pod
kubectl label node k8s-worker-1 disk=ssd  # Restore label
```

---

## 8. Affinity and Anti-Affinity — Expressive Placement Rules

### 8.1 Why Affinity Exists

`nodeSelector` was the original node-selection mechanism but is too limited for production use. `nodeAffinity` was introduced to provide:
- **Operators**: `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`
- **Preferences**: soft rules that influence but don't mandate placement
- **Combined expressions**: multiple terms with OR/AND logic

### 8.2 Affinity Types Overview

```
spec.affinity:
  ├── nodeAffinity
  │   ├── requiredDuringSchedulingIgnoredDuringExecution   ← HARD RULE
  │   │   └── nodeSelectorTerms (OR logic between terms)
  │   │       └── matchExpressions / matchFields (AND within term)
  │   │
  │   └── preferredDuringSchedulingIgnoredDuringExecution  ← SOFT PREFERENCE
  │       └── weight (1-100) + preference
  │
  ├── podAffinity
  │   ├── requiredDuringSchedulingIgnoredDuringExecution   ← HARD: co-locate
  │   └── preferredDuringSchedulingIgnoredDuringExecution  ← SOFT: prefer co-locate
  │
  └── podAntiAffinity
      ├── requiredDuringSchedulingIgnoredDuringExecution   ← HARD: separate
      └── preferredDuringSchedulingIgnoredDuringExecution  ← SOFT: prefer separate
```

### 8.3 "IgnoredDuringExecution" Explained

All current affinity types include "IgnoredDuringExecution" in their names. This means:

```
"requiredDuringSchedulingIgnoredDuringExecution":
              ─────────────────────────         ────────────────────────────────
  REQUIRED at scheduling time (filter)       IGNORED after pod is running
  → Pod won't be placed if not satisfied     → Pod won't be evicted if node changes

Future (alpha): "requiredDuringSchedulingRequiredDuringExecution"
→ Would evict pods if affinity is violated after scheduling
```

### 8.4 nodeAffinity Operators Reference

| Operator | Meaning | Example |
|---|---|---|
| `In` | Label value is in the specified list | `zone In [us-east-1a, us-east-1b]` |
| `NotIn` | Label value is NOT in the list | `tier NotIn [frontend]` |
| `Exists` | Label key exists (any value) | `accelerator Exists` |
| `DoesNotExist` | Label key does not exist | `maintenance DoesNotExist` |
| `Gt` | Label value is greater than (numeric string) | `storage-size Gt 100` |
| `Lt` | Label value is less than (numeric string) | `storage-size Lt 500` |

---

## 9. Lab: Deploying Pods with Required Node Affinity

### 9.1 Setup — Label Nodes

```bash
# Label nodes with zone and hardware information
kubectl label node k8s-worker-1 \
  topology.kubernetes.io/zone=us-east-1a \
  hardware/disk-type=ssd \
  hardware/cpu-generation=skylake

kubectl label node k8s-worker-2 \
  topology.kubernetes.io/zone=us-east-1b \
  hardware/disk-type=ssd \
  hardware/cpu-generation=broadwell

# Verify
kubectl get nodes \
  -o custom-columns="NAME:.metadata.name,ZONE:.metadata.labels.topology\.kubernetes\.io/zone,DISK:.metadata.labels.hardware/disk-type"
```

### 9.2 Required Node Affinity (Must Be Satisfied)

```bash
# ── EXAMPLE 1: Must be in specific zone ──────────────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: zone-required-pod
  labels:
    app: zone-required
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a    # Must be in us-east-1a
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
EOF

kubectl get pod zone-required-pod -o wide
# Running on k8s-worker-1 (us-east-1a)

kubectl delete pod zone-required-pod
```

### 9.3 Required Affinity with Multiple Expressions (AND logic)

```bash
# ── EXAMPLE 2: Must be SSD AND in east zone ───────────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ssd-east-required-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              # ALL expressions in a term must match (AND logic)
              - key: hardware/disk-type
                operator: In
                values:
                  - ssd
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a
                  - us-east-1b     # Either east zone is acceptable
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
EOF

kubectl get pod ssd-east-required-pod -o wide
# Scheduled on a node that is BOTH SSD AND in east zone

kubectl delete pod ssd-east-required-pod
```

### 9.4 Required Affinity with Multiple Terms (OR logic)

```bash
# ── EXAMPLE 3: OR logic between terms ─────────────────────────────────────
# Multiple items in nodeSelectorTerms = OR logic
# Pod can land on EITHER a GPU node OR a high-memory node
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: or-logic-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          # TERM 1: GPU node (OR)
          - matchExpressions:
              - key: hardware/gpu
                operator: Exists
          # TERM 2: High-memory node (OR)
          - matchExpressions:
              - key: hardware/high-memory
                operator: Exists
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
EOF

# Pod will land on ANY node that has EITHER hardware/gpu OR hardware/high-memory label
kubectl describe pod or-logic-pod | grep -A 5 "Events:"

kubectl delete pod or-logic-pod
```

---

## 10. Lab: Deploying Pods with Preferred Node Affinity

### 10.1 Preferred Affinity — Soft Placement Rules

Preferred affinity adds **weight** (1-100) to scoring. If no node satisfies the preference, the Pod still gets scheduled — it just might land on a less preferred node.

```bash
# ── EXAMPLE 1: Prefer SSD but will accept HDD ────────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: prefer-ssd-pod
spec:
  affinity:
    nodeAffinity:
      # Prefer SSD nodes but will accept others if no SSD available
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80    # Score bonus: 80 out of 100 for SSD nodes
          preference:
            matchExpressions:
              - key: hardware/disk-type
                operator: In
                values:
                  - ssd
        - weight: 20    # Smaller bonus for east zone preference
          preference:
            matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
EOF

kubectl get pod prefer-ssd-pod -o wide
# Likely on SSD node, but will still schedule if no SSD available

kubectl delete pod prefer-ssd-pod
```

### 10.2 Combining Required and Preferred Affinity

```bash
# ── EXAMPLE 2: MUST be Linux, PREFER specific zone ───────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: combined-affinity-pod
spec:
  affinity:
    nodeAffinity:
      # HARD RULE: Must be Linux (always true for standard nodes)
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                  - linux
      # SOFT RULE: Prefer east zone
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          preference:
            matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - us-east-1a
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
EOF

kubectl get pod combined-affinity-pod -o wide
kubectl delete pod combined-affinity-pod
```

### 10.3 Pod Affinity — Co-locate Pods

```bash
# ── EXAMPLE 3: Pod Affinity — run near the cache pod ─────────────────────
# First, create the cache pod
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: redis-cache
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
          memory: 64Mi
EOF

# Now create the app pod that wants to be co-located with cache
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-colocated
spec:
  affinity:
    podAffinity:
      # HARD RULE: Must be on same node as redis-cache
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - redis-cache
          topologyKey: kubernetes.io/hostname   # Same NODE
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
EOF

# Verify both pods are on the same node
kubectl get pods -o wide | grep -E "redis-cache|app-colocated"

kubectl delete pods redis-cache app-colocated
```

### 10.4 Pod Anti-Affinity — Spread Pods Across Nodes

```bash
# ── EXAMPLE 4: Anti-Affinity for HA ──────────────────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ha-web
  template:
    metadata:
      labels:
        app: ha-web
    spec:
      affinity:
        podAntiAffinity:
          # PREFER spreading across different nodes
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - ha-web
                topologyKey: kubernetes.io/hostname  # Prefer different nodes
          # HARD RULE: Must be in different ZONES (if zones exist)
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: ha-web
              topologyKey: topology.kubernetes.io/zone  # Different zone required
EOF

kubectl get pods -l app=ha-web -o wide
# Pods distributed across different zones/nodes

kubectl delete deployment ha-web-app
```

---

## 11. Advanced Affinity Options — Operators, Weights & Topology

### 11.1 Complete Operator Reference with Examples

```yaml
# ── In Operator: value must be one of the list ──────────────────────────
- key: topology.kubernetes.io/zone
  operator: In
  values:
    - us-east-1a
    - us-east-1b
    - us-east-1c
# Matches nodes in any of the three zones

# ── NotIn Operator: value must NOT be in the list ───────────────────────
- key: tier
  operator: NotIn
  values:
    - frontend
    - dmz
# Matches nodes NOT in frontend or dmz tier

# ── Exists Operator: key must exist (any value) ──────────────────────────
- key: hardware/gpu
  operator: Exists
# Matches any node that has the hardware/gpu label (regardless of value)

# ── DoesNotExist Operator: key must NOT exist ─────────────────────────────
- key: maintenance
  operator: DoesNotExist
# Matches nodes NOT in maintenance mode

# ── Gt Operator: numeric value must be greater than ─────────────────────
- key: storage-iops
  operator: Gt
  values:
    - "10000"   # Value must be a string containing a number
# Matches nodes where storage-iops > 10000

# ── Lt Operator: numeric value must be less than ─────────────────────────
- key: active-pods
  operator: Lt
  values:
    - "50"
# Matches nodes with fewer than 50 active pods label value
```

### 11.2 topologyKey — Defining Failure Domains

The `topologyKey` field in pod affinity and anti-affinity defines the **failure domain**:

| topologyKey | Granularity | Use Case |
|---|---|---|
| `kubernetes.io/hostname` | Single node | Co-locate/separate on same node |
| `topology.kubernetes.io/zone` | Availability zone | Spread across AZs |
| `topology.kubernetes.io/region` | Cloud region | Cross-region spread |
| `failure-domain.beta.kubernetes.io/rack` | Physical rack | Bare-metal rack spread |
| `custom/datacenter` | Custom domain | Any custom topology key |

```yaml
# Anti-affinity at the ZONE level (not hostname)
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: my-service
      topologyKey: topology.kubernetes.io/zone
# This ensures each replica is in a different zone
# (not just a different node — that's kubernetes.io/hostname)
```

### 11.3 Weight Calculation for Preferred Affinity

```
Nodes after Filter phase: [node-a, node-b, node-c]

For node-a (has ssd AND east zone):
  Base score from NodeResourcesFit: 60
  + Preferred affinity weight 80 (has disk=ssd): +80
  + Preferred affinity weight 20 (east zone):    +20
  = Total: 160

For node-b (has ssd, NOT east zone):
  Base score from NodeResourcesFit: 55
  + Preferred affinity weight 80 (has disk=ssd): +80
  + Preferred affinity weight 20 (NOT east zone): 0
  = Total: 135

For node-c (no ssd, NOT east zone):
  Base score from NodeResourcesFit: 70
  + Preferred affinity weight 80 (no ssd):        0
  + Preferred affinity weight 20 (NOT east zone):  0
  = Total: 70

Winner: node-a (total: 160)
```

### 11.4 topologySpreadConstraints — Modern Pod Spreading

`topologySpreadConstraints` is the modern replacement for pod anti-affinity for spreading use cases:

```yaml
# Spread pods evenly across zones with maximum skew of 1
spec:
  topologySpreadConstraints:
    - maxSkew: 1           # Max difference between zones
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule    # Hard rule
      labelSelector:
        matchLabels:
          app: my-service
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway   # Soft rule
      labelSelector:
        matchLabels:
          app: my-service
```

---

## 12. Lab: Multiple Affinity Selection

### 12.1 Combining Multiple Affinity Types

```bash
# ── SETUP ─────────────────────────────────────────────────────────────────
# Label nodes with multiple attributes
kubectl label node k8s-worker-1 \
  topology.kubernetes.io/zone=us-east-1a \
  hardware/disk-type=ssd \
  hardware/memory=high \
  tier=premium

kubectl label node k8s-worker-2 \
  topology.kubernetes.io/zone=us-east-1b \
  hardware/disk-type=hdd \
  hardware/memory=standard \
  tier=standard

# ── CREATE ANCHOR POD ─────────────────────────────────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: database-pod
  labels:
    app: database
    tier: data
spec:
  nodeSelector:
    hardware/disk-type: ssd    # Simple requirement: SSD only
  containers:
    - name: db
      image: postgres:14-alpine
      env:
        - name: POSTGRES_PASSWORD
          value: "test123"
      resources:
        requests:
          cpu: 250m
          memory: 256Mi
        limits:
          cpu: 500m
          memory: 512Mi
EOF

# ── CREATE APP WITH COMPLEX AFFINITY ──────────────────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: complex-affinity-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: complex-app
  template:
    metadata:
      labels:
        app: complex-app
        tier: application
    spec:
      affinity:
        # RULE 1: Node must be SSD or have high memory
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              # TERM 1 (OR): SSD node
              - matchExpressions:
                  - key: hardware/disk-type
                    operator: In
                    values:
                      - ssd
              # TERM 2 (OR): High memory node
              - matchExpressions:
                  - key: hardware/memory
                    operator: In
                    values:
                      - high
          # Prefer east zone
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 60
              preference:
                matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values:
                      - us-east-1a
                      - us-east-1b
        # RULE 2: Pod should be near the database
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 80
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: database
                topologyKey: kubernetes.io/hostname
        # RULE 3: Pods of this app should NOT be on the same node
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: complex-app
                topologyKey: kubernetes.io/hostname
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

# Observe placement decisions
kubectl get pods -l app=complex-app -o wide
kubectl describe pods -l app=complex-app | grep -E "Node:|Events:" -A 3

# Cleanup
kubectl delete deployment complex-affinity-app
kubectl delete pod database-pod
```

---

## 13. Lab: Using Not-Equal Operators in Affinity Selection

### 13.1 NotIn and DoesNotExist Operators

```bash
# ── SCENARIO: Avoid nodes in maintenance mode ─────────────────────────────

# Mark a node as under maintenance
kubectl label node k8s-worker-2 \
  maintenance=true \
  maintenance-reason=disk-replacement

# Create pod that avoids maintenance nodes
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: avoid-maintenance-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              # Avoid nodes with maintenance=true
              - key: maintenance
                operator: NotIn
                values:
                  - "true"
              # Also avoid nodes with ANY maintenance label
              # (belt and suspenders approach)
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
EOF

kubectl get pod avoid-maintenance-pod -o wide
# Should land on k8s-worker-1 (no maintenance label)

kubectl delete pod avoid-maintenance-pod
```

### 13.2 DoesNotExist Operator

```bash
# ── SCENARIO: Only schedule on 'clean' nodes (no special labels) ──────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: clean-node-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              # Node must NOT have maintenance label at all
              - key: maintenance
                operator: DoesNotExist
              # Node must NOT have decommissioning label
              - key: decommissioning
                operator: DoesNotExist
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
EOF

kubectl get pod clean-node-pod -o wide
# Lands on k8s-worker-1 (no maintenance or decommissioning labels)

kubectl delete pod clean-node-pod
```

### 13.3 Gt and Lt Operators for Range Selection

```bash
# ── SCENARIO: Select nodes based on custom numeric metrics ────────────────

# Label nodes with numeric values (stored as strings in labels)
kubectl label node k8s-worker-1 \
  storage-iops="50000" \
  cpu-speed-mhz="3200"

kubectl label node k8s-worker-2 \
  storage-iops="5000" \
  cpu-speed-mhz="2400"

# Create pod requiring high IOPS storage
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: high-iops-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: storage-iops
                operator: Gt
                values:
                  - "10000"   # Must have IOPS > 10000 (as string comparison)
  containers:
    - name: db
      image: postgres:14-alpine
      env:
        - name: POSTGRES_PASSWORD
          value: "test"
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
EOF

kubectl get pod high-iops-pod -o wide
# Lands on k8s-worker-1 (50000 > 10000)

kubectl delete pod high-iops-pod
```

### 13.4 Complex NOT Logic with Multiple Rules

```bash
# ── SCENARIO: Avoid certain node types for compliance ─────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: compliance-pod
  annotations:
    compliance.company.com/standard: "pci-dss"
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              # MUST be in a compliant region
              - key: topology.kubernetes.io/region
                operator: In
                values:
                  - us-east-1
                  - us-west-2
              # Must NOT be a spot/preemptible instance
              - key: node.kubernetes.io/instance-type
                operator: NotIn
                values:
                  - spot
                  - preemptible
              # Must NOT have the shared-tenant label
              - key: shared-tenant
                operator: DoesNotExist
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
EOF

kubectl get pod compliance-pod -o wide
kubectl describe pod compliance-pod | grep -A 5 "Events:"

kubectl delete pod compliance-pod

# Cleanup node labels
kubectl label node k8s-worker-2 maintenance- maintenance-reason-
```

---

## 14. Taints and Tolerations — Node-Side Repulsion Mechanism

### 14.1 Taints vs Affinity — Conceptual Difference

```
AFFINITY:          Pod says "I WANT to go to nodes with X"
                   → Pod drives the selection

TAINT/TOLERATION:  Node says "I REPEL pods that don't tolerate Y"
                   + Pod says "I can TOLERATE Y"
                   → Node drives exclusivity, Pod provides acceptance
```

### 14.2 Taint Structure

```
Format: key=value:Effect

key:     Identifier (required)
value:   Optional value (can be empty)
Effect:  What happens to non-tolerating pods:
  ├── NoSchedule     = New pods WITHOUT toleration: NOT scheduled here
  │                    Existing pods without toleration: CONTINUE running
  ├── PreferNoSchedule = New pods WITHOUT toleration: PREFER not to schedule here
  │                     (scheduler tries to avoid but will place if no other option)
  └── NoExecute      = New pods WITHOUT toleration: NOT scheduled here
                       Existing pods WITHOUT toleration: EVICTED
```

### 14.3 Taint Effects Comparison

| Effect | New Pods (no toleration) | Existing Pods (no toleration) | Use Case |
|---|---|---|---|
| `NoSchedule` | Won't be scheduled | Continue running | Dedicated nodes, gradual migration |
| `PreferNoSchedule` | Avoid if possible | Continue running | Soft node preference |
| `NoExecute` | Won't be scheduled | Evicted (with grace period) | Node maintenance, isolation |

### 14.4 Well-Known System Taints

| Taint | Applied By | Effect | Meaning |
|---|---|---|---|
| `node.kubernetes.io/not-ready` | Node controller | NoExecute | Node not yet ready |
| `node.kubernetes.io/unreachable` | Node controller | NoExecute | Node unreachable |
| `node.kubernetes.io/unschedulable` | kubectl cordon | NoSchedule | Node cordoned |
| `node.kubernetes.io/memory-pressure` | kubelet | NoSchedule | Low memory |
| `node.kubernetes.io/disk-pressure` | kubelet | NoSchedule | Low disk space |
| `node.kubernetes.io/pid-pressure` | kubelet | NoSchedule | Low PID space |
| `node.kubernetes.io/network-unavailable` | Cloud providers | NoSchedule | Network not ready |
| `node-role.kubernetes.io/control-plane` | kubeadm | NoSchedule | Control plane nodes |

### 14.5 Toleration Structure

```yaml
spec:
  tolerations:
    # Match specific key=value with specific effect
    - key: "dedicated"
      operator: "Equal"
      value: "gpu-team"
      effect: "NoSchedule"

    # Match any value for a key (operator: Exists)
    - key: "hardware/gpu"
      operator: "Exists"
      effect: "NoSchedule"

    # Tolerate NoExecute with time limit (pod evicted after 300s)
    - key: "node.kubernetes.io/not-ready"
      operator: "Exists"
      effect: "NoExecute"
      tolerationSeconds: 300    # Tolerate for 300 seconds before eviction

    # Tolerate all taints (special wildcard - dangerous!)
    - operator: "Exists"
    # No key or effect = match ALL taints
    # Used by: critical system pods like kube-proxy, node-exporter
```

---

## 15. Lab: Applying Taints on Nodes

### 15.1 Basic Taint Operations

```bash
# ── APPLYING TAINTS ────────────────────────────────────────────────────────

# NoSchedule: Prevent all new pods (existing stay)
kubectl taint node k8s-worker-2 \
  dedicated=gpu-team:NoSchedule

# Verify taint applied
kubectl describe node k8s-worker-2 | grep Taint
# Taints: dedicated=gpu-team:NoSchedule

# PreferNoSchedule: Soft repulsion
kubectl taint node k8s-worker-1 \
  hardware=experimental:PreferNoSchedule

# NoExecute: Evict existing pods AND prevent new
kubectl taint node k8s-worker-2 \
  maintenance=disk-replacement:NoExecute

# Taint with no value (just key + effect)
kubectl taint node k8s-worker-1 \
  spot-instance:NoSchedule

# ── VIEWING TAINTS ─────────────────────────────────────────────────────────
kubectl describe node k8s-worker-1 | grep -A 5 "Taints:"

# See taints for all nodes in one view
kubectl get nodes \
  -o custom-columns="NAME:.metadata.name,TAINTS:.spec.taints"

# Using jsonpath for structured output
kubectl get nodes \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints}{"\n"}{end}'

# ── REMOVING TAINTS (append - to the taint) ───────────────────────────────
kubectl taint node k8s-worker-2 dedicated=gpu-team:NoSchedule-
kubectl taint node k8s-worker-2 maintenance=disk-replacement:NoExecute-
kubectl taint node k8s-worker-1 hardware=experimental:PreferNoSchedule-
kubectl taint node k8s-worker-1 spot-instance:NoSchedule-

# Verify taints removed
kubectl describe node k8s-worker-1 | grep "Taints:"
kubectl describe node k8s-worker-2 | grep "Taints:"
```

### 15.2 Demonstrating Taint Effects

```bash
# ── DEMO 1: NoSchedule prevents scheduling ────────────────────────────────

# Apply taint
kubectl taint node k8s-worker-2 dedicated=database:NoSchedule

# Try to create pod without toleration
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: untolerated-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
EOF

# Pod will schedule ONLY to k8s-worker-1 (k8s-worker-2 is tainted)
kubectl get pod untolerated-pod -o wide
# NODE = k8s-worker-1 (the only untainted worker)

kubectl delete pod untolerated-pod

# ── DEMO 2: NoExecute evicts existing pods ────────────────────────────────

# First create a running pod on k8s-worker-2
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: evictable-pod
spec:
  nodeSelector:
    kubernetes.io/hostname: k8s-worker-2
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
EOF

# First, remove the existing NoSchedule taint so pod can schedule
kubectl taint node k8s-worker-2 dedicated=database:NoSchedule-
kubectl get pod evictable-pod -o wide
# Running on k8s-worker-2

# Now apply NoExecute taint
kubectl taint node k8s-worker-2 \
  emergency=evacuation:NoExecute

# Watch the pod get evicted!
kubectl get pod evictable-pod -w
# Running → Terminating → (gone)

# Cleanup
kubectl taint node k8s-worker-2 emergency=evacuation:NoExecute-
```

---

## 16. Lab: Scheduling Pods with Tolerations

### 16.1 Setup — Dedicated Nodes with Taints

```bash
# ── SETUP: Create dedicated node topology ────────────────────────────────

# Taint worker-2 as GPU-only dedicated node
kubectl taint node k8s-worker-2 \
  dedicated=gpu-workloads:NoSchedule

# Label it appropriately
kubectl label node k8s-worker-2 \
  hardware/gpu=nvidia-a100 \
  dedicated-to=ml-team

# Verify setup
kubectl describe node k8s-worker-2 | grep -A 5 "Taints:"
kubectl describe node k8s-worker-2 | grep -A 5 "Labels:"
```

### 16.2 Creating Tolerating Pods

```bash
# ── POD 1: Tolerates the GPU taint ────────────────────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-tolerating-pod
spec:
  tolerations:
    # Must tolerate the specific taint on the GPU node
    - key: "dedicated"
      operator: "Equal"
      value: "gpu-workloads"
      effect: "NoSchedule"
  # Also use nodeSelector to ONLY land on GPU node (not just tolerate)
  nodeSelector:
    hardware/gpu: nvidia-a100
  containers:
    - name: gpu-app
      image: nvidia/cuda:12.0-base-ubuntu22.04
      command: ["sh", "-c", "while true; do sleep 3600; done"]
      resources:
        requests:
          cpu: 500m
          memory: 1Gi
        limits:
          cpu: "2"
          memory: 4Gi
          # nvidia.com/gpu: "1"   # Uncomment when GPU device plugin installed
EOF

# Verify: pod scheduled on k8s-worker-2
kubectl get pod gpu-tolerating-pod -o wide
# NODE = k8s-worker-2 (tolerates taint AND matches nodeSelector)

# ── POD 2: NoExecute toleration with timeout ──────────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: graceful-evict-pod
spec:
  tolerations:
    # Tolerate not-ready for 60 seconds (then evict)
    - key: "node.kubernetes.io/not-ready"
      operator: "Exists"
      effect: "NoExecute"
      tolerationSeconds: 60
    # Tolerate unreachable for 60 seconds (then evict)
    - key: "node.kubernetes.io/unreachable"
      operator: "Exists"
      effect: "NoExecute"
      tolerationSeconds: 60
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
EOF

kubectl get pod graceful-evict-pod -o wide

# ── POD 3: Tolerate all taints (DaemonSet pattern) ────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: tolerates-all-pod
spec:
  tolerations:
    # Tolerate ALL taints (runs on every node including tainted ones)
    - operator: "Exists"   # No key = matches any key; no effect = matches any effect
  containers:
    - name: monitor-agent
      image: busybox:1.36
      command: ["sh", "-c", "while true; do echo monitoring; sleep 30; done"]
      resources:
        requests:
          cpu: 10m
          memory: 16Mi
EOF

# This pod can schedule on ANY node, including dedicated GPU nodes
kubectl get pod tolerates-all-pod -o wide

# Cleanup
kubectl delete pods gpu-tolerating-pod graceful-evict-pod tolerates-all-pod
kubectl taint node k8s-worker-2 dedicated=gpu-workloads:NoSchedule-
kubectl label node k8s-worker-2 hardware/gpu- dedicated-to-
```

### 16.3 Taints + Affinity — The Perfect Pair

```bash
# ── BEST PRACTICE: Taint + Toleration + NodeAffinity together ─────────────
# Taint: Repels pods without toleration
# Affinity: Attracts only intended pods
# Together: Exclusive dedicated node

# Setup
kubectl taint node k8s-worker-2 team=ml:NoSchedule
kubectl label node k8s-worker-2 team=ml hardware/gpu=present

# Pod that should ONLY run on ML team's GPU nodes
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ml-exclusive-pod
spec:
  tolerations:
    # Tolerate the ML team taint
    - key: "team"
      operator: "Equal"
      value: "ml"
      effect: "NoSchedule"
  affinity:
    nodeAffinity:
      # Also REQUIRE the ML team label (not just tolerate)
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: team
                operator: In
                values:
                  - ml
  containers:
    - name: training-job
      image: python:3.11-slim
      command: ["python", "-c", "import time; time.sleep(3600)"]
      resources:
        requests:
          cpu: 500m
          memory: 512Mi
        limits:
          cpu: "2"
          memory: 2Gi
EOF

# Verify: ONLY lands on k8s-worker-2 (has taint toleration AND affinity match)
kubectl get pod ml-exclusive-pod -o wide

# Cleanup
kubectl delete pod ml-exclusive-pod
kubectl taint node k8s-worker-2 team=ml:NoSchedule-
kubectl label node k8s-worker-2 team- hardware/gpu-
```

---

## 17. Understanding Pod Priority Classes

### 17.1 Why Priority Classes Exist

In a resource-constrained cluster, not all Pods are created equal. A payment service should not be denied resources because a batch analytics job is consuming everything. Priority Classes define a **numeric priority** for Pods, enabling:

1. **Scheduling order**: Higher-priority Pods are scheduled before lower-priority ones
2. **Preemption**: When no resources exist for a high-priority Pod, lower-priority Pods can be **evicted** to make room
3. **System component protection**: Control plane pods use extremely high priorities

### 17.2 Priority Class Hierarchy

```
INTEGER PRIORITY VALUES:
─────────────────────────────────────────────────────────
2,000,000,000  ← Maximum value (reserved for system)
1,000,000,000  ← system-cluster-critical  (built-in)
 999,999,999   ← system-node-critical     (built-in)
         ...
   1,000,000   ← High priority user Pods (example)
     100,000   ← Medium priority user Pods (example)
      10,000   ← Low priority user Pods (example)
           0   ← Default (no PriorityClass specified)
          -1   ← Can be negative (BestEffort batch)
─────────────────────────────────────────────────────────
Rule: Higher number = Higher priority = Scheduled first
      Higher priority Pod preempts lower priority when resources needed
```

### 17.3 Built-in Priority Classes

```bash
# View existing priority classes
kubectl get priorityclasses

# NAME                      VALUE        GLOBAL-DEFAULT   AGE
# system-cluster-critical   2000000000   false            5d
# system-node-critical      2000001000   false            5d

# Describe the built-in ones
kubectl describe priorityclass system-cluster-critical
# Value: 2000000000
# GlobalDefault: false
# PreemptionPolicy: PreemptLowerPriority
# Description: Used for system critical pods that must run...
```

### 17.4 Preemption Policy

| Policy | Behavior | Use Case |
|---|---|---|
| `PreemptLowerPriority` (default) | Can evict lower-priority Pods to get scheduled | Critical production services |
| `Never` | Will wait in queue if no resources; won't evict others | Batch jobs (don't want to disrupt running pods) |

---

## 18. Lab: Creating Pod Priority Classes

### 18.1 Define Your Priority Hierarchy

```bash
# ── PRODUCTION PRIORITY HIERARCHY ────────────────────────────────────────

# Tier 1: Critical infrastructure (highest user priority)
cat << 'EOF' | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: critical-infrastructure
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Used for critical infrastructure services like monitoring, logging.
  These pods can preempt lower priority workloads."
EOF

# Tier 2: Production services
cat << 'EOF' | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-service
value: 100000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Production-facing services. Will preempt batch and dev workloads."
EOF

# Tier 3: Standard development
cat << 'EOF' | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: development
value: 10000
globalDefault: true    # Applied to pods with no priorityClassName
preemptionPolicy: PreemptLowerPriority
description: "Default for development workloads."
EOF

# Tier 4: Batch jobs (won't preempt anyone)
cat << 'EOF' | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-job
value: 1000
globalDefault: false
preemptionPolicy: Never    # Will NOT evict other pods to get scheduled
description: "Non-critical batch jobs. Will be preempted by almost anything.
  Uses available resources only, does not disrupt running workloads."
EOF

# Tier 5: Best-effort background tasks (can even be negative)
cat << 'EOF' | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: background-besteffort
value: -100
globalDefault: false
preemptionPolicy: Never
description: "Background tasks that only run when cluster has spare capacity.
  Will never preempt any other workload."
EOF

# Verify all priority classes
kubectl get priorityclasses
# NAME                      VALUE        GLOBAL-DEFAULT
# background-besteffort     -100         false
# batch-job                 1000         false
# development               10000        true
# production-service        100000       false
# critical-infrastructure   1000000      false
# system-cluster-critical   2000000000   false
# system-node-critical      2000001000   false
```

### 18.2 View Priority Class Details

```bash
# Describe all user-defined priority classes
kubectl describe priorityclass critical-infrastructure
kubectl describe priorityclass production-service
kubectl describe priorityclass batch-job

# Check which pods use each priority class
kubectl get pods --all-namespaces \
  -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,PRIORITY:.spec.priorityClassName,VALUE:.spec.priority"
```

---

## 19. Lab: Scheduling Pods with Priority Classes

### 19.1 Basic Priority Class Usage

```bash
# ── ASSIGN PRIORITY CLASSES TO PODS ───────────────────────────────────────

# Critical production service
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: payment-service-pod
  labels:
    app: payment-service
    tier: production
spec:
  priorityClassName: production-service   # Priority: 100000
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
EOF

# Batch analytics job
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: analytics-batch-pod
  labels:
    app: analytics
    tier: batch
spec:
  priorityClassName: batch-job    # Priority: 1000
  containers:
    - name: analytics
      image: python:3.11-slim
      command: ["python", "-c", "import time; time.sleep(7200)"]
      resources:
        requests:
          cpu: 500m
          memory: 512Mi
        limits:
          cpu: "1"
          memory: 1Gi
EOF

# Check priorities assigned
kubectl get pods \
  -o custom-columns="NAME:.metadata.name,PRIORITY:.spec.priority,CLASS:.spec.priorityClassName"
# NAME                    PRIORITY   CLASS
# analytics-batch-pod     1000       batch-job
# payment-service-pod     100000     production-service
```

### 19.2 Demonstrating Preemption

```bash
# ── PREEMPTION DEMO ────────────────────────────────────────────────────────
# This demo shows a high-priority pod preempting a low-priority pod

# Step 1: Fill up node resources with low-priority pods
# (Create enough to take up most resources)
for i in 1 2 3 4 5; do
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: batch-pod-${i}
  labels:
    app: batch
spec:
  priorityClassName: batch-job
  containers:
    - name: batch
      image: nginx:1.25
      resources:
        requests:
          cpu: 300m
          memory: 256Mi
        limits:
          cpu: 500m
          memory: 512Mi
EOF
done

# Wait for them to schedule
kubectl get pods -l app=batch -o wide
kubectl top nodes

# Step 2: Create a high-priority pod that needs resources
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: critical-high-priority-pod
  labels:
    app: critical
spec:
  priorityClassName: production-service   # Priority: 100000 >> 1000
  containers:
    - name: critical-app
      image: nginx:1.25
      resources:
        requests:
          cpu: 800m    # Needs significant CPU
          memory: 512Mi
        limits:
          cpu: "1"
          memory: 1Gi
EOF

# Watch what happens:
# 1. Scheduler finds no room for production-service pod
# 2. Scheduler identifies lower-priority pods (batch-job) to preempt
# 3. Batch pod is gracefully terminated (preemption)
# 4. Production pod is scheduled

kubectl get pods -w
# batch-pod-3   Running → Terminating   (preempted!)
# critical-high-priority-pod   Pending → Running

# Check the events
kubectl describe pod critical-high-priority-pod | grep -A 10 "Events:"
# Preempted pods are mentioned in events

# Cleanup
kubectl delete pods -l app=batch
kubectl delete pods critical-high-priority-pod payment-service-pod analytics-batch-pod
```

### 19.3 Priority Classes with Deployments

```bash
# Production deployment with priority
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: production-api
  template:
    metadata:
      labels:
        app: production-api
    spec:
      priorityClassName: production-service  # All replicas get this priority
      containers:
        - name: api
          image: nginx:1.25
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
---
# Background worker with low priority
apiVersion: apps/v1
kind: Deployment
metadata:
  name: background-worker
spec:
  replicas: 5
  selector:
    matchLabels:
      app: background-worker
  template:
    metadata:
      labels:
        app: background-worker
    spec:
      priorityClassName: batch-job    # Lower priority; will be preempted
      containers:
        - name: worker
          image: busybox:1.36
          command: ["sh", "-c", "while true; do echo working; sleep 10; done"]
          resources:
            requests:
              cpu: 200m
              memory: 128Mi
            limits:
              cpu: 400m
              memory: 256Mi
EOF

kubectl get deployments
kubectl get pods -o custom-columns="NAME:.metadata.name,NODE:.spec.nodeName,PRIORITY:.spec.priority"

# Cleanup
kubectl delete deployments production-api background-worker
kubectl delete priorityclass critical-infrastructure production-service batch-job background-besteffort
# Note: 'development' was set as globalDefault - safe to keep or delete
kubectl delete priorityclass development
```

---

## 20. Built-in Controllers Deep Dive

### 20.1 How Controllers Interact with Scheduling

Each controller creates Pods that are then processed by the scheduler. Understanding what each controller does helps you design scheduling constraints correctly.

### 20.2 ReplicaSet Controller

```bash
# RS controller ensures N pods; scheduling determines WHERE they go
# RS controller watches: ReplicaSet + Pods with matching labels
# RS reconciliation:
#   desired = rs.spec.replicas
#   current = count(pods matching rs.selector, not terminating)
#   delta > 0 → create pods (scheduler places them)
#   delta < 0 → delete pods (prefers older, unscheduled, failed first)

kubectl get replicasets -n <namespace>
kubectl describe rs <rs-name>
# Shows: Replicas, Pods Status (running/waiting/succeeded/failed)
```

### 20.3 Deployment Controller

```bash
# Deployment controller manages RS for rolling updates
# Key interaction with scheduling:
#   During rolling update: new RS pods go through scheduler
#   maxSurge: temporary extra pods (scheduler must find space)
#   maxUnavailable: pods removed before new ones ready

kubectl rollout status deployment/<n>
kubectl get rs -l app=<n>   # Old RS (scaling down) + New RS (scaling up)
```

### 20.4 Node Controller

The Node controller directly interacts with scheduling via **automatic taints**:

```bash
# Node controller applies these taints automatically:
# - node.kubernetes.io/not-ready:NoSchedule     (node not ready)
# - node.kubernetes.io/not-ready:NoExecute      (after grace period)
# - node.kubernetes.io/unreachable:NoSchedule
# - node.kubernetes.io/unreachable:NoExecute

kubectl get nodes -o wide
kubectl describe node <n> | grep -A 5 "Taints:"

# Node controller timeline:
# T+0: Heartbeat stops
# T+40s (node-monitor-grace-period): condition Unknown
# T+40s: Taint not-ready:NoSchedule applied
# T+5m (pod-eviction-timeout): Taint not-ready:NoExecute applied
# T+5m+: Pods begin eviction
```

### 20.5 DaemonSet Controller and Scheduling

DaemonSets interact with the scheduler differently — they bypass the normal scheduling queue:

```bash
# DaemonSet pods use a special scheduler path:
# DaemonSet controller directly sets spec.nodeName (bypasses scheduler)
# OR uses the scheduler with special tolerations

# This explains why DaemonSet pods run on tainted nodes
# (they have tolerations for system taints built-in)

kubectl get pods -n kube-system -l k8s-app=calico-node -o wide
# Each pod on a different node (including control-plane)

kubectl get pod -n kube-system calico-node-XXXXX -o yaml | grep tolerations -A 20
```

### 20.6 Job Controller

```bash
# Job controller creates pods for batch work
# Scheduling constraints apply normally (nodeSelector, affinity, etc.)
# Important: Job pods must have restartPolicy: Never or OnFailure

cat << 'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: scheduled-job
spec:
  template:
    spec:
      restartPolicy: OnFailure
      # Scheduling constraints work here too
      nodeSelector:
        hardware/disk-type: ssd
      priorityClassName: batch-job
      containers:
        - name: worker
          image: busybox:1.36
          command: ["sh", "-c", "echo 'job done' && exit 0"]
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
EOF

kubectl get jobs
kubectl get pods -l job-name=scheduled-job -o wide
kubectl delete job scheduled-job
```

### 20.7 StatefulSet, PersistentVolume, and Service Controllers

These controllers have direct implications for scheduling:
- **StatefulSet**: Pods must often be placed near their PVs (volume-affinity)
- **PV controller**: Binds PVCs to PVs; topology constraints limit which nodes can use a PV
- **Service controller**: Creates cloud LBs; not directly scheduling-related

```bash
# PV topology constraints
kubectl get pv -o yaml | grep nodeAffinity
# PVs can have nodeAffinity that limits which pods can bind to them
# This creates implicit scheduling constraints

kubectl get storageclass <name> -o yaml | grep volumeBindingMode
# WaitForFirstConsumer: PV created/bound only when pod is scheduled
# → Scheduler considers volume topology when placing pod
```

---

## 21. Internal Working Concepts — Informers, Work Queues & Reconciliation

### 21.1 Scheduler Informer Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SCHEDULER INFORMER SETUP                         │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Pod Informer                                                │  │
│  │  LIST + WATCH /api/v1/pods?fieldSelector=spec.nodeName=""   │  │
│  │  → Updates scheduling queue with new pods                   │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Node Informer                                               │  │
│  │  LIST + WATCH /api/v1/nodes                                  │  │
│  │  → Updates node cache: labels, taints, capacity             │  │
│  │  → Node added: trigger retry of pending pods                │  │
│  │  → Node label changed: may enable scheduling of stuck pods  │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Local Cache (Snapshot)                                      │  │
│  │  Scheduler takes a SNAPSHOT of node state at start of cycle  │  │
│  │  All filter/score decisions use the snapshot (not live API)  │  │
│  │  Prevents race conditions with other schedulers              │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### 21.2 Scheduling Queue Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SCHEDULING QUEUE                                 │
│                                                                     │
│  ActiveQ (sorted by priority + timestamp):                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                         │
│  │ Pod A    │  │ Pod B    │  │ Pod C    │                         │
│  │ Pri:1000 │  │ Pri:100  │  │ Pri:10   │  → Processed in order   │
│  └──────────┘  └──────────┘  └──────────┘                         │
│                                                                     │
│  BackoffQ (failed scheduling attempts):                            │
│  Pods that failed scheduling wait here with exponential backoff    │
│  Retried periodically or when cluster state changes                │
│                                                                     │
│  UnschedulableQ (temporarily unschedulable):                       │
│  Moved to ActiveQ when cluster state changes                       │
│  (new node joins, taint removed, pod deleted freeing resources)    │
└─────────────────────────────────────────────────────────────────────┘
```

### 21.3 Reconciliation Loop for Scheduling

```bash
# Watch the scheduling queue in action
kubectl get pods --field-selector=status.phase=Pending -w &

# Create a pod that can't be scheduled immediately
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: queue-demo-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: soon-to-exist
                operator: Exists
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
EOF

# Pod is in Pending state, in the UnschedulableQ

# Now add the label to a node → scheduler detects state change
# → moves pod from UnschedulableQ to ActiveQ → reschedules successfully
kubectl label node k8s-worker-1 soon-to-exist=yes

# Watch pod get scheduled
kubectl get pod queue-demo-pod -o wide

# Cleanup
kubectl delete pod queue-demo-pod
kubectl label node k8s-worker-1 soon-to-exist-
kill %1
```

---

## 22. API Server and etcd Interaction

### 22.1 The Golden Rule for Scheduling

```
╔═══════════════════════════════════════════════════════════════════════╗
║                                                                       ║
║  THE SCHEDULER NEVER ACCESSES etcd DIRECTLY.                         ║
║                                                                       ║
║  Scheduling flow:                                                     ║
║                                                                       ║
║  1. Scheduler reads node state from LOCAL CACHE (informer-backed)    ║
║     → Not from API server per request                                ║
║     → Not from etcd at all                                           ║
║                                                                       ║
║  2. Scheduler reads Pod specs from LOCAL CACHE                       ║
║     → Including: nodeSelector, affinity, tolerations, priority        ║
║                                                                       ║
║  3. Scheduler WRITES only one thing: the Binding object              ║
║     POST /api/v1/namespaces/{ns}/pods/{name}/binding                 ║
║     This writes spec.nodeName to etcd via API server                 ║
║                                                                       ║
║  4. Controllers follow the same rule:                                 ║
║     kube-controller-manager → kube-apiserver → etcd                  ║
║     NEVER: kube-controller-manager → etcd directly                   ║
║                                                                       ║
╚═══════════════════════════════════════════════════════════════════════╝
```

### 22.2 What the Scheduler Reads vs Writes

```bash
# READS (via informer cache, not direct API calls per scheduling cycle):
# - Pod specs: nodeSelector, affinity, tolerations, priorityClassName
# - Node objects: labels, taints, capacity, allocatable, conditions
# - PersistentVolumes: topology constraints
# - PriorityClass objects: value, preemptionPolicy

# WRITES (via API server, not etcd):
# - Binding object: spec.nodeName = selectedNode
# POST /api/v1/namespaces/{ns}/pods/{name}/binding
# This is the only write operation the scheduler makes

# See the binding call in verbose mode
kubectl run test --image=nginx --dry-run=server -v=9 2>&1 | \
  grep "POST\|binding" | head -5
```

---

## 23. Leader Election for HA Scheduling

### 23.1 Why the Scheduler Needs Leader Election

In a 3-node HA control plane, running 3 active schedulers simultaneously would cause:
- Race conditions: Two schedulers both try to bind the same Pod to different nodes
- Duplicate bindings: Pod bound twice (API server rejects second binding, but waste occurs)
- Resource conflicts: Both schedulers reserve resources on different nodes for the same Pod

**Solution:** Only ONE scheduler instance is active (the leader). Others are warm standbys.

### 23.2 Leader Election via Lease Object

```bash
# View the scheduler's Lease object
kubectl get lease kube-scheduler -n kube-system -o yaml
```

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-scheduler
  namespace: kube-system
spec:
  acquireTime: "2025-01-01T10:00:00.000000Z"
  holderIdentity: "k8s-cp-1_abc123-def456"  # Current leader
  leaseDurationSeconds: 15                    # Must renew within 15s
  leaseTransitions: 3                         # Leadership changes count
  renewTime: "2025-01-01T14:30:00.123000Z"   # Last renewed at this time
```

```bash
# Watch for leadership changes
kubectl get lease kube-scheduler -n kube-system -w

# Simulate leader failure (in lab only!)
# kubectl delete pod -n kube-system kube-scheduler-k8s-cp-1
# Leader changes within --leader-elect-lease-duration (default 15s)
```

### 23.3 Leader Election Configuration Flags

| Flag | Default | Description |
|---|---|---|
| `--leader-elect` | `true` | Enable leader election |
| `--leader-elect-lease-duration` | `15s` | How long a lease is valid |
| `--leader-elect-renew-deadline` | `10s` | Leader must renew before this |
| `--leader-elect-retry-period` | `2s` | Standby retry interval |
| `--leader-elect-resource-lock` | `leases` | Lease object type |
| `--leader-elect-resource-namespace` | `kube-system` | Lease namespace |
| `--leader-elect-resource-name` | `kube-scheduler` | Lease name |

### 23.4 HA Best Practices

```bash
# Run 3 schedulers (odd number for quorum)
kubectl get pods -n kube-system | grep scheduler
# kube-scheduler-k8s-cp-1   1/1   Running
# kube-scheduler-k8s-cp-2   1/1   Running
# kube-scheduler-k8s-cp-3   1/1   Running

# Monitor lease stability (high leaseTransitions = instability)
kubectl get lease kube-scheduler -n kube-system \
  -o jsonpath='{.spec.leaseTransitions}' && echo

# Tune for large/active clusters (avoid thrashing)
# --leader-elect-lease-duration=30s
# --leader-elect-renew-deadline=25s
# --leader-elect-retry-period=5s
```

---

## 24. Performance Tuning for the Scheduler

### 24.1 Key Scheduler Performance Flags

```yaml
# KubeSchedulerConfiguration
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler

# Percentage of cluster nodes to score (reduces CPU on large clusters)
percentageOfNodesToScore: 50
# Default: auto-calculates based on cluster size
# 50 = score only 50% of feasible nodes, pick best from those
# Lower = faster scheduling, potentially suboptimal placement
# Higher = more optimal placement, more CPU cost

# Parallel evaluation of pods
parallelism: 16
# Default: GOMAXPROCS (number of CPUs)
# Increase for high Pod creation rates
```

### 24.2 Filter Plugin Performance

```bash
# Check scheduler metrics for slow plugins
kubectl get --raw /metrics 2>/dev/null | \
  grep 'scheduler_framework_extension_point_duration' | \
  grep -v "^#" | sort -t= -k4 -rn | head -10

# Look for plugins with high execution time
# Common culprits on large clusters:
# - InterPodAffinity (complex label matching)
# - NodeResourcesFit (resource calculation)
# - VolumeBinding (volume topology checking)
```

### 24.3 Affinity Performance Considerations

```
PERFORMANCE IMPACT OF AFFINITY RULES:
─────────────────────────────────────────────────────────────────────
Required Node Affinity (Filter phase):
  → O(nodes * expressions) evaluation
  → Very fast for simple expressions (In, Exists)
  → Slightly slower for complex matchFields

Pod Affinity/Anti-Affinity (Filter + Score):
  → O(pods * pods) in worst case
  → EXPENSIVE for large clusters with many pods
  → Prefer topologySpreadConstraints for simple spread use cases

InterPod Affinity can cause:
  → Scheduling thrashing when pods can't be placed
  → Deadlocks with anti-affinity rules (avoid circular constraints)

Best practices for performance:
  → Use required affinity sparingly (each adds filter overhead)
  → Prefer preferred affinity when hard rules aren't needed
  → topologySpreadConstraints is more performant than pod anti-affinity
  → Avoid pod affinity in clusters >500 nodes
```

### 24.4 Resource Limits for kube-scheduler

```yaml
# In /etc/kubernetes/manifests/kube-scheduler.yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 2000m    # Allow bursting during heavy scheduling activity
    memory: 2Gi
```

---

## 25. Security Hardening Practices

### 25.1 RBAC for Scheduling-Related Operations

```yaml
# Role: Can add/remove node labels (nodeSelector management)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-labeler
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "patch", "update"]
    # WARNING: node labeling gives scheduling control — restrict carefully

---
# Role: Can manage taints (powerful — restricts workload placement)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-tainter
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "patch", "update"]
    # Also highly sensitive — tainting nodes can disrupt workloads

---
# Role: Can create/manage PriorityClasses (controls preemption)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: priority-class-manager
rules:
  - apiGroups: ["scheduling.k8s.io"]
    resources: ["priorityclasses"]
    verbs: ["get", "list", "create", "update", "patch", "delete"]
```

### 25.2 Node Tainting Security

```bash
# Audit all node taints (taints are security controls)
kubectl get nodes -o json | \
  jq '.items[] | {node: .metadata.name, taints: .spec.taints}'

# Verify control-plane taint is present (prevents workloads on CP)
kubectl describe node k8s-cp-1 | grep "Taint"
# Taints: node-role.kubernetes.io/control-plane:NoSchedule

# Check who can taint nodes (only platform/ops teams)
kubectl auth can-i taint nodes --as=system:serviceaccount:default:my-app
# no

kubectl auth can-i taint nodes --as=admin@company.com
# yes (only if explicitly granted)
```

### 25.3 PriorityClass Security

```bash
# High-priority classes can cause preemption = service disruption
# Restrict who can set priorityClassName

# Admission policy: Only allow known priority classes
# (Use OPA Gatekeeper or Kyverno for this)

# Check who is using high-priority classes
kubectl get pods --all-namespaces \
  -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}: {.spec.priorityClassName}{"\n"}{end}' | \
  grep -E "critical-infrastructure|production-service"
```

---

## 26. Monitoring and Observability

### 26.1 Key Scheduling Metrics

| Metric | Description | Alert |
|---|---|---|
| `scheduler_pending_pods{queue="active"}` | Pods in active queue | > 0 for > 5min |
| `scheduler_pending_pods{queue="backoff"}` | Pods in backoff | High rate = scheduling failures |
| `scheduler_pending_pods{queue="unschedulable"}` | Unschedulable pods | > 0 for > 5min |
| `scheduler_scheduling_attempt_duration_seconds` | Per-pod scheduling time | p99 > 100ms |
| `scheduler_preemption_attempts_total` | Total preemption attempts | High rate = resource pressure |
| `scheduler_preemption_victims` | Pods evicted due to preemption | Histogram of victims |
| `scheduler_pod_scheduling_duration_seconds` | End-to-end pod scheduling | p99 > 1s |
| `scheduler_volume_scheduling_duration_seconds` | Volume scheduling time | p99 > 5s |

### 26.2 Monitoring Node Labels and Taints

```bash
# Check for nodes without expected labels
kubectl get nodes --show-labels | grep -v "zone="
# Nodes missing zone label can't be targeted by zone-based affinity

# Check for unexpected taints
kubectl get nodes -o json | \
  jq '.items[] | select(.spec.taints != null) | {node: .metadata.name, taints: .spec.taints}'

# Check for nodes under pressure (auto-tainted)
kubectl get nodes -o json | \
  jq '.items[] | select(.status.conditions[] | 
    select(.type != "Ready" and .status == "True")) | 
    .metadata.name'
```

### 26.3 Prometheus Alert Rules for Scheduling

```yaml
groups:
  - name: kubernetes-scheduling
    rules:
      # Alert on high unschedulable pods
      - alert: KubePodsUnschedulable
        expr: |
          sum(kube_pod_status_unschedulable) > 5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "More than 5 pods are unschedulable for 10+ minutes"

      # Alert on scheduling queue backlog
      - alert: KubeSchedulerPendingQueue
        expr: |
          scheduler_pending_pods{queue="active"} > 50
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Scheduler active queue has 50+ pods"

      # Alert on high preemption rate (resource pressure indicator)
      - alert: KubeHighPreemptionRate
        expr: |
          rate(scheduler_preemption_attempts_total[5m]) > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High pod preemption rate - cluster may be resource constrained"

      # Alert on scheduling latency
      - alert: KubeSchedulingLatencyHigh
        expr: |
          histogram_quantile(0.99, 
            rate(scheduler_scheduling_attempt_duration_seconds_bucket[5m])
          ) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Scheduler p99 latency > 500ms"
```

---

## 27. Troubleshooting Scheduling Issues

### 27.1 Pods Not Created / Stuck in Pending

```bash
# ── DIAGNOSTIC TREE ──────────────────────────────────────────────────────

# Step 1: Why is the pod Pending?
kubectl describe pod <pending-pod> | grep -A 20 "Events:"

# Step 2: Decode the message
# "0/3 nodes are available: 
#  1 node(s) had taint {dedicated: gpu-team}, 
#  2 node(s) didn't match node selector"
# → Remove taint or add toleration; fix nodeSelector

# Step 3: Check nodeSelector
kubectl get pod <pending-pod> -o yaml | grep -A 5 "nodeSelector:"
kubectl get nodes --show-labels | grep <required-label>

# Step 4: Check node affinity
kubectl get pod <pending-pod> -o yaml | grep -A 30 "affinity:"

# Step 5: Check available nodes
kubectl get nodes -o wide
kubectl get nodes --show-labels

# Step 6: Check node conditions (not-ready, pressure)
kubectl get nodes -o json | jq \
  '.items[] | {name: .metadata.name, 
               conditions: [.status.conditions[] | 
               select(.type != "Ready" or .status != "True")]}'

# Step 7: Check if resources are available
kubectl describe nodes | grep -A 8 "Allocated resources:"
kubectl top nodes

# Step 8: Check for taints
kubectl get nodes -o json | jq \
  '.items[] | {name: .metadata.name, taints: .spec.taints}'
```

### 27.2 Affinity Rules Causing Pending

```bash
# Debugging affinity-caused scheduling failures

# Check if pods matching the affinity selector exist
kubectl get pod <pending-pod> -o yaml | \
  grep -A 30 "podAffinity:\|podAntiAffinity:"

# For pod affinity: check if target pods exist
kubectl get pods -l app=database   # Must exist for podAffinity to work

# For pod anti-affinity required: check if constraint is satisfiable
kubectl get pods -l app=my-service -o wide
# If all replicas are on the same node, required anti-affinity may be unsatisfiable
# Solution: Add more nodes, or change required to preferred

# Simulate scheduling decision
kubectl get events --field-selector reason=FailedScheduling \
  --sort-by='.lastTimestamp' -n <namespace>
```

### 27.3 Deployment Stuck / Rolling Update Issues

```bash
# Check rollout status
kubectl rollout status deployment/<n> --timeout=2m

# View ReplicaSets
kubectl get rs -l app=<n>

# Check why new pods aren't scheduling
NEW_RS=$(kubectl get rs -l app=<n> \
  --sort-by='.metadata.creationTimestamp' \
  -o name | tail -1)
kubectl get pods -l app=<n> -o wide

# Inspect new pod events
kubectl describe pod $(kubectl get pods -l app=<n> \
  --field-selector status.phase=Pending \
  -o name | head -1) | grep -A 10 "Events:"

# Common scheduling issues during rolling update:
# 1. Node selector changed → new pods have different requirements
# 2. Resources increased → not enough capacity for new pods
# 3. Affinity changed → new constraint can't be satisfied
# 4. Priority class changed → unexpected preemption
```

### 27.4 Node NotReady — Scheduling Impact

```bash
# Check which nodes are NotReady
kubectl get nodes | grep -v " Ready"

# See what taints auto-applied to NotReady nodes
kubectl describe node <notready-node> | grep "Taint"
# node.kubernetes.io/not-ready:NoSchedule

# See if pods were evicted
kubectl get events --all-namespaces | grep -i "evict\|Evicted"

# Check if replacement pods are scheduling
kubectl get pods -l app=<n> --all-namespaces -o wide
# Some may still be Pending if remaining nodes are full

# Manual intervention: cordon and drain
kubectl cordon <notready-node>    # Stop new scheduling
kubectl drain <notready-node> \
  --ignore-daemonsets \
  --delete-emptydir-data

# After fixing: uncordon
kubectl uncordon <notready-node>
```

### 27.5 Priority and Preemption Debugging

```bash
# Find pods being preempted
kubectl get events --all-namespaces | grep -i "preempt"

# Find what priority levels are in use
kubectl get pods --all-namespaces \
  -o custom-columns="NAMESPACE:.metadata.namespace,NAME:.metadata.name,PRIORITY:.spec.priority" | \
  sort -k3 -rn | head -20

# Check if a pod triggered preemption
kubectl describe pod <high-priority-pod> | grep "preempt\|nominatedNode"

# Find pods that could be preempted
kubectl get pods --all-namespaces \
  -o custom-columns="NAME:.metadata.name,PRIORITY:.spec.priority,CLASS:.spec.priorityClassName" | \
  awk '$2 < 10000'  # Low priority pods
```

---

## 28. Disaster Recovery Concepts

### 28.1 Scheduler State — Completely Stateless

```
The kube-scheduler maintains NO persistent state.
All scheduling decisions are derived from:
  1. Node labels and taints (in etcd via API server)
  2. Pod specs: nodeSelector, affinity, tolerations (in etcd)
  3. PriorityClass definitions (in etcd)
  4. Node capacity and allocated resources (reported by kubelets)

If the scheduler crashes:
  - Running pods continue unaffected (kubelet manages them)
  - New pods stay Pending (no one to bind them)
  - After leader election (< 30s): new leader starts fresh
  - New leader re-lists all nodes and pending pods
  - Scheduling resumes from scratch — no work is lost

This is the power of the reconciliation pattern:
  Stateless controllers + stateful etcd = resilient system
```

### 28.2 etcd Backup and Scheduling Configuration

```bash
# Scheduling-related objects stored in etcd (included in etcd backup):
# - Node labels and taints
# - Pod specs (including affinity, nodeSelector, tolerations)
# - PriorityClass objects
# - Lease objects (leader election state)

# etcd backup protects all scheduling configuration
ETCDCTL="etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key"

kubectl exec -n kube-system etcd-k8s-cp-1 -- \
  $ETCDCTL snapshot save /tmp/scheduling-config-backup.db

# Check specific scheduling objects in etcd
kubectl exec -n kube-system etcd-k8s-cp-1 -- \
  $ETCDCTL get /registry/priorityclasses --prefix --keys-only

# GitOps as backup: All PriorityClasses, node labels in Git
# After DR: apply all manifests to restore exact scheduling config
```

### 28.3 Recovery Procedure for Lost Scheduling Configuration

```bash
# If scheduling config is lost (nodes relabeled, taints removed):
# 1. Restore from etcd backup OR
# 2. Apply from GitOps repository

# From GitOps:
kubectl apply -f k8s/priority-classes/
kubectl apply -f k8s/node-config/

# Re-label nodes (in a script)
./scripts/apply-node-labels.sh

# Re-apply taints
kubectl taint node k8s-worker-2 dedicated=gpu:NoSchedule

# Verify scheduling resumes normally
kubectl get pods --field-selector=status.phase=Pending
# Should be empty or quickly clearing
```

---

## 29. Comparison — kube-apiserver vs kube-scheduler vs kube-controller-manager

| Dimension | kube-apiserver | kube-scheduler | kube-controller-manager |
|---|---|---|---|
| **Primary function** | REST API gateway; only etcd accessor | Assigns Pods to Nodes | Runs all reconciliation controllers |
| **Scheduling role** | Validates and stores scheduling spec | Evaluates and enforces placement rules | Creates Pods for workloads (RS, DS, Job) |
| **Node label awareness** | Stores labels | Reads labels for scheduling decisions | Node controller updates node status |
| **Taint awareness** | Stores taints | TaintToleration filter plugin | Node controller adds system taints |
| **Priority awareness** | Stores PriorityClass + Pod priority | Sorts scheduling queue by priority | Not directly involved |
| **etcd access** | YES (only component) | NO (via API server informers) | NO (via API server) |
| **HA model** | Active-Active | Active-Passive (leader election) | Active-Passive (leader election) |
| **Failure impact** | Cluster management stops | New Pods not placed | Workloads not reconciled |
| **Recovery time** | Immediate (behind LB) | ~30s (leader election) | ~30s (leader election) |
| **Port** | 6443 | 10259 | 10257 |

---

## 30. ASCII Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║         KUBERNETES ADVANCED SCHEDULING — COMPLETE ARCHITECTURE                       ║
╠══════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                      ║
║  USER / DEVOPS ENGINEER                                                              ║
║  kubectl apply -f pod-with-affinity.yaml                                             ║
║  kubectl taint node worker-2 dedicated=gpu:NoSchedule                                ║
║  kubectl label node worker-1 topology.kubernetes.io/zone=us-east-1a                  ║
║           │                                                                          ║
║           │ HTTPS :6443                                                              ║
║           ▼                                                                          ║
║  ┌──────────────────────────────────────────────────────────────────────────────┐    ║
║  │                         CONTROL PLANE                                        │    ║
║  │                                                                              │    ║
║  │  ┌──────────────────────────────────────────────────────────────────────┐   │    ║
║  │  │                    kube-apiserver :6443                              │   │    ║
║  │  │  AuthN → AuthZ → Admission → Audit                                  │   │    ║
║  │  │  Stores: Pod specs (nodeSelector, affinity, tolerations, priority)  │   │    ║
║  │  │  Stores: Node labels, taints, PriorityClass objects                 │   │    ║
║  │  └──────────────────────────────┬───────────────────────────────────────┘   │    ║
║  │                                 │                                            │    ║
║  │              ┌──────────────────▼──────────────────────┐                    │    ║
║  │              │                etcd :2379                │                    │    ║
║  │              │  /registry/pods/       /registry/nodes/  │                    │    ║
║  │              │  /registry/priorityclasses/              │                    │    ║
║  │              │  SINGLE SOURCE OF TRUTH                  │                    │    ║
║  │              └─────────────────────────────────────────┘                    │    ║
║  │                                                                              │    ║
║  │  ┌──────────────────────────────────────────────────────────────────────┐   │    ║
║  │  │                   kube-scheduler :10259                              │   │    ║
║  │  │                   (Leader-elected, Active-Passive)                   │   │    ║
║  │  │                                                                      │   │    ║
║  │  │  Informers: WATCHES unscheduled Pods, Node state                    │   │    ║
║  │  │                                                                      │   │    ║
║  │  │  FILTER PHASE:  NodeSelector ✓  NodeAffinity ✓  TaintToleration ✓  │   │    ║
║  │  │                 PodAffinity ✓   ResourceFit ✓   VolumeBinding ✓    │   │    ║
║  │  │                                                                      │   │    ║
║  │  │  SCORE PHASE:   PreferredAffinity ✓  ResourceBalance ✓             │   │    ║
║  │  │                 ImageLocality ✓  InterPodAffinity ✓                │   │    ║
║  │  │                                                                      │   │    ║
║  │  │  PRIORITY QUEUE: High-priority pods scheduled first                 │   │    ║
║  │  │  PREEMPTION:     Evict low-priority pods if needed                  │   │    ║
║  │  │                                                                      │   │    ║
║  │  │  BIND: Write spec.nodeName to API server                            │   │    ║
║  │  └──────────────────────────────────────────────────────────────────────┘   │    ║
║  │                                                                              │    ║
║  │  ┌──────────────────────────────────────────────────────────────────────┐   │    ║
║  │  │               kube-controller-manager :10257                         │   │    ║
║  │  │  Node Controller: applies not-ready/unreachable taints automatically │   │    ║
║  │  │  RS/Deploy: Creates Pods (scheduler decides WHERE)                   │   │    ║
║  │  │  DaemonSet: Creates Pods per-node (scheduler bypass or aware)       │   │    ║
║  │  └──────────────────────────────────────────────────────────────────────┘   │    ║
║  └──────────────────────────────────────────────────────────────────────────────┘    ║
║                               │ kubelet watches for bound Pods                       ║
║              ┌────────────────┼────────────────────────┐                            ║
║              │                │                        │                            ║
║  ┌───────────▼─────────┐ ┌────▼─────────────────┐ ┌────▼─────────────────────┐    ║
║  │  WORKER NODE 1      │ │  WORKER NODE 2        │ │  WORKER NODE 3           │    ║
║  │  Labels:            │ │  Labels:              │ │  Labels:                 │    ║
║  │  zone=us-east-1a    │ │  zone=us-east-1b      │ │  zone=us-east-1c         │    ║
║  │  disk=ssd           │ │  dedicated=gpu        │ │  disk=ssd                │    ║
║  │                     │ │  Taints:              │ │                          │    ║
║  │  Taint: none        │ │  dedicated=gpu:       │ │  Taint: none             │    ║
║  │                     │ │  NoSchedule           │ │                          │    ║
║  │  ┌───────────────┐  │ │                       │ │  ┌────────────────────┐  │    ║
║  │  │ Regular Pods  │  │ │  ┌─────────────────┐  │ │  │  Regular Pods     │  │    ║
║  │  │ (no taint)    │  │ │  │ GPU Pods ONLY   │  │ │  │  (no taint)       │  │    ║
║  │  │ - ssd apps    │  │ │  │ (have toleration│  │ │  │  - ssd apps       │  │    ║
║  │  │ - east zone   │  │ │  │  + node affinity│  │ │  │  - zone spread    │  │    ║
║  │  └───────────────┘  │ │  │  for dedicated) │  │ │  └────────────────────┘  │    ║
║  │                     │ │  └─────────────────┘  │ │                          │    ║
║  │  Priority/Eviction: │ │                       │ │  Priority/Eviction:      │    ║
║  │  PriorityClass →    │ │  PriorityClass →      │ │  PriorityClass →         │    ║
║  │  critical > prod >  │ │  critical > prod >    │ │  critical > prod >       │    ║
║  │  dev > batch        │ │  dev > batch          │ │  dev > batch             │    ║
║  └─────────────────────┘ └───────────────────────┘ └──────────────────────────┘    ║
╚══════════════════════════════════════════════════════════════════════════════════════╝
```

---

## 31. Real-World Production Use Cases

### 31.1 Multi-Tenant GPU Cluster (ML Platform)

```yaml
# Dedicated GPU nodes for ML team using taint + affinity + priority
---
# Taint GPU nodes
# kubectl taint node gpu-node-1 hardware/gpu=nvidia-a100:NoSchedule
# kubectl label node gpu-node-1 hardware/gpu=nvidia-a100 team=ml

# ML training job manifest
apiVersion: batch/v1
kind: Job
metadata:
  name: llm-fine-tuning
  namespace: ml-production
spec:
  template:
    spec:
      priorityClassName: critical-ml-training
      tolerations:
        - key: "hardware/gpu"
          operator: "Equal"
          value: "nvidia-a100"
          effect: "NoSchedule"
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: hardware/gpu
                    operator: In
                    values:
                      - nvidia-a100
      containers:
        - name: trainer
          image: pytorch:2.0-gpu
          resources:
            limits:
              nvidia.com/gpu: "4"
              memory: 64Gi
            requests:
              cpu: "16"
              memory: 64Gi
      restartPolicy: OnFailure
```

### 31.2 HA Production Services with Zone Spread

```yaml
# Production service spread across 3 AZs with hard anti-affinity
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
spec:
  replicas: 6   # 2 per zone
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
    spec:
      priorityClassName: production-critical
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: payment-service
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
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
                      - spot
                      - preemptible
      containers:
        - name: payment-api
          image: payment-service:v3.1
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: "2"
              memory: 2Gi
```

### 31.3 Node Maintenance with Zero Downtime

```bash
# Production procedure for node maintenance using taints

# 1. Cordon the node (NoSchedule but existing pods stay)
kubectl cordon k8s-worker-1

# 2. Apply custom maintenance taint with NoExecute + 60s grace
kubectl taint node k8s-worker-1 \
  maintenance=scheduled:NoExecute

# 3. Pods without toleration get 300s grace period (default)
# Then evicted and rescheduled on remaining nodes

# 4. Verify drain (DaemonSets stay — they're node-critical)
kubectl drain k8s-worker-1 \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --timeout=300s

# 5. Verify all workload pods moved
kubectl get pods --all-namespaces \
  --field-selector spec.nodeName=k8s-worker-1 | \
  grep -v "kube-system"

# 6. Perform maintenance...

# 7. Return node to service
kubectl uncordon k8s-worker-1
kubectl taint node k8s-worker-1 maintenance=scheduled:NoExecute-
```

---

## 32. Best Practices for Production Environments

### 32.1 nodeSelector and Affinity

- **Use nodeSelector for simple, single-label constraints** — easier to read and debug
- **Use nodeAffinity when you need OR logic, NotIn, or preferences** — don't abuse nodeSelector for complex rules
- **Always use topology.kubernetes.io/zone for zone constraints** — cloud providers set this automatically; be consistent
- **Use `preferredDuringScheduling` for cost optimization** — allows scheduling even when preferred nodes aren't available
- **Avoid deeply nested affinity rules** — they're hard to debug and can cause unexpected Pending states
- **Document all affinity rules** — use annotations to explain why placement constraints exist

### 32.2 Taints and Tolerations

- **Use NoSchedule for new workload exclusion** — doesn't disrupt existing Pods
- **Use NoExecute only for emergency evacuation** — immediate Pod disruption
- **Pair taints with matching node labels** — use both for full placement control (taint repels, affinity attracts)
- **Keep system taints intact** — never remove node-role.kubernetes.io/control-plane:NoSchedule from control plane nodes
- **Document taint purpose** — taint keys should be self-explaining (team=gpu, maintenance=true)

### 32.3 Priority Classes

- **Design a clear priority hierarchy** — define levels for: system > production > staging > batch
- **Use `preemptionPolicy: Never` for batch jobs** — prevents disruption to existing workloads
- **Never give all teams access to high-priority classes** — RBAC protects PriorityClass creation
- **Set a sensible globalDefault** — `development` priority (10000) is a reasonable default

### 32.4 General Scheduling

- **Use topologySpreadConstraints over pod anti-affinity for spreading** — more performant, clearer intent
- **Label nodes consistently before deploying workloads** — unlabeled nodes mean lost scheduling opportunities
- **Test scheduling constraints with `--dry-run=server`** — catches issues before production
- **Monitor scheduling queue depth** — high unschedulable count signals need for cluster expansion

---

## 33. Common Mistakes and Pitfalls

### 33.1 Required Anti-Affinity Deadlock

**Mistake:** Requiring all replicas to be on different nodes, but you have fewer nodes than replicas.
```yaml
# BAD: If you have 2 nodes and replicas: 3, some pods are always Pending
podAntiAffinity:
  requiredDuringSchedulingIgnoredDuringExecution:
    - topologyKey: kubernetes.io/hostname
```
**Fix:** Use `preferredDuringSchedulingIgnoredDuringExecution` or ensure replica count ≤ node count.

---

### 33.2 Forgetting Tolerations for DaemonSet on Tainted Nodes

**Mistake:** Taint a node for GPU workloads, then deploy monitoring DaemonSet without toleration — monitoring misses GPU nodes.
**Fix:** Always add `operator: Exists` toleration in DaemonSets to tolerate all taints.

---

### 33.3 nodeSelector vs Affinity Confusion

**Mistake:** Using nodeSelector when you need OR logic between values.
```yaml
# WRONG: Can't express "zone A or zone B" with nodeSelector
nodeSelector:
  zone: us-east-1a   # AND us-east-1b is impossible!
```
**Fix:** Use nodeAffinity with `In` operator.

---

### 33.4 Circular Pod Affinity

**Mistake:** Pod A must be with Pod B, and Pod B must be with Pod C, but Pod C has anti-affinity with Pod A. None can schedule.
**Fix:** Design affinity rules as a DAG (no cycles); test with `--dry-run=server`.

---

### 33.5 High Priority Class = No Preemption Configured

**Mistake:** High priority class but preemptionPolicy: Never — high-priority Pod waits in queue indefinitely when cluster is full.
**Fix:** Set `preemptionPolicy: PreemptLowerPriority` for critical services that must run.

---

### 33.6 Relabeling Nodes Without Considering Running Pods

**Mistake:** Removing a label from a node that running pods have in their nodeSelector.
**Consequence:** Running pods are NOT evicted (IgnoredDuringExecution). But if they restart, they may not be able to reschedule.
**Fix:** Drain the node before relabeling; verify no pods have the label as a hard nodeSelector requirement.

---

## 34. Interview Questions — Beginner to Advanced

### Beginner Level

**Q1: What is the difference between nodeSelector and nodeAffinity?**

**A:** `nodeSelector` is the simpler, older mechanism. It takes a flat map of key-value pairs, and the node must have ALL of them (AND logic only). It cannot express OR conditions, negative matches, or preferences.

`nodeAffinity` is the expressive, modern replacement. It provides:
- Set-based operators: `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`
- Soft preferences: `preferredDuringSchedulingIgnoredDuringExecution` with weights
- Hard requirements: `requiredDuringSchedulingIgnoredDuringExecution`
- OR logic between `nodeSelectorTerms`

Rule of thumb: Use `nodeSelector` for single-label constraints. Use `nodeAffinity` for anything more complex.

---

**Q2: What are the three Taint effects and when would you use each?**

**A:**
- `NoSchedule`: New pods without matching toleration won't be scheduled here. Existing pods without toleration continue running. **Use for:** Dedicating nodes to specific workloads without disrupting what's already running.
- `PreferNoSchedule`: Scheduler tries to avoid placing pods here but will do so if no other option. **Use for:** Soft node preferences, cost optimization.
- `NoExecute`: New pods won't schedule AND existing pods without toleration are evicted. Has optional `tolerationSeconds` for grace period. **Use for:** Emergency node evacuation, critical maintenance.

---

**Q3: What is a PriorityClass and how does preemption work?**

**A:** A PriorityClass is a cluster-wide object that assigns an integer priority to Pods. Higher number = higher priority. Pods without a priorityClassName get the globalDefault value (or 0 if none).

When a high-priority Pod can't be scheduled (no resources), and `preemptionPolicy: PreemptLowerPriority` is set, the scheduler will:
1. Find nodes where evicting lower-priority Pods would create enough space
2. Gracefully evict those lower-priority Pods (with their terminationGracePeriodSeconds)
3. Schedule the high-priority Pod in the freed space

This ensures critical services can always get resources, even in a full cluster.

---

### Intermediate Level

**Q4: Explain the difference between `requiredDuringSchedulingIgnoredDuringExecution` and `preferredDuringSchedulingIgnoredDuringExecution`.**

**A:**
- **Required**: Applied during scheduling as a **hard filter**. If no node satisfies the rule, the Pod stays `Pending` indefinitely. The rule is evaluated by the Filter plugins in the scheduler pipeline. No node that fails this check can receive the Pod.
- **Preferred**: Applied during scheduling as a **soft weight**. Nodes satisfying the preference get bonus score points (1-100 per rule). If no preferred node exists, the Pod still schedules on a non-preferred node. The rule is evaluated by Score plugins.

The "IgnoredDuringExecution" part of both names means: **after a Pod is running, the affinity rules are NOT continuously enforced**. If you change a node's labels so they no longer satisfy the Pod's affinity, the Pod keeps running on that node. It would only be affected if rescheduled (after crash, manual delete, etc.).

---

**Q5: How do you create a dedicated node that ONLY accepts specific workloads?**

**A:** Use both a Taint and Node Affinity together:

1. **Taint the node** (repels pods without toleration):
   ```bash
   kubectl taint node gpu-node-1 dedicated=gpu-team:NoSchedule
   ```

2. **Label the node** (for affinity matching):
   ```bash
   kubectl label node gpu-node-1 dedicated=gpu-team
   ```

3. **In the Pod spec** — add both toleration AND affinity:
   ```yaml
   tolerations:
     - key: "dedicated"
       operator: "Equal"
       value: "gpu-team"
       effect: "NoSchedule"
   affinity:
     nodeAffinity:
       requiredDuringSchedulingIgnoredDuringExecution:
         nodeSelectorTerms:
           - matchExpressions:
               - key: dedicated
                 operator: In
                 values:
                   - gpu-team
   ```

**Why both?** Taint alone prevents OTHER pods from landing here, but doesn't ATTRACT your pods. Affinity attracts your pods but doesn't REPEL others. Together they achieve true exclusivity.

---

**Q6: Explain the scheduling queue and how priority affects it.**

**A:** The scheduler uses a multi-queue system:

- **ActiveQ**: Pods ready to be scheduled, sorted by priority descending. High-priority pods jump to the front. Pods with equal priority are FIFO.
- **BackoffQ**: Pods that failed scheduling are moved here with exponential backoff. They're retried periodically.
- **UnschedulableQ**: Pods that the scheduler determined cannot be scheduled right now. They're moved back to ActiveQ when cluster state changes (new node joins, taint removed, resource freed).

When a high-priority Pod arrives:
1. It's added to the front of ActiveQ (highest priority = processed first)
2. If no nodes are feasible, scheduler runs **PostFilter** (preemption): identifies lowest-priority running pods that, if evicted, would free enough space
3. Those pods are evicted, the high-priority pod is marked with a `nominatedNodeName`
4. In the next scheduling cycle, the pod is placed on the freed node

---

### Advanced Level

**Q7: A pod with required pod anti-affinity is stuck in Pending. How do you diagnose and fix it?**

**A:** Diagnosis:
```bash
kubectl describe pod <pending-pod> | grep -A 10 "Events:"
# "0/3 nodes are available: 3 node(s) didn't match pod anti-affinity rules"
```

Root causes:
1. **Not enough nodes**: Required anti-affinity with `topologyKey: kubernetes.io/hostname` means you need as many nodes as replicas. If you have 3 replicas but all 3 workers already have a conflicting pod, no node is available.
2. **Missing topology key**: The `topologyKey` must be a label that exists on the nodes. If nodes don't have the label, no node matches.
3. **Circular affinity**: Pod A requires being with Pod B, but Pod B has anti-affinity against Pod A.

Fixes:
- Add more nodes (if it's a capacity issue)
- Change `requiredDuringScheduling` to `preferredDuringScheduling` (if hard constraint isn't truly needed)
- Use `topologySpreadConstraints` instead (better for spreading use cases)
- Debug by temporarily removing the anti-affinity and observing where the pod schedules

**Q8: Describe how the scheduler handles a scenario where a high-priority Pod needs to preempt, but the eviction of lower-priority Pods would violate a PodDisruptionBudget (PDB).**

**A:** The scheduler respects PDBs during preemption. When considering which low-priority pods to evict:

1. Scheduler identifies candidate victims (lower-priority pods that would free enough resources)
2. For each candidate, it checks if evicting that pod would violate any PDB (`spec.minAvailable` or `spec.maxUnavailable`)
3. Pods protected by PDB that would be violated are NOT evicted
4. Scheduler looks for a combination of lower-priority pods that CAN be evicted without violating any PDB

Result possibilities:
- **Some pods evicted**: High-priority pod is scheduled on the freed node
- **No valid victims found**: High-priority pod stays Pending (even with preemption, sometimes there's no way to make room without PDB violation)
- **Partial placement**: In large clusters, preemption on a different node may be possible

This is why PDBs are essential: they protect critical services from being preempted even by higher-priority workloads.

---

## 35. Cheat Sheet — Commands, Flags & Manifests

### 35.1 Node Label Management

```bash
# Add labels
kubectl label node <n> key=value [key2=value2...]
kubectl label node <n> topology.kubernetes.io/zone=us-east-1a

# Remove labels (append - to key)
kubectl label node <n> key-

# Update existing (requires --overwrite)
kubectl label node <n> key=newvalue --overwrite

# View labels
kubectl get nodes --show-labels
kubectl get nodes -L key1,key2  # Show as columns
kubectl get node <n> -o jsonpath='{.metadata.labels}' | jq
```

### 35.2 Taint Management

```bash
# Apply taint
kubectl taint node <n> key=value:Effect
kubectl taint node <n> key:Effect         # No value

# Remove taint (append - to taint string)
kubectl taint node <n> key=value:Effect-
kubectl taint node <n> key:Effect-

# View taints
kubectl describe node <n> | grep -A 5 "Taints:"
kubectl get nodes -o custom-columns="NAME:.metadata.name,TAINTS:.spec.taints"

# Common effects:
# NoSchedule         - New pods only; existing stay
# PreferNoSchedule   - Soft; avoid if possible
# NoExecute          - Evict existing + block new
```

### 35.3 PriorityClass Management

```bash
# View priority classes
kubectl get priorityclasses
kubectl describe priorityclass <n>

# Create (via YAML)
kubectl apply -f priorityclass.yaml

# Delete
kubectl delete priorityclass <n>

# Check which pods use each priority class
kubectl get pods --all-namespaces \
  -o custom-columns="NS:.metadata.namespace,NAME:.metadata.name,PCLASS:.spec.priorityClassName"
```

### 35.4 Quick Scheduling Snippets

```yaml
# nodeSelector (simple)
spec:
  nodeSelector:
    key: value

# Required node affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - {key: zone, operator: In, values: [us-east-1a]}

# Preferred node affinity
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - {key: disk, operator: In, values: [ssd]}

# Pod anti-affinity (spread across nodes)
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: my-app
            topologyKey: kubernetes.io/hostname

# Toleration
spec:
  tolerations:
    - key: "dedicated"
      operator: "Equal"
      value: "gpu"
      effect: "NoSchedule"
    - operator: "Exists"   # Tolerate ALL taints

# Priority class
spec:
  priorityClassName: production-service
```

### 35.5 Troubleshooting Commands

```bash
# Why is pod Pending?
kubectl describe pod <n> | grep -A 20 "Events:"

# What nodes match a nodeSelector?
kubectl get nodes -l disk=ssd,zone=us-east-1a

# What pods can schedule on a node?
kubectl describe node <n> | grep -A 10 "Conditions:\|Taints:\|Allocated:"

# Scheduling events cluster-wide
kubectl get events --field-selector reason=FailedScheduling -A

# Node resource availability
kubectl top nodes
kubectl describe nodes | grep -A 5 "Allocated resources:"

# Check preemption
kubectl get events -A | grep -i preempt

# Scheduler leader
kubectl get lease kube-scheduler -n kube-system -o yaml
```

### 35.6 Affinity Operators Quick Reference

| Operator | Logic | Values Required |
|---|---|---|
| `In` | value in list | Yes (1+) |
| `NotIn` | value NOT in list | Yes (1+) |
| `Exists` | key exists (any value) | No |
| `DoesNotExist` | key doesn't exist | No |
| `Gt` | value > (numeric string) | Yes (1) |
| `Lt` | value < (numeric string) | Yes (1) |

---

## 36. Key Takeaways & Summary

### The Advanced Scheduling Mental Model

```
PLACEMENT DECISION FLOW:
─────────────────────────────────────────────────────────────────────────────
1. NODE PRE-FILTERING (eliminates infeasible nodes):
   
   nodeSelector         → Simple: ALL labels must match (AND only)
   Required nodeAffinity → Expressive: In/NotIn/Exists with OR between terms
   Taint/Toleration     → Node's taint must be tolerated by Pod
   Resource check       → Node must have enough CPU/memory for requests

2. NODE SCORING (ranks feasible nodes):
   
   Preferred nodeAffinity → Weight bonus for matching nodes
   Pod affinity           → Bonus for co-location with target pods
   Resource balance       → Favor least/most allocated based on strategy
   Image locality         → Bonus if image already cached on node

3. PRIORITY & PREEMPTION:
   
   PriorityClass         → Higher value = processed first in queue
   PreemptLowerPriority  → Can evict lower-priority pods to make room
   preemptionPolicy=Never → Won't evict; waits for natural availability

4. RESULT:
   
   Winning node's name written to spec.nodeName → kubelet starts Pod
─────────────────────────────────────────────────────────────────────────────
```

### The 10 Rules of Advanced Scheduling

1. **nodeSelector is AND-only.** For OR logic or negative matching, use nodeAffinity.
2. **Required affinity = hard filter.** Pod stays Pending if no node satisfies it.
3. **Preferred affinity = score boost.** Pod schedules on any node; preferred nodes just rank higher.
4. **Taints repel; tolerations accept.** Pair taints on nodes with tolerations in Pods for dedicated nodes.
5. **Use NoSchedule for soft dedication; NoExecute for emergency evacuation.** Know the difference.
6. **Combine taint + affinity for true exclusivity.** Taint repels others; affinity attracts intended workloads.
7. **topologyKey defines the failure domain.** hostname = node-level; zone = AZ-level; region = region-level.
8. **Priority determines scheduling order AND preemption eligibility.** Design a clear hierarchy.
9. **Controllers never touch etcd directly.** All scheduling writes go through the API server.
10. **The scheduler is stateless.** All state is in etcd. Scheduler crash = pods stay where they are; new scheduling resumes after leader election.

---

> **This guide covers Kubernetes v1.29+ scheduling mechanisms. Consult the official documentation at https://kubernetes.io/docs/concepts/scheduling-eviction/ for the most current information.**

---

*End of: Kubernetes Pod Scheduling — Part 2: Advanced Scheduling, Affinity, Taints & Priority Classes*
