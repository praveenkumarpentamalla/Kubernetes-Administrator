## Table of Contents

1. [Introduction & Strategic Importance](#1-introduction--strategic-importance)
2. [Core Identity Table](#2-core-identity-table)
3. [ASCII Architecture Diagram](#3-ascii-architecture-diagram)
4. [The Controller Pattern: Watch → Compare → Act → Loop](#4-the-controller-pattern-watch--compare--act--loop)
5. [Kubelet Lifecycle & Startup Sequence](#5-kubelet-lifecycle--startup-sequence)
6. [Pod Lifecycle Management by Kubelet](#6-pod-lifecycle-management-by-kubelet)
7. [Container Runtime Interface (CRI)](#7-container-runtime-interface-cri)
8. [Static Pods](#8-static-pods)
9. [Node Registration & Status Reporting](#9-node-registration--status-reporting)
10. [Resource Management: CPU, Memory & QoS](#10-resource-management-cpu-memory--qos)
11. [Volume Management & CSI Integration](#11-volume-management--csi-integration)
12. [Health Probes: Liveness, Readiness & Startup](#12-health-probes-liveness-readiness--startup)
13. [Garbage Collection](#13-garbage-collection)
14. [Eviction Manager & Node Pressure](#14-eviction-manager--node-pressure)
15. [Interaction with API Server & etcd](#15-interaction-with-api-server--etcd)
16. [Leader Election (N/A for Kubelet) vs. Other Control Plane Components](#16-leader-election-na-for-kubelet-vs-other-control-plane-components)
17. [Security Hardening Practices](#17-security-hardening-practices)
18. [Performance Tuning & Flags](#18-performance-tuning--flags)
19. [Monitoring & Observability](#19-monitoring--observability)
20. [Troubleshooting Guide with Real kubectl Commands](#20-troubleshooting-guide-with-real-kubectl-commands)
21. [Comparison with kube-apiserver & kube-scheduler](#21-comparison-with-kube-apiserver--kube-scheduler)
22. [Disaster Recovery Concepts](#22-disaster-recovery-concepts)
23. [Real-World Production Use Cases](#23-real-world-production-use-cases)
24. [Best Practices for Production Environments](#24-best-practices-for-production-environments)
25. [Common Mistakes and Pitfalls](#25-common-mistakes-and-pitfalls)
26. [Hands-On Labs & Mini Practical Exercises](#26-hands-on-labs--mini-practical-exercises)
27. [Interview Questions: Beginner to Advanced](#27-interview-questions-beginner-to-advanced)
28. [Cheat Sheet: Commands & Flags](#28-cheat-sheet-commands--flags)
29. [Key Takeaways & Summary](#29-key-takeaways--summary)

---

## 1. Introduction & Strategic Importance

The **Kubelet** is the primary node agent in Kubernetes — the single most critical process running on every node in the cluster, including both worker nodes and (in most configurations) control plane nodes. It is the bridge between the cluster's desired state, which is defined in the API server, and the actual reality of what is running on each machine.

### What is the Kubelet?

The Kubelet is a **daemon process** that runs as a systemd service (or equivalent) directly on every Kubernetes node. It is not containerized itself — it runs natively on the host OS. Its primary responsibility is to ensure that the containers described in PodSpecs assigned to its node are running and healthy.

When the Kubernetes scheduler assigns a Pod to a node, it is the Kubelet on that node that actually performs the work: pulling images, creating containers, attaching volumes, configuring networking, running health checks, and reporting status back to the control plane.

### Why Kubelet Matters in the Architecture

Understanding the Kubelet is non-negotiable for anyone operating Kubernetes in production because:

- **Without the Kubelet, no workloads run** — even if the control plane is perfectly healthy, a node with a broken Kubelet cannot host any Pods.
- **It is the trust boundary between the cluster and the node** — the Kubelet enforces resource limits, security policies, and health gates.
- **It is the primary source of node telemetry** — metrics, conditions, and capacity data all originate from the Kubelet.
- **Most production outages trace back to node-level failures** that the Kubelet was the first to detect or was unable to recover from.
- **It directly integrates with CRI, CNI, and CSI** — the three foundational plugin interfaces of Kubernetes.

In short, the Kubelet is the "hands" of Kubernetes — the component that physically does the work on each node that the brain (control plane) has decided must happen.

---

## 2. Core Identity Table

| Field | Value |
|---|---|
| **Binary Name** | `kubelet` |
| **Default Manifest Location** | `/etc/kubernetes/manifests/` (static pods) |
| **Config File** | `/var/lib/kubelet/config.yaml` |
| **Service Unit** | `kubelet.service` (systemd) |
| **Primary Port (HTTPS, Read/Write)** | `10250` |
| **Read-Only Port (deprecated)** | `10255` (disabled in hardened setups) |
| **Health Check Port** | `10248` (`/healthz` endpoint) |
| **Process Name on Node** | `kubelet` |
| **Runs On** | Every node (worker + control plane) |
| **Runs As** | `root` (or privileged user) |
| **Communicates With** | kube-apiserver (via HTTPS), container runtime (via CRI/gRPC), CNI plugins, CSI plugins |
| **Does NOT Communicate With** | etcd directly |
| **Authentication to API Server** | TLS client certificates (node bootstrapping) |
| **Authorization Mode** | Node Authorizer (RBAC node mode) |
| **Key Dependency** | Container Runtime (containerd, CRI-O) |
| **Config Hot-Reload** | Partial (some flags require restart) |
| **Process Manager** | systemd (recommended) |
| **High Availability** | Not applicable — one per node, no leader election needed |
| **Log Location** | `journalctl -u kubelet` |
| **State Storage** | `/var/lib/kubelet/` |
| **Pod State Checkpoint** | `/var/lib/kubelet/pods/` |

---

## 3. ASCII Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                         KUBERNETES CLUSTER ARCHITECTURE                     ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║   ┌─────────────────────────────── CONTROL PLANE ──────────────────────┐   ║
║   │                                                                      │   ║
║   │  ┌──────────────┐   ┌──────────────┐   ┌───────────────────────┐   │   ║
║   │  │ kube-apiserver│   │kube-scheduler│   │ kube-controller-manager│  │   ║
║   │  │   :6443       │   │              │   │                       │   │   ║
║   │  └──────┬───────┘   └──────┬───────┘   └───────────┬───────────┘   │   ║
║   │         │                  │                         │               │   ║
║   │         └──────────────────┴──────────┬─────────────┘               │   ║
║   │                                        │                             │   ║
║   │  ┌──────────────────────┐    ┌─────────┴──────────┐                 │   ║
║   │  │       etcd           │◄───│   API Server Watch  │                 │   ║
║   │  │  (Source of Truth)   │    └────────────────────┘                 │   ║
║   │  └──────────────────────┘                                            │   ║
║   └──────────────────────────────────────────────────────────────────────┘   ║
║                         │  HTTPS :6443 (Watch/Get/Update)                    ║
║   ┌─────────────────────┼──────────────────────────────────────────────┐    ║
║   │                WORKER NODE                                           │    ║
║   │                     │                                                │    ║
║   │   ┌─────────────────▼─────────────────────────────────────────┐    │    ║
║   │   │                     KUBELET :10250                         │    │    ║
║   │   │                                                            │    │    ║
║   │   │  ┌──────────────┐  ┌───────────────┐  ┌───────────────┐  │    │    ║
║   │   │  │ Pod Manager  │  │ Volume Manager│  │ Probe Manager │  │    │    ║
║   │   │  └──────┬───────┘  └───────┬───────┘  └───────┬───────┘  │    │    ║
║   │   │         │                  │                   │           │    │    ║
║   │   │  ┌──────▼───────┐  ┌───────▼───────┐  ┌───────▼───────┐  │    │    ║
║   │   │  │Eviction Mgr  │  │Image Manager  │  │  PLEG Engine  │  │    │    ║
║   │   │  └──────────────┘  └───────────────┘  └───────────────┘  │    │    ║
║   │   └───────────────────────────┬────────────────────────────────┘    │    ║
║   │                               │ gRPC (CRI)                          │    ║
║   │           ┌───────────────────▼──────────────────────┐              │    ║
║   │           │          Container Runtime                │              │    ║
║   │           │         (containerd / CRI-O)              │              │    ║
║   │           └──────────────┬───────────────────────────┘              │    ║
║   │                          │                                           │    ║
║   │           ┌──────────────▼──────────────────────────┐               │    ║
║   │           │              OCI Runtime                 │               │    ║
║   │           │          (runc / kata-runtime)           │               │    ║
║   │           └──────────────────────────────────────────┘               │    ║
║   │                                                                       │    ║
║   │   ┌─────────────────┐   ┌────────────────┐   ┌──────────────────┐   │    ║
║   │   │   CNI Plugin    │   │   CSI Plugin   │   │  cAdvisor/Stats  │   │    ║
║   │   │  (Pod Network)  │   │   (Volumes)    │   │   (Metrics)      │   │    ║
║   │   └─────────────────┘   └────────────────┘   └──────────────────┘   │    ║
║   └───────────────────────────────────────────────────────────────────────┘   ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

**Key Data Flows:**
- `API Server → Kubelet`: Pod assignments, ConfigMap/Secret updates, node drain signals
- `Kubelet → API Server`: Node status updates, Pod status updates, lease renewals
- `Kubelet → Container Runtime`: Start/stop containers, exec, logs, image pull
- `Kubelet → CNI`: Set up Pod network namespace
- `Kubelet → CSI`: Mount/unmount volumes

---

## 4. The Controller Pattern: Watch → Compare → Act → Loop

The Kubelet, like all controllers in Kubernetes, follows a universal control-plane pattern known as the **reconciliation loop** or **control loop**. This is the foundational pattern of Kubernetes itself.

```
┌─────────────────────────────────────────────────────────────────┐
│                   RECONCILIATION CONTROL LOOP                    │
│                                                                   │
│   ┌──────────┐      ┌──────────────┐      ┌───────────────┐     │
│   │  WATCH   │─────►│   COMPARE    │─────►│     ACT       │     │
│   │          │      │              │      │               │     │
│   │ Observe  │      │ Desired State│      │ Create/Update │     │
│   │ API      │      │ vs           │      │ Delete/Patch  │     │
│   │ Server   │      │ Actual State │      │ Containers    │     │
│   └──────────┘      └──────────────┘      └───────┬───────┘     │
│        ▲                                           │              │
│        └───────────────────────────────────────────┘              │
│                          LOOP                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Phase 1: WATCH

The Kubelet maintains a persistent **watch connection** to the kube-apiserver. Specifically, it watches for:

- `Pod` objects where `.spec.nodeName` equals the current node's name
- `Node` objects (its own node object)
- `ConfigMap` and `Secret` objects (referenced by its Pods, through projected volumes)

This watch is implemented using the **Informer Pattern** — a shared in-memory cache that reduces load on the API server by coalescing watch events and caching objects locally.

```
Informer Architecture:
  ListWatch → Reflector → Delta FIFO Queue → Indexer (Local Store)
                                           └→ Event Handlers (callbacks)
```

### Phase 2: COMPARE

On every reconciliation cycle (and when a watch event is received), the Kubelet compares:

- **Desired State**: What Pods should be running (from API server)
- **Actual State**: What containers are actually running (from Container Runtime via CRI)

Differences (called **drift**) trigger action.

### Phase 3: ACT

Based on the comparison, the Kubelet takes one or more actions:

| Drift Detected | Action Taken |
|---|---|
| Pod assigned but not running | Pull images, create containers, set up network |
| Container crashed (CrashLoopBackOff) | Restart container (with exponential backoff) |
| Pod deleted from API server | Terminate containers, clean up volumes |
| Liveness probe failing | Restart the specific container |
| Readiness probe failing | Remove Pod from Endpoints (no traffic) |
| Resource limit exceeded | OOM kill or CPU throttling |
| Pod evicted by Eviction Manager | Terminate and update Pod status |

### Phase 4: LOOP

The cycle repeats continuously. Key loop timing parameters:

| Parameter | Default | Description |
|---|---|---|
| `syncFrequency` | 1 minute | How often to full-sync pod state |
| `fileCheckFrequency` | 20 seconds | How often to check static pod manifests |
| `httpCheckFrequency` | 20 seconds | How often to check HTTP pod sources |
| Node status update frequency | 10 seconds | How often to update node status |
| Node lease renewal | 10 seconds | Heartbeat to API server |

### The Informer Pattern (Deep Dive)

The Informer is a higher-level abstraction over the raw `List + Watch` API. It is the backbone of how every Kubernetes controller (including Kubelet's internal watchers) observes cluster state efficiently.

```
┌──────────────────────────────────────────────────────────────┐
│                    INFORMER INTERNALS                         │
│                                                               │
│  API Server                                                   │
│     │                                                         │
│     │  List (initial full state)                             │
│     ▼                                                         │
│  ┌──────────┐                                                │
│  │ Reflector │◄── Watch (streaming deltas after List)        │
│  └────┬─────┘                                                │
│       │ Adds events                                           │
│  ┌────▼──────────┐                                           │
│  │ Delta FIFO    │  (Added, Updated, Deleted, Sync events)   │
│  │   Queue       │                                           │
│  └────┬──────────┘                                           │
│       │ Processes events                                      │
│  ┌────▼──────────┐     ┌──────────────────┐                 │
│  │  Indexer      │     │ Event Handlers   │                 │
│  │ (Local Cache) │     │ OnAdd()          │                 │
│  │               │     │ OnUpdate()       │                 │
│  │ Thread-safe   │     │ OnDelete()       │                 │
│  └───────────────┘     └────────┬─────────┘                 │
│                                  │                           │
│                         ┌────────▼────────┐                 │
│                         │   Work Queue    │                 │
│                         │ (Rate-limited)  │                 │
│                         └────────┬────────┘                 │
│                                  │                           │
│                         ┌────────▼────────┐                 │
│                         │   Worker Loop   │                 │
│                         │  (reconcile)    │                 │
│                         └─────────────────┘                 │
└──────────────────────────────────────────────────────────────┘
```

**Benefits of the Informer Pattern:**
- Reduces API server load — only one `List` at startup, then streaming `Watch`
- Provides local in-memory cache for fast reads
- Handles watch reconnections automatically
- Deduplicates events (if a key is already in the work queue, it's not added again)

### Work Queues

Work queues are the buffer between the informer's event handlers and the reconciliation workers. The Kubelet uses **rate-limited work queues** to:

- **Prevent thundering herd**: Multiple rapid changes to the same object result in a single reconciliation
- **Implement backoff**: Failed reconciliations are retried with exponential backoff
- **Decouple producers from consumers**: Event handlers (producers) and reconcile loops (consumers) operate independently

```go
// Conceptual work queue operation:
queue.AddRateLimited(key)       // Producer (informer event handler)
key := queue.Get()             // Consumer (reconcile loop)
queue.Done(key)                // Signal completion
queue.Forget(key)              // Reset backoff counter on success
queue.AddRateLimited(key)      // Re-add on failure (triggers backoff)
```

---

## 5. Kubelet Lifecycle & Startup Sequence

Understanding the Kubelet startup sequence is essential for diagnosing node-level problems.

```
Kubelet Startup Sequence:
─────────────────────────

1. Parse flags and load KubeletConfiguration
   └─ /etc/kubernetes/kubelet.conf (kubeconfig)
   └─ /var/lib/kubelet/config.yaml (KubeletConfiguration)

2. Initialize the node object
   └─ Determine node name (hostname or --hostname-override)
   └─ Discover node IP address
   └─ Read machine info (CPU, memory, OS info via cadvisor)

3. Authenticate to API Server
   └─ Use TLS client certificate from kubeconfig
   └─ Node bootstrapping (if TLS bootstrap enabled)

4. Register node with API server (if not already registered)
   └─ POST /api/v1/nodes with node spec
   └─ Set initial node labels and taints

5. Initialize Container Runtime Interface (CRI)
   └─ Connect to container runtime via gRPC socket
   └─ Verify runtime version compatibility

6. Initialize internal managers:
   └─ Volume Manager
   └─ Image Manager (garbage collection)
   └─ Eviction Manager
   └─ OOM Watcher
   └─ PLEG (Pod Lifecycle Event Generator)
   └─ Probe Manager (liveness, readiness, startup)
   └─ Certificate Manager

7. Start the sync loop (main reconciliation loop)
   └─ Begin watching for pods assigned to this node
   └─ Start static pod file watcher

8. Begin reporting node status (every 10s by default)
   └─ Update node conditions (Ready, MemoryPressure, DiskPressure, PIDPressure)
   └─ Renew node Lease object (heartbeat)

9. Serve HTTPS API on :10250
   └─ /metrics
   └─ /pods
   └─ /exec
   └─ /logs
   └─ /healthz
```

### PLEG: Pod Lifecycle Event Generator

PLEG is one of the most critical internal components of the Kubelet. It is a polling mechanism that:

- Queries the Container Runtime (via CRI) for the state of all containers on the node every second
- Compares the current state with the previous state
- Generates lifecycle events (ContainerStarted, ContainerDied, ContainerRemoved)
- These events drive the sync loop

**PLEG Unhealthy** is a common production issue. When the CRI cannot respond to PLEG's queries within 3 minutes, the Kubelet marks itself as unhealthy:

```
# Symptom in logs:
PLEG is not healthy: pleg was last seen active 3m5s ago; threshold is 3m0s

# Result on node:
kubectl get nodes
NAME       STATUS     ROLES    AGE   VERSION
worker-1   NotReady   <none>   10d   v1.28.0
```

---

## 6. Pod Lifecycle Management by Kubelet

The Kubelet is the sole component responsible for the complete lifecycle of Pods on its node.

### Pod Admission

Before creating a Pod, the Kubelet runs it through **admission handlers**:

| Admission Handler | Purpose |
|---|---|
| `NamespaceLifecycle` | Reject Pods in terminating namespaces |
| `LimitRanger` | Enforce LimitRange defaults |
| `ServiceAccount` | Mount service account tokens |
| `NodeRestriction` | Node can only manage its own Pods |
| `AlwaysPullImages` (optional) | Force image pull on every restart |

### Pod State Machine

```
                      ┌─────────┐
                      │ Pending │◄──────────────────────────┐
                      └────┬────┘                           │
                           │ All containers created          │
                           ▼                                 │
                      ┌─────────┐                           │
                      │ Running │                           │
                      └────┬────┘                           │
               ┌───────────┴───────────┐                   │
               │                       │                   │
               ▼                       ▼                   │
         ┌──────────┐          ┌─────────────┐            │
         │ Succeeded │          │   Failed    │            │
         │(all done, │          │(container   │            │
         │  exit 0) │          │  exit != 0) │            │
         └──────────┘          └─────────────┘            │
                                                           │
                    CrashLoopBackOff ──────────────────────┘
                    (restart with exponential backoff)
```

### Container Restart Policy

| RestartPolicy | Behavior on Container Exit |
|---|---|
| `Always` (default) | Always restart, regardless of exit code |
| `OnFailure` | Restart only if exit code != 0 |
| `Never` | Never restart |

### Restart Backoff

The Kubelet uses an exponential backoff for container restarts to prevent rapid restart loops from consuming resources:

```
Backoff durations: 10s → 20s → 40s → 80s → 160s → 300s (max)
Reset after: 10 minutes of successful running
```

---

## 7. Container Runtime Interface (CRI)

The Kubelet does not directly manage containers. It communicates with the **Container Runtime** via the **CRI (Container Runtime Interface)** — a gRPC API defined by Kubernetes.

### CRI Architecture

```
Kubelet
  │
  │ gRPC
  ▼
┌─────────────────────────────────────┐
│     Container Runtime (CRI server) │
│                                     │
│  ┌─────────────────────────────┐   │
│  │ RuntimeService              │   │   ← Manage Pods/Containers
│  │  - RunPodSandbox()          │   │
│  │  - CreateContainer()        │   │
│  │  - StartContainer()         │   │
│  │  - StopContainer()          │   │
│  │  - RemoveContainer()        │   │
│  │  - Exec()                   │   │
│  └─────────────────────────────┘   │
│  ┌─────────────────────────────┐   │
│  │ ImageService                │   │   ← Manage Images
│  │  - PullImage()              │   │
│  │  - ListImages()             │   │
│  │  - RemoveImage()            │   │
│  └─────────────────────────────┘   │
└─────────────────────────────────────┘
         │
         │ OCI (runc, kata, etc.)
         ▼
    Actual Containers
```

### Supported CRI Implementations

| Runtime | CRI Socket | Notes |
|---|---|---|
| **containerd** | `/run/containerd/containerd.sock` | Default in modern K8s |
| **CRI-O** | `/var/run/crio/crio.sock` | Lightweight, OCI-focused |
| **Docker (via cri-dockerd)** | `/var/run/cri-dockerd.sock` | Deprecated since K8s 1.24 |

### CRI gRPC Socket Configuration

```yaml
# /var/lib/kubelet/config.yaml
containerRuntimeEndpoint: "unix:///run/containerd/containerd.sock"
imageServiceEndpoint: "unix:///run/containerd/containerd.sock"
```

---

## 8. Static Pods

Static Pods are a critical mechanism for running control plane components. They are managed **directly by the Kubelet** — not by the API server.

### How Static Pods Work

```
1. Kubelet watches directory: /etc/kubernetes/manifests/
2. Finds YAML manifest files (e.g., kube-apiserver.yaml)
3. Creates the Pod without consulting API server
4. Creates a "mirror Pod" in API server (read-only, reflects status)
5. Kubelet automatically restarts the Pod if it crashes
6. Deleting the mirror Pod from API server has no effect
```

### Static Pod Files on Control Plane Node

```bash
ls /etc/kubernetes/manifests/
# kube-apiserver.yaml
# kube-controller-manager.yaml
# kube-scheduler.yaml
# etcd.yaml
```

### Why Static Pods are Critical

- They allow the control plane to bootstrap without itself being available
- The Kubelet can run control plane components even before etcd is available
- They are the foundation of `kubeadm`-managed clusters
- They provide self-healing for control plane components

### Modifying Static Pods

```bash
# Edit on the node directly (Kubelet detects changes automatically)
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml

# Kubelet will terminate old Pod and create new one within seconds
# Monitor the mirror Pod
kubectl get pod -n kube-system kube-apiserver-<node-name> -w
```

---

## 9. Node Registration & Status Reporting

### Node Registration

When a Kubelet starts for the first time, it registers itself with the API server by creating a `Node` object. This process is called **node registration**.

```bash
# Node object created by Kubelet:
kubectl get node worker-1 -o yaml
```

Key fields the Kubelet populates:

| Field | Source | Example |
|---|---|---|
| `metadata.name` | Hostname / `--hostname-override` | `worker-1` |
| `status.capacity.cpu` | `/proc/cpuinfo` | `4` |
| `status.capacity.memory` | `/proc/meminfo` | `8Gi` |
| `status.capacity.pods` | `--max-pods` flag | `110` |
| `status.nodeInfo.osImage` | OS detection | `Ubuntu 22.04.3 LTS` |
| `status.nodeInfo.kernelVersion` | `uname -r` | `5.15.0-1034-aws` |
| `status.nodeInfo.containerRuntimeVersion` | CRI query | `containerd://1.7.2` |
| `status.nodeInfo.kubeletVersion` | Binary version | `v1.28.0` |

### Node Conditions

The Kubelet continuously updates `Node.status.conditions`:

| Condition | Type | Meaning when True |
|---|---|---|
| `Ready` | Normal | Node is healthy and can accept Pods |
| `MemoryPressure` | Problem | Available memory is low |
| `DiskPressure` | Problem | Available disk space is low |
| `PIDPressure` | Problem | Too many processes on the node |
| `NetworkUnavailable` | Problem | Network is not configured correctly |

```bash
# View node conditions
kubectl describe node worker-1 | grep -A 20 "Conditions:"
```

### Node Lease (Heartbeat)

Since Kubernetes 1.14, Kubelet uses a **Lease object** in the `kube-node-lease` namespace as a lightweight heartbeat mechanism:

```bash
# View node leases
kubectl get lease -n kube-node-lease

# Inspect a specific lease
kubectl get lease worker-1 -n kube-node-lease -o yaml
```

```yaml
# Example Lease object:
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: worker-1
  namespace: kube-node-lease
spec:
  holderIdentity: worker-1
  leaseDurationSeconds: 40
  renewTime: "2024-01-15T10:30:45.000000Z"
```

**Why Leases instead of Node status updates?**

| Method | API Server Load | Update Frequency | Data Size |
|---|---|---|---|
| Node status update | High (full status) | Every 10s | ~3-5 KB |
| Lease renewal | Very low (single timestamp) | Every 10s | ~200 bytes |

Leases reduce etcd write load significantly in large clusters. The node controller uses lease age to determine if a node is dead (default: 40s without renewal → node is considered unhealthy).

---

## 10. Resource Management: CPU, Memory & QoS

The Kubelet is responsible for enforcing resource requests and limits via Linux kernel mechanisms.

### How Resource Limits are Enforced

| Resource | Mechanism | Effect of Exceeding |
|---|---|---|
| CPU Request | CFS (Completely Fair Scheduler) weight | Guaranteed minimum CPU share |
| CPU Limit | CFS quota (`cpu.cfs_quota_us`) | CPU throttling (not killed) |
| Memory Request | Used for scheduling decisions | No direct kernel enforcement |
| Memory Limit | cgroup memory limit | OOMKill (container killed) |
| Ephemeral Storage | Kubelet policing | Pod eviction |

### Quality of Service (QoS) Classes

The Kubelet assigns QoS classes to Pods based on their resource specifications:

| QoS Class | Condition | Eviction Priority |
|---|---|---|
| **Guaranteed** | All containers have `requests == limits` for CPU and memory | Last to be evicted |
| **Burstable** | At least one container has CPU or memory requests set | Evicted based on usage vs request |
| **BestEffort** | No container has CPU or memory requests or limits set | First to be evicted |

```yaml
# Example: Guaranteed QoS
resources:
  requests:
    memory: "128Mi"
    cpu: "500m"
  limits:
    memory: "128Mi"   # Must equal requests
    cpu: "500m"       # Must equal requests
```

```bash
# Check QoS class of a running Pod
kubectl get pod my-pod -o jsonpath='{.status.qosClass}'
```

### cgroup Hierarchy

The Kubelet organizes resources using cgroup v2 (or v1 on older nodes):

```
/sys/fs/cgroup/
└── kubepods/
    ├── besteffort/
    │   └── pod-<uid>/
    │       └── container-<id>/
    ├── burstable/
    │   └── pod-<uid>/
    │       └── container-<id>/
    └── guaranteed/
        └── pod-<uid>/
            └── container-<id>/
```

---

## 11. Volume Management & CSI Integration

The Kubelet's Volume Manager is responsible for the complete lifecycle of storage for each Pod.

### Volume Lifecycle Phases

```
1. PROVISION  ─ (Done by external provisioner or cloud provider)
2. ATTACH     ─ Attach block device to node (CSI ControllerPublish)
3. MOUNT      ─ Mount the device into the Pod's filesystem (CSI NodePublish)
                 └─ Also: Mount ConfigMaps, Secrets, Projected volumes
4. UNMOUNT    ─ Unmount when Pod terminates (CSI NodeUnpublish)
5. DETACH     ─ Detach block device from node (CSI ControllerUnpublish)
6. DEPROVISION─ (Done by external provisioner based on reclaim policy)
```

### CSI Plugin Communication

```
Kubelet Volume Manager
        │
        │ gRPC
        ▼
CSI Node Plugin (DaemonSet on each node)
        │
        ├─ NodeStageVolume()    ← Mount to staging path
        ├─ NodePublishVolume()  ← Bind mount to Pod path
        ├─ NodeUnpublishVolume()
        └─ NodeUnstageVolume()
```

### Volume Mount Path Structure

```
/var/lib/kubelet/pods/<pod-uid>/
├── volumes/
│   ├── kubernetes.io~configmap/
│   ├── kubernetes.io~secret/
│   ├── kubernetes.io~projected/
│   └── kubernetes.io~csi/<pvc-name>/
├── plugins/
└── containers/
```

### Common Volume Types and Their Handlers

| Volume Type | Plugin Handler | Notes |
|---|---|---|
| `configMap` | Built-in | Mounted as files, auto-updated |
| `secret` | Built-in | Mounted as files (tmpfs), auto-updated |
| `emptyDir` | Built-in | Created on node, deleted with Pod |
| `hostPath` | Built-in | Direct node filesystem access |
| `persistentVolumeClaim` | CSI | Delegates to CSI driver |
| `projected` | Built-in | Combines multiple sources |
| `downwardAPI` | Built-in | Exposes Pod/Node metadata as files |

---

## 12. Health Probes: Liveness, Readiness & Startup

The Kubelet's **Probe Manager** runs health checks against containers and takes action based on results.

### Probe Types Comparison

| Probe | Purpose | Action on Failure | Default |
|---|---|---|---|
| **Liveness** | Is the container alive? | Restart the container | No probe |
| **Readiness** | Is the container ready to serve traffic? | Remove from Endpoints (no kill) | No probe |
| **Startup** | Has the container finished starting? | Restart (blocks liveness/readiness) | No probe |

### Probe Mechanisms

| Mechanism | Description | Example Use Case |
|---|---|---|
| `httpGet` | HTTP GET to endpoint, success if 200-399 | REST API health endpoint |
| `tcpSocket` | TCP connection check | Database port check |
| `exec` | Run command in container, success if exit 0 | Script-based check |
| `grpc` | gRPC health check protocol | gRPC services |

### Probe Configuration Parameters

| Parameter | Default | Description |
|---|---|---|
| `initialDelaySeconds` | 0 | Wait before first probe |
| `periodSeconds` | 10 | How often to probe |
| `timeoutSeconds` | 1 | Probe timeout |
| `successThreshold` | 1 | Consecutive successes to pass |
| `failureThreshold` | 3 | Consecutive failures to fail |

```yaml
# Production-grade probe configuration example:
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  successThreshold: 1
  failureThreshold: 3

startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30    # 30 * 10s = 300s max startup time
  periodSeconds: 10
```

### Probe Execution by Kubelet

```bash
# Kubelet executes probes inside the container namespace:
# For exec probe:
nsenter --target <container-pid> --mount --uts --ipc --net --pid -- <command>

# The probe result is logged in Kubelet's logs:
journalctl -u kubelet | grep "Liveness probe"
```

---

## 13. Garbage Collection

The Kubelet performs garbage collection to reclaim disk space consumed by unused container images and terminated containers.

### Image Garbage Collection

The Kubelet's Image Manager runs GC based on disk usage thresholds:

| Flag | Default | Description |
|---|---|---|
| `imageGCHighThresholdPercent` | 85 | Start GC when disk usage hits this % |
| `imageGCLowThresholdPercent` | 80 | GC until disk usage drops to this % |
| `imageMinimumGCAge` | 2m | Minimum age of unused image before GC |

**GC Order**: Oldest unused images are deleted first (LRU — Least Recently Used).

```bash
# Images in use are NEVER deleted. Check image usage:
crictl images
crictl ps -a  # All containers (running + stopped)
```

### Container Garbage Collection

Terminated containers are also GC'd:

| Flag | Default | Description |
|---|---|---|
| `minimumContainerTTL` | 0 | Min time before GC of stopped container |
| `maximumDeadContainersPerContainer` | 1 | Max dead containers kept per Pod container |
| `maximumDeadContainers` | -1 | Max total dead containers on node (-1 = unlimited) |

---

## 14. Eviction Manager & Node Pressure

When a node is running low on resources, the Kubelet's **Eviction Manager** evicts Pods to reclaim resources and prevent the node from becoming completely unstable.

### Eviction Signals

| Signal | Description | Eviction Triggered |
|---|---|---|
| `memory.available` | Available memory on node | When below threshold |
| `nodefs.available` | Available disk on root FS | When below threshold |
| `nodefs.inodesFree` | Free inodes on root FS | When below threshold |
| `imagefs.available` | Disk for container images | When below threshold |
| `pid.available` | Available process IDs | When below threshold |

### Eviction Thresholds

```yaml
# /var/lib/kubelet/config.yaml
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
  nodefs.inodesFree: "5%"
  imagefs.available: "15%"

evictionSoft:
  memory.available: "200Mi"
  nodefs.available: "15%"

evictionSoftGracePeriod:
  memory.available: "90s"
  nodefs.available: "2m"
```

### Eviction Types

| Type | Behavior | Configuration |
|---|---|---|
| **Hard Eviction** | Immediate, no grace period | `evictionHard` |
| **Soft Eviction** | Waits for grace period before evicting | `evictionSoft` + `evictionSoftGracePeriod` |

### Pod Eviction Order

When eviction occurs, the Kubelet evicts Pods in this order:

```
1. BestEffort Pods (no resource requests)
2. Burstable Pods exceeding their requests (highest usage first)
3. Guaranteed Pods (only if nothing else can be evicted)
```

```bash
# Check eviction events
kubectl get events --field-selector reason=Evicted --all-namespaces

# Check node pressure conditions
kubectl describe node worker-1 | grep -E "MemoryPressure|DiskPressure|PIDPressure"
```

---

## 15. Interaction with API Server & etcd

This section is critical for understanding the Kubernetes architecture correctly.

### The Golden Rule

> **The Kubelet NEVER talks directly to etcd. Ever.**

All Kubelet communication with cluster state goes through the kube-apiserver. This is a non-negotiable architectural boundary.

```
CORRECT:  Kubelet ────HTTPS────► kube-apiserver ────► etcd
INCORRECT: Kubelet ─────────────────────────────────► etcd  ✗
```

### Why This Architecture Matters

| Concern | How API Server Solves It |
|---|---|
| **Authentication** | API server validates Kubelet's TLS certificate |
| **Authorization** | Node Authorizer limits what each Kubelet can access |
| **Validation** | API server validates all object writes |
| **Audit Logging** | API server logs all Kubelet API calls |
| **Rate Limiting** | API server enforces rate limits |
| **etcd Protection** | etcd is only exposed to the API server |

### Kubelet API Server Interactions

| Operation | HTTP Method | API Path | Frequency |
|---|---|---|---|
| Watch Pods on this node | GET (watch) | `/api/v1/pods?fieldSelector=spec.nodeName=<node>` | Persistent |
| Update Pod status | PATCH | `/api/v1/namespaces/<ns>/pods/<name>/status` | Per status change |
| Update Node status | PATCH | `/api/v1/nodes/<name>/status` | Every 10s |
| Renew Node Lease | PUT | `/apis/coordination.k8s.io/v1/namespaces/kube-node-lease/leases/<node>` | Every 10s |
| Get ConfigMap | GET | `/api/v1/namespaces/<ns>/configmaps/<name>` | When mounting |
| Get Secret | GET | `/api/v1/namespaces/<ns>/secrets/<name>` | When mounting |
| Create Event | POST | `/api/v1/namespaces/<ns>/events` | On significant events |

### Node Authorizer

The **Node Authorizer** is a special authorization mode that limits Kubelet permissions using the principle of least privilege:

```
Each Kubelet can only:
  ✅ Read Pods assigned to its own node
  ✅ Read Secrets/ConfigMaps referenced by Pods on its node
  ✅ Update status of its own node
  ✅ Update status of Pods on its node
  ✅ Write events for its own node
  ❌ Read Pods on other nodes
  ❌ Read Secrets not referenced by its Pods
  ❌ Modify other nodes
```

This is enforced by the `--authorization-mode=Node,RBAC` flag on the API server.

### Kubelet kubeconfig

```bash
# Kubelet's kubeconfig (credentials to talk to API server):
cat /etc/kubernetes/kubelet.conf

# Contains:
# - API server URL
# - CA certificate (to verify API server)
# - Client certificate + key (Kubelet's identity)
```

---

## 16. Leader Election (N/A for Kubelet) vs. Other Control Plane Components

### Why Kubelet Does NOT Need Leader Election

The Kubelet is a **per-node agent** — there is exactly one Kubelet per node by design. Leader election is only needed when multiple instances of a component might run simultaneously and conflict on shared state.

| Component | Runs | Leader Election | Reason |
|---|---|---|---|
| **kubelet** | One per node | ❌ No | Each manages only its own node |
| **kube-controller-manager** | Multiple (HA) | ✅ Yes | All instances watch same objects |
| **kube-scheduler** | Multiple (HA) | ✅ Yes | Only one should schedule each Pod |
| **etcd** | Multiple | ✅ Yes (Raft) | Consensus protocol |
| **kube-apiserver** | Multiple (HA) | ❌ No | Stateless, load-balanced |

### How Leader Election Works (for kube-controller-manager reference)

Since Kubelet doesn't use it, this section provides context for the full control plane picture:

```yaml
# Leader Election uses a Lease object:
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  holderIdentity: controller-manager-pod-xyz   # Current leader
  leaseDurationSeconds: 15                      # Lease expires after 15s without renewal
  acquireTime: "2024-01-15T10:00:00.000000Z"
  renewTime: "2024-01-15T10:30:45.000000Z"     # Leader must renew before expiry
  leaderTransitions: 5                          # How many times leadership changed
```

**Leader Election Process:**
1. All candidates attempt to acquire the Lease by writing their identity
2. First writer wins (atomic write via etcd)
3. Leader continuously renews the lease (every `renewDeadline` seconds)
4. If leader fails to renew within `leaseDuration`, others race to acquire
5. New leader is elected

**Important Flags:**

| Flag | Default | Description |
|---|---|---|
| `--leader-elect` | `true` | Enable leader election |
| `--leader-elect-lease-duration` | `15s` | How long non-leader waits before acquiring |
| `--leader-elect-renew-deadline` | `10s` | How often leader renews the lease |
| `--leader-elect-retry-period` | `2s` | How often candidates retry acquiring |

**HA Best Practices for leader-elected components:**
- Run 3 replicas of kube-controller-manager and kube-scheduler
- Ensure `renewDeadline < leaseDuration` to prevent double-leader scenarios
- Monitor `leader_election_master_status` Prometheus metric

---

## 17. Security Hardening Practices

### TLS Configuration

The Kubelet serves its API over HTTPS with mutual TLS.

```yaml
# /var/lib/kubelet/config.yaml — TLS configuration
tlsCertFile: /var/lib/kubelet/pki/kubelet.crt
tlsPrivateKeyFile: /var/lib/kubelet/pki/kubelet.key
tlsMinVersion: VersionTLS12
tlsCipherSuites:
  - TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
  - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
  - TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
  - TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
```

### Node TLS Bootstrapping

```bash
# Kubelet uses bootstrap token to get a certificate:
# 1. Kubelet starts with bootstrap kubeconfig (token auth)
# 2. Sends CSR (CertificateSigningRequest) to API server
# 3. controller-manager or admin approves CSR
# 4. Kubelet gets signed certificate and uses it henceforth

# Check pending CSRs:
kubectl get csr

# Approve a CSR:
kubectl certificate approve <csr-name>

# Enable auto-approval (for production automation):
kubectl create clusterrolebinding node-client-auto-approve \
  --clusterrole=system:certificates.k8s.io:certificatesigningrequests:nodeclient \
  --group=system:bootstrappers
```

### Disable Anonymous Authentication

```yaml
# /var/lib/kubelet/config.yaml
authentication:
  anonymous:
    enabled: false          # Disable anonymous access
  webhook:
    enabled: true           # Enable webhook auth (via API server)
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt  # CA for client certs
authorization:
  mode: Webhook             # Delegate authorization to API server
```

### Disable Read-Only Port

```yaml
# /var/lib/kubelet/config.yaml
readOnlyPort: 0             # Disable port 10255 (unauthenticated)
```

### Enable Seccomp and AppArmor

```yaml
# /var/lib/kubelet/config.yaml
seccompDefault: true        # Apply RuntimeDefault seccomp to all Pods
```

```yaml
# Pod-level security:
spec:
  securityContext:
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 1000
      capabilities:
        drop:
          - ALL
```

### RBAC for Kubelet API Access

```bash
# The system:node ClusterRole grants proper permissions:
kubectl describe clusterrole system:node

# Kubelets are identified by certificate CN: system:node:<nodename>
# And group: system:nodes

# Node Authorizer enforces that each Kubelet only accesses its own resources
# This is enabled with:
# kube-apiserver --authorization-mode=Node,RBAC
```

### Service Account Token Security

```yaml
# /var/lib/kubelet/config.yaml
# Service account token projection (bound tokens — preferred):
serviceAccountTokenAudience: https://kubernetes.default.svc  # Audience bound
# Tokens expire after X seconds and are auto-rotated by Kubelet:
# Default expiration: 3607 seconds (~1 hour)
```

### NodeRestriction Admission Plugin

```
# Enabled on kube-apiserver:
--enable-admission-plugins=NodeRestriction

# Effect: A Kubelet cannot modify node or pod objects
# for nodes/pods that are not its own — even with valid credentials
```

### Security Checklist

| Practice | Config | Status |
|---|---|---|
| Disable anonymous auth | `authentication.anonymous.enabled: false` | Must enable |
| Disable read-only port | `readOnlyPort: 0` | Must enable |
| Use Webhook auth | `authorization.mode: Webhook` | Must enable |
| TLS minimum v1.2 | `tlsMinVersion: VersionTLS12` | Must enable |
| Certificate rotation | `rotateCertificates: true` | Must enable |
| Protect kernel defaults | `protectKernelDefaults: true` | Must enable |
| Enable seccomp default | `seccompDefault: true` | Recommended |
| Event recording | `eventRecordQPS: 50` | Tune |

---

## 18. Performance Tuning & Flags

### Key KubeletConfiguration Parameters

```yaml
# /var/lib/kubelet/config.yaml — Production tuning
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

# Pod limits
maxPods: 110                      # Max Pods per node (default: 110)
podsPerCore: 0                    # Pods per CPU core (0 = use maxPods)

# Sync frequencies
syncFrequency: 1m                 # Full sync interval
fileCheckFrequency: 20s           # Static pod check interval
httpCheckFrequency: 20s           # HTTP pod source check interval
nodeStatusUpdateFrequency: 10s    # Node status update interval
nodeStatusReportFrequency: 5m     # Full node status report interval

# Garbage Collection
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m

# Eviction
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
  imagefs.available: "15%"

# Resource reservation (prevents node from overcommitting OS resources)
systemReserved:
  cpu: "200m"
  memory: "200Mi"
  ephemeral-storage: "1Gi"
kubeReserved:
  cpu: "100m"
  memory: "100Mi"
  ephemeral-storage: "1Gi"
enforceNodeAllocatable:
  - pods
  - system-reserved
  - kube-reserved

# API Server connection
kubeAPIQPS: 50                    # API server QPS limit
kubeAPIBurst: 100                 # API server burst limit

# Logging
logging:
  verbosity: 2                    # Log level (0-10, 2=info, 5=debug)
```

### CPU & Memory Reservation Math

```
Node Allocatable = Node Capacity - kube-reserved - system-reserved - eviction-threshold

Example:
  Node capacity:     CPU=4, Memory=8Gi
  kube-reserved:     CPU=100m, Memory=100Mi
  system-reserved:   CPU=200m, Memory=200Mi
  eviction-threshold: Memory=100Mi

  Allocatable CPU    = 4000m - 100m - 200m = 3700m
  Allocatable Memory = 8Gi - 100Mi - 200Mi - 100Mi = 7.6Gi
```

```bash
# Verify allocatable resources:
kubectl describe node worker-1 | grep -A 10 "Allocatable:"
```

### High Pod Density Tuning

For nodes running many pods (100+):

```yaml
# Increase API server connection limits
kubeAPIQPS: 100
kubeAPIBurst: 200

# Tune PLEG (if using high pod counts)
# Default PLEG relist period is 1s — acceptable for most workloads

# Increase max pods if hardware supports it
maxPods: 250   # Requires networking plugin support
```

### CPU Manager (CPU Pinning)

For latency-sensitive workloads (NFV, ML inference):

```yaml
cpuManagerPolicy: static    # Pin containers to dedicated CPUs
cpuManagerReconcilePeriod: 10s

# Pod must be Guaranteed QoS to benefit:
resources:
  requests:
    cpu: "2"      # Integer CPU required for static policy
  limits:
    cpu: "2"
```

### Memory Manager (NUMA-aware)

```yaml
memoryManagerPolicy: Static
reservedMemory:
  - numaNode: 0
    limits:
      memory: 1100Mi
```

### Topology Manager

```yaml
topologyManagerPolicy: best-effort  # or: none, restricted, single-numa-node
topologyManagerScope: container     # or: pod
```

---

## 19. Monitoring & Observability

### Kubelet Metrics Endpoint

The Kubelet exposes Prometheus metrics on port **10250** (authenticated):

```bash
# Scrape metrics (with auth):
curl -sk --cert /var/lib/kubelet/pki/kubelet.crt \
         --key /var/lib/kubelet/pki/kubelet.key \
         https://localhost:10250/metrics

# From inside cluster (with ServiceAccount token):
curl -sk -H "Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  https://10.0.0.1:10250/metrics
```

### Key Prometheus Metrics

**Pod & Container Metrics:**

| Metric | Type | Description |
|---|---|---|
| `kubelet_running_pods` | Gauge | Number of pods currently running |
| `kubelet_running_containers` | Gauge | Number of containers currently running |
| `kubelet_pod_start_duration_seconds` | Histogram | Time to start a pod |
| `kubelet_pod_worker_duration_seconds` | Histogram | Time for pod sync work |
| `kubelet_container_log_filesystem_used_bytes` | Gauge | Container log disk usage |

**Probe Metrics:**

| Metric | Type | Description |
|---|---|---|
| `prober_probe_total` | Counter | Total probe attempts by type and result |
| `kubelet_liveness_probe_duration_seconds` | Histogram | Liveness probe duration |

**PLEG Metrics:**

| Metric | Type | Description |
|---|---|---|
| `kubelet_pleg_relist_duration_seconds` | Histogram | PLEG relist duration (critical for health) |
| `kubelet_pleg_relist_interval_seconds` | Histogram | Interval between PLEG relists |

**Volume & Storage Metrics:**

| Metric | Type | Description |
|---|---|---|
| `kubelet_volume_stats_capacity_bytes` | Gauge | Volume capacity |
| `kubelet_volume_stats_used_bytes` | Gauge | Volume used bytes |
| `kubelet_volume_stats_available_bytes` | Gauge | Volume available bytes |

**Node Metrics (via cAdvisor):**

| Metric | Type | Description |
|---|---|---|
| `container_cpu_usage_seconds_total` | Counter | CPU usage per container |
| `container_memory_usage_bytes` | Gauge | Memory usage per container |
| `container_network_receive_bytes_total` | Counter | Network receive bytes |
| `container_network_transmit_bytes_total` | Counter | Network transmit bytes |
| `container_fs_usage_bytes` | Gauge | Filesystem usage per container |

**Garbage Collection Metrics:**

| Metric | Type | Description |
|---|---|---|
| `kubelet_image_garbage_collector_age_hours` | Histogram | Age of GC'd images |
| `kubelet_evictions_total` | Counter | Number of pod evictions |

### Critical Alert Rules

```yaml
# Prometheus alerting rules for Kubelet:
groups:
- name: kubelet.rules
  rules:
  - alert: KubeletDown
    expr: absent(up{job="kubelet"} == 1)
    for: 5m
    annotations:
      summary: "Kubelet is down on {{ $labels.node }}"

  - alert: KubeletPLEGUnhealthy
    expr: kubelet_pleg_relist_duration_seconds{quantile="0.99"} >= 10
    for: 5m
    annotations:
      summary: "PLEG relist duration too high: {{ $value }}s"

  - alert: KubeletTooManyPods
    expr: max by(node) (kubelet_running_pods) / max by(node) (kube_node_status_capacity{resource="pods"}) > 0.95
    for: 15m
    annotations:
      summary: "Node {{ $labels.node }} near pod capacity"

  - alert: NodeNotReady
    expr: kube_node_status_condition{condition="Ready", status="true"} == 0
    for: 2m
    annotations:
      summary: "Node {{ $labels.node }} is NotReady"
```

### Observability Stack Integration

```yaml
# Prometheus scrape config for Kubelet:
scrape_configs:
  - job_name: "kubelet"
    kubernetes_sd_configs:
      - role: node
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        regex: (.+)
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics
```

---

## 20. Troubleshooting Guide with Real kubectl Commands

### Scenario 1: Node NotReady

```bash
# Step 1: Check node status
kubectl get nodes
kubectl describe node <node-name>

# Step 2: Look at node conditions
kubectl get node <node-name> -o jsonpath='{.status.conditions[*].message}'

# Step 3: SSH to node and check Kubelet service
systemctl status kubelet
journalctl -u kubelet -n 100 --no-pager

# Step 4: Check if Kubelet can reach API server
curl -sk https://10.96.0.1:443/healthz  # From the node

# Step 5: Check container runtime
systemctl status containerd
crictl info
crictl ps

# Step 6: Check PLEG
journalctl -u kubelet | grep -i "pleg"

# Step 7: Check disk/memory pressure
df -h
free -h
kubectl describe node <node-name> | grep -E "Pressure|Conditions" -A 20
```

### Scenario 2: Pod Stuck in Pending

```bash
# Step 1: Describe the Pod
kubectl describe pod <pod-name> -n <namespace>
# Look at: Events section, Conditions section

# Step 2: Check if Pod was scheduled
kubectl get pod <pod-name> -o jsonpath='{.spec.nodeName}'

# Step 3: If not scheduled — scheduler issue:
kubectl get events -n <namespace> --field-selector involvedObject.name=<pod-name>

# Step 4: If scheduled but not running — Kubelet issue:
# SSH to the target node and check:
journalctl -u kubelet | grep <pod-name>

# Step 5: Check image pull issues
kubectl describe pod <pod-name> | grep -A 5 "Events:"
# Common: ErrImagePull, ImagePullBackOff

# Step 6: Check resource availability
kubectl describe node <node-name> | grep -A 10 "Allocated resources:"

# Step 7: Check PVC binding (if applicable)
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>
```

### Scenario 3: Deployment Stuck (No New Pods)

```bash
# Step 1: Check rollout status
kubectl rollout status deployment/<name> -n <namespace>

# Step 2: Check ReplicaSet
kubectl get rs -n <namespace>
kubectl describe rs <rs-name> -n <namespace>

# Step 3: Check events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Step 4: Check if old pods are terminating
kubectl get pods -n <namespace>
# Stuck in Terminating? Could be finalizers or volume unmount issues

# Step 5: Force delete stuck terminating pod (LAST RESORT):
kubectl delete pod <pod-name> --grace-period=0 --force -n <namespace>

# Step 6: Check resource quotas
kubectl describe resourcequota -n <namespace>

# Step 7: Check LimitRange
kubectl describe limitrange -n <namespace>
```

### Scenario 4: CrashLoopBackOff

```bash
# Step 1: Check current and previous logs
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> --previous

# Step 2: Check restart count and last state
kubectl describe pod <pod-name> -n <namespace> | grep -A 20 "Last State:"

# Step 3: Exec into container if it starts briefly
kubectl exec -it <pod-name> -- /bin/sh

# Step 4: Override entrypoint for debugging
# Edit deployment temporarily:
command: ["sleep", "infinity"]

# Step 5: Check OOMKill (memory limit too low)
kubectl describe pod <pod-name> | grep -i "OOMKilled"
dmesg | grep -i "oom" | tail -20  # On the node

# Step 6: Check liveness probe causing kills
kubectl describe pod <pod-name> | grep -A 5 "Liveness"
```

### Scenario 5: Container Runtime Issues

```bash
# Check containerd status on node:
systemctl status containerd
journalctl -u containerd -n 50 --no-pager

# Check CRI connectivity:
crictl info
crictl version

# List all containers (including stopped):
crictl ps -a

# List all pods from runtime perspective:
crictl pods

# Pull an image manually:
crictl pull nginx:latest

# Check image list:
crictl images

# Inspect a container:
crictl inspect <container-id>
```

### Scenario 6: Volume Mount Issues

```bash
# Pod stuck in ContainerCreating (often a volume issue):
kubectl describe pod <pod-name> | grep -A 20 "Events:"
# Look for: FailedMount, FailedAttachVolume

# Check PVC status:
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>

# Check PV:
kubectl get pv
kubectl describe pv <pv-name>

# Check CSI driver:
kubectl get pods -n kube-system | grep csi
kubectl logs <csi-node-driver-pod> -n kube-system

# On the node, check mounted volumes:
ls /var/lib/kubelet/pods/<pod-uid>/volumes/
mount | grep kubernetes
```

---

## 21. Comparison with kube-apiserver & kube-scheduler

### Feature Comparison

| Feature | kubelet | kube-apiserver | kube-scheduler |
|---|---|---|---|
| **Role** | Node agent, runs containers | Cluster API gateway | Pod-to-node assignment |
| **Runs On** | Every node | Control plane nodes | Control plane nodes |
| **Instances** | 1 per node | 1+ (HA, stateless) | 1+ (HA, leader election) |
| **Leader Election** | Not applicable | Not needed (stateless) | Required |
| **Talks to etcd** | Never | Yes (only component) | Never |
| **Primary Input** | Pod spec from API server | HTTP/HTTPS requests | Unscheduled Pods |
| **Primary Output** | Running containers + status | etcd writes + responses | Pod.spec.nodeName |
| **Port** | 10250 | 6443 | 10259 |
| **State** | Stateless (persists in etcd via API) | Stateless | Stateless |
| **Restart Impact** | Pods stop receiving management | All cluster ops fail | New Pods not scheduled |
| **Auth Method** | Node TLS certs | RBAC, OIDC, certs | Service account |
| **Plugin Interface** | CRI, CNI, CSI | Admission controllers | Scheduler plugins |

### Interaction Flow

```
User/Operator
     │
     │  kubectl apply
     ▼
kube-apiserver   ←── Validates, stores, serves all API requests
     │
     │  Watch for unscheduled pods
     ▼
kube-scheduler   ─── Selects node based on resources/constraints
     │
     │  Write spec.nodeName to Pod
     ▼
kube-apiserver   ←── Store the binding
     │
     │  Kubelet watches for pods on its node
     ▼
kubelet          ─── Actually creates containers, volumes, network
     │
     │  Update Pod/Node status
     ▼
kube-apiserver   ←── Store status updates back to etcd
```

---

## 22. Disaster Recovery Concepts

### Kubelet is Stateless by Design

The Kubelet itself does not store any cluster state. Its "truth" comes from:

1. **The API server** — which Pods should run on this node
2. **The container runtime** — what is actually running
3. **Local disk** — `/var/lib/kubelet/pods/` for checkpoint data only

This means: **restoring the Kubelet is as simple as reinstalling it and pointing it at the API server**. No Kubelet-specific backup is needed.

### What Happens When Kubelet Restarts

```
Kubelet restarts:
  1. Reads kubeconfig (credentials)
  2. Connects to API server
  3. Lists all Pods assigned to this node
  4. Queries container runtime for actual state
  5. Reconciles — starts missing containers, stops orphans
  6. No data loss (containers continue running during restart)
```

### Pod Checkpointing (Kubernetes 1.25+)

The Kubelet supports **Pod checkpointing** to enable recovery even when the API server is unavailable:

```yaml
# /var/lib/kubelet/config.yaml
featureGates:
  ContainerCheckpoint: true
```

```bash
# Checkpoint a running container:
POST /checkpoint/{namespace}/{pod}/{container}

# Example:
curl -X POST https://localhost:10250/checkpoint/default/my-pod/my-container
```

### Node Recovery Scenarios

| Scenario | Impact | Recovery |
|---|---|---|
| Kubelet process crash | Pods keep running, no new mgmt | Restart kubelet service |
| Node reboot | All pods terminate and restart | Kubelet auto-restarts pods on boot |
| etcd backup restore | State reverts to backup time | Kubelet reconciles with restored state |
| Node disk corruption | Pods may fail to start | Replace node, drain workloads |
| Node total loss | Pods declared "Unknown" | Mark node as deleted, reschedule pods |

### Relation to etcd Backups

```bash
# etcd backup (CRITICAL — this is the true disaster recovery):
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Restore etcd from snapshot:
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored

# After etcd restore, Kubelets will reconcile automatically
# No Kubelet-specific recovery needed
```

### When a Node Goes Down: What the Cluster Does

```
Node stops responding (Kubelet lease not renewed):
  T+0s    : Node stops renewing lease
  T+40s   : API server marks node as unhealthy (lease expired)
  T+40s   : node-controller begins monitoring
  T+5m    : node-controller marks Pods on node as "Unknown"
  T+5m+30s: node-controller deletes Pods (triggers rescheduling)
  T+~6m   : Replacement Pods scheduled on healthy nodes
  
Configurable via kube-controller-manager:
  --node-monitor-grace-period (default: 40s)
  --pod-eviction-timeout (default: 5m — soft eviction)
```

---

## 23. Real-World Production Use Cases

### Use Case 1: Large-Scale Node Fleet Management

**Scenario**: Running 500+ nodes across multiple availability zones.

**Kubelet configurations applied:**
- `systemReserved` and `kubeReserved` set to prevent resource starvation of Kubelet itself
- `evictionHard` tuned per node type (memory-optimized vs compute-optimized)
- Node labels and taints set via `--node-labels` and `--register-with-taints`
- Custom `--image-gc-high-threshold-percent` for nodes with limited disk

### Use Case 2: GPU Workloads (ML/AI)

**Scenario**: Running PyTorch training jobs on GPU nodes.

```yaml
# Device plugin reports GPU resources via Kubelet's device plugin API
# Kubelet exposes: nvidia.com/gpu resource

# GPU Pod configuration:
resources:
  limits:
    nvidia.com/gpu: 2    # Request 2 GPUs

# Kubelet topology manager for NUMA-aware GPU scheduling:
# topologyManagerPolicy: single-numa-node
```

### Use Case 3: Edge Computing

**Scenario**: Kubelet running on resource-constrained edge devices (Raspberry Pi, industrial controllers).

**Optimizations:**
```yaml
maxPods: 20          # Fewer pods on edge node
kubeAPIQPS: 5        # Reduce API server pressure
imageGCHighThresholdPercent: 70   # More aggressive GC (limited disk)
evictionHard:
  memory.available: "50Mi"        # Tighter for constrained memory
```

### Use Case 4: Multi-Tenancy & Workload Isolation

**Scenario**: Hosting multiple teams' workloads on shared node pools.

```yaml
# CPU Manager for dedicated CPUs per team:
cpuManagerPolicy: static

# Topology Manager for NUMA-aware placement:
topologyManagerPolicy: restricted

# Memory Manager for guaranteed memory NUMA locality:
memoryManagerPolicy: Static
```

### Use Case 5: Security-Hardened Regulated Environments

**Scenario**: Running Kubernetes in PCI-DSS / HIPAA compliant environment.

- Anonymous auth disabled
- Read-only port disabled
- Seccomp enabled by default
- AppArmor profiles applied
- Certificate rotation enabled
- NodeRestriction admission enabled
- All Kubelet API calls audited via API server audit log

---

## 24. Best Practices for Production Environments

### Node Configuration

- Always set `systemReserved` and `kubeReserved` to protect OS and Kubernetes daemons
- Use `enforceNodeAllocatable: ["pods", "system-reserved", "kube-reserved"]`
- Set appropriate `maxPods` based on CNI plugin limits and node resources
- Enable `protectKernelDefaults: true` to prevent pods from changing kernel parameters

### Security

- Disable `readOnlyPort` (set to 0)
- Disable anonymous authentication
- Use Webhook authorization (delegates to API server)
- Enable certificate rotation (`rotateCertificates: true`)
- Use Node TLS bootstrapping for automated certificate management
- Keep Kubelet version within N-2 of kube-apiserver version

### Monitoring

- Always monitor `kubelet_pleg_relist_duration_seconds` — high values (>10s) indicate CRI issues
- Alert on `kubelet_running_pods` approaching `maxPods`
- Track `kubelet_pod_start_duration_seconds` for performance regressions
- Monitor node conditions: `MemoryPressure`, `DiskPressure`, `PIDPressure`

### Upgrades

```bash
# Drain node before Kubelet upgrade:
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Upgrade Kubelet:
apt-get install kubelet=1.29.0-00
systemctl daemon-reload
systemctl restart kubelet

# Verify:
kubectl get node <node-name>
kubectl describe node <node-name>

# Uncordon node after verification:
kubectl uncordon <node-name>
```

### Probes

- Always set `startupProbe` for slow-starting applications
- Never set `initialDelaySeconds` as a substitute for `startupProbe`
- Set `readinessProbe` on all production services to prevent traffic to unready pods
- Keep `liveness` and `readiness` probes on separate endpoints with different logic

---

## 25. Common Mistakes and Pitfalls

### Mistake 1: Not Setting Resource Requests and Limits

```yaml
# WRONG — no requests/limits
containers:
- name: app
  image: myapp:latest

# RIGHT — always set both
containers:
- name: app
  image: myapp:latest
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "256Mi"
```

**Consequence**: Without limits, a misbehaving container can consume all node resources and cause evictions of other Pods.

### Mistake 2: Using initialDelaySeconds Instead of startupProbe

```yaml
# WRONG — guessing startup time:
livenessProbe:
  initialDelaySeconds: 120  # Magic number, fragile

# RIGHT — use startupProbe:
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10  # Up to 300s startup time, precisely controlled
```

### Mistake 3: Not Reserving Resources for System and Kubelet

Without `systemReserved` and `kubeReserved`, the Kubelet itself can be OOM-killed when workloads consume all memory.

### Mistake 4: Ignoring PLEG Health

Many teams only monitor Pod-level metrics and miss PLEG degradation. A slow CRI causes PLEG relist to slow, which delays all pod lifecycle events across the node.

### Mistake 5: Disabling Eviction Thresholds

```yaml
# WRONG — setting thresholds too low:
evictionHard:
  memory.available: "1Mi"  # Node will OOM before eviction kicks in!

# RIGHT:
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
```

### Mistake 6: Using hostPath Volumes in Production

```yaml
# AVOID in production:
volumes:
- name: data
  hostPath:
    path: /data

# USE instead (with proper PVC):
volumes:
- name: data
  persistentVolumeClaim:
    claimName: my-pvc
```

**Consequence**: hostPath bypasses Kubelet's volume management, doesn't move with pods, and creates security risks.

### Mistake 7: Running Kubelet as Least-Privileged Without Understanding Implications

The Kubelet requires root or near-root access to:
- Create cgroups for containers
- Mount volumes
- Configure network namespaces
- Write to `/proc` and `/sys` for container isolation

Attempts to restrict Kubelet's permissions break container creation in subtle ways.

### Mistake 8: Forgetting to Drain Before Node Maintenance

```bash
# ALWAYS drain before node maintenance:
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

# NOT just cordon (cordon only prevents new scheduling, doesn't move pods):
kubectl cordon <node>  # This alone is NOT enough for maintenance
```

---

## 26. Hands-On Labs & Mini Practical Exercises

### Lab 1: Inspect Kubelet Configuration

```bash
# Exercise: Understand your Kubelet's current config

# 1. SSH to a node and view Kubelet config:
cat /var/lib/kubelet/config.yaml

# 2. View Kubelet command-line flags:
ps aux | grep kubelet
# OR:
systemctl show kubelet | grep ExecStart

# 3. Check Kubelet health:
curl -sk https://localhost:10250/healthz

# 4. List pods from Kubelet's perspective:
curl -sk --cert /var/lib/kubelet/pki/kubelet.crt \
         --key /var/lib/kubelet/pki/kubelet.key \
         https://localhost:10250/pods | python3 -m json.tool | grep '"name"' | head -20
```

### Lab 2: Observe the Reconciliation Loop

```bash
# Exercise: Watch Kubelet react to pod changes in real-time

# Terminal 1: Watch Kubelet logs
journalctl -u kubelet -f | grep "SyncLoop\|syncPod\|startContainer"

# Terminal 2: Create a pod
kubectl run test-pod --image=nginx

# Observe in Terminal 1:
# - Kubelet detects new pod assignment
# - Kubelet starts container
# - Kubelet updates pod status

# Delete the pod and observe cleanup:
kubectl delete pod test-pod

# Kubelet detects deletion → stops container → cleans up volumes
```

### Lab 3: Trigger an Eviction

```bash
# Exercise: Understand eviction by creating memory pressure

# 1. Check current eviction thresholds:
kubectl describe node <node> | grep -A 10 "Conditions:"

# 2. Deploy a memory-hungry pod:
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: memory-hog
spec:
  containers:
  - name: mem
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "500M"]
    resources:
      requests:
        memory: "10Mi"
      limits:
        memory: "600Mi"
EOF

# 3. Watch the pod consume memory and potentially get evicted:
kubectl get events -w --field-selector reason=Evicted
kubectl describe pod memory-hog | grep -A 5 "OOMKilled"
```

### Lab 4: Explore Static Pods

```bash
# Exercise: Understand static pods (on a kubeadm cluster)

# 1. List static pod manifests:
ls -la /etc/kubernetes/manifests/

# 2. Check that these are running:
kubectl get pods -n kube-system

# 3. Try to delete a static pod (delete mirror pod):
kubectl delete pod kube-apiserver-<node> -n kube-system
# Observe: it immediately comes back (Kubelet re-creates it)

# 4. Create your own static pod:
cat > /etc/kubernetes/manifests/test-static.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-static
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx
EOF

# 5. Check it appears in API server:
kubectl get pod test-static-<node-name>

# 6. Clean up:
rm /etc/kubernetes/manifests/test-static.yaml
```

### Lab 5: Debug a CrashLoopBackOff

```bash
# Exercise: Debug a crashing container

# 1. Deploy a crashing pod:
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: crasher
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo 'Starting...'; sleep 5; exit 1"]
EOF

# 2. Watch it crash and backoff:
kubectl get pod crasher -w

# 3. Check current logs:
kubectl logs crasher

# 4. Check previous container logs:
kubectl logs crasher --previous

# 5. Check restart count:
kubectl get pod crasher -o jsonpath='{.status.containerStatuses[0].restartCount}'

# 6. Check backoff time:
kubectl describe pod crasher | grep -A 5 "Back-off"

# 7. Clean up:
kubectl delete pod crasher
```

### Lab 6: Resource Reservation Verification

```bash
# Exercise: Verify resource reservation on a node

# 1. Check node capacity:
kubectl get node <node> -o jsonpath='{.status.capacity}' | python3 -m json.tool

# 2. Check allocatable (after reservations):
kubectl get node <node> -o jsonpath='{.status.allocatable}' | python3 -m json.tool

# 3. Calculate the difference (reserved resources):
# This shows what systemReserved + kubeReserved + evictionThreshold =

# 4. Check current allocation:
kubectl describe node <node> | grep -A 20 "Allocated resources:"
```

---

## 27. Interview Questions: Beginner to Advanced

### Beginner Level

**Q1: What is the Kubelet and what does it do?**

**A:** The Kubelet is the primary node agent in Kubernetes. It runs on every node (worker and control plane) and is responsible for ensuring that containers described in PodSpecs are running and healthy on its node. It communicates with the API server to get Pod assignments and with the container runtime (via CRI) to start/stop containers. It also reports node and Pod status back to the API server.

---

**Q2: On which port does the Kubelet serve its HTTPS API?**

**A:** Port **10250** for the authenticated HTTPS API. Port 10255 (read-only, unauthenticated) is deprecated and should be disabled in hardened environments. Port 10248 serves the `/healthz` endpoint.

---

**Q3: What is a Static Pod and how is it different from a regular Pod?**

**A:** A Static Pod is defined by a manifest file placed in the Kubelet's static pod directory (typically `/etc/kubernetes/manifests/`). The Kubelet reads these manifests and creates the Pods directly, without involving the API server. A mirror Pod (read-only) appears in the API server for visibility. Unlike regular Pods, static Pods cannot be deleted via `kubectl delete pod` — you must remove the manifest file. Control plane components (kube-apiserver, etcd, scheduler) are commonly run as static Pods.

---

**Q4: What happens if the Kubelet process crashes on a node?**

**A:** Running containers continue running uninterrupted — they are managed by the container runtime, not the Kubelet process itself. However, the Kubelet stops reconciling: new Pod assignments are not processed, health probes stop running, and node status is not updated. The node will be marked NotReady after the lease expires (~40 seconds). When the Kubelet restarts, it reconnects to the API server, queries the container runtime, and reconciles the current state.

---

### Intermediate Level

**Q5: Does the Kubelet communicate directly with etcd?**

**A:** **No.** This is a fundamental architectural principle. The Kubelet only communicates with the kube-apiserver via HTTPS. All reads and writes to cluster state go through the API server, which is the only component that communicates with etcd. This design provides security (API server enforces auth/authz), validation, audit logging, and rate limiting.

---

**Q6: What is PLEG and why is it important?**

**A:** PLEG (Pod Lifecycle Event Generator) is a Kubelet component that polls the container runtime every second to detect changes in container state (started, stopped, crashed). It compares the current state with the previous state and generates lifecycle events. These events drive the Kubelet's sync loop. If the container runtime is slow or unresponsive, PLEG becomes unhealthy (after 3 minutes), which causes the Kubelet to report itself as unhealthy and the node becomes NotReady.

---

**Q7: Explain the difference between Liveness, Readiness, and Startup probes.**

**A:**
- **Liveness Probe**: Checks if the container is alive. Failure causes the Kubelet to restart the container. Used to recover from deadlocks or corrupted state.
- **Readiness Probe**: Checks if the container is ready to serve traffic. Failure removes the Pod from Service Endpoints (no traffic is sent) but does NOT restart the container. Used for graceful startup and temporary unavailability.
- **Startup Probe**: Blocks liveness and readiness probes until it succeeds. Used for slow-starting applications. Failure causes a restart. Once it succeeds, it stops running.

---

**Q8: What are QoS classes in Kubernetes, and how does the Kubelet use them?**

**A:** QoS (Quality of Service) classes are assigned by Kubernetes based on resource requests/limits:
- **Guaranteed**: All containers have `requests == limits`. Never OOM-killed unless limits are exceeded.
- **Burstable**: Some containers have requests set. Evicted based on usage vs requests.
- **BestEffort**: No requests or limits. First to be evicted under pressure.

The Kubelet uses QoS classes to determine eviction order when the node is under resource pressure. BestEffort Pods are evicted first, Guaranteed last.

---

### Advanced Level

**Q9: How does the Kubelet's Informer pattern reduce load on the API server?**

**A:** The Informer pattern uses a two-phase approach: (1) An initial `List` to get all relevant objects, followed by (2) a streaming `Watch` that delivers incremental changes (add/update/delete). All objects are cached in a local in-memory store (Indexer). Subsequent reads are served from the cache, eliminating repeated API calls. Multiple controllers sharing the same Informer only result in one `List+Watch` per resource type. Rate-limited work queues prevent reconciliation storms when many changes arrive simultaneously.

---

**Q10: A node shows NotReady and you suspect a PLEG issue. Walk me through your diagnosis.**

**A:**
```bash
# 1. Check node conditions:
kubectl describe node <node> | grep -A 10 "Conditions:"

# 2. SSH to node, check Kubelet logs:
journalctl -u kubelet -n 200 | grep -i "pleg\|not healthy"
# Look for: "PLEG is not healthy: pleg was last seen active Xs ago"

# 3. Check container runtime:
systemctl status containerd
crictl info
crictl ps  # Can it list containers?

# 4. Check PLEG relist duration metric:
curl -sk https://localhost:10250/metrics | grep pleg_relist_duration

# 5. Common causes:
# - containerd/CRI-O unresponsive (restart it)
# - Too many containers on node (PLEG relist takes too long)
# - Disk I/O saturation causing CRI timeouts

# 6. If CRI is the issue:
systemctl restart containerd
# Kubelet may restart itself or you restart it:
systemctl restart kubelet
```

---

**Q11: How does the CPU Manager in the Kubelet work, and when should you use it?**

**A:** The CPU Manager is a Kubelet feature that provides dedicated CPU core allocation for containers using the `static` policy. When enabled, Guaranteed QoS Pods requesting integer CPU counts are pinned to specific CPUs using cgroups `cpuset.cpus`. Other pods share the remaining CPUs in a shared pool. This eliminates CPU scheduling jitter and cache thrashing — critical for latency-sensitive workloads like telco VNFs, real-time data processing, and ML inference. The `none` policy (default) allows the Linux CFS scheduler to freely allocate CPU across all pods.

---

**Q12: Explain the Node Authorizer and why it matters for security.**

**A:** The Node Authorizer is a special authorization plugin in the kube-apiserver that implements the principle of least privilege for Kubelet access. Each Kubelet is identified by its TLS certificate (CN: `system:node:<nodename>`, Group: `system:nodes`). The Node Authorizer ensures each Kubelet can only:
- Read Pods assigned to its own node
- Read Secrets and ConfigMaps referenced by its own Pods  
- Update status for its own node and its Pods
- Cannot access resources of other nodes

This prevents a compromised Kubelet on one node from reading secrets intended for Pods on other nodes — a critical security boundary in multi-tenant clusters.

---

**Q13: What is the difference between `evictionHard` and `evictionSoft`, and what are the operational trade-offs?**

**A:** `evictionHard` triggers immediate eviction without any grace period the moment a threshold is crossed. `evictionSoft` evicts only after the threshold has been breached continuously for the duration specified in `evictionSoftGracePeriod`.

Trade-offs:
- **Hard eviction** is aggressive — protects the node immediately but may cause unexpected pod terminations. Best for critical thresholds where delay risks node stability.
- **Soft eviction** gives workloads time to shed load naturally (e.g., a traffic spike that'll pass). Better for transient pressure but risks node instability if the pressure doesn't resolve.

In production, use both: soft eviction with generous thresholds for early warning/graceful handling, and hard eviction as a safety net at tighter thresholds.

---

## 28. Cheat Sheet: Commands & Flags

### Essential kubectl Commands

```bash
# Node inspection
kubectl get nodes -o wide
kubectl describe node <node-name>
kubectl get node <node-name> -o yaml
kubectl top nodes

# Check node conditions
kubectl get nodes -o custom-columns=NAME:.metadata.name,STATUS:.status.conditions[-1].type,REASON:.status.conditions[-1].reason

# Check node allocatable vs capacity
kubectl get node <node> -o jsonpath='{.status.allocatable}' | python3 -m json.tool
kubectl get node <node> -o jsonpath='{.status.capacity}' | python3 -m json.tool

# Check node leases (Kubelet heartbeats)
kubectl get lease -n kube-node-lease
kubectl get lease <node-name> -n kube-node-lease -o yaml

# Pod debugging
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous
kubectl exec -it <pod> -n <ns> -- /bin/sh
kubectl get events -n <ns> --sort-by='.lastTimestamp'

# Node maintenance
kubectl cordon <node>
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>

# CSRs (certificate signing requests for new nodes)
kubectl get csr
kubectl certificate approve <csr-name>
kubectl certificate deny <csr-name>
```

### Kubelet Direct API Commands (From Node)

```bash
# Health check
curl -sk https://localhost:10250/healthz

# List pods on this node
curl -sk --cert /var/lib/kubelet/pki/kubelet.crt \
         --key /var/lib/kubelet/pki/kubelet.key \
         https://localhost:10250/pods

# Get metrics
curl -sk --cert /var/lib/kubelet/pki/kubelet.crt \
         --key /var/lib/kubelet/pki/kubelet.key \
         https://localhost:10250/metrics

# Run pod stats
curl -sk --cert /var/lib/kubelet/pki/kubelet.crt \
         --key /var/lib/kubelet/pki/kubelet.key \
         https://localhost:10250/stats/summary
```

### CRI / crictl Commands (On Node)

```bash
# List running pods (runtime view)
crictl pods

# List all containers
crictl ps -a

# List images
crictl images

# Inspect container
crictl inspect <container-id>

# Get container logs
crictl logs <container-id>

# Execute command in container
crictl exec -it <container-id> /bin/sh

# Pull image
crictl pull nginx:latest

# Remove stopped containers
crictl rm <container-id>

# Remove image
crictl rmi <image-id>

# Runtime info
crictl info
crictl version
```

### Systemd / Kubelet Service Commands

```bash
# Check Kubelet status
systemctl status kubelet

# View Kubelet logs (live)
journalctl -u kubelet -f

# View recent Kubelet logs
journalctl -u kubelet -n 100 --no-pager

# View Kubelet logs since restart
journalctl -u kubelet --since "10 minutes ago"

# Restart Kubelet
systemctl restart kubelet

# Reload config and restart
systemctl daemon-reload && systemctl restart kubelet

# Check Kubelet command line
systemctl show kubelet | grep ExecStart
ps aux | grep kubelet | grep -v grep
```

### Key KubeletConfiguration Flags Reference

| Flag | Default | Purpose |
|---|---|---|
| `maxPods` | 110 | Max pods per node |
| `kubeAPIQPS` | 50 | API server QPS rate limit |
| `kubeAPIBurst` | 100 | API server burst limit |
| `nodeStatusUpdateFrequency` | 10s | How often to update node status |
| `imageGCHighThresholdPercent` | 85 | Start image GC at this % disk |
| `imageGCLowThresholdPercent` | 80 | Stop image GC at this % disk |
| `evictionHard` | (see above) | Hard eviction thresholds |
| `evictionSoft` | (see above) | Soft eviction thresholds |
| `cpuManagerPolicy` | none | CPU pinning policy |
| `topologyManagerPolicy` | none | NUMA topology policy |
| `readOnlyPort` | 10255 | Set to 0 to disable |
| `authentication.anonymous.enabled` | true | Set to false to harden |
| `authorization.mode` | AlwaysAllow | Set to Webhook for production |
| `tlsMinVersion` | VersionTLS10 | Set to VersionTLS12 |
| `rotateCertificates` | false | Enable cert rotation |
| `seccompDefault` | false | Enable RuntimeDefault seccomp |
| `protectKernelDefaults` | false | Prevent pods changing kernel params |

---

## 29. Key Takeaways & Summary

### The Essential Mental Model

Think of the Kubelet as the **node's operating system integration layer for Kubernetes**. The control plane decides what should happen; the Kubelet makes it happen on each physical or virtual node.

### The 10 Most Important Things to Know About Kubelet

1. **One per node** — the Kubelet is a per-node agent, not a centralized service. It runs as a systemd service directly on the host OS.

2. **Never talks to etcd directly** — all communication with cluster state flows through the kube-apiserver, which is the only component that reads and writes etcd.

3. **Drives the reconciliation loop** — watch desired state from API server → compare with actual state from CRI → act to close the gap → repeat forever.

4. **PLEG is its heartbeat to the container runtime** — a slow or unresponsive CRI causes PLEG to fail, making the node NotReady even if containers are running fine.

5. **Enforces resource limits via cgroups** — CPU throttling and OOM kills are enforced by Linux kernel cgroups, configured by the Kubelet based on Pod resource specs.

6. **Manages the complete Pod lifecycle** — from image pull and container creation to health probing, volume mounting, and cleanup on deletion.

7. **Runs control plane components as static Pods** — the kube-apiserver, kube-scheduler, and kube-controller-manager are often managed by the Kubelet via manifests in `/etc/kubernetes/manifests/`.

8. **Security is configured at the Kubelet level** — anonymous auth, read-only port, TLS version, and certificate rotation must all be explicitly hardened.

9. **Resource reservation protects the node** — always configure `systemReserved` and `kubeReserved` so the Kubelet and OS don't get OOM-killed by workloads.

10. **Stateless by design** — the Kubelet's state is in etcd (via API server). Recovery from Kubelet failures requires only a service restart, not data restoration.

### Quick Decision Framework

```
Pod not running?
  └─ Was it scheduled?
       ├─ No  → kube-scheduler issue
       └─ Yes → Kubelet issue on the target node
                  └─ Check: journalctl -u kubelet
                            crictl ps -a
                            systemctl status containerd

Node NotReady?
  └─ Is Kubelet running?
       ├─ No  → systemctl start kubelet
       └─ Yes → Is PLEG healthy?
                  ├─ No  → CRI issue (check containerd)
                  └─ Yes → Check network, certificates, API server connectivity

CrashLoopBackOff?
  └─ kubectl logs <pod> --previous
  └─ Check: OOMKilled? → Increase memory limits
            Exit code 1? → Application bug, check app logs
            Exit code 137? → OOM from OS
            Liveness probe? → Extend initialDelaySeconds or use startupProbe
```

---

*This document represents production-grade Kubernetes knowledge compiled for DevOps engineers, SREs, and platform engineers managing Kubernetes at scale. All configurations should be tested in a non-production environment before applying to production clusters.*

---

**Document Version:** 1.0  
**Last Updated:** April 2026  
**Kubernetes Reference Version:** v1.28+

---
*End of Document*
