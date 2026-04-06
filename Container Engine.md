## Table of Contents

1. [Introduction & Strategic Importance](#1-introduction--strategic-importance)
2. [Core Identity Table](#2-core-identity-table)
3. [ASCII Architecture Diagram](#3-ascii-architecture-diagram)
4. [The Container Engine Ecosystem: Layers Explained](#4-the-container-engine-ecosystem-layers-explained)
5. [The Controller Pattern: Watch → Compare → Act → Loop](#5-the-controller-pattern-watch--compare--act--loop)
6. [Deep Dive: Major Built-in Controllers & Container Engine Interactions](#6-deep-dive-major-built-in-controllers--container-engine-interactions)
7. [Container Runtime Interface (CRI): The Critical Bridge](#7-container-runtime-interface-cri-the-critical-bridge)
8. [OCI: Open Container Initiative Standards](#8-oci-open-container-initiative-standards)
9. [containerd: Architecture & Internals](#9-containerd-architecture--internals)
10. [CRI-O: Architecture & Internals](#10-cri-o-architecture--internals)
11. [Docker Engine & cri-dockerd (Legacy Path)](#11-docker-engine--cri-dockerd-legacy-path)
12. [Internal Working Concepts: Reconciliation, Informer Pattern & Work Queues](#12-internal-working-concepts-reconciliation-informer-pattern--work-queues)
13. [Leader Election in the Container Engine Ecosystem](#13-leader-election-in-the-container-engine-ecosystem)
14. [Interaction with API Server and etcd](#14-interaction-with-api-server-and-etcd)
15. [Image Management: Pull, Cache, Garbage Collection](#15-image-management-pull-cache-garbage-collection)
16. [Namespace & cgroup Isolation](#16-namespace--cgroup-isolation)
17. [Security Hardening Practices](#17-security-hardening-practices)
18. [Performance Tuning & Configuration](#18-performance-tuning--configuration)
19. [Monitoring & Observability](#19-monitoring--observability)
20. [Troubleshooting Guide with Real Commands](#20-troubleshooting-guide-with-real-commands)
21. [Comparison: Container Engine vs kube-apiserver vs kube-scheduler](#21-comparison-container-engine-vs-kube-apiserver-vs-kube-scheduler)
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

The **Container Engine** is the foundational software layer that makes containerized workloads physically possible on any Kubernetes node. While Kubernetes orchestrates *what* should run and *where*, the container engine determines *how* containers are actually created, isolated, started, stopped, and managed on the host operating system.

### What Is a Container Engine?

A container engine is a software system that:

- Accepts container image references and runtime configurations
- Pulls and manages container images from registries
- Creates and manages container lifecycle (start, stop, pause, kill)
- Sets up Linux kernel primitives (namespaces, cgroups, seccomp, capabilities) for isolation
- Exposes a standardized API (CRI — Container Runtime Interface) that Kubernetes uses

In the Kubernetes world, the term "container engine" encompasses both the **higher-level runtime** (like containerd or CRI-O) and the **lower-level OCI runtime** (like runc or crun) that actually calls Linux kernel syscalls to spawn processes in isolated environments.

### The Evolution of Container Engines in Kubernetes

```
Pre-2016:     Docker was the only option. Kubernetes called Docker directly.
2016:         Container Runtime Interface (CRI) introduced (Kubernetes 1.5)
              → Abstraction layer allowing any CRI-compliant runtime
2017:         containerd donated to CNCF. CRI-O emerges as lightweight option.
2020:         containerd 1.4 becomes production-grade with full CRI support
2022 (K8s 1.24): Dockershim removed from Kubernetes
              → Docker containers still work, but via cri-dockerd shim
              → containerd and CRI-O become the primary runtimes
2023+:        containerd dominates (used in EKS, GKE, AKS by default)
```

### Why Container Engines Matter in Kubernetes Architecture

Understanding container engines is non-negotiable for production engineers because:

- **Every workload depends on them** — a broken container runtime means no pods can start on that node, regardless of control plane health.
- **They are the security enforcement layer** — seccomp profiles, AppArmor, Linux capabilities, and namespace isolation are all enforced here, not in Kubernetes.
- **Performance characteristics differ** — image pull speed, memory overhead, startup latency, and concurrency limits vary significantly between runtimes.
- **Debugging requires runtime knowledge** — `kubectl` gives you Kubernetes-level visibility; `crictl` and `ctr` give you runtime-level visibility that Kubernetes cannot expose.
- **Node stability depends on runtime health** — a hung containerd shim can cause PLEG failures, making the entire node NotReady.
- **Registry interaction is runtime-handled** — authentication, pull retry logic, and image caching are runtime responsibilities, not Kubernetes responsibilities.

---

## 2. Core Identity Table

### containerd

| Field | Value |
|---|---|
| **Binary Name** | `containerd` |
| **Version Command** | `containerd --version` |
| **gRPC Socket** | `/run/containerd/containerd.sock` |
| **CRI Socket (for Kubelet)** | `/run/containerd/containerd.sock` |
| **Metrics Port** | `localhost:1338` (Prometheus) |
| **Config File** | `/etc/containerd/config.toml` |
| **Service Unit** | `containerd.service` (systemd) |
| **Runs On** | Every Kubernetes worker and control plane node |
| **Runs As** | `root` |
| **OCI Runtime Default** | `runc` |
| **Namespace (internal)** | `k8s.io` (Kubernetes workloads) |
| **Image Store** | `/var/lib/containerd/` |
| **Snapshotter Default** | `overlayfs` |
| **CNCF Project** | Graduated |
| **Used By Default In** | EKS (1.24+), GKE, AKS, kind, k3s |
| **Primary API** | gRPC (CRI) + containerd client API |
| **Log Driver** | via shim → journald or file |
| **Primary Port** | No TCP port; uses Unix socket |

### CRI-O

| Field | Value |
|---|---|
| **Binary Name** | `crio` |
| **Version Command** | `crio --version` |
| **CRI Socket** | `/var/run/crio/crio.sock` |
| **Metrics Port** | `localhost:9090` (Prometheus) |
| **Config File** | `/etc/crio/crio.conf` |
| **Service Unit** | `crio.service` (systemd) |
| **OCI Runtime Default** | `runc` |
| **Used By Default In** | OpenShift (OKD/RHCOS), some on-prem setups |
| **CNCF Project** | Incubating |
| **Design Philosophy** | Kubernetes-only, minimal footprint |

---

## 3. ASCII Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║                    KUBERNETES + CONTAINER ENGINE ARCHITECTURE                    ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║                                                                                  ║
║   ┌─────────────────────── CONTROL PLANE ──────────────────────────────────┐   ║
║   │  kube-apiserver  │  kube-scheduler  │  kube-controller-manager  │ etcd │   ║
║   └──────────────────────────────┬──────────────────────────────────────────┘   ║
║                                  │ HTTPS :6443                                  ║
║   ┌──────────────────────────────▼──────────────────────────────────────────┐   ║
║   │                        WORKER NODE                                       │   ║
║   │                                                                          │   ║
║   │  ┌──────────────────────────────────────────────────────────────────┐   │   ║
║   │  │                         KUBELET :10250                            │   │   ║
║   │  │    PodManager │ VolumeManager │ ProbeManager │ EvictionManager   │   │   ║
║   │  └──────────────────────────┬───────────────────────────────────────┘   │   ║
║   │                             │ CRI gRPC (Unix Socket)                    │   ║
║   │  ┌──────────────────────────▼───────────────────────────────────────┐   │   ║
║   │  │              CONTAINER ENGINE (High-Level Runtime)                │   │   ║
║   │  │                                                                   │   │   ║
║   │  │   containerd  ──OR──  CRI-O  ──OR──  cri-dockerd (legacy)        │   │   ║
║   │  │                                                                   │   │   ║
║   │  │  ┌─────────────────┐   ┌─────────────────┐   ┌───────────────┐  │   │   ║
║   │  │  │  CRI Service    │   │  Image Service  │   │  Snapshotter  │  │   │   ║
║   │  │  │  (gRPC server)  │   │  (Pull/Cache)   │   │  (overlayfs)  │  │   │   ║
║   │  │  └────────┬────────┘   └────────┬────────┘   └───────┬───────┘  │   │   ║
║   │  │           │                     │                     │           │   │   ║
║   │  │  ┌────────▼─────────────────────▼─────────────────────▼───────┐  │   │   ║
║   │  │  │              containerd shim v2  (per container)            │  │   │   ║
║   │  │  │              (containerd-shim-runc-v2)                      │  │   │   ║
║   │  │  └────────────────────────────┬────────────────────────────────┘  │   │   ║
║   │  └───────────────────────────────│───────────────────────────────────┘   │   ║
║   │                                  │ OCI Runtime Spec                      │   ║
║   │  ┌───────────────────────────────▼───────────────────────────────────┐   │   ║
║   │  │                  OCI RUNTIME (Low-Level Runtime)                   │   │   ║
║   │  │         runc  ──OR──  crun  ──OR──  kata-containers               │   │   ║
║   │  └───────────────────────────────┬───────────────────────────────────┘   │   ║
║   │                                  │ syscalls                              │   ║
║   │  ┌───────────────────────────────▼───────────────────────────────────┐   │   ║
║   │  │                       LINUX KERNEL                                 │   │   ║
║   │  │  namespaces │ cgroups │ seccomp │ capabilities │ OverlayFS        │   │   ║
║   │  └───────────────────────────────────────────────────────────────────┘   │   ║
║   │                                                                          │   ║
║   │  ┌──────────────┐  ┌──────────────────┐  ┌──────────────────────────┐   │   ║
║   │  │ CNI Plugin   │  │  CSI Plugin      │  │  Container Registry      │   │   ║
║   │  │ (Networking) │  │  (Storage)       │  │  (Image Pull source)     │   │   ║
║   │  └──────────────┘  └──────────────────┘  └──────────────────────────┘   │   ║
║   └──────────────────────────────────────────────────────────────────────────┘   ║
╚══════════════════════════════════════════════════════════════════════════════════╝

Data Flow:
  API Server → Kubelet (pod assignment via watch)
  Kubelet → Container Engine (CRI gRPC: RunPodSandbox, CreateContainer, StartContainer)
  Container Engine → OCI Runtime (spawn isolated process)
  OCI Runtime → Linux Kernel (namespaces, cgroups, seccomp)
  Container Engine → Registry (pull image layers)
  Container Engine → Snapshotter (overlay filesystem setup)
```

---

## 4. The Container Engine Ecosystem: Layers Explained

One of the most common sources of confusion in Kubernetes is the layering of container-related components. Here is a precise breakdown:

### Layer Model

```
┌──────────────────────────────────────────────────────────┐
│  Layer 0: Orchestration (Kubernetes)                      │
│  Kubernetes API, Scheduler, Controllers                   │
├──────────────────────────────────────────────────────────┤
│  Layer 1: Node Agent (Kubelet)                            │
│  Translates pod specs into CRI calls                      │
├──────────────────────────────────────────────────────────┤
│  Layer 2: High-Level Runtime / Container Engine           │
│  containerd, CRI-O — implements CRI gRPC interface        │
│  Handles: image pull, snapshots, shim management          │
├──────────────────────────────────────────────────────────┤
│  Layer 3: Runtime Shim                                    │
│  containerd-shim-runc-v2                                  │
│  Keeps container alive after parent dies, manages I/O     │
├──────────────────────────────────────────────────────────┤
│  Layer 4: Low-Level OCI Runtime                           │
│  runc, crun, kata-containers, gVisor                      │
│  Creates the isolated process using kernel primitives     │
├──────────────────────────────────────────────────────────┤
│  Layer 5: Linux Kernel                                    │
│  namespaces, cgroups v1/v2, OverlayFS, seccomp, LSMs     │
└──────────────────────────────────────────────────────────┘
```

### Component Responsibility Matrix

| Responsibility | Kubernetes | Kubelet | Container Engine | OCI Runtime | Linux Kernel |
|---|---|---|---|---|---|
| Schedule Pod to node | ✅ | ❌ | ❌ | ❌ | ❌ |
| Translate Pod spec → CRI calls | ❌ | ✅ | ❌ | ❌ | ❌ |
| Pull container images | ❌ | ❌ | ✅ | ❌ | ❌ |
| Manage image cache | ❌ | ❌ | ✅ | ❌ | ❌ |
| Create pod network sandbox | ❌ | ❌ | ✅ (via CNI) | ❌ | ❌ |
| Create container filesystem | ❌ | ❌ | ✅ (snapshotter) | ❌ | ❌ |
| Spawn container process | ❌ | ❌ | ❌ | ✅ | ❌ |
| Apply namespace isolation | ❌ | ❌ | ❌ | ✅ | ✅ |
| Enforce cgroup limits | ❌ | ❌ | ❌ | ✅ | ✅ |
| Apply seccomp/AppArmor | ❌ | ❌ | ❌ | ✅ | ✅ |
| Manage container I/O streams | ❌ | ❌ | ✅ (shim) | ❌ | ❌ |
| Report container status | ❌ | ✅ (via CRI) | ✅ | ❌ | ❌ |

---

## 5. The Controller Pattern: Watch → Compare → Act → Loop

The container engine itself is not a "controller" in the Kubernetes sense, but the **Kubelet** that drives it absolutely follows the controller pattern. Understanding this pattern is essential because it explains *why* container engines are called in the way they are.

### The Reconciliation Control Loop

```
┌──────────────────────────────────────────────────────────────────┐
│                 KUBELET RECONCILIATION LOOP                       │
│                 (drives all Container Engine calls)               │
│                                                                   │
│  ┌──────────┐      ┌──────────────────┐      ┌───────────────┐  │
│  │  WATCH   │─────►│    COMPARE       │─────►│     ACT       │  │
│  │          │      │                  │      │               │  │
│  │ API Srv  │      │ Desired (API)    │      │ Call CRI:     │  │
│  │ for Pods │      │   vs             │      │ RunPodSandbox │  │
│  │ on node  │      │ Actual (CRI)     │      │ CreateContainer│ │
│  └──────────┘      └──────────────────┘      │ StartContainer│  │
│       ▲                                       │ StopContainer │  │
│       └───────────────────────────────────────│ RemoveContainer│ │
│                        LOOP                   └───────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

### Phase 1: WATCH — Desired State from API Server

The Kubelet watches the API server for PodSpecs assigned to its node. This is done via an **Informer** — a `List + Watch` mechanism backed by a local cache.

```go
// Conceptual: What Kubelet watches
podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    kubelet.handlePodAdditions,
    UpdateFunc: kubelet.handlePodUpdates,
    DeleteFunc: kubelet.handlePodDeletions,
})
```

### Phase 2: COMPARE — Desired vs Actual

The Kubelet queries the container engine via CRI to understand the **actual state**:

```bash
# Kubelet internally does the equivalent of:
crictl pods        # What pod sandboxes exist?
crictl ps -a       # What containers exist (running + stopped)?
```

The diff between what the API server says should exist and what CRI reports exists drives all actions.

### Phase 3: ACT — CRI Calls to Container Engine

| Desired State | Actual State | CRI Action |
|---|---|---|
| Pod assigned to node | No pod sandbox | `RunPodSandbox()` → then `CreateContainer()` → `StartContainer()` |
| Pod running | Container crashed | `CreateContainer()` → `StartContainer()` (restart) |
| Pod deleted | Pod running | `StopContainer()` → `RemoveContainer()` → `RemovePodSandbox()` |
| Image not present | Image needed | `PullImage()` |
| Container should exec | — | `ExecSync()` |

### Phase 4: LOOP

This cycle runs continuously. Key timing:

| Event | Trigger | Frequency |
|---|---|---|
| Pod added via Watch | Immediate | Event-driven |
| Full sync (reconcile all pods) | `syncFrequency` | 1 minute (default) |
| PLEG relist (query CRI for all container states) | `plegRelistPeriod` | 1 second |
| Node status update (report to API server) | `nodeStatusUpdateFrequency` | 10 seconds |

---

## 6. Deep Dive: Major Built-in Controllers & Container Engine Interactions

The **kube-controller-manager** runs controllers that define the desired state of Pods. The container engine is ultimately what *realizes* that desired state on the node. Here's how each major controller connects to container engine behavior:

### 6.1 ReplicaSet Controller

**Purpose**: Ensures a specified number of identical Pod replicas are running at all times.

```
ReplicaSet Controller (in kube-controller-manager)
  → Watches ReplicaSet objects and Pod objects
  → Creates/Deletes Pod objects to match .spec.replicas
  → Does NOT talk to container engine directly

Kubelet (on each node)
  → Watches new Pod assignments
  → Calls container engine: RunPodSandbox → CreateContainer → StartContainer
```

**Container Engine Impact**: When a ReplicaSet scales up from 3 to 5, two new Pods are created in the API server. The Kubelet on the assigned nodes detects these, then calls the container engine to pull images (if not cached) and start containers. Image pull time is the dominant latency factor here.

```bash
# Observe container engine activity during ReplicaSet scale:
kubectl scale rs my-replicaset --replicas=5
# On the node:
crictl ps -w   # Watch new containers appear
```

### 6.2 Deployment Controller

**Purpose**: Manages rolling updates, rollbacks, and ReplicaSet lifecycle.

```
Deployment Controller
  → Creates/scales ReplicaSets (old and new)
  → Implements maxSurge and maxUnavailable policies

During rolling update:
  1. New ReplicaSet created (new pod spec)
  2. Kubelet calls container engine: PullImage (new image)
  3. New containers start
  4. readinessProbe passes (Kubelet drives this via CRI exec)
  5. Old ReplicaSet scaled down
  6. Kubelet calls container engine: StopContainer + RemoveContainer
```

**Container Engine Impact**: Rolling update speed is directly affected by image pull time. If the new image is large and not pre-pulled, each node's container engine must pull it fresh, creating a waterfall delay.

```bash
# Monitor image pulls during deployment rollout:
kubectl rollout status deployment/my-app
# On node:
journalctl -u containerd | grep "pulling image"
crictl images  # Check if new image is cached
```

### 6.3 Node Controller

**Purpose**: Monitors node health; evicts pods from unhealthy nodes.

```
Node Controller
  → Watches Node objects
  → Monitors Kubelet heartbeats (via Node Lease)
  → After node goes dark:
      T+40s: Node marked Unknown
      T+5m:  Pods on node marked for deletion
      → New Pods scheduled elsewhere
      → Container engine on NEW node starts the replacement containers
```

**Container Engine Impact**: When a node fails and pods are rescheduled, the container engine on healthy nodes must pull all images needed for the rescheduled pods. This is why image pre-pulling (or using a registry mirror/cache) is critical.

### 6.4 Service Controller (Endpoints)

**Purpose**: Maintains Endpoints objects (IP:Port mappings for Pods backing a Service).

```
EndpointSlice Controller
  → Watches Pods and Services
  → When Pod IP is assigned (by container engine's CNI setup) and
    readinessProbe passes → adds Pod to EndpointSlice

Container Engine role:
  → CNI plugin is invoked by container engine when RunPodSandbox() runs
  → CNI assigns Pod IP and sets up network namespace
  → Only after IP is assigned can the Endpoint be created
```

**Container Engine Impact**: The container engine's invocation of the CNI plugin is the gate that determines when a Pod gets an IP address. Slow CNI calls = delayed endpoint registration = traffic disruption during rollouts.

### 6.5 Namespace Controller

**Purpose**: Manages Namespace lifecycle; ensures cleanup when a namespace is deleted.

```
Namespace Controller
  → When namespace is deleted: marks all resources for deletion
  → Other controllers (ReplicaSet, StatefulSet) delete their Pods
  → Kubelet detects Pod deletions
  → Container engine: StopContainer → RemoveContainer → RemovePodSandbox
  → Container engine: Cleans up volumes, network namespace
```

### 6.6 Job Controller

**Purpose**: Ensures a specified number of Pods run to completion.

```
Job Controller
  → Creates Pods with RestartPolicy: Never or OnFailure
  → Container engine starts container
  → Container exits (success: exit 0, failure: exit != 0)
  → Container engine reports exit code via CRI ContainerStatus
  → Kubelet reports to API server
  → Job Controller reads exit status, decides to create retry or mark complete
```

**Container Engine Impact**: For Jobs, the container engine's ability to accurately report exit codes is critical. Also, `RestartPolicy: Never` means the container engine will NOT restart a crashed container — the Job controller creates a new Pod instead.

### 6.7 StatefulSet Controller

**Purpose**: Manages stateful applications with stable network IDs and persistent storage.

```
StatefulSet Controller
  → Creates Pods with predictable names (pod-0, pod-1, pod-2)
  → Ordered startup: pod-0 must be Running+Ready before pod-1 starts

Container Engine sequence:
  1. pod-0: RunPodSandbox → CNI (get IP) → mount PVC → CreateContainer → Start
  2. Readiness probe passes (Kubelet probes via CRI exec/http)
  3. pod-1: Same sequence starts
```

**Container Engine Impact**: Volume mount failures (CSI issues called during `RunPodSandbox`) will stall the entire StatefulSet because ordered startup blocks on the previous pod being ready. The container engine's volume attachment step is the critical path.

### 6.8 DaemonSet Controller

**Purpose**: Ensures a Pod runs on every (or selected) node.

```
DaemonSet Controller
  → When a new node joins the cluster:
      → Creates a Pod for the DaemonSet on that node
      → Kubelet detects it
      → Container engine starts it

DaemonSets often run infrastructure services:
  → Logging agents (Fluentd, Vector)
  → Monitoring agents (Prometheus Node Exporter, Datadog)
  → Network plugins (Calico, Cilium — these are DaemonSets)
  → Storage plugins (CSI drivers)
```

**Container Engine Impact**: DaemonSet pods on new nodes must start before other workloads are useful. The CNI DaemonSet (which IS a DaemonSet itself) must start and configure networking before any other pod can get an IP. This bootstrap ordering is managed by Kubelet's `--network-plugin` readiness checks and container engine startup order.

### 6.9 Garbage Collector Controller

**Purpose**: Cleans up orphaned resources (Pods owned by deleted ReplicaSets, etc.).

```
Garbage Collector
  → Uses owner references (ownerReferences field)
  → When owner is deleted: deletes all owned objects
  → Pod deletion triggers:
      Kubelet → Container Engine: StopContainer (SIGTERM → SIGKILL after grace period)
                                  RemoveContainer
                                  RemovePodSandbox
                                  (CNI teardown: remove network namespace)
```

**Container Engine Impact**: Container engine's handling of `StopContainer()` determines whether application code runs its shutdown hooks properly. The `terminationGracePeriodSeconds` is translated by the Kubelet into a timeout for the container engine's stop operation.

### 6.10 PersistentVolume Controller

**Purpose**: Binds PersistentVolumeClaims to PersistentVolumes.

```
PV Controller
  → Matches PVCs to available PVs (or triggers dynamic provisioning)
  → Does NOT mount volumes (that's Kubelet + container engine)

Volume Mount Flow:
  1. PV Controller: PVC bound to PV
  2. AD Controller: Attaches block device to node (cloud provider call)
  3. Kubelet Volume Manager: Detects PVC-bound pod
  4. Kubelet calls CSI node plugin: NodeStageVolume, NodePublishVolume
  5. Container Engine: RunPodSandbox includes volume mounts
  6. Container sees the mounted volume at the specified path
```

**Container Engine Impact**: Volume mount paths are injected into the container's filesystem by the container engine during `CreateContainer()`. If the Kubelet has not yet mounted the volume to the host path, `CreateContainer()` will fail.

---

## 7. Container Runtime Interface (CRI): The Critical Bridge

The CRI is the standardized gRPC API that enables Kubernetes (via the Kubelet) to work with any compliant container engine without modification. It was introduced in Kubernetes 1.5 and is the single most important interface in the container ecosystem.

### CRI Protocol Definition

```protobuf
// Simplified CRI proto definition:
service RuntimeService {
  // Sandbox operations (Pod-level network namespace)
  rpc RunPodSandbox(RunPodSandboxRequest) returns (RunPodSandboxResponse);
  rpc StopPodSandbox(StopPodSandboxRequest) returns (StopPodSandboxResponse);
  rpc RemovePodSandbox(RemovePodSandboxRequest) returns (RemovePodSandboxResponse);
  rpc PodSandboxStatus(PodSandboxStatusRequest) returns (PodSandboxStatusResponse);
  rpc ListPodSandbox(ListPodSandboxRequest) returns (ListPodSandboxResponse);

  // Container operations
  rpc CreateContainer(CreateContainerRequest) returns (CreateContainerResponse);
  rpc StartContainer(StartContainerRequest) returns (StartContainerResponse);
  rpc StopContainer(StopContainerRequest) returns (StopContainerResponse);
  rpc RemoveContainer(RemoveContainerRequest) returns (RemoveContainerResponse);
  rpc ListContainers(ListContainersRequest) returns (ListContainersResponse);
  rpc ContainerStatus(ContainerStatusRequest) returns (ContainerStatusResponse);
  rpc UpdateContainerResources(UpdateContainerResourcesRequest) returns (UpdateContainerResourcesResponse);

  // Exec / Attach / Port-forward
  rpc ExecSync(ExecSyncRequest) returns (ExecSyncResponse);
  rpc Exec(ExecRequest) returns (ExecResponse);
  rpc Attach(AttachRequest) returns (AttachResponse);
  rpc PortForward(PortForwardRequest) returns (PortForwardResponse);

  // Stats
  rpc ContainerStats(ContainerStatsRequest) returns (ContainerStatsResponse);
  rpc ListContainerStats(ListContainerStatsRequest) returns (ListContainerStatsResponse);
}

service ImageService {
  rpc ListImages(ListImagesRequest) returns (ListImagesResponse);
  rpc ImageStatus(ImageStatusRequest) returns (ImageStatusResponse);
  rpc PullImage(PullImageRequest) returns (PullImageResponse);
  rpc RemoveImage(RemoveImageRequest) returns (RemoveImageResponse);
  rpc ImageFsInfo(ImageFsInfoRequest) returns (ImageFsInfoResponse);
}
```

### CRI Call Sequence: Pod Creation

```
Kubelet decides to create a Pod
  │
  ├─1─► ImageService.PullImage("nginx:1.25")
  │       └─ Container engine: check local cache, pull from registry if needed
  │
  ├─2─► RuntimeService.RunPodSandbox(PodSandboxConfig)
  │       └─ Container engine: create network namespace, call CNI, get Pod IP
  │       └─ Returns: sandbox_id
  │
  ├─3─► RuntimeService.CreateContainer(sandbox_id, ContainerConfig)
  │       └─ Container engine: create container filesystem via snapshotter
  │       └─ Configure: env vars, mounts, resource limits, security context
  │       └─ Returns: container_id
  │
  └─4─► RuntimeService.StartContainer(container_id)
          └─ Container engine: call OCI runtime (runc) to spawn process
          └─ OCI runtime: unshare namespaces, apply cgroups, exec entrypoint
```

### CRI Socket Configuration

```yaml
# /var/lib/kubelet/config.yaml
containerRuntimeEndpoint: "unix:///run/containerd/containerd.sock"
imageServiceEndpoint: "unix:///run/containerd/containerd.sock"
```

```bash
# Verify CRI connection:
crictl --runtime-endpoint unix:///run/containerd/containerd.sock version
```

### CRI Implementation Comparison

| Feature | containerd | CRI-O | cri-dockerd |
|---|---|---|---|
| **CRI Compliance** | Full | Full | Full |
| **Image Format** | OCI + Docker | OCI + Docker | Docker |
| **Snapshotter** | overlayfs, btrfs, zfs, native | overlayfs | overlayfs |
| **Multi-runtime support** | ✅ (RuntimeClass) | ✅ | ❌ |
| **Memory footprint** | Low | Very low | High (Docker daemon) |
| **Plugin ecosystem** | NRI, containerd plugins | ✅ | Docker plugins |
| **Image build support** | ❌ (not its job) | ❌ | ✅ (via Docker) |
| **Kubernetes-only design** | ❌ (general purpose) | ✅ | ❌ |
| **Production adoption** | Very High | High (OpenShift) | Low (legacy) |
| **Default in major clouds** | EKS, GKE, AKS | OpenShift | — |

---

## 8. OCI: Open Container Initiative Standards

The OCI (Open Container Initiative) defines the open standards that enable container interoperability across all engines and runtimes.

### OCI Specifications

| Specification | What It Defines | File/Format |
|---|---|---|
| **OCI Image Spec** | Container image format (layers, manifest, config) | `manifest.json`, `config.json` |
| **OCI Runtime Spec** | How to run a container (filesystem bundle + config) | `config.json` (runtime) |
| **OCI Distribution Spec** | How to push/pull images from registries | HTTP API protocol |

### OCI Image Structure

```
OCI Image:
├── index.json             ← Entry point, references manifests
├── blobs/
│   └── sha256/
│       ├── <manifest-digest>   ← Image manifest (layer list)
│       ├── <config-digest>     ← Image config (env, cmd, entrypoint)
│       ├── <layer1-digest>     ← Compressed filesystem layer (tar.gz)
│       ├── <layer2-digest>     ← Compressed filesystem layer (tar.gz)
│       └── <layer3-digest>     ← Compressed filesystem layer (tar.gz)
└── oci-layout             ← OCI layout marker file
```

### OCI Runtime Bundle

When the container engine calls the OCI runtime, it provides a **bundle**:

```
/run/containerd/io.containerd.runtime.v2.task/k8s.io/<container-id>/
├── config.json          ← OCI runtime spec (namespaces, cgroups, mounts, hooks)
└── rootfs/              ← Merged overlay filesystem (container root)
```

```json
// Simplified config.json structure:
{
  "ociVersion": "1.0.2",
  "process": {
    "user": {"uid": 1000, "gid": 1000},
    "args": ["/bin/nginx", "-g", "daemon off;"],
    "env": ["PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"],
    "cwd": "/"
  },
  "root": {"path": "rootfs", "readonly": false},
  "mounts": [
    {"destination": "/proc", "type": "proc", "source": "proc"},
    {"destination": "/tmp", "type": "tmpfs", "source": "tmpfs"}
  ],
  "linux": {
    "namespaces": [
      {"type": "pid"},
      {"type": "network", "path": "/var/run/netns/cni-<uuid>"},
      {"type": "ipc"},
      {"type": "uts"},
      {"type": "mount"}
    ],
    "cgroupsPath": "/kubepods/burstable/pod<uid>/<container-id>",
    "resources": {
      "memory": {"limit": 268435456},
      "cpu": {"quota": 50000, "period": 100000}
    },
    "seccomp": {"defaultAction": "SCMP_ACT_ERRNO", "syscalls": [...]}
  }
}
```

### Low-Level OCI Runtime Comparison

| Runtime | Language | Use Case | Notes |
|---|---|---|---|
| **runc** | Go | Default, general purpose | Reference implementation by OCI |
| **crun** | C | Performance-critical | Lower memory, faster startup |
| **kata-containers** | Go/Rust | Strong isolation (VM-based) | Hardware virtualization per pod |
| **gVisor (runsc)** | Go | Sandboxed (syscall interception) | Google-developed |
| **youki** | Rust | Modern alternative to runc | High performance |
| **Firecracker** | Rust | Serverless (AWS Lambda) | MicroVM, very fast boot |

---

## 9. containerd: Architecture & Internals

containerd is the most widely deployed container engine in Kubernetes environments. Originally extracted from Docker, it was donated to the CNCF and graduated in 2019.

### containerd Internal Architecture

```
containerd daemon
├── gRPC Server
│   ├── CRI Plugin (serves Kubelet)         ← CRI gRPC API
│   ├── containerd API (serves ctr CLI)     ← containerd native API
│   └── Introspection API
│
├── Metadata Plugin
│   └── BoltDB (local metadata store)       ← Image, container, snapshot metadata
│
├── Content Store
│   └── /var/lib/containerd/io.containerd.content.v1.content/
│       └── Image layer blobs (raw OCI content)
│
├── Snapshotter (overlayfs by default)
│   └── /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/
│       └── Filesystem layers for each container
│
├── Runtime Plugin
│   └── io.containerd.runtime.v2.task       ← Manages shims
│       └── containerd-shim-runc-v2         ← One shim per container
│
├── Events Plugin
│   └── Publish container lifecycle events
│
└── GC (Garbage Collector)
    └── Removes unused snapshots and content
```

### containerd Namespaces

containerd uses internal namespaces to isolate clients:

```bash
# List containerd namespaces:
ctr namespaces list

# Kubernetes uses the 'k8s.io' namespace:
ctr --namespace k8s.io containers list
ctr --namespace k8s.io images list

# Other namespaces (e.g., for Docker compatibility via moby):
# default    ← used by ctr CLI default
# k8s.io     ← used by Kubelet/CRI
# moby       ← used by Docker daemon
```

### containerd Snapshotter (Overlay Filesystem)

The snapshotter manages the layered filesystem for containers:

```
Image Layers (read-only, shared across containers):
  layer1 (base OS)     → /var/lib/containerd/.../sha256/<digest>/
  layer2 (nginx binary)→ /var/lib/containerd/.../sha256/<digest>/

Container Snapshot (read-write, per container):
  upperdir (container writes) → /var/lib/containerd/.../snapshots/<id>/fs/
  workdir (overlay temp)      → /var/lib/containerd/.../snapshots/<id>/work/

Merged view (what container sees):
  /run/containerd/io.containerd.runtime.v2.task/k8s.io/<id>/rootfs/
  (OverlayFS mount: lowerdir=layer1:layer2, upperdir=container-rw, workdir=work)
```

```bash
# Inspect snapshots:
ctr --namespace k8s.io snapshots list

# Show OverlayFS mount for a running container:
cat /proc/mounts | grep overlay
```

### containerd Shim (containerd-shim-runc-v2)

The shim is one of containerd's most important design decisions:

```
Why a shim?
  ┌─────────────────────────────────────────────────────────┐
  │ Without shim: containerd crash = all containers die     │
  │ With shim:    containerd crash = containers keep running │
  └─────────────────────────────────────────────────────────┘

Shim responsibilities:
  1. Act as parent process for container (inherit container stdio)
  2. Keep container alive if containerd restarts
  3. Reap zombie processes (container's init)
  4. Report exit status back to containerd after reconnection
  5. Implement OCI Runtime API (call runc)

One shim process per container:
  PID 1234: containerd
  PID 2345: containerd-shim-runc-v2 (for container abc123)
  PID 2346: nginx (container abc123, PID 1 inside namespace)
  PID 2347: containerd-shim-runc-v2 (for container def456)
  PID 2348: redis (container def456, PID 1 inside namespace)
```

```bash
# See shim processes:
ps aux | grep containerd-shim
# or:
pstree -p $(pidof containerd) | head -50
```

### containerd Configuration

```toml
# /etc/containerd/config.toml — Production configuration

version = 2

[grpc]
  address = "/run/containerd/containerd.sock"
  max_recv_message_size = 16777216
  max_send_message_size = 16777216

[debug]
  level = "info"   # Use "debug" for troubleshooting only

[metrics]
  address = "127.0.0.1:1338"   # Prometheus metrics endpoint

[plugins."io.containerd.grpc.v1.cri"]
  # Sandbox image (pause container):
  sandbox_image = "registry.k8s.io/pause:3.9"

  [plugins."io.containerd.grpc.v1.cri".containerd]
    snapshotter = "overlayfs"
    default_runtime_name = "runc"

    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
      runtime_type = "io.containerd.runc.v2"

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        SystemdCgroup = true     # CRITICAL: must match kubelet cgroupDriver

    # Additional runtime for kata containers:
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
      runtime_type = "io.containerd.kata.v2"

  [plugins."io.containerd.grpc.v1.cri".registry]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
        endpoint = ["https://my-registry-mirror.example.com"]

  [plugins."io.containerd.grpc.v1.cri".image_decryption]
    key_model = "node"

[plugins."io.containerd.gc.v1.scheduler"]
  pause_threshold = 0.02
  deletion_threshold = 0
  mutation_threshold = 100
  schedule_delay = "0s"
  startup_delay = "100ms"
```

---

## 10. CRI-O: Architecture & Internals

CRI-O was built specifically for Kubernetes — it implements exactly the CRI and nothing more. It is the default runtime in Red Hat OpenShift.

### CRI-O Design Philosophy

```
containerd: General-purpose container engine (can be used outside Kubernetes)
CRI-O:      Kubernetes-only (CRI is its entire purpose)

CRI-O's philosophy:
  → Minimal codebase (fewer features = smaller attack surface)
  → Kubernetes version aligned (CRI-O 1.28 → Kubernetes 1.28)
  → No image building, no compose, no swarm — just CRI
```

### CRI-O Architecture

```
crio daemon
├── CRI gRPC Server (sole purpose)
│   ├── RuntimeService implementation
│   └── ImageService implementation
│
├── Image Management
│   └── Uses: containers/image library (shares with Podman, Skopeo)
│   └── Storage: /var/lib/containers/storage/
│
├── Runtime Management
│   └── Default: runc
│   └── Configurable per runtime class
│
├── CNI Integration
│   └── Calls CNI plugins for network setup
│
└── Metrics (port 9090)
```

### CRI-O Configuration

```toml
# /etc/crio/crio.conf
[crio]
  log_level = "info"
  log_dir = "/var/log/crio/pods"

[crio.api]
  listen = "/var/run/crio/crio.sock"

[crio.runtime]
  default_runtime = "runc"
  cgroup_manager = "systemd"   # Match kubelet cgroupDriver
  seccomp_profile = "/etc/crio/seccomp.json"
  apparmor_profile = "crio-default"
  default_capabilities = [
    "CHOWN", "DAC_OVERRIDE", "FSETID", "FOWNER",
    "SETGID", "SETUID", "SETPCAP", "NET_BIND_SERVICE",
    "KILL"
  ]

  [crio.runtime.runtimes.runc]
    runtime_path = "/usr/bin/runc"
    runtime_type = "oci"
    runtime_root = "/run/runc"

  # Kata containers:
  [crio.runtime.runtimes.kata-runtime]
    runtime_path = "/usr/bin/kata-runtime"
    runtime_type = "vm"

[crio.image]
  default_transport = "docker://"
  pause_image = "registry.k8s.io/pause:3.9"
  pause_command = "/pause"

[crio.network]
  cni_default_network = ""
  network_dir = "/etc/cni/net.d/"
  plugin_dirs = ["/opt/cni/bin/"]
```

---

## 11. Docker Engine & cri-dockerd (Legacy Path)

Since Kubernetes 1.24, Docker Engine is no longer directly supported by Kubernetes (dockershim was removed). However, Docker containers (OCI images) still work perfectly — only the daemon interface changed.

### The Historical Context

```
Pre-K8s 1.24:
  Kubelet → dockershim (built into kubelet) → Docker Engine → containerd → runc

Post-K8s 1.24:
  Kubelet → containerd (CRI native) → runc
  OR
  Kubelet → cri-dockerd (community shim) → Docker Engine → containerd → runc
```

### Why Docker Was Removed

| Issue | Details |
|---|---|
| **Maintenance burden** | Dockershim required K8s team to maintain Docker-specific code |
| **Redundant layers** | Kubelet → dockershim → Docker → containerd → runc (4 hops) |
| **Performance overhead** | Multiple IPC boundaries vs direct CRI to containerd |
| **Feature gap** | Docker didn't expose all CRI features natively |
| **containerd was already inside Docker** | Docker used containerd internally — going directly to containerd eliminates Docker |

### Migration Path

```bash
# Check current runtime:
kubectl get nodes -o wide  # Shows CONTAINER-RUNTIME column
# OR:
crictl info | grep -i runtime

# Verify runtime version:
kubectl get node <node> -o jsonpath='{.status.nodeInfo.containerRuntimeVersion}'
# docker://20.10.x    ← Old (Docker)
# containerd://1.7.x  ← New (containerd) ✅
# cri-o://1.28.x      ← New (CRI-O) ✅
```

---

## 12. Internal Working Concepts: Reconciliation, Informer Pattern & Work Queues

### The Informer Pattern (Container Engine Context)

The Informer pattern is the backbone of how Kubernetes components watch for changes. While this runs in the Kubelet (not the container engine), it is what drives all container engine calls.

```
┌──────────────────────────────────────────────────────────────────────┐
│                    INFORMER INTERNALS (in Kubelet)                    │
│                                                                        │
│  kube-apiserver                                                        │
│       │                                                                │
│       │  1. List (initial full state of pods on this node)            │
│       ▼                                                                │
│  ┌────────────┐                                                        │
│  │  Reflector │◄── 2. Watch (streaming deltas: Add/Update/Delete)     │
│  └─────┬──────┘                                                        │
│         │  Produces events                                             │
│  ┌──────▼──────────┐                                                   │
│  │  Delta FIFO     │  ← Thread-safe queue of change events             │
│  │    Queue        │                                                    │
│  └──────┬──────────┘                                                   │
│          │  Processes events                                            │
│  ┌───────▼───────┐     ┌───────────────────────────────────────┐      │
│  │   Indexer     │     │         Event Handlers                │      │
│  │ (Local Cache) │     │  OnAdd()    → enqueue pod key         │      │
│  │               │     │  OnUpdate() → enqueue pod key         │      │
│  │  Thread-safe  │     │  OnDelete() → enqueue pod key         │      │
│  └───────────────┘     └─────────────────┬─────────────────────┘      │
│                                           │                            │
│                              ┌────────────▼──────────────┐            │
│                              │  Rate-Limited Work Queue  │            │
│                              └────────────┬──────────────┘            │
│                                           │                            │
│                              ┌────────────▼──────────────┐            │
│                              │   syncPod() Worker Loop   │            │
│                              │  ┌─────────────────────┐  │            │
│                              │  │ Compare desired vs   │  │            │
│                              │  │ actual (via CRI)     │  │            │
│                              │  └──────────┬──────────┘  │            │
│                              │             │              │            │
│                              │  ┌──────────▼──────────┐  │            │
│                              │  │ Call CRI if delta    │  │            │
│                              │  │ PullImage()          │  │            │
│                              │  │ RunPodSandbox()      │  │            │
│                              │  │ CreateContainer()    │  │            │
│                              │  │ StartContainer()     │  │            │
│                              │  └─────────────────────-┘  │            │
│                              └───────────────────────────--┘            │
└──────────────────────────────────────────────────────────────────────--┘
```

### Work Queue Behavior

Work queues provide critical properties for reliability:

| Property | Behavior | Benefit |
|---|---|---|
| **Deduplication** | If pod key already in queue, not added again | Prevents redundant CRI calls |
| **Rate limiting** | Limits how fast items are processed | Protects container engine from overload |
| **Exponential backoff** | Failed items re-added with increasing delay | Handles transient CRI failures gracefully |
| **FIFO for unique items** | Ordering within unique keys | Prevents race conditions |

```
Example: Pod "nginx-abc" crashes and is restarted rapidly:
  Event 1: Container died → enqueue "nginx-abc"
  Event 2: Probe failed → try to enqueue "nginx-abc" → DEDUPLICATED (already in queue)
  Event 3: Status update → try to enqueue "nginx-abc" → DEDUPLICATED

  Worker processes "nginx-abc" once → calls CRI CreateContainer + StartContainer
  On failure: re-enqueue with 10s backoff
  On next failure: re-enqueue with 20s backoff (CrashLoopBackOff behavior)
```

### Reconciliation in Container Engine Context

The container engine itself also maintains internal reconciliation:

```
containerd internal reconciliation:
  → GC scheduler: periodically checks for unreferenced content/snapshots
  → Shim monitor: detects if shim processes have died unexpectedly
  → Lease system: prevents GC from deleting content that's being pulled

CRI-O internal reconciliation:
  → Periodic sync of container state with OCI runtime
  → Ensures metadata consistency in storage
```

---

## 13. Leader Election in the Container Engine Ecosystem

### Does the Container Engine Use Leader Election?

**No.** The container engine (containerd, CRI-O) does not use leader election. It is a per-node daemon — exactly one instance per node, by design.

| Component | Leader Election | Reason |
|---|---|---|
| containerd | ❌ No | One per node; manages only local containers |
| CRI-O | ❌ No | One per node; Kubernetes-only scope |
| kubelet | ❌ No | One per node; node-scoped agent |
| kube-controller-manager | ✅ Yes | Multiple instances, shared cluster state |
| kube-scheduler | ✅ Yes | Multiple instances, shared Pod queue |
| etcd | ✅ Yes (Raft) | Distributed consensus |
| kube-apiserver | ❌ No | Stateless, load-balanced |

### Leader Election for Related Components

While the container engine itself does not elect a leader, the **kube-controller-manager** (which drives the state that the container engine ultimately fulfills) does use leader election. Understanding this is important for HA setups.

#### How Lease-Based Leader Election Works

```yaml
# Leader Election Lease object (for kube-controller-manager):
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  holderIdentity: "controller-manager-pod-abc-on-master-1"
  leaseDurationSeconds: 15      # Lease valid for 15s without renewal
  acquireTime: "2024-01-15T09:00:00.000000Z"
  renewTime: "2024-01-15T10:30:45.000000Z"  # Must renew before expiry
  leaderTransitions: 3          # Times leadership changed
```

#### Leader Election Process

```
Step 1: All kube-controller-manager pods start simultaneously
Step 2: Each attempts atomic write to Lease object in API server
        (API server writes to etcd with compare-and-swap)
Step 3: First writer wins → becomes leader
Step 4: Leader runs all controllers (which create Pods)
Step 5: Leader renews Lease every renewDeadline (10s default)
Step 6: Non-leaders watch Lease; if not renewed within leaseDuration (15s):
        → Race to acquire; new leader elected
Step 7: New leader starts processing (brief gap of up to leaseDuration)
```

#### Important Flags for Leader Election

```bash
# kube-controller-manager flags:
--leader-elect=true                     # Enable leader election
--leader-elect-lease-duration=15s       # How long lease is valid
--leader-elect-renew-deadline=10s       # How often leader renews
--leader-elect-retry-period=2s          # How often non-leaders retry
--leader-elect-resource-name=kube-controller-manager
--leader-elect-resource-namespace=kube-system
```

#### HA Best Practices

```yaml
# Deploy kube-controller-manager with:
replicas: 3                        # Odd number for clean election
--leader-elect=true                # Always enable in HA
--leader-elect-lease-duration=15s  # Default is fine
--leader-elect-renew-deadline=10s  # Must be < lease-duration
--leader-elect-retry-period=2s     # Must be < renew-deadline

# Monitor leader status:
kubectl get lease kube-controller-manager -n kube-system -o yaml
kubectl get lease kube-scheduler -n kube-system -o yaml
```

---

## 14. Interaction with API Server and etcd

### The Immutable Rule

> **Container engines (containerd, CRI-O) NEVER talk to the API server or etcd. Ever.**

The container engine's world is entirely local to the node. It knows nothing about Kubernetes objects, namespaces, or the cluster. It only knows about containers, images, and sandboxes on the local machine.

```
CORRECT communication flow:
  kubectl → kube-apiserver → etcd (stores desired state)
  kube-apiserver → kubelet (delivers pod spec via watch)
  kubelet → containerd (CRI gRPC: "start this container")
  containerd → runc → Linux kernel (actually runs it)

INCORRECT (never happens):
  containerd → kube-apiserver ✗
  containerd → etcd           ✗
  CRI-O → kube-apiserver      ✗
```

### What the Container Engine Does Know About

| Knows | Does NOT Know |
|---|---|
| Container IDs | Pod names |
| Image digests and tags | Namespace names |
| Sandbox IDs | Service accounts |
| Local filesystem paths | RBAC policies |
| cgroup paths | ReplicaSet names |
| Network namespace paths | Deployment rollout state |
| OCI runtime config | etcd |
| Registry endpoints (for pull) | API server |

### The Kubelet as Translator

The Kubelet is the critical translation layer:

```
Kubernetes world (API Server):
  Pod spec:
    containers:
    - name: nginx
      image: nginx:1.25
      resources:
        limits:
          memory: 256Mi
          cpu: 500m
      securityContext:
        runAsUser: 1000

                    ↓ Kubelet translates

Container Engine world (CRI):
  CreateContainerRequest:
    image: {image: "nginx:1.25"}
    config:
      metadata: {name: "nginx"}
      envs: [...]
      mounts: [...]
      linux:
        resources:
          memory_limit_in_bytes: 268435456
          cpu_quota: 50000
          cpu_period: 100000
        security_context:
          run_as_user: 1000
          seccomp_profile_type: RUNTIME_DEFAULT
```

### API Server Interactions (Kubelet on behalf of container engine)

The Kubelet calls the API server to:

| Operation | Why | CRI Trigger |
|---|---|---|
| Watch Pods | Get pod assignments | Drives all CRI calls |
| Get Secret | Registry pull credentials | Before `PullImage()` |
| Get ConfigMap | Projected volume content | Before `CreateContainer()` |
| Update Pod status | Report container state | After CRI state changes |
| Create Event | Report image pull, OOM kill | After container events |
| Update Node status | Report runtime version, allocatable | Periodic |

---

## 15. Image Management: Pull, Cache, Garbage Collection

Image management is one of the most operationally important responsibilities of the container engine. Slow image pulls, cache misses, and disk exhaustion are among the most common causes of pod start failures in production.

### Image Pull Flow

```
Kubelet calls: ImageService.PullImage("nginx:1.25", authConfig, sandboxConfig)
  │
  ├─1─► Check local content store (is image already present?)
  │      └─ Yes: return immediately ← cache HIT
  │      └─ No: proceed to pull    ← cache MISS
  │
  ├─2─► Authenticate to registry
  │      └─ Use authConfig (from imagePullSecrets → kubelet → CRI)
  │
  ├─3─► Fetch image manifest from registry
  │      └─ GET https://registry.example.com/v2/nginx/manifests/1.25
  │
  ├─4─► For each layer in manifest:
  │      └─ Check if layer blob already exists (content-addressed by sha256)
  │      └─ If missing: GET https://registry.example.com/v2/nginx/blobs/<digest>
  │      └─ Decompress and verify sha256 checksum
  │      └─ Store in content store
  │
  └─5─► Create snapshot (prepare overlay layers)
         └─ Image ready for container creation
```

### Image Pull Policy

| `imagePullPolicy` | Behavior | When to Use |
|---|---|---|
| `IfNotPresent` | Pull only if image not cached locally | Production (avoid unnecessary pulls) |
| `Always` | Always pull (but may use cached layers) | CI/CD with `latest` tags |
| `Never` | Never pull; fail if not present | Air-gapped environments |

```yaml
# Production recommendation:
image: myapp:v1.2.3         # Use specific digest or version tag
imagePullPolicy: IfNotPresent # Never use 'latest' in production
```

### Image Garbage Collection

```
Image GC thresholds (Kubelet flags → implemented by container engine):

  imageGCHighThresholdPercent: 85  ← Start GC when disk > 85% full
  imageGCLowThresholdPercent: 80   ← GC until disk < 80% full

GC algorithm:
  1. List all images from container engine (ImageService.ListImages)
  2. Identify images NOT referenced by any container
  3. Sort by last-used time (LRU)
  4. Delete oldest unused images until threshold is met
  5. Images in use are NEVER deleted
```

```bash
# View images and their sizes:
crictl images
ctr --namespace k8s.io images list

# Manually trigger image GC (via Kubelet API):
curl -X POST https://localhost:10250/imagegc \
  --cert /var/lib/kubelet/pki/kubelet.crt \
  --key /var/lib/kubelet/pki/kubelet.key -k

# View disk usage by container engine:
du -sh /var/lib/containerd/
ctr --namespace k8s.io images check   # Check image integrity
```

### Registry Mirror Configuration

```toml
# containerd mirror config:
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
  endpoint = [
    "https://my-internal-mirror.example.com",
    "https://mirror.gcr.io",          # GCR mirror for docker.io
    "https://registry-1.docker.io"    # Fallback to origin
  ]

[plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.k8s.io"]
  endpoint = ["https://my-internal-mirror.example.com"]
```

### Private Registry Authentication

```bash
# Method 1: imagePullSecret (per-pod)
kubectl create secret docker-registry regcred \
  --docker-server=registry.example.com \
  --docker-username=myuser \
  --docker-password=mypassword

# Method 2: containerd static credentials
# /etc/containerd/config.toml:
[plugins."io.containerd.grpc.v1.cri".registry.configs."registry.example.com".auth]
  username = "myuser"
  password = "mypassword"

# Method 3: Node-level credentials file
# /root/.docker/config.json — kubelet reads this automatically
```

---

## 16. Namespace & cgroup Isolation

The container engine's most critical security and isolation function is configuring Linux kernel primitives for each container.

### Linux Namespaces Used Per Container

| Namespace | Isolates | Kubernetes Control |
|---|---|---|
| **pid** | Process IDs | `shareProcessNamespace: true/false` |
| **net** | Network interfaces, routes | Shared at Pod level (all containers share) |
| **ipc** | Inter-process communication | Shared at Pod level |
| **uts** | Hostname, domain name | Per-pod (hostname = pod name) |
| **mnt** | Filesystem mounts | Per-container |
| **user** | User/group IDs | `runAsUser`, `runAsGroup` |
| **cgroup** | cgroup root (v2 only) | Kubernetes-managed |

### Namespace Sharing in Pods

```
Pod (shared namespaces):
  ├── Network namespace (shared by all containers in pod)
  │    └─ All containers can communicate via localhost
  │    └─ Only one container can bind a given port
  ├── IPC namespace (shared by all containers in pod)
  │    └─ Containers can use shared memory, semaphores
  │
  └── Per-container namespaces:
       ├── Container A: [pid ns A] [mnt ns A]
       └── Container B: [pid ns B] [mnt ns B]

Exception: shareProcessNamespace: true
  └─ All containers share PID namespace
  └─ Can see and signal each other's processes
```

### cgroup Hierarchy for Kubernetes

```
/sys/fs/cgroup/
└── kubepods/                              ← Kubelet creates this
    ├── besteffort/                        ← BestEffort QoS pods
    │   └── pod<uid>/
    │       ├── <container-id>/
    │       │   ├── cpu.max               ← CPU quota
    │       │   ├── memory.max            ← Memory limit
    │       │   └── memory.min            ← Memory request guarantee
    │       └── memory.max                ← Pod-level limit
    ├── burstable/                         ← Burstable QoS pods
    │   └── pod<uid>/
    │       └── <container-id>/
    └── guaranteed/                        ← Guaranteed QoS pods
        └── pod<uid>/
            └── <container-id>/
```

```bash
# Inspect cgroup for a container:
CONTAINER_ID=$(crictl ps | grep my-container | awk '{print $1}')
CGROUP_PATH=$(crictl inspect $CONTAINER_ID | jq -r '.info.runtimeSpec.linux.cgroupsPath')
cat /sys/fs/cgroup/${CGROUP_PATH}/memory.max
cat /sys/fs/cgroup/${CGROUP_PATH}/cpu.max
```

### seccomp Profile Application

```
Container Engine applies seccomp via OCI runtime:
  RuntimeDefault profile: allowlist of ~300 safe syscalls
  Custom profile: specified in pod securityContext

# What gets blocked by RuntimeDefault (examples):
  - keyctl (kernel keyring)
  - ptrace (debugging/tracing — can be enabled for debugging pods)
  - mount (prevent privilege escalation via mount)
  - reboot (prevent node shutdown from container)
```

---

## 17. Security Hardening Practices

### RBAC for Container Engine Adjacent Resources

The container engine itself doesn't use RBAC, but resources it depends on (image pull secrets, runtime classes) do.

```yaml
# RuntimeClass — controls which OCI runtime is used:
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata-containers
handler: kata            # Must match containerd runtime name
scheduling:
  nodeSelector:
    kata-runtime: "true"  # Only schedule on nodes with kata
overhead:
  podFixed:
    memory: "120Mi"       # Extra overhead for VM-based runtime
    cpu: "250m"
```

```yaml
# Pod using RuntimeClass:
spec:
  runtimeClassName: kata-containers  # Use kata instead of runc
```

```bash
# RBAC for imagePullSecrets:
kubectl create role image-puller \
  --verb=get --resource=secrets \
  --resource-name=registry-credentials

kubectl create rolebinding image-puller-binding \
  --role=image-puller \
  --serviceaccount=default:default
```

### TLS and Socket Security

```bash
# containerd socket permissions (restrict access):
ls -la /run/containerd/containerd.sock
# Should be: srw-rw---- root containerd-group
# Kubelet must be in the containerd-group OR run as root

# Verify socket is not world-accessible:
stat -c "%a %G" /run/containerd/containerd.sock
# Should show: 660 containerd (or similar restricted permissions)
```

### Seccomp Hardening

```yaml
# Enable RuntimeDefault seccomp for ALL pods by default:
# /var/lib/kubelet/config.yaml
seccompDefault: true

# Per-pod override (stricter profile):
spec:
  securityContext:
    seccompProfile:
      type: Localhost
      localhostProfile: profiles/my-strict-profile.json
```

### AppArmor Integration

```yaml
# Pod-level AppArmor (containerd applies this via OCI runtime):
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/nginx: runtime/default
```

```bash
# Verify AppArmor profile is loaded:
aa-status | grep -i docker
# or for containerd containers:
cat /proc/<container-pid>/attr/current
```

### Dropping Linux Capabilities

```yaml
# Security best practice: drop all, add only what's needed
spec:
  containers:
  - name: app
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      runAsUser: 10001
      capabilities:
        drop:
          - ALL
        add:
          - NET_BIND_SERVICE   # Only if binding port < 1024
```

### Container Engine Security Checklist

| Practice | containerd Config | Risk if Missing |
|---|---|---|
| Use SystemdCgroup | `SystemdCgroup = true` | cgroup hierarchy mismatch → OOM issues |
| Restrict socket permissions | chmod 660 on socket | Unauthorized container creation |
| Enable seccomp default | `seccompDefault: true` in Kubelet | Wide attack surface |
| Use read-only registry | Configure pull-only credentials | Image tampering |
| Sign and verify images | Cosign + policy controller | Supply chain attacks |
| Use sandbox runtime (kata/gVisor) | RuntimeClass for sensitive workloads | Kernel exploit escape |
| Disable privileged containers | Admission controller | Full node compromise |
| Set non-root user | `runAsNonRoot: true` | Privilege escalation |
| Drop all capabilities | `capabilities.drop: [ALL]` | Capability abuse |
| Read-only root FS | `readOnlyRootFilesystem: true` | Persistent malware |

### Image Signing and Verification

```bash
# Sign an image with Cosign:
cosign sign --key cosign.key registry.example.com/myapp:v1.0

# Verify before pull (integrated with containerd via policy):
cosign verify --key cosign.pub registry.example.com/myapp:v1.0

# Kubernetes policy enforcement via Sigstore Policy Controller:
# (admission webhook that verifies image signatures before pod creation)
kubectl apply -f https://github.com/sigstore/policy-controller/releases/latest/download/policy_controller.yaml
```

---

## 18. Performance Tuning & Configuration

### containerd Performance Flags

```toml
# /etc/containerd/config.toml — Performance tuning

[grpc]
  max_recv_message_size = 67108864   # 64MB (for large exec outputs)
  max_send_message_size = 67108864   # 64MB

# Snapshotter tuning:
[plugins."io.containerd.snapshotter.v1.overlayfs"]
  # Enable async image layer unpacking (faster image pulls):
  # (containerd 1.7+)
  slow_chown = false                 # Use faster chown path

# GC tuning:
[plugins."io.containerd.gc.v1.scheduler"]
  pause_threshold = 0.02             # Pause GC if mutation rate < 2%
  deletion_threshold = 0             # Delete aggressively
  mutation_threshold = 100           # Trigger GC after 100 mutations
  schedule_delay = "0s"              # Start GC immediately when triggered
  startup_delay = "100ms"            # Delay after startup before first GC
```

### Kubelet Flags Affecting Container Engine

```yaml
# /var/lib/kubelet/config.yaml — Container engine interaction tuning

# Pod concurrency:
maxPods: 110                           # Increase for large nodes
serializeImagePulls: false             # Allow parallel image pulls (default: true)
maxParallelImagePulls: 5               # Max simultaneous pulls (when serialization disabled)

# Image management:
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m

# Container log:
containerLogMaxSize: "50Mi"            # Max size per log file
containerLogMaxFiles: 5                # Max rotated log files

# Eviction:
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
  imagefs.available: "15%"             # Protect image disk space

# cgroup driver (MUST match containerd's SystemdCgroup setting):
cgroupDriver: systemd                  # or: cgroupfs

# Concurrency tuning:
kubeAPIQPS: 50
kubeAPIBurst: 100
```

### Image Pull Performance Optimization

| Strategy | Implementation | Impact |
|---|---|---|
| **Pre-pull images** | DaemonSet or node startup script | Eliminates pull latency |
| **Registry mirror** | containerd mirror config | Reduces WAN latency |
| **Image layer caching** | Shared base images | Pulls only changed layers |
| **Parallel pulls** | `serializeImagePulls: false` | Faster multi-container pods |
| **Image streaming** | EKS image acceleration, Stargz | Start before pull complete |
| **Slim images** | Distroless, Alpine base | Smaller layers to pull |
| **Pull-through cache** | Harbor, ECR pull-through | Centralized caching |

```bash
# Enable parallel image pulls (disable serialization):
# /var/lib/kubelet/config.yaml
serializeImagePulls: false
maxParallelImagePulls: 3  # Tune based on network bandwidth

# Result: Pods with multiple containers start pulling all images simultaneously
# rather than sequentially
```

### Node-Level Performance Tuning

```bash
# Increase inotify limits (important for many containers):
echo "fs.inotify.max_user_instances=8192" >> /etc/sysctl.conf
echo "fs.inotify.max_user_watches=524288" >> /etc/sysctl.conf
sysctl -p

# Increase PID limit for many containers:
echo "kernel.pid_max=4194304" >> /etc/sysctl.conf

# Tune OverlayFS (if many layers):
echo "vm.dirty_ratio=40" >> /etc/sysctl.conf
echo "vm.dirty_background_ratio=10" >> /etc/sysctl.conf

# Verify containerd socket response time:
time crictl info
# Should respond in < 100ms; > 500ms indicates containerd load
```

### Resource Limits for the Container Engine Itself

```ini
# /etc/systemd/system/containerd.service.d/override.conf
[Service]
LimitNOFILE=1048576          # Max open files
LimitNPROC=infinity          # Max processes
LimitCORE=infinity           # Allow core dumps for debugging
TasksMax=infinity             # No task limit
Restart=always               # Auto-restart on crash
RestartSec=5s                # Wait 5s before restart
```

---

## 19. Monitoring & Observability

### containerd Metrics Endpoint

```bash
# containerd Prometheus metrics (default: localhost:1338):
curl http://localhost:1338/v1/metrics

# Verify metrics endpoint is up:
curl -s http://localhost:1338/v1/metrics | head -30
```

### Key Prometheus Metrics

**containerd Runtime Metrics:**

| Metric | Type | Description |
|---|---|---|
| `container_runtime_crio_operations_total` | Counter | Total CRI operations by type |
| `container_runtime_crio_operations_errors_total` | Counter | Failed CRI operations |
| `container_runtime_crio_operations_latency_microseconds` | Histogram | CRI operation latency |
| `containerd_snapshot_op_duration_seconds` | Histogram | Snapshot operation duration |
| `containerd_content_store_duration_seconds` | Histogram | Content store op duration |

**Kubelet Container Metrics (exposed on :10250/metrics):**

| Metric | Type | Description |
|---|---|---|
| `kubelet_running_containers` | Gauge | Number of running containers |
| `kubelet_running_pods` | Gauge | Number of running pods |
| `kubelet_container_log_filesystem_used_bytes` | Gauge | Container log disk usage |
| `kubelet_pleg_relist_duration_seconds` | Histogram | PLEG relist (CRI query) duration |
| `kubelet_pleg_relist_interval_seconds` | Histogram | Interval between PLEG relists |
| `kubelet_pod_start_duration_seconds` | Histogram | Total pod start time |
| `kubelet_pod_worker_duration_seconds` | Histogram | Pod sync worker duration |
| `kubelet_image_garbage_collected_total` | Counter | Images GC'd |

**Image Pull Metrics:**

| Metric | Type | Description |
|---|---|---|
| `kubelet_image_pull_duration_seconds` | Histogram | Time to pull an image |
| `kubelet_image_garbage_collected_total` | Counter | Images removed by GC |

**cAdvisor Container Resource Metrics (most important for alerting):**

| Metric | Type | Description |
|---|---|---|
| `container_cpu_usage_seconds_total` | Counter | CPU usage per container |
| `container_memory_usage_bytes` | Gauge | Current memory usage |
| `container_memory_working_set_bytes` | Gauge | Working set memory (used for OOM) |
| `container_network_receive_bytes_total` | Counter | Network RX per container |
| `container_network_transmit_bytes_total` | Counter | Network TX per container |
| `container_fs_usage_bytes` | Gauge | Filesystem usage per container |
| `container_oom_events_total` | Counter | OOM kill events |

### Critical Alert Rules

```yaml
groups:
- name: container-engine.rules
  rules:

  # Container runtime down:
  - alert: ContainerRuntimeDown
    expr: absent(up{job="kubelet"} == 1)
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Container runtime may be down on {{ $labels.node }}"

  # PLEG unhealthy (CRI unresponsive):
  - alert: PLEGUnhealthy
    expr: kubelet_pleg_relist_duration_seconds{quantile="0.99"} > 10
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "PLEG relist too slow: {{ $value }}s on {{ $labels.node }}"
      description: "Container runtime likely overloaded or hung"

  # Image pull slow (impacting pod starts):
  - alert: SlowImagePull
    expr: histogram_quantile(0.95, kubelet_image_pull_duration_seconds_bucket) > 60
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Image pulls taking > 60s (p95)"

  # Image disk pressure:
  - alert: ImageDiskPressure
    expr: (node_filesystem_avail_bytes{mountpoint="/var/lib/containerd"} /
           node_filesystem_size_bytes{mountpoint="/var/lib/containerd"}) < 0.15
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Container image disk space < 15% on {{ $labels.node }}"

  # Container OOM:
  - alert: ContainerOOMKilled
    expr: increase(container_oom_events_total[5m]) > 0
    labels:
      severity: warning
    annotations:
      summary: "Container {{ $labels.container }} OOM killed in pod {{ $labels.pod }}"

  # Too many container restarts:
  - alert: HighContainerRestartRate
    expr: increase(kube_pod_container_status_restarts_total[15m]) > 5
    labels:
      severity: warning
    annotations:
      summary: "Container {{ $labels.container }} restarting frequently"
```

### Prometheus Scrape Configuration

```yaml
# prometheus.yml — Scraping containerd and kubelet:
scrape_configs:
  # Kubelet metrics (includes cAdvisor):
  - job_name: "kubelet"
    kubernetes_sd_configs:
      - role: node
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      insecure_skip_verify: true
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics

  # cAdvisor metrics:
  - job_name: "cadvisor"
    kubernetes_sd_configs:
      - role: node
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
      insecure_skip_verify: true
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - target_label: __address__
        replacement: kubernetes.default.svc:443
      - source_labels: [__meta_kubernetes_node_name]
        target_label: __metrics_path__
        replacement: /api/v1/nodes/${1}/proxy/metrics/cadvisor
```

---

## 20. Troubleshooting Guide with Real Commands

### Scenario 1: Pods Not Created / ContainerCreating Stuck

```bash
# Step 1: Describe the pod (look at Events section)
kubectl describe pod <pod-name> -n <namespace>
# Common events to look for:
# - "Failed to pull image" → registry/auth issue
# - "Back-off pulling image" → repeated pull failures
# - "Failed to create pod sandbox" → CNI or runtime issue
# - "MountVolume.SetUp failed" → CSI/volume issue

# Step 2: Check container runtime directly on the node
# First, find which node the pod is on:
kubectl get pod <pod-name> -n <namespace> -o wide

# SSH to the node, then:
# Check containerd status:
systemctl status containerd
journalctl -u containerd -n 100 --no-pager

# List pod sandboxes:
crictl pods | grep <pod-name>

# List containers (including failed ones):
crictl ps -a | grep <pod-name>

# Step 3: Check for image pull issues
crictl pull <image-name>  # Test pull manually
# If fails, check registry auth:
crictl auth  # Check configured credentials

# Check image is available:
crictl images | grep <image-name>

# Step 4: Check CNI (if pod sandbox creation fails)
# Verify CNI plugin binaries:
ls /opt/cni/bin/
# Verify CNI config:
ls /etc/cni/net.d/
cat /etc/cni/net.d/*.conf*

# Test CNI directly:
journalctl -u kubelet | grep -i "cni\|network\|sandbox" | tail -30

# Step 5: Check containerd logs for specific errors:
journalctl -u containerd | grep -E "error|failed|panic" | tail -50
```

### Scenario 2: Deployment Stuck During Rollout

```bash
# Step 1: Check rollout status
kubectl rollout status deployment/<name> -n <namespace>
kubectl rollout history deployment/<name> -n <namespace>

# Step 2: Check new pods
kubectl get pods -n <namespace> -l app=<label> --show-labels

# Step 3: If new pods are stuck in ContainerCreating:
# → Follow Scenario 1 steps
# Check if image pull is the bottleneck:
kubectl describe pod <new-pod> | grep -A 5 "Events:"

# Step 4: If old pods are stuck in Terminating:
kubectl get pods -n <namespace>
# Stuck Terminating? Check for:
# - Finalizers blocking deletion:
kubectl get pod <stuck-pod> -o jsonpath='{.metadata.finalizers}'

# - PreStop hook hanging:
kubectl describe pod <stuck-pod> | grep -i "prestop\|lifecycle"

# - Volume unmount issue:
journalctl -u kubelet | grep -i "unmount\|detach" | tail -30

# Force delete (LAST RESORT):
kubectl delete pod <pod-name> --grace-period=0 --force

# Step 5: On the node — force kill container
crictl stop <container-id>
crictl rm <container-id>

# Step 6: Check readiness probe (new pods may not be passing it)
kubectl describe pod <new-pod> | grep -A 10 "Readiness:"
# Check readiness probe logs:
kubectl logs <new-pod> -n <namespace>
```

### Scenario 3: Node NotReady (Container Runtime Related)

```bash
# Step 1: Check node conditions
kubectl describe node <node-name>
# Look for: "container runtime is down" or PLEG errors

# Step 2: SSH to node

# Step 3: Check containerd
systemctl status containerd
# If stopped or failed:
systemctl start containerd
journalctl -u containerd -n 200 | grep -E "error|panic|fatal"

# Step 4: Check if containerd can respond:
crictl info
# If this hangs: containerd is frozen

# Force restart containerd:
systemctl restart containerd

# Wait 30s and check if Kubelet reconnects:
journalctl -u kubelet -f | grep -i "pleg\|runtime\|connected"

# Step 5: If containerd is stuck:
# Check for zombie shim processes:
ps aux | grep containerd-shim | grep -v grep

# Kill stuck shims:
pkill -f containerd-shim-runc-v2
# Then restart containerd:
systemctl restart containerd

# Step 6: Check disk space (common cause of runtime failure):
df -h /var/lib/containerd
df -h /run

# If disk full, free space:
crictl rmi --prune   # Remove unused images
crictl rm $(crictl ps -a -q --state=exited)  # Remove stopped containers

# Step 7: Check PLEG health via metrics
curl -sk https://localhost:10250/metrics | grep pleg_relist_duration
# Values > 3 seconds indicate CRI performance issues

# Step 8: Check for OverlayFS issues
dmesg | grep -i "overlay\|filesystem" | tail -20
# OverlayFS kernel bugs can cause containerd to hang
```

### Scenario 4: OOMKilled Containers

```bash
# Step 1: Confirm OOMKill
kubectl describe pod <pod-name> | grep -A 3 "Last State:"
# Look for: "Reason: OOMKilled"

# Step 2: Check current memory usage
kubectl top pod <pod-name> -n <namespace>
kubectl top pods -n <namespace> --containers

# Step 3: Check cAdvisor metrics
# From Prometheus: container_oom_events_total
# From kubectl:
kubectl get pod <pod-name> -o json | jq '.status.containerStatuses[].lastState'

# Step 4: On node - check kernel OOM log
dmesg | grep -i "oom-killer\|killed process" | tail -20

# Step 5: Check container cgroup memory limit
CONTAINER_ID=$(crictl ps | grep <container-name> | awk '{print $1}')
crictl inspect $CONTAINER_ID | jq '.info.runtimeSpec.linux.resources.memory'

# Step 6: Fix - increase memory limit in deployment
kubectl set resources deployment/<name> \
  --containers=<container-name> \
  --requests=memory=256Mi \
  --limits=memory=512Mi
```

### Scenario 5: Image Pull Authentication Failure

```bash
# Symptom: ErrImagePull or ImagePullBackOff

# Step 1: Check the error
kubectl describe pod <pod> | grep -A 10 "Events:"
# Look for: "401 Unauthorized" or "403 Forbidden"

# Step 2: Verify imagePullSecret exists
kubectl get secret <pull-secret-name> -n <namespace>

# Step 3: Verify secret content (decode)
kubectl get secret <pull-secret-name> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq

# Step 4: Test credentials manually on node
crictl pull --creds username:password <image>
# OR configure in containerd:
# /etc/containerd/config.toml registry auth section

# Step 5: Verify secret is referenced in pod spec:
kubectl get pod <pod> -o jsonpath='{.spec.imagePullSecrets}'

# Step 6: If using service account:
kubectl get serviceaccount <sa-name> -o yaml | grep imagePullSecrets
```

---

## 21. Comparison: Container Engine vs kube-apiserver vs kube-scheduler

### Architectural Role Comparison

| Dimension | Container Engine (containerd) | kube-apiserver | kube-scheduler |
|---|---|---|---|
| **Primary Role** | Run containers on local node | Cluster API gateway, state store | Assign pods to nodes |
| **Scope** | Node-local | Cluster-wide | Cluster-wide |
| **Instances Per Cluster** | One per node | 1-5 (HA) | 1-5 (HA with leader election) |
| **Leader Election** | No (per-node daemon) | No (stateless) | Yes (one active leader) |
| **State Storage** | Local metadata (BoltDB) | etcd (via API server) | Stateless (reads from API) |
| **Talks to etcd** | Never | Yes (only component) | Never |
| **Talks to API Server** | Never | Receives requests | Yes (reads pods, writes binding) |
| **Talks to Kubelet** | Receives CRI calls | Receives status updates | Never directly |
| **Restart Impact on Pods** | Containers keep running | All API calls fail | No new scheduling |
| **Restart Impact on Cluster** | Node goes NotReady (~40s) | Full cluster outage | Pending pods stay pending |
| **Primary Interface** | gRPC (CRI) / Unix socket | HTTPS REST + gRPC | Kubernetes API (internal) |
| **Config Format** | TOML (containerd) | Flags + YAML | Flags + KubeSchedulerConfig YAML |
| **Metrics Port** | 1338 (containerd) | 2381 | 10259 |
| **Authentication** | Socket file permissions | x509, OIDC, Bearer tokens | TLS cert to API server |
| **Plugin System** | NRI, runtime plugins | Admission webhooks | Scheduler plugins/extenders |

### Interaction Flow

```
┌────────────────────────────────────────────────────────────────────┐
│                    Request Flow for Pod Creation                    │
│                                                                     │
│  1. User: kubectl apply -f pod.yaml                                │
│            │                                                        │
│            ▼                                                        │
│  2. kube-apiserver: Validate, store Pod in etcd (Pending)          │
│            │                                                        │
│            ▼                                                        │
│  3. kube-scheduler: Watch unscheduled pods                         │
│     → Score nodes (resources, affinity, taints)                    │
│     → Write pod.spec.nodeName = "worker-1"                        │
│     → kube-apiserver stores binding in etcd                        │
│            │                                                        │
│            ▼                                                        │
│  4. Kubelet on worker-1: Watch detects pod assignment              │
│     → Compute desired state                                         │
│     → Compare with CRI actual state                                │
│            │                                                        │
│            ▼                                                        │
│  5. Container Engine (containerd):                                  │
│     PullImage() → RunPodSandbox() → CreateContainer()              │
│     → StartContainer()                                              │
│            │                                                        │
│            ▼                                                        │
│  6. OCI Runtime (runc):                                             │
│     clone() → unshare namespaces → apply cgroups → exec()          │
│            │                                                        │
│            ▼                                                        │
│  7. Kubelet: Detect container running (via PLEG/CRI)               │
│     → Update pod status to Running                                  │
│     → kube-apiserver stores status in etcd                         │
│            │                                                        │
│            ▼                                                        │
│  8. User sees: kubectl get pod → Running                           │
└────────────────────────────────────────────────────────────────────┘
```

---

## 22. Disaster Recovery Concepts

### Container Engine is Stateless (from Kubernetes perspective)

The container engine does not own any Kubernetes state. Its local state (BoltDB metadata in containerd, image cache, container metadata) is **derived state** — it can be rebuilt from the source of truth (etcd, container registries).

```
True Source of Truth:
  etcd → kube-apiserver → Kubelet → Container Engine actions

Container Engine local state (all recoverable):
  /var/lib/containerd/     ← Image layers (re-pullable from registry)
  BoltDB (metadata)        ← Rebuilt from container runtime state
  Running containers       ← Recreated from pod specs in etcd
```

### Node Failure Scenarios

| Scenario | Container Engine State | Recovery |
|---|---|---|
| containerd process crash | Containers keep running (shim survives) | `systemctl restart containerd` |
| Node reboot | All containers stop | Kubelet + containerd restart, pods recreated |
| Disk corruption on /var/lib/containerd | Images lost, containers lost | Node replaced; pods rescheduled elsewhere |
| containerd upgrade | Service restart required | Drain node first |
| OCI runtime (runc) crash | Affected container dies | Container restarted by Kubelet (per restartPolicy) |

### Node Recovery Procedure

```bash
# Scenario: Node disk /var/lib/containerd corrupted

# Step 1: Drain node (move workloads away)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Step 2: On the node — clean slate
systemctl stop kubelet containerd

# Step 3: Wipe corrupted data
rm -rf /var/lib/containerd/*
rm -rf /var/lib/kubelet/pods/*

# Step 4: Restart services
systemctl start containerd
systemctl start kubelet

# Step 5: Verify node comes back
kubectl get node <node-name>
# Wait for Ready status

# Step 6: Uncordon to allow workload scheduling
kubectl uncordon <node-name>

# Step 7: Kubernetes will reschedule pods; container engine will pull images
# Monitor:
kubectl get pods --all-namespaces -o wide | grep <node-name>
watch -n 5 'crictl ps | wc -l'
```

### etcd Backup Relationship

```bash
# etcd backup (cluster-level DR):
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# After etcd restore:
# 1. etcd has desired state restored
# 2. API server reads restored state
# 3. Kubelet watches reconcile (may need pod restarts)
# 4. Container engine receives new CRI calls
# 5. Pods are recreated on nodes

# Container engine does NOT need a separate backup
# Images: pulled from registry as needed
# Container state: derived from pod specs in etcd
```

### Container Image Backup Strategy

```bash
# While the container engine doesn't need backup, images should be:

# 1. Stored in HA registry (ECR, GCR, Harbor with replication)
# 2. Pinned by digest (not just tag) for reproducibility:
image: nginx@sha256:abc123...

# 3. Mirror to offline registry for air-gapped DR:
# Save image:
crictl pull nginx:1.25
ctr --namespace k8s.io images export nginx-1.25.tar nginx:1.25

# Load image (on DR site):
ctr --namespace k8s.io images import nginx-1.25.tar
```

---

## 23. Real-World Production Use Cases

### Use Case 1: Multi-Runtime Cluster (Security Tiering)

**Scenario**: SaaS platform with mixed workloads — trusted internal services and untrusted customer code.

```yaml
# RuntimeClass for sandboxed customer code:
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: gvisor
handler: runsc   # gVisor
---
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata    # Kata containers (full VM isolation)
---
# Customer code Pod (sandboxed):
spec:
  runtimeClassName: gvisor
  containers:
  - name: customer-function
    image: customer-registry/func:latest
---
# Internal service Pod (standard runc):
spec:
  # No runtimeClassName = default runc
  containers:
  - name: internal-api
    image: internal/api:v1
```

**Container Engine Setup (containerd)**:
```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runsc]
  runtime_type = "io.containerd.runsc.v1"
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.kata]
  runtime_type = "io.containerd.kata.v2"
```

### Use Case 2: Air-Gapped / Offline Cluster

**Scenario**: Government or enterprise cluster with no internet access.

```bash
# Offline image workflow:
# 1. On internet-connected machine: pull and export images
crictl pull registry.k8s.io/pause:3.9
ctr images export pause.tar registry.k8s.io/pause:3.9

# 2. Transfer to air-gapped environment
# 3. On each node: import images
ctr --namespace k8s.io images import pause.tar

# 4. Configure containerd to use internal registry:
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.k8s.io"]
  endpoint = ["https://internal-registry.example.com"]

# 5. Disable external pulls entirely (firewall + registry config)
```

### Use Case 3: High-Density Node (1000+ Pods per Node)

**Scenario**: Serverless platform running thousands of short-lived containers per node.

```yaml
# Tuning for high density:
# Kubelet:
maxPods: 1000
serializeImagePulls: false
maxParallelImagePulls: 10
kubeAPIQPS: 200
kubeAPIBurst: 400
```

```toml
# containerd tuning for high density:
[grpc]
  max_recv_message_size = 67108864
  max_send_message_size = 67108864

# Use crun instead of runc (faster startup, lower memory):
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.crun]
  runtime_type = "io.containerd.runc.v2"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.crun.options]
    BinaryName = "/usr/bin/crun"
    SystemdCgroup = true
```

### Use Case 4: GPU Workloads (ML Training)

```toml
# NVIDIA Container Runtime for GPU access:
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
  runtime_type = "io.containerd.runc.v2"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia.options]
    BinaryName = "/usr/bin/nvidia-container-runtime"
    SystemdCgroup = true
```

```yaml
# RuntimeClass for GPU pods:
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: nvidia
handler: nvidia
---
spec:
  runtimeClassName: nvidia
  containers:
  - name: training
    image: pytorch/pytorch:2.0.0-cuda11.7-cudnn8-runtime
    resources:
      limits:
        nvidia.com/gpu: 4
```

---

## 24. Best Practices for Production Environments

### Container Engine Configuration

- **Always use SystemdCgroup = true** in containerd when Kubelet uses `cgroupDriver: systemd` (the modern default). Mismatch causes subtle resource accounting bugs and OOM issues.
- **Pin the pause/sandbox image** (`sandbox_image`) to a specific digest, not just a tag. The pause container is the anchor of every pod network namespace.
- **Configure registry mirrors** to reduce dependency on public registries and improve pull latency.
- **Separate the image disk** (`/var/lib/containerd`) from the root filesystem to prevent container image bloat from filling the OS disk.
- **Enable metrics** (`address = "127.0.0.1:1338"` in containerd) and scrape them with Prometheus.

### Image Management

- Never use `latest` or mutable tags in production — use specific version tags or image digests.
- Pre-pull large images using a DaemonSet or node startup script to eliminate pull latency from your pod startup critical path.
- Implement a registry mirror (Harbor, ECR pull-through cache) to avoid DockerHub pull rate limits and reduce WAN latency.
- Set `serializeImagePulls: false` with `maxParallelImagePulls: 3-5` for faster multi-image pod starts.

### Security

- Enable seccomp defaults (`seccompDefault: true` in Kubelet).
- Use `readOnlyRootFilesystem: true`, `runAsNonRoot: true`, and `capabilities.drop: [ALL]` for all production containers.
- Use image signing (Cosign) and a policy controller to prevent unsigned images from running.
- Use RuntimeClass to run sensitive workloads with kata-containers or gVisor isolation.
- Regularly scan images for CVEs (Trivy, Grype) before admission.

### Operations

- **Always drain a node before upgrading the container engine** — `kubectl drain <node>` first.
- Monitor `kubelet_pleg_relist_duration_seconds` — values above 3 seconds indicate CRI performance degradation.
- Set up alerting for `container_oom_events_total` to catch memory-starved workloads before they cause cascading failures.
- Implement log rotation (`containerLogMaxSize`, `containerLogMaxFiles`) to prevent log files from filling the disk.

---

## 25. Common Mistakes and Pitfalls

### Mistake 1: cgroup Driver Mismatch

```toml
# containerd config.toml:
SystemdCgroup = false   # WRONG when Kubelet uses systemd cgroupDriver

# This causes: Pods start but have incorrect resource accounting
# Symptoms: CPU/memory metrics wrong, OOM kills at wrong thresholds
# Fix:
SystemdCgroup = true    # Match kubelet's cgroupDriver: systemd
```

### Mistake 2: Not Setting the Correct Pause/Sandbox Image

```toml
# DEFAULT (may be wrong for your K8s version):
sandbox_image = "registry.k8s.io/pause:3.6"

# CORRECT for K8s 1.28+:
sandbox_image = "registry.k8s.io/pause:3.9"

# Why it matters: pause container version must match K8s version
# In air-gapped: if pause image is wrong version and can't be pulled → ALL pods fail to start
```

### Mistake 3: Using `latest` Tags in Production

```yaml
# WRONG:
image: myapp:latest        # Which version? Unknown. Updates silently.
imagePullPolicy: IfNotPresent  # Won't pull new version anyway!

# RIGHT:
image: myapp:v1.2.3        # Explicit version
# OR use digest for immutability:
image: myapp@sha256:abc123def456...
```

### Mistake 4: Not Draining Before Runtime Upgrade

```bash
# WRONG — upgrading containerd on a live node:
apt-get upgrade containerd.io
systemctl restart containerd
# Result: All containers momentarily disrupted, pod failures, node briefly NotReady

# RIGHT — drain first:
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
apt-get upgrade containerd.io
systemctl restart containerd
kubectl uncordon worker-1
```

### Mistake 5: Ignoring Image Disk Pressure

```bash
# Symptom: Node DiskPressure condition = True
# Cause: /var/lib/containerd disk is full
# Effect: No new pods can start; existing pods may be evicted

# Prevention: Separate partition for /var/lib/containerd
# AND: Set imageGCHighThresholdPercent appropriately
# AND: Set up disk monitoring alert
```

### Mistake 6: Privileged Containers in Production

```yaml
# WRONG — opens full node access:
securityContext:
  privileged: true

# RIGHT — only grant specific permissions needed:
securityContext:
  capabilities:
    add:
      - NET_ADMIN   # Only if truly needed
  allowPrivilegeEscalation: false
```

### Mistake 7: Not Setting terminationGracePeriodSeconds

```yaml
# Default is 30 seconds. For containers that need longer:
spec:
  terminationGracePeriodSeconds: 90   # Give app 90s to drain connections
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 5 && nginx -s quit"]
# Without this: SIGTERM followed by SIGKILL after 30s → dropped requests
```

### Mistake 8: Running Everything as Root

```yaml
# WRONG:
# (no securityContext = root by default in most images)

# RIGHT:
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  runAsGroup: 10001
  fsGroup: 10001
```

---

## 26. Hands-On Labs & Mini Practical Exercises

### Lab 1: Inspect Your Container Engine

```bash
# Exercise: Get familiar with your cluster's container engine

# 1. Find what runtime each node uses:
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
RUNTIME:.status.nodeInfo.containerRuntimeVersion

# 2. SSH to a worker node

# 3. Check containerd version and status:
containerd --version
systemctl status containerd

# 4. List all containers from runtime perspective:
crictl ps

# 5. List all pod sandboxes:
crictl pods

# 6. List cached images:
crictl images

# 7. Inspect a running container:
CONTAINER_ID=$(crictl ps | awk 'NR==2{print $1}')
crictl inspect $CONTAINER_ID | jq '.info.runtimeSpec.linux.namespaces'

# 8. View the OCI config for a container:
crictl inspect $CONTAINER_ID | jq '.info.runtimeSpec.process'
```

### Lab 2: Observe Image Pull in Real Time

```bash
# Exercise: Watch what happens when a new image is pulled

# Terminal 1: Watch containerd logs
journalctl -u containerd -f | grep -i "pull\|fetch\|download"

# Terminal 2: Watch crictl images
watch -n 1 'crictl images'

# Terminal 3: Deploy a pod with an image not on the node
kubectl run pull-test \
  --image=nginx:1.25.3 \
  --image-pull-policy=Always

# Observe:
# - containerd fetching manifest
# - containerd downloading each layer
# - Image appearing in crictl images
# - Pod moving from ContainerCreating to Running

# Clean up:
kubectl delete pod pull-test
```

### Lab 3: Explore the OverlayFS Filesystem

```bash
# Exercise: Understand container filesystem layering

# 1. Find a running nginx container:
crictl ps | grep nginx
CONTAINER_ID=<from above>

# 2. Get the OCI bundle path:
crictl inspect $CONTAINER_ID | jq -r '.info.runtimeSpec.root.path'

# 3. Look at OverlayFS mounts:
mount | grep overlay

# 4. Find the upper (writable) layer:
# The upperdir is where container writes go
mount | grep overlay | grep $CONTAINER_ID

# 5. Create a file inside the container:
crictl exec $CONTAINER_ID touch /tmp/test-file

# 6. Find the file in the upper layer on the host:
find /var/lib/containerd -name "test-file" 2>/dev/null

# 7. Observe that base image layers are unchanged (read-only):
# The test-file only exists in the container's upper layer
```

### Lab 4: RuntimeClass in Action

```bash
# Exercise: Deploy pods with different runtimes (if kata or gVisor installed)
# For clusters without kata: use this to understand the concept with runc

# 1. List available RuntimeClasses:
kubectl get runtimeclass

# 2. Create a RuntimeClass:
kubectl apply -f - <<EOF
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: runc-explicit
handler: runc
EOF

# 3. Deploy pod with explicit RuntimeClass:
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: runc-test
spec:
  runtimeClassName: runc-explicit
  containers:
  - name: test
    image: busybox
    command: ["sh", "-c", "echo 'Runtime: runc-explicit'; sleep 3600"]
EOF

# 4. Verify it's running:
kubectl describe pod runc-test | grep "Runtime Class"

# 5. From the node, verify the shim:
ps aux | grep containerd-shim

# Clean up:
kubectl delete pod runc-test
kubectl delete runtimeclass runc-explicit
```

### Lab 5: Simulate Container Engine Failure and Recovery

```bash
# Exercise: Understand impact of container runtime failure
# WARNING: This will temporarily impact node health; do in test environment

# 1. Note running pods on your test node:
kubectl get pods -o wide | grep <test-node>

# 2. Stop containerd:
systemctl stop containerd

# 3. Observe:
# - Containers keep running (shims survive!)
ps aux | grep nginx  # Still there

# - Node goes NotReady after ~40s:
kubectl get nodes --watch

# - Kubelet logs show CRI errors:
journalctl -u kubelet | grep "PLEG\|runtime\|error" | tail -20

# 4. Restart containerd:
systemctl start containerd

# 5. Observe recovery:
kubectl get nodes --watch
# Node returns to Ready within ~30s
kubectl get pods -o wide | grep <test-node>
# Pods return to Running
```

### Lab 6: Container Security Context Exploration

```bash
# Exercise: Understand how security context maps to Linux primitives

# 1. Deploy a privileged pod:
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: security-test
spec:
  containers:
  - name: test
    image: busybox
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 1000
      readOnlyRootFilesystem: true
      capabilities:
        drop: [ALL]
EOF

# 2. Verify the user inside the container:
kubectl exec security-test -- id
# Should show: uid=1000

# 3. Try to write to root filesystem:
kubectl exec security-test -- touch /test-file
# Should fail: Read-only file system

# 4. Try to write to /tmp (emptyDir or tmpfs):
kubectl exec security-test -- touch /tmp/test-file
# Should succeed if /tmp is mounted as tmpfs

# 5. On the node, verify the process runs as uid 1000:
ps aux | grep sleep
# Should show uid 1000 for the container's process

# 6. Check capabilities:
kubectl exec security-test -- cat /proc/1/status | grep Cap
# CapEff should show 0000000000000000 (no capabilities)

# Clean up:
kubectl delete pod security-test
```

---

## 27. Interview Questions: Beginner to Advanced

### Beginner Level

**Q1: What is a container engine and how does it relate to Kubernetes?**

**A:** A container engine is software that manages the full container lifecycle — pulling images, creating containers, setting up isolation, and managing container processes. In Kubernetes, the container engine (typically containerd or CRI-O) is the component that actually creates and runs containers on each node. Kubernetes itself (the control plane) only defines *what* should run and *where*; the container engine is what makes it physically happen on the node through the CRI interface.

---

**Q2: What is the CRI and why was it introduced?**

**A:** CRI (Container Runtime Interface) is a standardized gRPC API introduced in Kubernetes 1.5 that defines how the Kubelet communicates with any container runtime. Before CRI, Kubernetes had Docker-specific code (dockershim) built directly into the Kubelet. CRI abstracted this so any runtime (containerd, CRI-O, etc.) can work with Kubernetes without modifying Kubernetes code. The Kubelet calls CRI methods like `RunPodSandbox()`, `CreateContainer()`, and `StartContainer()` and doesn't care what's implementing them.

---

**Q3: What is the difference between a container engine and an OCI runtime?**

**A:** A container engine (like containerd) is a higher-level component that handles image management, CRI interface, filesystem snapshotting, and runtime shim management. An OCI runtime (like runc) is a lower-level component that the container engine calls to actually spawn the isolated process using Linux kernel primitives (namespaces, cgroups, seccomp). containerd manages the lifecycle and then delegates actual process creation to runc via the OCI runtime spec. Think of containerd as the manager and runc as the worker that digs the actual hole.

---

**Q4: Why was Docker removed from Kubernetes 1.24?**

**A:** The Kubernetes team removed dockershim (the Docker-specific code in the Kubelet) because: (1) it was a maintenance burden requiring K8s team to maintain Docker-specific code; (2) Docker itself uses containerd internally, so Kubelet → dockershim → Docker → containerd → runc was 4 hops vs. Kubelet → containerd → runc (2 hops); (3) Docker had features not needed by Kubernetes (build, compose, swarm) that added unnecessary complexity. Importantly, Docker-built images (which are OCI images) still work perfectly — only the daemon interface changed.

---

### Intermediate Level

**Q5: What is the pause container (sandbox container) and why does every pod have one?**

**A:** The pause container (also called the "infra container") is a minimal container (a few KB binary) that holds the network namespace for a pod. Every pod gets a pause container first (via `RunPodSandbox()`). All other containers in the pod join this network namespace, giving them a shared network stack (shared IP, localhost communication). If one app container dies and restarts, it rejoins the existing pause container's network namespace — so the pod IP doesn't change. The pause container runs forever (just `pause()` syscall) and is the anchor of the pod.

---

**Q6: What is the containerd shim and why does it exist?**

**A:** The containerd shim (`containerd-shim-runc-v2`) is a lightweight process that runs as the parent of each container process. It exists to decouple container lifecycle from containerd's lifecycle: if containerd crashes or is upgraded and restarted, running containers are not affected because their direct parent (the shim) is still alive. The shim also: (1) manages container stdio (logging); (2) reaps zombie child processes; (3) reports exit status back to containerd when it reconnects. There is one shim process per container.

---

**Q7: How does the container engine handle image layers and why is it efficient?**

**A:** Container images are composed of read-only layers (OCI specification). The container engine stores these layers content-addressed (by SHA256 hash) so layers can be shared across multiple images and containers. OverlayFS is used to merge these layers into a single filesystem view for each container. A writable layer (upperdir) is added on top for each container instance. When pulling an image, only layers not already present are downloaded — if the base OS layer is shared with 10 other images, it's only stored once. This dramatically reduces both disk space and pull time.

---

**Q8: What happens to containers when the container engine (containerd) crashes?**

**A:** Running containers continue running because they are child processes of the containerd-shim, not of containerd itself. The shims survive the containerd crash. However: (1) the Kubelet loses its CRI connection and can't receive container state updates; (2) PLEG relists fail; (3) after ~3 minutes of PLEG failure, the Kubelet marks itself unhealthy; (4) the node is marked NotReady. When containerd restarts, it reconnects to existing shims and resumes management. The Kubelet reconnects and resumes normal operation. No containers are lost in a clean containerd restart.

---

### Advanced Level

**Q9: Explain the full pod creation sequence from API server to running container.**

**A:** (1) User submits pod manifest; (2) kube-apiserver validates and stores in etcd (pod status: Pending); (3) kube-scheduler watches for unscheduled pods, scores nodes, assigns `spec.nodeName`; (4) Kubelet watches pods on its node, detects new assignment; (5) Kubelet calls `ImageService.PullImage()` — containerd checks cache, pulls from registry if needed; (6) Kubelet calls `RuntimeService.RunPodSandbox()` — containerd creates network namespace, calls CNI plugin for IP assignment; (7) Kubelet calls `RuntimeService.CreateContainer()` — containerd creates OCI bundle (config.json + rootfs overlay), applies security context; (8) Kubelet calls `RuntimeService.StartContainer()` — containerd calls `containerd-shim-runc-v2`, which calls runc; (9) runc calls `clone()` with namespace flags, sets up cgroups, applies seccomp, executes container entrypoint; (10) Kubelet detects running state via PLEG, updates pod status via API server.

---

**Q10: How does RuntimeClass work and what security benefits does it provide?**

**A:** RuntimeClass is a Kubernetes API object that maps a name (like "kata") to a container runtime handler configured in containerd or CRI-O. When a pod's `spec.runtimeClassName` references a RuntimeClass, the Kubelet passes the handler name in the `RunPodSandbox()` CRI call, and the container engine uses the corresponding OCI runtime instead of the default (runc). Security benefits: (1) **VM-based isolation** (kata-containers): each pod runs in its own lightweight VM — a container escape can't reach the host kernel; (2) **Syscall interception** (gVisor/runsc): a user-space kernel intercepts all syscalls — the container never directly touches the Linux kernel; (3) **Security tiering**: trusted internal services use fast runc, untrusted customer code uses kata or gVisor.

---

**Q11: How would you diagnose and fix a "PLEG is not healthy" situation in production?**

**A:** PLEG (Pod Lifecycle Event Generator) fails when the container runtime (CRI) can't respond to list-container queries within 3 minutes. Diagnosis steps: (1) Check `journalctl -u kubelet | grep PLEG` for "pleg was last seen active Xs ago"; (2) Test CRI responsiveness: `time crictl ps` — if this hangs > 5s, containerd is overloaded; (3) Check containerd logs: `journalctl -u containerd | grep -E "error|slow|timeout"`; (4) Check system resources: `top`, `iostat -x 1`, `df -h /var/lib/containerd`; (5) Check shim count: `ps aux | grep containerd-shim | wc -l` — thousands of shims can slow PLEG relist; (6) Check for disk I/O saturation (overlayfs operations are I/O intensive). Fix: (a) if disk full: GC images `crictl rmi --prune`; (b) if too many containers: reduce pod density; (c) if containerd hung: `systemctl restart containerd`; (d) if kernel/overlayfs bug: upgrade kernel; (e) long-term: switch to crun (faster than runc), separate image disk, tune GC settings.

---

**Q12: Explain how seccomp, AppArmor, and Linux capabilities are applied by the container engine.**

**A:** All three are applied during container creation via the OCI runtime spec (config.json): (1) **seccomp**: The container engine writes a seccomp BPF filter specification into config.json's `linux.seccomp` field. runc calls `prctl(PR_SET_SECCOMP)` to attach the BPF filter, which then intercepts every syscall from the container and allows/denies based on the profile. `RuntimeDefault` allows ~300 safe syscalls; (2) **AppArmor**: The OCI spec includes `linux.process.apparmorProfile`. runc writes the profile name to the container's `/proc/<pid>/attr/exec` before `exec()`ing the entrypoint, causing the Linux kernel to enforce the AppArmor policy from process start; (3) **Capabilities**: OCI spec specifies `process.capabilities` (permitted, effective, inheritable, bounding sets). runc calls `capset()` to reduce the process's capabilities to only what's specified. The Kubelet translates pod `securityContext.capabilities` into these OCI capability sets. All three mechanisms are kernel-enforced — the container engine configures them but the kernel enforces them.

---

## 28. Cheat Sheet: Commands & Flags

### crictl Commands (Container Runtime CLI)

```bash
# === CONTAINERS ===
crictl ps                          # List running containers
crictl ps -a                       # List all containers (including stopped)
crictl ps -a --state=exited        # List only stopped containers
crictl inspect <container-id>      # Inspect container (full OCI spec + state)
crictl logs <container-id>         # Get container logs
crictl logs -f <container-id>      # Follow container logs
crictl exec <container-id> <cmd>   # Execute command in container
crictl exec -it <container-id> sh  # Interactive shell
crictl stop <container-id>         # Stop container
crictl rm <container-id>           # Remove stopped container

# === PODS (SANDBOXES) ===
crictl pods                        # List pod sandboxes
crictl pods --name=<pod-name>      # Filter by pod name
crictl pods --namespace=<ns>       # Filter by namespace
crictl inspectp <pod-id>           # Inspect pod sandbox
crictl stopp <pod-id>              # Stop pod sandbox
crictl rmp <pod-id>                # Remove pod sandbox

# === IMAGES ===
crictl images                      # List images
crictl pull <image>                # Pull image
crictl pull --creds user:pass <img># Pull with credentials
crictl rmi <image>                 # Remove image
crictl rmi --prune                 # Remove all unused images
crictl inspecti <image>            # Inspect image

# === INFO ===
crictl info                        # Runtime info (version, config)
crictl version                     # CRI version
crictl stats                       # Container resource stats
crictl statsp                      # Pod resource stats

# === RUNTIME ENDPOINT ===
crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps
crictl --runtime-endpoint unix:///var/run/crio/crio.sock ps
```

### ctr Commands (containerd Native CLI)

```bash
# Always specify --namespace k8s.io for Kubernetes containers:
NAMESPACE="--namespace k8s.io"

# === IMAGES ===
ctr $NAMESPACE images list         # List images
ctr $NAMESPACE images pull nginx   # Pull image
ctr $NAMESPACE images push <img>   # Push image
ctr $NAMESPACE images export img.tar <image>  # Export image
ctr $NAMESPACE images import img.tar          # Import image
ctr $NAMESPACE images rm <image>   # Remove image

# === CONTAINERS ===
ctr $NAMESPACE containers list     # List containers
ctr $NAMESPACE containers info <id># Container details

# === SNAPSHOTS ===
ctr $NAMESPACE snapshots list      # List snapshots (layers)
ctr $NAMESPACE snapshots info <key># Snapshot details

# === TASKS (running processes) ===
ctr $NAMESPACE tasks list          # List running tasks (containers)
ctr $NAMESPACE tasks kill <id> --signal=SIGKILL

# === CONTENT (image blobs) ===
ctr $NAMESPACE content list        # List content blobs
ctr $NAMESPACE content active      # List active ingests (pulling)
```

### containerd Management Commands

```bash
# Service management:
systemctl status containerd        # Check service status
systemctl start containerd         # Start containerd
systemctl stop containerd          # Stop containerd
systemctl restart containerd       # Restart containerd
journalctl -u containerd -f        # Follow logs
journalctl -u containerd --since "10 minutes ago"

# Configuration:
containerd config default > /etc/containerd/config.toml  # Generate default config
containerd config dump             # Show current effective config

# Metrics:
curl http://localhost:1338/v1/metrics  # Prometheus metrics

# Disk usage:
du -sh /var/lib/containerd/
du -sh /var/lib/containerd/io.containerd.content.v1.content/  # Image blobs
du -sh /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/  # Snapshots
```

### Key Kubelet Flags for Container Engine

```yaml
# /var/lib/kubelet/config.yaml — Container engine relevant settings:

containerRuntimeEndpoint: "unix:///run/containerd/containerd.sock"
imageServiceEndpoint: "unix:///run/containerd/containerd.sock"
cgroupDriver: systemd              # Must match containerd's SystemdCgroup

# Image management:
serializeImagePulls: false         # Allow parallel pulls
maxParallelImagePulls: 3           # Max simultaneous pulls
imageGCHighThresholdPercent: 85    # Start GC at 85% disk
imageGCLowThresholdPercent: 80     # GC until 80% disk
imageMinimumGCAge: 2m              # Min age before GC

# Container logs:
containerLogMaxSize: "50Mi"        # Per-file log size limit
containerLogMaxFiles: 5            # Max rotated files

# Security:
seccompDefault: true               # RuntimeDefault seccomp for all pods

# Pods:
maxPods: 110                       # Max pods per node
```

### Useful Debug Commands

```bash
# Check which node a pod is on:
kubectl get pod <pod> -o wide

# Get container ID for a pod:
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].containerID}'
# Output: containerd://abc123...

# Get pod UID (used in cgroup paths):
kubectl get pod <pod> -o jsonpath='{.metadata.uid}'

# Check cgroup for a container:
CONTAINER_ID=$(crictl ps | grep <name> | awk '{print $1}')
CGROUP=$(crictl inspect $CONTAINER_ID | jq -r '.info.runtimeSpec.linux.cgroupsPath')
cat /sys/fs/cgroup/$CGROUP/memory.max

# Check OverlayFS mount for container rootfs:
mount | grep overlay | grep <container-id-prefix>

# Find container process on host:
crictl inspect <container-id> | jq '.info.pid'
nsenter --target <pid> --mount --pid --net -- ps aux  # Enter container namespaces

# Check what kernel namespaces a container uses:
crictl inspect <container-id> | jq '.info.runtimeSpec.linux.namespaces'

# Verify image signature:
cosign verify --key cosign.pub <image>

# Check image layers:
ctr --namespace k8s.io content ls | grep <image-digest>

# Force image GC via Kubelet:
curl -X POST -k --cert /var/lib/kubelet/pki/kubelet.crt \
  --key /var/lib/kubelet/pki/kubelet.key \
  https://localhost:10250/imagegc
```

### containerd config.toml Key Sections Reference

```toml
# /etc/containerd/config.toml — Key settings reference

version = 2

[grpc]
  address = "/run/containerd/containerd.sock"    # CRI socket path

[metrics]
  address = "127.0.0.1:1338"                    # Prometheus metrics

[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "registry.k8s.io/pause:3.9"   # Pause container image

  [plugins."io.containerd.grpc.v1.cri".containerd]
    snapshotter = "overlayfs"                    # Filesystem snapshotter
    default_runtime_name = "runc"               # Default OCI runtime

    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
      runtime_type = "io.containerd.runc.v2"

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        SystemdCgroup = true                     # CRITICAL: match kubelet cgroupDriver

  [plugins."io.containerd.grpc.v1.cri".registry]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
      endpoint = ["https://mirror.example.com"]

    [plugins."io.containerd.grpc.v1.cri".registry.configs."registry.example.com".auth]
      username = "user"
      password = "pass"
```

---

## 29. Key Takeaways & Summary

### The Essential Mental Model

The container engine is the **execution layer** of Kubernetes — the component that bridges the declarative world of Kubernetes API objects and the imperative world of Linux kernel primitives. When you `kubectl apply` a deployment, you're writing into etcd. When that deployment's pod actually runs as a process with a filesystem, network, and resource limits, that's the container engine's work.

### The 12 Most Critical Things to Know About Container Engines

1. **The CRI is the contract** — any CRI-compliant runtime (containerd, CRI-O) is interchangeable from Kubernetes's perspective. Kubernetes calls `RunPodSandbox()`, `CreateContainer()`, `StartContainer()` — it doesn't care what's underneath.

2. **Container engines never talk to etcd or the API server** — their world is entirely local. All cluster state flows: etcd → API server → Kubelet → Container Engine (CRI calls).

3. **Two layers: high-level runtime + low-level OCI runtime** — containerd manages images, snapshots, and shims; runc (or crun, kata) actually creates isolated processes using kernel primitives.

4. **The pause container is the pod's anchor** — every pod gets a pause container first that holds the network namespace. All app containers join this namespace. Pod IP is the pause container's IP.

5. **The shim decouples container lifecycle from containerd** — one `containerd-shim-runc-v2` per container ensures containers survive a containerd restart/upgrade.

6. **OverlayFS enables image layer sharing** — image layers are stored once and shared across multiple containers. Only the per-container writable layer is unique. This is the foundation of image efficiency.

7. **cgroup driver MUST match between containerd and Kubelet** — `SystemdCgroup = true` in containerd must align with `cgroupDriver: systemd` in Kubelet config. Mismatch causes silent resource accounting errors.

8. **PLEG health is the proxy for container runtime health** — PLEG relist duration tells you if your CRI is responsive. High values (>3s) are an early warning; >3min causes node NotReady.

9. **Security is enforced at the OCI runtime level** — seccomp BPF filters, AppArmor profiles, and Linux capabilities are applied by runc via the OCI spec. Kubernetes security policies are only effective if the container engine correctly translates them into the OCI spec.

10. **RuntimeClass enables security tiering** — use runc for trusted workloads, kata-containers or gVisor for untrusted/sensitive workloads, all on the same cluster with no changes to the application.

11. **Image pull is a hidden latency source** — pre-pull images, use registry mirrors, enable parallel pulls (`serializeImagePulls: false`), and use slim base images. Image pull is often the largest contributor to pod startup time.

12. **Container engines are recoverable without data loss** — their local state (image cache, metadata) is derived from etcd + container registries. Node recovery requires only restarting the service, not restoring data.

### Quick Decision Reference

```
Pod stuck in ContainerCreating?
  → Check: kubectl describe pod (Events section)
  → On node: crictl ps -a | grep <pod>
             journalctl -u containerd | grep error
  → Image pull issue? → crictl pull <image> manually
  → CNI issue? → check /etc/cni/net.d/, crictl pods

Node NotReady after 40s?
  → Check: systemctl status containerd
  → If stopped: systemctl start containerd
  → If hung: systemctl restart containerd
  → Check PLEG: time crictl ps (should be < 1s)
  → Check disk: df -h /var/lib/containerd

OOMKilled container?
  → Check: kubectl describe pod | grep OOMKilled
  → Check: container_memory_working_set_bytes (Prometheus)
  → Fix: increase memory limit in pod spec

Image pull slow?
  → Enable: serializeImagePulls: false
  → Configure: registry mirror in containerd config.toml
  → Pre-pull: DaemonSet or node bootstrap script

Security concern with container?
  → Enforce: runAsNonRoot: true, readOnlyRootFilesystem: true
  → Drop: capabilities.drop: [ALL]
  → Enable: seccompDefault: true
  → Isolate: runtimeClassName: kata (for high-security)
```

---

*This document represents production-grade Kubernetes container engine knowledge compiled for DevOps engineers, SREs, and platform engineers managing containerized workloads at scale. All configurations should be validated in a non-production environment before applying to production clusters.*

---

**Document Version:** 1.0
**Last Updated:** April 2026
**Kubernetes Reference Version:** v1.28+
**Container Engine Reference Versions:** containerd v1.7+, CRI-O v1.28+

---
*End of Document*
