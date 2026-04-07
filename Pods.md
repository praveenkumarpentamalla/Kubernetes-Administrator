## Table of Contents

1. [Introduction & Strategic Importance](#1-introduction--strategic-importance)
2. [Core Identity Table](#2-core-identity-table)
3. [ASCII Architecture Diagram](#3-ascii-architecture-diagram)
4. [Pod Anatomy: Deep Structural Breakdown](#4-pod-anatomy-deep-structural-breakdown)
5. [The Controller Pattern: Watch → Compare → Act → Loop](#5-the-controller-pattern-watch--compare--act--loop)
6. [Deep Dive: Major Built-in Controllers & Pod Management](#6-deep-dive-major-built-in-controllers--pod-management)
7. [Pod Lifecycle: Phases, Conditions & Container States](#7-pod-lifecycle-phases-conditions--container-states)
8. [Init Containers, Sidecar Containers & Ephemeral Containers](#8-init-containers-sidecar-containers--ephemeral-containers)
9. [Pod Networking: IPs, DNS & Inter-Pod Communication](#9-pod-networking-ips-dns--inter-pod-communication)
10. [Pod Storage: Volumes, PVCs & Projected Volumes](#10-pod-storage-volumes-pvcs--projected-volumes)
11. [Internal Working Concepts: Reconciliation, Informer Pattern & Work Queues](#11-internal-working-concepts-reconciliation-informer-pattern--work-queues)
12. [Leader Election in Pod Management Context](#12-leader-election-in-pod-management-context)
13. [Interaction with API Server and etcd](#13-interaction-with-api-server-and-etcd)
14. [Resource Management: Requests, Limits & QoS](#14-resource-management-requests-limits--qos)
15. [Pod Scheduling: Affinity, Taints & Topology](#15-pod-scheduling-affinity-taints--topology)
16. [Security Hardening Practices](#16-security-hardening-practices)
17. [Performance Tuning & Configuration](#17-performance-tuning--configuration)
18. [Monitoring & Observability](#18-monitoring--observability)
19. [Troubleshooting Guide with Real Commands](#19-troubleshooting-guide-with-real-commands)
20. [Comparison: Pods vs kube-apiserver vs kube-scheduler Perspective](#20-comparison-pods-vs-kube-apiserver-vs-kube-scheduler-perspective)
21. [Disaster Recovery Concepts](#21-disaster-recovery-concepts)
22. [Real-World Production Use Cases](#22-real-world-production-use-cases)
23. [Best Practices for Production Environments](#23-best-practices-for-production-environments)
24. [Common Mistakes and Pitfalls](#24-common-mistakes-and-pitfalls)
25. [Hands-On Labs & Mini Practical Exercises](#25-hands-on-labs--mini-practical-exercises)
26. [Interview Questions: Beginner to Advanced](#26-interview-questions-beginner-to-advanced)
27. [Cheat Sheet: Commands & Flags](#27-cheat-sheet-commands--flags)
28. [Key Takeaways & Summary](#28-key-takeaways--summary)

---

## 1. Introduction & Strategic Importance

The **Pod** is the most fundamental building block of Kubernetes. Every workload that runs in a Kubernetes cluster — whether it is a web server, a database, a batch job, or an infrastructure agent — ultimately runs inside a Pod. Understanding Pods deeply is not optional for anyone who operates Kubernetes in production; it is the prerequisite for understanding every other Kubernetes concept.

### What Is a Pod?

A Pod is the **smallest deployable unit** in Kubernetes. It is a logical abstraction that wraps one or more containers (typically one) and gives them:

- A **shared network namespace** — all containers in the Pod share the same IP address and port space
- **Shared storage** — volumes are mounted at the Pod level and accessible to all containers
- A **shared lifecycle** — containers in a Pod are always co-scheduled on the same node, start together, and stop together
- A **shared Linux namespace context** — UTS namespace (hostname), IPC namespace (shared memory), and optionally PID namespace

Critically, a Pod is **not** a container. It is a layer of abstraction above containers that provides the primitives needed for co-located, tightly-coupled processes to work together — the way they would on a traditional virtual machine or physical server, but in a containerized world.

### The Philosophical Purpose of Pods

The Pod design answers a fundamental question: *What is the right unit of deployment?* A single container is too granular — it doesn't account for patterns where processes must be co-located (like a web server and its log shipper). A full VM is too coarse. The Pod is the answer: a group of containers that **must** run together because they form a single cohesive unit of work.

```
Traditional VM model:           Kubernetes Pod model:
  One VM, many processes          One Pod, one-or-few containers
  Shared IP, shared disk          Shared IP, shared volumes
  OS-level process isolation      Container-level isolation
  Hard to version/scale           Easy to scale, replace, version
```

### Why Pods Matter in Production

- **They are the atom of Kubernetes scheduling** — the scheduler assigns Pods to nodes, not containers.
- **They are the unit of resource management** — CPU/memory requests and limits are defined per container within a Pod.
- **They are ephemeral by design** — Pods are cattle, not pets. They can be killed and replaced at any time.
- **Their lifecycle directly impacts service availability** — understanding Pod startup, readiness, and termination is what enables zero-downtime deployments.
- **Every production incident eventually traces to Pod behavior** — OOMKills, CrashLoopBackOffs, pending Pods, evictions — all are Pod-level phenomena.
- **Security is enforced at the Pod level** — SecurityContext, ServiceAccounts, and network policies all apply at the Pod boundary.

### Evolution of Pods

```
Kubernetes 1.0 (2014):  Pods introduced as the fundamental unit
Kubernetes 1.3:         Init containers introduced (alpha)
Kubernetes 1.6:         Init containers stable
Kubernetes 1.14:        Startup probes added
Kubernetes 1.18:        Ephemeral containers (alpha) for debugging
Kubernetes 1.23:        Ephemeral containers stable
Kubernetes 1.28:        Sidecar containers (init containers with restart policy)
Kubernetes 1.29+:       Sidecar containers stable
```

---

## 2. Core Identity Table

| Field | Value |
|---|---|
| **API Version** | `v1` |
| **Kind** | `Pod` |
| **API Group** | Core group (no group prefix) |
| **Namespaced** | Yes |
| **REST API Path** | `/api/v1/namespaces/{namespace}/pods/{name}` |
| **Status Subresource** | `/api/v1/namespaces/{namespace}/pods/{name}/status` |
| **Exec Subresource** | `/api/v1/namespaces/{namespace}/pods/{name}/exec` |
| **Log Subresource** | `/api/v1/namespaces/{namespace}/pods/{name}/log` |
| **Created By** | kubelet (static pods) or controllers (ReplicaSet, etc.) |
| **Scheduled By** | kube-scheduler |
| **Run By** | kubelet + container engine (containerd/CRI-O) |
| **Lifecycle Controller** | kubelet (on the assigned node) |
| **Unique Identifier** | `metadata.uid` (UUID) |
| **Network Identity** | Pod IP (assigned by CNI plugin) |
| **Storage** | Volumes (defined in `spec.volumes`) |
| **Restart Policy** | `Always` / `OnFailure` / `Never` |
| **Termination Grace Period** | `spec.terminationGracePeriodSeconds` (default: 30s) |
| **Resource Isolation** | cgroups (CPU, memory, PIDs) |
| **Namespace Isolation** | Linux namespaces (net, pid, ipc, uts, mnt) |
| **DNS Name** | `<pod-ip-dashes>.<namespace>.pod.cluster.local` |
| **Max Pods per Node** | 110 (default kubelet), configurable |
| **Immutability** | Most spec fields are immutable after creation |
| **Primary Status Field** | `status.phase`: Pending, Running, Succeeded, Failed, Unknown |

---

## 3. ASCII Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║                    KUBERNETES POD ARCHITECTURE & CONTEXT                         ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║                                                                                  ║
║   ┌─────────────────────────── CONTROL PLANE ──────────────────────────────┐   ║
║   │                                                                          │   ║
║   │  ┌──────────────────┐  ┌──────────────────────────────────────────────┐│   ║
║   │  │  kube-apiserver  │  │         kube-controller-manager              ││   ║
║   │  │                  │  │  ┌──────────────┐   ┌────────────────────┐  ││   ║
║   │  │ Pod objects      │  │  │ReplicaSet Ctrl│   │  Deployment Ctrl   │  ││   ║
║   │  │ stored in etcd   │  │  │  watches pods │   │  watches RS + pods │  ││   ║
║   │  │                  │  │  └──────────────┘   └────────────────────┘  ││   ║
║   │  │ /api/v1/pods     │  │  ┌──────────────┐   ┌────────────────────┐  ││   ║
║   │  │ /api/v1/nodes    │  │  │ StatefulSet  │   │   DaemonSet Ctrl   │  ││   ║
║   │  │ /api/v1/services │  │  │    Ctrl      │   │                    │  ││   ║
║   │  └────────┬─────────┘  │  └──────────────┘   └────────────────────┘  ││   ║
║   │           │            │  ┌──────────────┐   ┌────────────────────┐  ││   ║
║   │  ┌────────▼────────┐   │  │  Job Ctrl    │   │  GC Controller     │  ││   ║
║   │  │     etcd        │   │  └──────────────┘   └────────────────────┘  ││   ║
║   │  │ (source of      │   └──────────────────────────────────────────────┘│   ║
║   │  │  truth for      │                                                    │   ║
║   │  │  all pod specs) │   ┌────────────────────────────────────────────┐  │   ║
║   │  └─────────────────┘   │           kube-scheduler                   │  │   ║
║   │                        │   Watches: Pods with no spec.nodeName       │  │   ║
║   │                        │   Writes:  spec.nodeName = "worker-1"       │  │   ║
║   │                        └────────────────────────────────────────────┘  │   ║
║   └──────────────────────────────────────────────────────────────────────────┘   ║
║                                        │ HTTPS Watch                             ║
║   ┌────────────────────────────────────┼─────────────────────────────────────┐  ║
║   │                          WORKER NODE                                      │  ║
║   │                                    │                                      │  ║
║   │   ┌────────────────────────────────▼──────────────────────────────────┐  │  ║
║   │   │                        KUBELET :10250                              │  │  ║
║   │   │   Watches Pods assigned to THIS node (spec.nodeName=worker-1)      │  │  ║
║   │   │   Drives Pod lifecycle via CRI, CNI, CSI                           │  │  ║
║   │   └───────────────────────────────────────────────────────────────────-┘  │  ║
║   │                                    │                                      │  ║
║   │   ┌─────────────────────────────────────────────────────────────────────┐ │  ║
║   │   │                         POD (unit of work)                           │ │  ║
║   │   │                                                                      │ │  ║
║   │   │  ┌─────────────────────────────────────────────────────────────┐    │ │  ║
║   │   │  │              PAUSE CONTAINER (infra container)              │    │ │  ║
║   │   │  │  Holds:  Network Namespace (Pod IP: 10.244.1.5)             │    │ │  ║
║   │   │  │          IPC Namespace, UTS Namespace (hostname=pod-name)   │    │ │  ║
║   │   │  └─────────────────────────────────────────────────────────────┘    │ │  ║
║   │   │                                                                      │ │  ║
║   │   │  ┌───────────────────┐  ┌───────────────────┐  ┌────────────────┐  │ │  ║
║   │   │  │  Init Container   │  │  App Container    │  │Sidecar Container│ │ │  ║
║   │   │  │  (runs first,     │  │  (main workload)  │  │(log shipper,   │ │ │  ║
║   │   │  │   exits before    │  │                   │  │ proxy, etc.)   │ │ │  ║
║   │   │  │   app starts)     │  │  Port: 8080       │  │ Port: 9090     │ │ │  ║
║   │   │  └───────────────────┘  └────────┬──────────┘  └────────────────┘  │ │  ║
║   │   │                                  │ shares network/storage            │ │  ║
║   │   │  ┌───────────────────────────────▼──────────────────────────────┐  │ │  ║
║   │   │  │                    SHARED VOLUMES                             │  │ │  ║
║   │   │  │  emptyDir / configMap / secret / PVC / projected             │  │ │  ║
║   │   │  └───────────────────────────────────────────────────────────────┘  │ │  ║
║   │   │                                                                      │ │  ║
║   │   │  Pod IP: 10.244.1.5  │  Node: worker-1  │  Namespace: production    │ │  ║
║   │   └─────────────────────────────────────────────────────────────────────┘ │  ║
║   │                                                                            │  ║
║   │   ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────────┐   │  ║
║   │   │  CNI Plugin     │  │  Container Engine │  │  cgroups (kernel)    │   │  ║
║   │   │  (Pod Network)  │  │  (containerd)     │  │  CPU/Memory limits   │   │  ║
║   │   └─────────────────┘  └──────────────────┘  └──────────────────────┘   │  ║
║   └────────────────────────────────────────────────────────────────────────────┘  ║
╚══════════════════════════════════════════════════════════════════════════════════╝

Pod Creation Flow:
  1. Controller (ReplicaSet) creates Pod object in API server (spec.nodeName = "")
  2. kube-scheduler assigns Pod to a node (writes spec.nodeName = "worker-1")
  3. Kubelet on worker-1 watches, detects the assigned Pod
  4. Kubelet calls CRI → containerd: PullImage, RunPodSandbox, CreateContainer, Start
  5. CNI plugin sets up network namespace, assigns Pod IP
  6. CSI plugin mounts volumes
  7. Kubelet runs liveness/readiness/startup probes
  8. Pod status updated to Running via API server
```

---

## 4. Pod Anatomy: Deep Structural Breakdown

A Pod is defined by a YAML/JSON manifest. Every field has production implications.

### Complete Annotated Pod Spec

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: production-app                         # Must be unique within namespace
  namespace: production                        # Logical isolation boundary
  labels:                                      # Used by selectors, Services, NetworkPolicy
    app: web-frontend
    version: v1.2.3
    environment: production
    tier: frontend
  annotations:                                 # Non-selector metadata
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
    deployment.kubernetes.io/revision: "3"
  ownerReferences:                             # Set by controller (e.g., ReplicaSet)
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: web-frontend-abc123
    uid: <uid>
    controller: true
    blockOwnerDeletion: true

spec:
  # ── SCHEDULING ────────────────────────────────────────────────
  nodeName: ""                                 # Set by scheduler; empty = not scheduled
  nodeSelector:                                # Simple node selection
    kubernetes.io/os: linux
    node.kubernetes.io/instance-type: m5.large

  affinity:                                    # Advanced scheduling rules
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values: [us-east-1a, us-east-1b]
    podAntiAffinity:                           # Don't co-locate with same app
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web-frontend
          topologyKey: kubernetes.io/hostname

  tolerations:                                 # Allow scheduling on tainted nodes
  - key: "node.kubernetes.io/not-ready"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 300

  topologySpreadConstraints:                   # Spread across failure domains
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web-frontend

  priorityClassName: high-priority             # Preemption priority
  schedulerName: default-scheduler             # Or custom scheduler

  # ── IDENTITY & ACCESS ─────────────────────────────────────────
  serviceAccountName: web-frontend-sa          # Identity for API server auth
  automountServiceAccountToken: true           # Mount SA token as projected volume

  # ── NETWORKING ────────────────────────────────────────────────
  hostNetwork: false                           # Use node's network namespace
  hostPID: false                               # Share host PID namespace
  hostIPC: false                               # Share host IPC namespace
  dnsPolicy: ClusterFirst                      # DNS resolution order
  dnsConfig:
    nameservers: []
    searches: []
    options:
    - name: ndots
      value: "5"

  # ── LIFECYCLE ─────────────────────────────────────────────────
  restartPolicy: Always                        # Always | OnFailure | Never
  terminationGracePeriodSeconds: 30            # SIGTERM → SIGKILL grace window
  activeDeadlineSeconds: null                  # Max runtime (for Jobs)

  # ── SECURITY ──────────────────────────────────────────────────
  securityContext:                             # Pod-level (applies to all containers)
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 2000                              # Volume ownership
    fsGroupChangePolicy: OnRootMismatch        # Efficient ownership change
    seccompProfile:
      type: RuntimeDefault
    sysctls:
    - name: net.core.somaxconn
      value: "1024"

  # ── INIT CONTAINERS ───────────────────────────────────────────
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']
    resources:
      requests: { cpu: "10m", memory: "16Mi" }
      limits: { cpu: "50m", memory: "32Mi" }

  # ── MAIN CONTAINERS ───────────────────────────────────────────
  containers:
  - name: web-frontend
    image: myrepo/web-frontend:v1.2.3          # Always use specific tag or digest
    imagePullPolicy: IfNotPresent              # Always | IfNotPresent | Never

    ports:
    - name: http
      containerPort: 8080
      protocol: TCP
    - name: metrics
      containerPort: 9090
      protocol: TCP

    env:
    - name: DB_HOST
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: host
    - name: LOG_LEVEL
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: log_level
    - name: POD_NAME                           # Downward API
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          fieldPath: spec.nodeName

    resources:
      requests:
        cpu: "250m"                            # Guaranteed minimum
        memory: "256Mi"
        ephemeral-storage: "1Gi"
      limits:
        cpu: "1000m"                           # Maximum allowed
        memory: "512Mi"
        ephemeral-storage: "2Gi"

    volumeMounts:
    - name: config
      mountPath: /app/config
      readOnly: true
    - name: data
      mountPath: /app/data
    - name: tmp
      mountPath: /tmp

    securityContext:                           # Container-level (overrides pod-level)
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsUser: 1000
      capabilities:
        drop: [ALL]
        add: [NET_BIND_SERVICE]

    lifecycle:
      preStop:                                 # Run before SIGTERM
        exec:
          command: ["/bin/sh", "-c", "sleep 5 && nginx -s quit"]
      postStart:                               # Run after container starts
        exec:
          command: ["/bin/sh", "-c", "echo 'started' > /tmp/started"]

    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 30
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 3
      successThreshold: 1

    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
      timeoutSeconds: 3
      failureThreshold: 3
      successThreshold: 1

    startupProbe:                              # Blocks liveness until passed
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30
      periodSeconds: 10                        # Max 300s startup time

    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: FallbackToLogsOnError

  # ── VOLUMES ───────────────────────────────────────────────────
  volumes:
  - name: config
    configMap:
      name: app-config
      defaultMode: 0440
  - name: data
    persistentVolumeClaim:
      claimName: web-frontend-data
  - name: tmp
    emptyDir:
      medium: Memory
      sizeLimit: 100Mi

  # ── IMAGE PULL ────────────────────────────────────────────────
  imagePullSecrets:
  - name: registry-credentials

  # ── RUNTIME ───────────────────────────────────────────────────
  runtimeClassName: runc                       # or: kata-containers, gvisor

status:
  phase: Running                               # Pending|Running|Succeeded|Failed|Unknown
  podIP: 10.244.1.5
  podIPs:
  - ip: 10.244.1.5
  hostIP: 192.168.1.10
  startTime: "2024-01-15T10:00:00Z"
  conditions:
  - type: PodScheduled       status: "True"
  - type: Initialized        status: "True"
  - type: ContainersReady    status: "True"
  - type: Ready              status: "True"
  containerStatuses:
  - name: web-frontend
    ready: true
    restartCount: 0
    state:
      running:
        startedAt: "2024-01-15T10:00:30Z"
    image: myrepo/web-frontend:v1.2.3
    imageID: docker-pullable://myrepo/web-frontend@sha256:abc123
    containerID: containerd://abc123def456
```

---

## 5. The Controller Pattern: Watch → Compare → Act → Loop

Pods are almost never created directly in production. They are created, monitored, and managed by **controllers** that follow the Kubernetes control loop pattern. Understanding this pattern explains why Pods are self-healing.

### The Reconciliation Control Loop

```
┌─────────────────────────────────────────────────────────────────────┐
│                   CONTROLLER RECONCILIATION LOOP                     │
│              (drives all Pod creation and management)                │
│                                                                       │
│  ┌──────────────┐     ┌──────────────────────┐     ┌─────────────┐  │
│  │    WATCH     │────►│      COMPARE         │────►│     ACT     │  │
│  │              │     │                      │     │             │  │
│  │ API Server   │     │ Desired Pods         │     │ Create Pod  │  │
│  │ for Pods,    │     │   (from controller   │     │ Delete Pod  │  │
│  │ ReplicaSets, │     │    spec/replicas)    │     │ Update Pod  │  │
│  │ Deployments  │     │        vs            │     │ Patch status│  │
│  │              │     │ Actual Pods          │     │             │  │
│  │              │     │   (running in cluster│     │             │  │
│  └──────────────┘     └──────────────────────┘     └──────┬──────┘  │
│          ▲                                                 │          │
│          └─────────────────────────────────────────────────┘          │
│                              LOOP                                      │
│                    (event-driven + periodic reconcile)                 │
└─────────────────────────────────────────────────────────────────────┘
```

### Phase 1: WATCH — Desired State from API Server

Controllers use the **Informer pattern** to maintain a local cache of relevant objects:

```go
// Conceptual: ReplicaSet controller informer setup
podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc:    func(obj interface{}) { rs.enqueueReplicaSet(obj) },
    UpdateFunc: func(old, new interface{}) { rs.enqueueReplicaSet(new) },
    DeleteFunc: func(obj interface{}) { rs.enqueueReplicaSet(obj) },
})
```

### Phase 2: COMPARE — Computing the Diff

```
ReplicaSet: desired replicas = 3
Actual pods matching selector: 2  (one was deleted by a node failure)

Diff: need to CREATE 1 more Pod
```

### Phase 3: ACT — Create/Delete/Update API Objects

```
Controller calls API server: POST /api/v1/namespaces/default/pods
API server validates, writes to etcd
Scheduler detects unscheduled pod → assigns node
Kubelet detects pod on its node → starts container
```

### Phase 4: LOOP — Continuous Reconciliation

```
Event-driven triggers:
  → Pod deleted externally
  → Node fails (pod goes to Unknown state)
  → ReplicaSet replicas changed (kubectl scale)
  → Pod fails liveness probe (container restarted)

Periodic full reconcile:
  → Every controller has a resync period (typically 10-30 minutes)
  → Catches any missed events, ensures consistency
```

---

## 6. Deep Dive: Major Built-in Controllers & Pod Management

### 6.1 ReplicaSet Controller

**Purpose**: Ensure exactly N identical Pod replicas are always running.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-frontend-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-frontend    # Must match pod template labels
  template:
    metadata:
      labels:
        app: web-frontend  # These labels identify pods owned by this RS
    spec:
      containers:
      - name: web
        image: nginx:1.25
```

**How it manages Pods:**

```
ReplicaSet controller loop:
  1. List pods matching .spec.selector
  2. Count: currentReplicas = len(matching pods)
  3. desiredReplicas = .spec.replicas = 3

  If currentReplicas < desiredReplicas:
    → Create (desiredReplicas - currentReplicas) new Pods
    → New pods inherit .spec.template
    → ownerReference set to this ReplicaSet

  If currentReplicas > desiredReplicas:
    → Delete excess pods (oldest first, then by lowest restart count)

  If currentReplicas == desiredReplicas:
    → No action; loop again on next event

Pod adoption: If an existing pod matches the selector and has no owner:
  → RS adopts it (sets ownerReference)
  → This prevents duplicate pod creation
```

**Production behavior:**

```bash
# Scale manually:
kubectl scale rs web-frontend-rs --replicas=5

# ReplicaSet doesn't do rolling updates — use Deployment for that
# If you update the pod template in a RS, existing pods are NOT updated
# Only NEW pods (created after the update) use the new template
```

### 6.2 Deployment Controller

**Purpose**: Manages ReplicaSets to provide rolling updates, rollbacks, and declarative updates.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # Allow 1 extra pod during update
      maxUnavailable: 0    # Never go below desired count (zero-downtime)
  selector:
    matchLabels:
      app: web-frontend
  template:
    metadata:
      labels:
        app: web-frontend
    spec:
      containers:
      - name: web
        image: nginx:1.25
```

**Rolling update Pod lifecycle:**

```
Initial state: RS-v1 has 3 pods running nginx:1.24

Update triggered (kubectl set image deployment/web nginx=nginx:1.25):

Step 1: Deployment controller creates RS-v2 (nginx:1.25), scales to 0
Step 2: RS-v2 scaled to 1 (maxSurge=1: total pods = 4 = 3+1)
Step 3: New pod becomes Ready (readinessProbe passes)
Step 4: RS-v1 scaled to 2 (maxUnavailable=0: still 3 ready pods)
Step 5: RS-v2 scaled to 2, RS-v1 scaled to 1
Step 6: RS-v2 scaled to 3, RS-v1 scaled to 0
Step 7: RS-v1 kept (for rollback), RS-v2 is the active RS

Rollback:
kubectl rollout undo deployment/web-frontend
→ RS-v1 scaled back to 3, RS-v2 scaled to 0
```

**Pod management flags:**

```bash
# Pause/resume rollout:
kubectl rollout pause deployment/web-frontend
kubectl rollout resume deployment/web-frontend

# Check rollout history:
kubectl rollout history deployment/web-frontend

# Rollback to specific revision:
kubectl rollout undo deployment/web-frontend --to-revision=2

# Watch rollout:
kubectl rollout status deployment/web-frontend -w
```

### 6.3 Node Controller

**Purpose**: Monitors node health; manages Pod eviction from failed nodes.

```
Node controller behavior for Pods:

Node heartbeat stops (kubelet lease not renewed):
  T+0:   Node stops renewing Lease
  T+40s: Node-lifecycle-controller marks node as Unknown
  T+5m:  Node-lifecycle-controller evicts Pods from Unknown node
         → Sets pod.status.conditions[Ready] = Unknown
         → Creates taint: node.kubernetes.io/unreachable:NoExecute
         → Pods with toleration: 300s → evicted after 5min
         → Pods without toleration → evicted immediately

Pod eviction:
  → Pod deleted from API server
  → If owned by ReplicaSet: RS controller creates replacement Pod
  → New Pod scheduled on healthy node
  → Kubelet on new node starts the replacement containers
```

```bash
# Check node conditions:
kubectl describe node <node-name> | grep -A 10 "Conditions:"

# View pods on a specific node:
kubectl get pods --field-selector spec.nodeName=<node-name> -A

# Manually evict a pod (respects PodDisruptionBudgets):
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
```

### 6.4 Service Controller (Endpoints)

**Purpose**: Ensures Services route to ready Pods via EndpointSlices.

```
Service → Pod connection via EndpointSlice controller:

  Service selector: app=web-frontend
  
  EndpointSlice controller:
    → Watches Pods and Services
    → When Pod becomes Ready AND labels match service selector:
        → Adds Pod IP:Port to EndpointSlice
    → When Pod becomes NotReady or Terminated:
        → Removes Pod IP:Port from EndpointSlice
    → kube-proxy reads EndpointSlices → programs iptables/IPVS

Critical for zero-downtime:
  Pod termination order:
  1. Pod marked for deletion
  2. Pod condition Ready=False (immediately)
  3. EndpointSlice: Pod IP removed (EndpointSlice controller reacts)
  4. kube-proxy: DNAT rule removed (reacts to EndpointSlice change)
  5. preStop hook runs (app drain time)
  6. SIGTERM sent to container
  7. Container shuts down (within terminationGracePeriodSeconds)
  8. SIGKILL if grace period exceeded
```

### 6.5 Namespace Controller

**Purpose**: Manages lifecycle of Pods when their namespace is deleted.

```
Namespace deletion cascade:
  kubectl delete namespace staging

  Namespace controller:
  1. Marks namespace as Terminating
  2. Removes all resources in namespace (Pods, Services, ConfigMaps, etc.)
  3. For each Pod: sets deletionTimestamp
  4. Kubelet detects pod deletion → stops containers
  5. Namespace finally deleted when all resources are gone

  Pods in Terminating namespace:
    → Cannot create new pods (namespace is terminating)
    → Existing pods get SIGTERM via kubelet
    → Force delete if finalizers block deletion:
       kubectl delete pod <name> --grace-period=0 --force
```

### 6.6 Job Controller

**Purpose**: Ensures a specified number of Pods run to successful completion.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
spec:
  completions: 5           # Run 5 successful completions total
  parallelism: 2           # Run 2 pods at a time
  backoffLimit: 4          # Retry up to 4 times per pod failure
  activeDeadlineSeconds: 600  # Kill job after 10 minutes
  template:
    spec:
      restartPolicy: OnFailure  # Never or OnFailure for Jobs
      containers:
      - name: migrate
        image: myapp:migrate
```

**Job Pod lifecycle:**

```
Job controller manages Pods differently from Deployment:

  restartPolicy: OnFailure:
    → Container exits with non-zero: kubelet restarts container (same pod)
    → After backoffLimit restarts: pod marked Failed, new pod created
    
  restartPolicy: Never:
    → Container exits with non-zero: pod marked Failed, new pod created immediately
    → Job controller tracks failures against backoffLimit

  Successful completion:
    → Container exits 0 → Pod phase = Succeeded
    → Job controller marks completion count
    → Keeps pod for log access (ttlSecondsAfterFinished cleans it up)

  CronJob:
    → Creates Job objects on schedule
    → Job controller creates Pods
    → Completed Jobs cleaned up per .spec.successfulJobsHistoryLimit
```

### 6.7 StatefulSet Controller

**Purpose**: Manages stateful Pods with stable network identity, ordered deployment, and persistent storage.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: "postgres"    # Headless service for stable DNS
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
        image: postgres:15
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:      # One PVC per pod, never shared
  - metadata:
      name: data
    spec:
      accessModes: [ReadWriteOnce]
      resources:
        requests:
          storage: 10Gi
```

**StatefulSet Pod identity:**

```
Pod names are ORDERED and STABLE:
  postgres-0    → DNS: postgres-0.postgres.default.svc.cluster.local
  postgres-1    → DNS: postgres-1.postgres.default.svc.cluster.local
  postgres-2    → DNS: postgres-2.postgres.default.svc.cluster.local

PVC names are STABLE (follow pod):
  data-postgres-0    (bound to postgres-0, always)
  data-postgres-1    (bound to postgres-1, always)
  data-postgres-2    (bound to postgres-2, always)

Ordered startup:
  postgres-0 must be Running+Ready BEFORE postgres-1 starts
  postgres-1 must be Running+Ready BEFORE postgres-2 starts

Ordered deletion (scale down):
  postgres-2 deleted first, then postgres-1, then postgres-0

Update strategy:
  RollingUpdate: Updates from highest ordinal to lowest (postgres-2 first)
  OnDelete: Only update when manually deleted
```

### 6.8 DaemonSet Controller

**Purpose**: Ensures exactly one Pod runs on every (or selected) node.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        effect: NoSchedule          # Run on control plane nodes too
      hostNetwork: true             # Use node's network
      hostPID: true                 # Access host PID namespace
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.6.1
        ports:
        - containerPort: 9100
          hostPort: 9100            # Binds to host network port
```

**DaemonSet Pod behavior:**

```
DaemonSet controller logic:
  For each node in cluster:
    If node matches nodeSelector/affinity:
      If no DaemonSet pod running on that node → CREATE pod
      If pod exists but spec changed → DELETE + CREATE (rolling update)
    
  New node joins cluster:
    → DaemonSet controller immediately creates pod on new node
    → No scheduler involvement (DaemonSet sets spec.nodeName directly)
    
  Node removed:
    → Pod is garbage collected (no restart/reschedule)

Unlike other controllers: DaemonSet does NOT use kube-scheduler for placement
  → DaemonSet controller sets spec.nodeName directly
  → Bypasses normal scheduling (needed for network plugins that MUST start before scheduler workloads)
```

### 6.9 Garbage Collector Controller

**Purpose**: Cleans up Pod objects when their owner (ReplicaSet, Job, etc.) is deleted.

```
Garbage Collector uses ownerReferences:

  ReplicaSet ownerReferences → Deployment
  Pod ownerReferences        → ReplicaSet

When Deployment deleted:
  Option 1: Cascade delete (default)
    → GC deletes ReplicaSet → deletes Pods
    kubectl delete deployment web --cascade=foreground

  Option 2: Orphan
    → RS and Pods remain, no owner
    kubectl delete deployment web --cascade=orphan

  Option 3: Background
    → Deployment object deleted immediately
    → GC async-cleans RS and Pods
    kubectl delete deployment web --cascade=background

Orphaned Pods:
  → Pod with no matching controller (owner deleted, orphaned)
  → Pod remains Running until node fails or manually deleted
  → NOT automatically restarted if it crashes
  → Can be re-adopted by new controller with matching selector
```

### 6.10 PersistentVolume Controller

**Purpose**: Manages PVC binding and volume provisioning for Pods.

```
PV Provisioning flow for Pods:

1. Pod spec has: volumes: [{name: data, persistentVolumeClaim: {claimName: my-pvc}}]
2. PVC object: my-pvc exists, status: Pending
3. PV controller: finds matching PV or triggers dynamic provisioning
4. PV controller: binds PVC to PV (PVC status → Bound)
5. Scheduler: can now schedule Pod (PVC must be Bound for most volumes)
6. Kubelet on assigned node: mounts PV to host path
7. Container engine: bind-mounts host path into container filesystem

Volume lifecycle for Pods:
  Pod created → PVC bound → volume attached to node → volume mounted in pod
  Pod deleted → volume unmounted → (if PV reclaim=Retain: kept)
               → (if PV reclaim=Delete: storage deleted)

ReadWriteOnce (RWO):
  → Can only be mounted on ONE node at a time
  → Pod migration to different node: must unmount from old, mount on new
  → Causes brief downtime during node changes for stateful pods

ReadWriteMany (RWX):
  → Can be mounted on MULTIPLE nodes simultaneously
  → NFS, CephFS, etc.
  → Allows pod to move without unmount/mount sequence
```

---

## 7. Pod Lifecycle: Phases, Conditions & Container States

### Pod Phases

```
┌─────────────────────────────────────────────────────────────────┐
│                      POD PHASE STATE MACHINE                     │
│                                                                   │
│   ┌─────────┐    scheduled     ┌─────────┐    all containers    │
│   │ Pending │─────────────────►│ Running │──────────────────────┤
│   │         │                  │         │    running           │
│   │ waiting │    pull fail /   └────┬────┘                     │
│   │ to be   │    cant sched         │                           │
│   │ scheduled│                      │ all containers exit 0    │
│   └─────────┘                  ┌────▼──────┐                   │
│                                 │ Succeeded │                   │
│                                 └───────────┘                   │
│                                      │                           │
│                                 ┌────▼──────┐                   │
│                                 │  Failed   │ (at least one     │
│                                 │           │  container non-0) │
│                                 └───────────┘                   │
│                                                                   │
│   ┌─────────┐                                                    │
│   │ Unknown │  (node unreachable, kubelet cannot report status) │
│   └─────────┘                                                    │
└─────────────────────────────────────────────────────────────────┘
```

| Phase | Meaning | Common Causes |
|---|---|---|
| `Pending` | Pod accepted, not yet running | Scheduling, image pull, volume mount |
| `Running` | At least one container running | Normal operation |
| `Succeeded` | All containers exited 0 | Job/batch workload completed |
| `Failed` | At least one container exited non-0 | Application error, OOMKill |
| `Unknown` | Cannot determine state | Node failure, network partition |

### Pod Conditions

```bash
kubectl get pod <pod> -o jsonpath='{.status.conditions}'
```

| Condition Type | True Means |
|---|---|
| `PodScheduled` | Scheduler has assigned pod to a node |
| `Initialized` | All init containers have completed successfully |
| `ContainersReady` | All containers are ready (probes passing) |
| `Ready` | Pod is ready to serve traffic (all ContainersReady) |
| `PodReadyToStartContainers` | Sandbox and network configured |
| `DisruptionTarget` | Pod is being evicted |

### Container States

| State | Meaning |
|---|---|
| `Waiting` | Not running yet (pulling image, pending init container, etc.) |
| `Running` | Container is executing |
| `Terminated` | Container has exited (with exit code and reason) |

```bash
# View container states:
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].state}'

# View restart count:
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].restartCount}'

# View last termination state:
kubectl get pod <pod> -o jsonpath='{.status.containerStatuses[*].lastState}'
```

### Pod Termination Sequence

```
kubectl delete pod <name> (or controller deletion)
  │
  ├─ 1. Pod marked for deletion:
  │       metadata.deletionTimestamp = now
  │       metadata.deletionGracePeriodSeconds = 30
  │
  ├─ 2. Endpoint controller: Pod removed from EndpointSlices
  │       (Pod no longer receives new traffic)
  │
  ├─ 3. Kubelet detects deletion via watch
  │
  ├─ 4. preStop hook executed (if defined)
  │       Pod waits for hook to complete
  │
  ├─ 5. SIGTERM sent to all containers (PID 1 in each container)
  │       Application should begin graceful shutdown
  │
  ├─ 6. Kubelet waits terminationGracePeriodSeconds (default 30s)
  │
  ├─ 7. If containers still running after grace period:
  │       SIGKILL sent (force kill)
  │
  ├─ 8. Containers stopped, volumes unmounted
  │
  └─ 9. Pod object deleted from API server
```

### CrashLoopBackOff: Understanding Backoff

```
Container exits with non-zero (or killed by OOM):
  Restart 1: immediate
  Restart 2: 10s wait
  Restart 3: 20s wait
  Restart 4: 40s wait
  Restart 5: 80s wait
  Restart 6: 160s wait
  Restart 7+: 300s wait (5 minutes, maximum backoff)

Backoff resets after container runs successfully for 10 minutes.

kubectl describe pod: shows "Back-off restarting failed container"
```

---

## 8. Init Containers, Sidecar Containers & Ephemeral Containers

### Init Containers

Init containers run **sequentially before** the main containers start. They always exit before the application starts.

```yaml
spec:
  initContainers:
  # Run in order: wait-for-db runs first, then migrate-db
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c',
      'until nc -z postgres-svc 5432; do echo "Waiting for DB"; sleep 2; done']

  - name: migrate-db
    image: myapp:migrate
    command: ['sh', '-c', '/app/migrate.sh']
    env:
    - name: DB_HOST
      value: postgres-svc

  containers:
  - name: app
    image: myapp:v1.0
    # Only starts AFTER both init containers succeed
```

**Init container use cases:**

| Use Case | Example |
|---|---|
| Wait for dependency | Wait for database, message queue, service to be available |
| Database migration | Run schema migrations before app starts |
| Fetch secrets | Pull secrets from Vault, write to shared volume |
| Register with service | Register in service discovery before main app starts |
| Set up config | Generate config files from templates into shared volume |
| Clone git repo | Pull latest code/config into emptyDir |

**Init container properties:**

- Run sequentially (not in parallel)
- If an init container fails: pod remains Pending, init container is retried (per restartPolicy)
- Can use different images from the main app (lean tools like busybox, curl)
- Share volumes with main containers (communication mechanism)
- Do NOT share network ports with main containers (they are separate containers)

### Sidecar Containers (Kubernetes 1.28+)

Sidecar containers are init containers with `restartPolicy: Always` — they start before the main app and run alongside it for the Pod's entire lifetime.

```yaml
spec:
  initContainers:
  - name: log-shipper          # Sidecar: init container with restartPolicy
    image: fluentd:v1.16
    restartPolicy: Always      # This makes it a sidecar, not a regular init container
    volumeMounts:
    - name: log-dir
      mountPath: /var/log/app

  containers:
  - name: app
    image: myapp:v1.0
    volumeMounts:
    - name: log-dir
      mountPath: /var/log/app

  volumes:
  - name: log-dir
    emptyDir: {}
```

**Sidecar vs traditional patterns:**

| Aspect | Traditional Sidecar (regular container) | New Sidecar (init + restartPolicy:Always) |
|---|---|---|
| Startup order | Parallel with main container | Starts BEFORE main container |
| Shutdown order | Parallel termination | Stops AFTER main container |
| Job compatibility | Keeps Job running after main exits | Terminated properly when main exits |
| Liveness probe | Independent | Independent |

### Ephemeral Containers

Ephemeral containers are temporary containers added to a running Pod for debugging. They cannot be removed once added and do not appear in the pod spec.

```bash
# Add a debug container to a running pod:
kubectl debug -it my-pod --image=busybox:1.36 --target=my-container
# --target: share the process namespace of the target container

# Use a debug image with more tools:
kubectl debug -it my-pod --image=nicolaka/netshoot

# Copy pod for debugging (creates new pod with debug container):
kubectl debug my-pod --copy-to=debug-pod --image=busybox

# Add ephemeral container directly:
kubectl exec my-pod -- sh   # Won't work if no shell in production image

# Ephemeral containers can share:
#   - Process namespace (if shareProcessNamespace: true on pod)
#   - Network namespace (always shared in pod)
#   - Volumes (if the volume is listed in ephemeral container's volumeMounts)
```

---

## 9. Pod Networking: IPs, DNS & Inter-Pod Communication

### Pod Network Model

```
Kubernetes Network Model (fundamental rules):
  1. Every Pod gets its OWN IP address
  2. All Pods can communicate with all other Pods WITHOUT NAT
  3. All Nodes can communicate with all Pods WITHOUT NAT
  4. The IP a Pod sees as its own is the same IP others use to reach it

This means:
  Pod A (10.244.1.5) can connect directly to Pod B (10.244.2.3)
  No NAT required — the overlay network (CNI) handles routing
```

### Pod IP Assignment

```
Pod creation:
  1. Kubelet calls CRI: RunPodSandbox (creates pause container + network namespace)
  2. CRI calls CNI plugin: ADD command with network namespace path
  3. CNI plugin allocates IP from node's Pod CIDR (e.g., 10.244.1.0/24)
  4. CNI sets up veth pair: container side (eth0) + node side (cali1234 or similar)
  5. CNI writes IP to pod's network namespace
  6. Pod gets: eth0 with 10.244.1.5/32, default route via node's gateway

Node Pod CIDRs (example):
  worker-1: 10.244.1.0/24    (Pods: 10.244.1.1 - 10.244.1.254)
  worker-2: 10.244.2.0/24    (Pods: 10.244.2.1 - 10.244.2.254)
  worker-3: 10.244.3.0/24    (Pods: 10.244.3.1 - 10.244.3.254)
```

### DNS Resolution for Pods

```
Pod DNS configuration (/etc/resolv.conf inside pod):
  nameserver 10.96.0.10         ← CoreDNS ClusterIP
  search default.svc.cluster.local svc.cluster.local cluster.local
  options ndots:5

DNS records for Pods:
  pod-ip-dashes.namespace.pod.cluster.local
  Example: 10-244-1-5.default.pod.cluster.local → 10.244.1.5

DNS records for Services:
  service-name.namespace.svc.cluster.local
  Example: postgres.default.svc.cluster.local → 10.96.10.5 (ClusterIP)

  Headless service (pods directly):
  pod-name.service-name.namespace.svc.cluster.local
  Example: postgres-0.postgres.default.svc.cluster.local → 10.244.1.5

ndots:5 behavior:
  "curl postgres" → searches "postgres.default.svc.cluster.local" (match!)
  "curl postgres.default" → searches "postgres.default.svc.cluster.local" (match!)
  "curl external.example.com" → 5 dots check fails all search domains → hits external DNS
```

### Inter-Container Communication Within a Pod

```
Containers in the SAME Pod:
  → Share network namespace → same IP, same localhost
  → Container A can reach Container B via localhost:port
  → Example: nginx sidecar can call app on 127.0.0.1:8080

  Port collision:
  → Two containers CANNOT bind the same port number
  → This is a design constraint: use different ports

Shared IPC:
  → Containers share IPC namespace (POSIX shared memory, semaphores)
  → Useful for high-performance inter-process communication

Shared PID (optional):
  spec:
    shareProcessNamespace: true
  → All containers can see each other's processes
  → Container A can signal Container B's processes
  → Useful for: debugging, graceful shutdown coordination
```

---

## 10. Pod Storage: Volumes, PVCs & Projected Volumes

### Volume Types and Use Cases

| Volume Type | Lifecycle | Use Case | Cross-Node? |
|---|---|---|---|
| `emptyDir` | Pod lifetime | Scratch space, inter-container sharing | No |
| `hostPath` | Node-persistent | Access node filesystem, DaemonSets | No (tied to node) |
| `configMap` | ConfigMap lifetime | Configuration files | Yes |
| `secret` | Secret lifetime | Sensitive data (tokens, certs) | Yes |
| `persistentVolumeClaim` | PV lifetime | Persistent application data | Depends on PV |
| `projected` | Varies | Combine multiple sources | Yes |
| `downwardAPI` | Pod lifetime | Expose pod metadata as files | Yes |
| `emptyDir (Memory)` | Pod lifetime | High-speed scratch space | No |
| `nfs` | External | Shared filesystem | Yes |
| `csi` | Plugin-defined | Cloud volumes (EBS, GCE PD, etc.) | Depends |

### emptyDir: The Workhorse Volume

```yaml
# emptyDir: created when pod starts, deleted when pod terminates
volumes:
- name: shared-data
  emptyDir: {}              # Disk-backed scratch space

- name: fast-cache
  emptyDir:
    medium: Memory          # tmpfs (RAM-backed, faster, counts against memory limit)
    sizeLimit: 256Mi        # Optional: evict pod if exceeds this

# Use case: Share data between containers in same pod
containers:
- name: producer
  volumeMounts:
  - name: shared-data
    mountPath: /output
- name: consumer
  volumeMounts:
  - name: shared-data
    mountPath: /input
```

### Projected Volumes: Combining Sources

```yaml
# Combine multiple volume sources into a single mount:
volumes:
- name: combined
  projected:
    sources:
    - configMap:
        name: app-config
    - secret:
        name: app-secret
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600   # Short-lived, auto-rotated token
        audience: api
    - downwardAPI:
        items:
        - path: pod-name
          fieldRef:
            fieldPath: metadata.name
        - path: cpu-limit
          resourceFieldRef:
            containerName: app
            resource: limits.cpu
```

### PVC Lifecycle with Pods

```bash
# Create PVC:
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 10Gi
EOF

# Reference in Pod:
# spec:
#   volumes:
#   - name: data
#     persistentVolumeClaim:
#       claimName: app-data   # PVC must be in same namespace

# Check PVC status:
kubectl get pvc app-data
# STATUS: Bound (ready) or Pending (not yet provisioned/bound)

# Describe PVC issues:
kubectl describe pvc app-data

# Check PV:
kubectl get pv
kubectl describe pv <pv-name>
```

---

## 11. Internal Working Concepts: Reconciliation, Informer Pattern & Work Queues

### The Informer Pattern (for Pod controllers)

The Informer pattern is the backbone of how all Pod controllers maintain awareness of cluster state without overloading the API server.

```
┌──────────────────────────────────────────────────────────────────────┐
│              INFORMER PATTERN (used by all Pod controllers)           │
│                                                                        │
│  kube-apiserver                                                        │
│       │                                                                │
│       │  1. List (initial: all pods in namespace/cluster)             │
│       ▼                                                                │
│  ┌────────────┐                                                        │
│  │  Reflector │◄── 2. Watch (streaming: Add/Update/Delete events)     │
│  │            │        for Pods, ReplicaSets, etc.                    │
│  └──────┬─────┘                                                        │
│          │ Puts events in queue                                         │
│  ┌───────▼──────────┐                                                  │
│  │  Delta FIFO      │  ← Thread-safe event queue                      │
│  │    Queue         │    Events: ADDED, UPDATED, DELETED, SYNC        │
│  └───────┬──────────┘                                                  │
│           │ Processes events                                            │
│  ┌────────▼──────────┐     ┌─────────────────────────────────────┐   │
│  │   Indexer         │     │        Event Handlers               │   │
│  │  (Local Cache)    │     │  OnAdd(pod)    → enqueue RS key     │   │
│  │                   │     │  OnUpdate(pod) → enqueue RS key     │   │
│  │  Thread-safe      │     │  OnDelete(pod) → enqueue RS key     │   │
│  │  in-memory store  │     └────────────────────┬────────────────┘   │
│  └───────────────────┘                          │                     │
│                                      ┌──────────▼──────────┐         │
│                                      │  Rate-Limited        │         │
│                                      │  Work Queue          │         │
│                                      │  (deduplicated)      │         │
│                                      └──────────┬──────────┘         │
│                                                 │                     │
│                                      ┌──────────▼──────────┐         │
│                                      │  Worker Goroutines   │         │
│                                      │  processNextItem()   │         │
│                                      │  → reconcile(key)    │         │
│                                      │  → read from cache   │         │
│                                      │  → compute diff      │         │
│                                      │  → call API server   │         │
│                                      └─────────────────────-┘         │
└──────────────────────────────────────────────────────────────────────┘
```

### Reconciliation Loop in Detail

```go
// Conceptual ReplicaSet reconciliation:
func (r *ReplicaSetController) syncReplicaSet(key string) error {
    namespace, name, _ := cache.SplitMetaNamespaceKey(key)

    // 1. Get desired state from LOCAL CACHE (not API server — no extra call)
    rs, err := r.rsLister.ReplicaSets(namespace).Get(name)
    if err != nil {
        return err
    }

    // 2. Get actual pods matching selector from LOCAL CACHE
    selector, _ := metav1.LabelSelectorAsSelector(rs.Spec.Selector)
    pods, err := r.podLister.Pods(namespace).List(selector)

    // 3. Filter: only count non-terminating pods as "active"
    activePods := filterActivePods(pods)

    // 4. Compute diff
    diff := len(activePods) - int(rs.Spec.Replicas)

    // 5. Act: call API server only if there's a diff
    if diff < 0 {
        // Need more pods: create -diff pods
        r.createPods(namespace, rs, -diff)
    } else if diff > 0 {
        // Too many pods: delete diff pods
        r.deletePods(activePods[:diff])
    }

    // 6. Update ReplicaSet status
    r.updateReplicaSetStatus(rs, activePods)

    return nil
}
```

### Work Queue Properties

| Property | Behavior | Pod Management Benefit |
|---|---|---|
| **Deduplication** | Same RS key not added twice | Rapid pod events don't cause duplicate reconciles |
| **Rate limiting** | Token bucket / exponential backoff | API server not overwhelmed during mass pod creation |
| **Exponential backoff** | Failed items retry with increasing delay | Transient API errors handled gracefully |
| **FIFO** | Ordered processing per unique key | Predictable ordering |

```
Example: 5 pods of a ReplicaSet crash simultaneously:
  Event 1: pod-a Deleted → enqueue RS-key
  Event 2: pod-b Deleted → try enqueue RS-key → DEDUPLICATED (already in queue)
  Event 3: pod-c Deleted → try enqueue RS-key → DEDUPLICATED
  Event 4: pod-d Deleted → try enqueue RS-key → DEDUPLICATED
  Event 5: pod-e Deleted → try enqueue RS-key → DEDUPLICATED

  Worker processes RS-key ONCE:
  → Counts pods: 0 (all deleted)
  → Creates 5 new pods in one reconcile
  → Single batch API call (not 5 separate reconciles)
```

---

## 12. Leader Election in Pod Management Context

### Does a Pod Use Leader Election?

**No.** Individual Pods are not controller processes and do not participate in leader election. However, the **kube-controller-manager** — which runs all Pod-managing controllers — uses leader election in HA setups.

### Leader Election for kube-controller-manager

```
Why needed:
  In HA, 3+ kube-controller-manager instances run simultaneously
  If all were active: race conditions on Pod creation/deletion
  Example: RS needs 1 new pod; all 3 controllers create 1 each → 3 pods created!

Solution: Only ONE instance is the leader; others are on standby.
```

```yaml
# Leader election Lease object:
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  holderIdentity: "controller-manager-pod-xyz-on-master-1"
  leaseDurationSeconds: 15      # Lease expires if not renewed
  acquireTime: "2024-01-15T09:00:00.000000Z"
  renewTime: "2024-01-15T10:30:45.000000Z"
  leaderTransitions: 3
```

**Leader election process:**

```
Step 1: All 3 kube-controller-manager pods start
Step 2: Each atomically tries to write its identity to the Lease object
        (etcd compare-and-swap ensures only ONE wins)
Step 3: Winner = leader → runs all controllers (RS, Deployment, Job, etc.)
Step 4: Leader renews Lease every renewDeadline (10s default)
Step 5: Followers watch Lease; if not renewed within leaseDuration (15s):
        → Race to acquire; new leader elected
Step 6: New leader takes over; resumes all controller reconciliation loops

Impact on Pods:
  During failover (~15s): No new Pods created, no old Pods deleted
  After new leader elected: Controllers resume, catch up to desired state
```

**Important flags:**

```bash
# kube-controller-manager flags:
--leader-elect=true                    # Enable (default for kubeadm)
--leader-elect-lease-duration=15s      # How long lease is valid
--leader-elect-renew-deadline=10s      # How often leader renews
--leader-elect-retry-period=2s         # How often followers retry
--concurrent-replicaset-syncs=5        # Parallel RS reconciliations
--concurrent-deployment-syncs=5        # Parallel Deployment reconciliations
--concurrent-pod-gc-syncs=20           # Parallel Pod GC operations
```

**HA Best Practices:**

```yaml
# Deploy kube-controller-manager with:
replicas: 3                    # Odd number for clean election
--leader-elect=true            # Always on
--leader-elect-lease-duration=15s
--leader-elect-renew-deadline=10s  # Must be < lease-duration
--leader-elect-retry-period=2s     # Must be < renew-deadline

# Monitor leader:
kubectl get lease kube-controller-manager -n kube-system -o yaml
kubectl get lease kube-scheduler -n kube-system -o yaml

# Watch for leader changes:
kubectl get lease -n kube-system -w
```

---

## 13. Interaction with API Server and etcd

### The Fundamental Rule

> **Controllers, Kubelet, and kube-proxy NEVER communicate directly with etcd. All cluster state flows through the kube-apiserver.**

```
CORRECT:
  Controller → API server → etcd (read/write)
  Kubelet    → API server (watch pods, update status)
  Pod        → API server (via ServiceAccount token, if needed)

INCORRECT:
  Controller → etcd  ✗
  Kubelet    → etcd  ✗
  Pod        → etcd  ✗
```

### How Pods Are Stored in etcd

```bash
# Pod objects stored in etcd at:
/registry/pods/<namespace>/<pod-name>

# View raw etcd content (from control plane node):
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/pods/default/my-pod | hexdump -C | head -30
# Note: stored as protobuf (binary), not JSON

# List all pod keys:
ETCDCTL_API=3 etcdctl ... \
  get /registry/pods/ --prefix --keys-only
```

### API Server Interactions for Pods

| Who | Operation | API Path | Frequency |
|---|---|---|---|
| Controller (RS) | Create Pod | `POST /api/v1/namespaces/{ns}/pods` | When diff < 0 |
| Controller (RS) | Delete Pod | `DELETE /api/v1/namespaces/{ns}/pods/{name}` | When diff > 0 |
| Controller (RS) | Watch Pods | `GET /api/v1/namespaces/{ns}/pods?watch=true` | Persistent |
| kube-scheduler | Bind Pod to Node | `POST /api/v1/namespaces/{ns}/pods/{name}/binding` | Per unscheduled pod |
| kubelet | Watch Pods on node | `GET /api/v1/pods?fieldSelector=spec.nodeName={node}` | Persistent |
| kubelet | Update Pod Status | `PATCH /api/v1/namespaces/{ns}/pods/{name}/status` | Per state change |
| kubelet | Create Event | `POST /api/v1/namespaces/{ns}/events` | On significant events |
| Pod (app) | Any API call | Varies | On demand (ServiceAccount) |

### Optimistic Concurrency in Pod Updates

```
Pods use resourceVersion for optimistic concurrency:

  1. Controller reads Pod: resourceVersion = "12345"
  2. Controller modifies Pod spec
  3. Controller sends PATCH with resourceVersion: "12345"
  4. API server checks: is current resourceVersion still "12345"?
     - YES: Apply update, increment resourceVersion to "12346"
     - NO (someone else updated): Return 409 Conflict
  5. Controller retries with fresh read

This prevents two controllers from simultaneously overwriting each other's changes.
```

### Pod API Admission Pipeline

```
kubectl apply -f pod.yaml → API Server

API Server pipeline:
  1. Authentication:   Who are you? (cert, token, OIDC)
  2. Authorization:    Can you create pods? (RBAC check)
  3. Admission:        Is this pod spec valid?
     a. Mutating Admission:
        - LimitRanger: inject default resource limits
        - ServiceAccount: inject service account token
        - Custom webhooks: inject sidecars, labels, etc.
     b. Validating Admission:
        - Schema validation (is the YAML valid?)
        - Business rules (PodSecurityAdmission: is pod secure enough?)
        - Custom webhooks: org-specific policies
  4. Write to etcd
  5. Respond: 201 Created

Common admission errors:
  "pods ... is forbidden: unable to validate against any security policy"
  → PodSecurityAdmission or PSP blocking the pod spec

  "exceeded quota"
  → ResourceQuota in namespace exceeded

  "no space left on device" (etcd)
  → etcd disk full; emergency: increase disk, run defrag
```

---

## 14. Resource Management: Requests, Limits & QoS

### CPU and Memory Semantics

```
CPU units:
  1000m = 1 CPU core = 1 vCPU
  250m  = 0.25 CPU core (25% of one core)
  100m  = 100 millicores

CPU Request:
  → Guaranteed minimum CPU share via cgroup CFS weight
  → Used by scheduler for pod placement

CPU Limit:
  → Maximum CPU throttled via cgroup CFS quota
  → Container gets throttled (NOT killed) when limit exceeded
  → WARNING: CPU throttling causes latency, not OOMKill

Memory Request:
  → Used by scheduler for pod placement
  → NOT enforced by kernel (memory can exceed request)

Memory Limit:
  → Enforced by cgroup memory limit
  → Container OOMKilled when limit exceeded (SIGKILL, no grace period)
  → restartCount++ on OOMKill
```

### QoS Classes

| QoS Class | Condition | Eviction Priority | Behavior |
|---|---|---|---|
| **Guaranteed** | ALL containers: `requests == limits` for CPU AND memory | Last evicted | Never throttled if within limits |
| **Burstable** | At least one container has requests | Middle priority | Evicted based on usage vs requests |
| **BestEffort** | No containers have ANY requests or limits | First evicted | Can use any available resources |

```yaml
# Guaranteed QoS:
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"     # Must equal requests
    memory: "256Mi" # Must equal requests

# Burstable QoS:
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
  limits:
    cpu: "1000m"    # Different from requests
    memory: "512Mi"

# BestEffort QoS (no resources specified):
# resources: {}   ← not set at all
```

```bash
# Check QoS class of a pod:
kubectl get pod my-pod -o jsonpath='{.status.qosClass}'

# View pod resource usage:
kubectl top pod my-pod --containers
```

### Resource Quotas and LimitRanges

```yaml
# ResourceQuota: namespace-level limits
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    pods: "50"                        # Max pod count
    requests.cpu: "20"                # Total CPU requests
    requests.memory: "40Gi"           # Total memory requests
    limits.cpu: "40"
    limits.memory: "80Gi"
    persistentvolumeclaims: "10"
    count/secrets: "20"
---
# LimitRange: per-pod/container defaults and limits
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:                          # Injected if not specified
      cpu: "200m"
      memory: "256Mi"
    defaultRequest:                   # Injected if request not specified
      cpu: "100m"
      memory: "128Mi"
    max:                              # Cannot exceed
      cpu: "4"
      memory: "4Gi"
    min:                              # Cannot be less than
      cpu: "50m"
      memory: "64Mi"
  - type: Pod
    max:
      cpu: "8"
      memory: "8Gi"
```

---

## 15. Pod Scheduling: Affinity, Taints & Topology

### Scheduling Pipeline

```
Unscheduled Pod → kube-scheduler

Scheduler pipeline:
  1. FILTERING (find feasible nodes):
     - NodeResourcesFit: node has enough CPU/memory
     - NodeName: spec.nodeName matches (if set)
     - NodeSelector: node labels match
     - Taints/Tolerations: pod tolerates node taints
     - PodFitsHostPorts: required ports available
     - VolumeZone: volume available in node's zone
     - NodeAffinity: node affinity rules satisfied

  2. SCORING (rank feasible nodes):
     - LeastAllocated: prefer least-used nodes
     - NodeAffinity: prefer nodes matching preferred rules
     - PodTopologySpread: spread pods across topology domains
     - InterPodAffinity: co-locate or spread based on other pods
     - ImageLocality: prefer nodes that already have the image

  3. BINDING: Write spec.nodeName = top-scored node

  4. PREEMPTION (if no feasible node found):
     - Find lower-priority pods to evict
     - Make room for this pod on a node
```

### Node Affinity

```yaml
spec:
  affinity:
    nodeAffinity:
      # HARD requirement (must be satisfied):
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/os
            operator: In
            values: [linux]
          - key: node.kubernetes.io/instance-type
            operator: In
            values: [m5.2xlarge, m5.4xlarge]

      # SOFT preference (try to satisfy, but not required):
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80      # Higher weight = stronger preference
        preference:
          matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values: [us-east-1a]
      - weight: 20
        preference:
          matchExpressions:
          - key: disk-type
            operator: In
            values: [ssd]
```

### Pod Anti-Affinity for High Availability

```yaml
# Spread pods across nodes (never co-locate same app):
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: web-frontend
        topologyKey: kubernetes.io/hostname    # Different physical nodes
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web-frontend
          topologyKey: topology.kubernetes.io/zone  # Prefer different AZs
```

### Taints and Tolerations

```bash
# Taint a node (prevent most pods from scheduling there):
kubectl taint nodes gpu-node-1 nvidia.com/gpu=present:NoSchedule

# Pod must tolerate the taint to schedule there:
spec:
  tolerations:
  - key: "nvidia.com/gpu"
    operator: "Equal"
    value: "present"
    effect: "NoSchedule"

# Common taints:
kubectl taint node master-1 node-role.kubernetes.io/control-plane:NoSchedule
kubectl taint node spot-node-1 spot=true:NoExecute  # Evict existing pods too

# Taint effects:
# NoSchedule:    New pods won't schedule (existing pods unaffected)
# PreferNoSchedule: Try not to schedule (soft version)
# NoExecute:    New pods won't schedule AND existing pods evicted
```

### TopologySpreadConstraints

```yaml
# Evenly spread pods across availability zones:
spec:
  topologySpreadConstraints:
  - maxSkew: 1              # Max difference between zone counts
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule   # Or: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: web-frontend
    minDomains: 3           # Require at least 3 AZs

  # Also spread across nodes within each zone:
  - maxSkew: 2
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway   # Soft (don't block scheduling)
    labelSelector:
      matchLabels:
        app: web-frontend
```

---

## 16. Security Hardening Practices

### Pod Security Standards

Kubernetes defines three security profiles:

| Profile | Restriction Level | Use Case |
|---|---|---|
| `Privileged` | Unrestricted | System-level workloads (CNI, CSI drivers) |
| `Baseline` | Minimally restrictive | General workloads, prevents known privilege escalation |
| `Restricted` | Heavily restricted | Security-sensitive workloads |

```yaml
# Enable at namespace level:
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
```

### Minimal SecurityContext

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001
    seccompProfile:
      type: RuntimeDefault    # Apply default seccomp filter

  containers:
  - name: app
    image: myapp:v1.0
    securityContext:
      allowPrivilegeEscalation: false    # Can't gain more privileges than parent
      readOnlyRootFilesystem: true       # Container can't write to root FS
      runAsNonRoot: true
      runAsUser: 10001
      capabilities:
        drop:
          - ALL                          # Drop ALL Linux capabilities
        add:
          - NET_BIND_SERVICE             # Only if binding port < 1024

    # For read-only root FS: mount writable volumes for app data:
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /app/cache

  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
```

### RBAC for Pod Access

```yaml
# ServiceAccount for the pod:
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-frontend-sa
  namespace: production
automountServiceAccountToken: false    # Don't auto-mount (opt-in security)
---
# Role: what this service account can do
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: web-frontend-role
  namespace: production
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["app-config"]       # Only THIS specific configmap
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["app-secret"]
  verbs: ["get"]
---
# RoleBinding: bind SA to Role
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: web-frontend-binding
  namespace: production
subjects:
- kind: ServiceAccount
  name: web-frontend-sa
  namespace: production
roleRef:
  kind: Role
  name: web-frontend-role
  apiGroup: rbac.authorization.k8s.io
```

```yaml
# Pod spec using the SA:
spec:
  serviceAccountName: web-frontend-sa
  automountServiceAccountToken: true  # Now opt-in (not auto)
  volumes:
  - name: kube-api-token
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600   # Short-lived token
          audience: kubernetes.default.svc
```

### Network Policies

```yaml
# Default deny all ingress in a namespace:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}    # Applies to ALL pods in namespace
  policyTypes: [Ingress]
---
# Allow specific traffic to web-frontend:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: web-frontend
  policyTypes: [Ingress, Egress]
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
      podSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx
    ports:
    - port: 8080
      protocol: TCP
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - port: 5432
  - to:                    # Allow DNS
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
```

### Image Security

```bash
# Use specific image digests (immutable):
image: nginx@sha256:abc123def456...   # Digest is immutable; tag can change

# Enforce image signing with Sigstore policy controller:
kubectl apply -f https://github.com/sigstore/policy-controller/...

# Scan images in CI/CD before pushing:
trivy image myapp:v1.0
grype myapp:v1.0

# Never use latest tag in production:
# BAD:
image: nginx:latest
# GOOD:
image: nginx:1.25.3
# BEST:
image: nginx@sha256:abc123...

# Private registry credentials:
kubectl create secret docker-registry registry-creds \
  --docker-server=registry.example.com \
  --docker-username=<user> \
  --docker-password=<pass>
# Reference in pod:
imagePullSecrets:
- name: registry-creds
```

---

## 17. Performance Tuning & Configuration

### Pod Startup Optimization

```yaml
# Fast pod startup:

# 1. Use small, layer-cached images:
image: myapp:v1.0   # Should be < 100MB for fast pulls

# 2. Use startupProbe to replace initialDelaySeconds:
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10    # Up to 300s startup time, precisely controlled

# 3. Avoid initialDelaySeconds (fragile):
# BAD:
livenessProbe:
  initialDelaySeconds: 120   # What if app starts in 5s? What if it takes 180s?
# GOOD: Use startupProbe instead

# 4. Pre-pull images (DaemonSet image warmup):
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: image-prepuller
spec:
  template:
    spec:
      initContainers:
      - name: prepull
        image: myapp:v1.1    # New version pre-pulled to all nodes
        command: ["true"]
      containers:
      - name: pause
        image: gcr.io/google-containers/pause:3.1
```

### Resource Tuning for Production

```yaml
# Identify right-sizing with VPA (Vertical Pod Autoscaler):
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web-frontend-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-frontend
  updatePolicy:
    updateMode: "Off"    # Recommendation only (don't auto-apply)

# Read VPA recommendations:
kubectl describe vpa web-frontend-vpa

# HorizontalPodAutoscaler:
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-frontend-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-frontend
  minReplicas: 3
  maxReplicas: 20
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
        type: AverageValue
        averageValue: 400Mi
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300    # Don't scale down too fast
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60                # Max 10% scale-down per minute
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15               # Can double quickly on traffic spike
```

### Pod Disruption Budgets

```yaml
# Ensure minimum availability during node drains/upgrades:
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-frontend-pdb
spec:
  minAvailable: 2       # Always keep at least 2 pods running
  # OR:
  # maxUnavailable: 1   # Never take down more than 1 at a time
  selector:
    matchLabels:
      app: web-frontend

# Check PDB status:
kubectl get pdb
kubectl describe pdb web-frontend-pdb
```

### Kubelet Performance Flags Affecting Pods

```yaml
# /var/lib/kubelet/config.yaml
maxPods: 110                           # Default; tune based on node size
serializeImagePulls: false             # Allow parallel image pulls
maxParallelImagePulls: 5               # Concurrent image pulls

# Pod eviction thresholds:
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
  imagefs.available: "15%"

# Reserve resources for kubelet/OS:
systemReserved:
  cpu: "200m"
  memory: "200Mi"
kubeReserved:
  cpu: "100m"
  memory: "100Mi"

# Container log rotation:
containerLogMaxSize: "50Mi"
containerLogMaxFiles: 5
```

---

## 18. Monitoring & Observability

### Key Pod Metrics (cAdvisor / kubelet)

**CPU Metrics:**

| Metric | Type | Description |
|---|---|---|
| `container_cpu_usage_seconds_total` | Counter | Total CPU time used by container |
| `container_cpu_throttled_seconds_total` | Counter | Time CPU was throttled |
| `container_cpu_cfs_throttled_periods_total` | Counter | Count of throttled CFS periods |

**Memory Metrics:**

| Metric | Type | Description |
|---|---|---|
| `container_memory_usage_bytes` | Gauge | Current memory usage |
| `container_memory_working_set_bytes` | Gauge | Working set (used for OOM decisions) |
| `container_memory_rss` | Gauge | RSS memory |
| `container_oom_events_total` | Counter | OOM kill events |

**Pod Status Metrics (kube-state-metrics):**

| Metric | Type | Description |
|---|---|---|
| `kube_pod_status_phase` | Gauge | Pod phase (1=true, 0=false per phase) |
| `kube_pod_status_ready` | Gauge | Is pod ready? |
| `kube_pod_container_status_restarts_total` | Counter | Container restart count |
| `kube_pod_container_status_waiting_reason` | Gauge | Reason container is waiting |
| `kube_pod_container_resource_requests` | Gauge | Requested resources |
| `kube_pod_container_resource_limits` | Gauge | Resource limits |

**Network Metrics:**

| Metric | Type | Description |
|---|---|---|
| `container_network_receive_bytes_total` | Counter | Network bytes received |
| `container_network_transmit_bytes_total` | Counter | Network bytes transmitted |
| `container_network_receive_errors_total` | Counter | Network receive errors |

### Critical Alert Rules

```yaml
groups:
- name: pod.rules
  rules:

  # CrashLoopBackOff detection:
  - alert: PodCrashLooping
    expr: |
      increase(kube_pod_container_status_restarts_total[15m]) > 3
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} in {{ $labels.namespace }} is crash-looping"

  # Pod not ready:
  - alert: PodNotReady
    expr: |
      kube_pod_status_ready{condition="true"} == 0
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} not ready for 5 minutes"

  # OOMKill:
  - alert: ContainerOOMKilled
    expr: |
      increase(container_oom_events_total[5m]) > 0
    labels:
      severity: warning
    annotations:
      summary: "Container {{ $labels.container }} OOM killed"

  # High CPU throttling (latency indicator):
  - alert: HighCPUThrottling
    expr: |
      rate(container_cpu_cfs_throttled_periods_total[5m])
      / rate(container_cpu_cfs_periods_total[5m]) > 0.25
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Container {{ $labels.container }} CPU throttled > 25%"
      description: "CPU limit may be too low. Consider increasing CPU limit or requests."

  # Pod stuck in Pending:
  - alert: PodPending
    expr: |
      kube_pod_status_phase{phase="Pending"} == 1
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} stuck in Pending for 10 minutes"

  # ImagePullBackOff:
  - alert: ImagePullBackOff
    expr: |
      kube_pod_container_status_waiting_reason{reason="ImagePullBackOff"} == 1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} has ImagePullBackOff"

  # High memory usage (approaching limit):
  - alert: PodMemoryNearLimit
    expr: |
      container_memory_working_set_bytes
      / on(pod, container, namespace)
      kube_pod_container_resource_limits{resource="memory"} > 0.85
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Pod {{ $labels.pod }} memory > 85% of limit"
```

### Practical Observability Commands

```bash
# Real-time pod resource usage:
kubectl top pods -n production --sort-by=memory
kubectl top pods -n production --containers

# Watch pod events:
kubectl get events -n production --sort-by='.lastTimestamp' -w

# Pod metrics from API server:
kubectl get --raw /api/v1/nodes/<node>/proxy/metrics/resource | grep container_cpu

# Custom metrics query (if Prometheus is installed):
kubectl port-forward -n monitoring svc/prometheus 9090:9090
# Then visit localhost:9090

# Stern for multi-pod log streaming:
stern -n production -l app=web-frontend --tail 50

# JSON logs parsing:
kubectl logs -n production my-pod | jq '.level, .message'
```

---

## 19. Troubleshooting Guide with Real Commands

### Scenario 1: Pods Not Created (Pending State)

```bash
# Step 1: Describe the pod (Events section is critical):
kubectl describe pod <pod-name> -n <namespace>
# Look at bottom "Events:" section for specific errors

# Step 2: Common Pending causes and diagnosis:

# CAUSE 1: Insufficient resources
kubectl describe pod <n> | grep "Insufficient"
# Solution: Check node resources:
kubectl describe nodes | grep -A 5 "Allocated resources"
kubectl top nodes

# CAUSE 2: No matching nodes (affinity/selector issue)
kubectl describe pod <n> | grep "didn't match"
# Check node labels:
kubectl get nodes --show-labels
kubectl describe node <node> | grep Labels

# CAUSE 3: PVC not bound
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>
# Check storage class:
kubectl get storageclass

# CAUSE 4: Image pull issue
kubectl describe pod <n> | grep -A 5 "Failed to pull"
# Test pull manually (on node):
crictl pull <image>
# Check imagePullSecrets:
kubectl get pod <n> -o jsonpath='{.spec.imagePullSecrets}'

# CAUSE 5: Taints preventing scheduling
kubectl describe pod <n> | grep "Taints"
kubectl describe nodes | grep Taints

# CAUSE 6: ResourceQuota exceeded
kubectl describe quota -n <namespace>
kubectl get resourcequota -n <namespace>

# CAUSE 7: PodSecurityAdmission blocking
kubectl create pod -f pod.yaml --dry-run=server -v=5 2>&1 | grep -i "security\|forbidden"
```

### Scenario 2: Deployment Stuck / Rolling Update Hung

```bash
# Step 1: Check rollout status:
kubectl rollout status deployment/<n> -n <namespace>

# Step 2: Check deployment conditions:
kubectl describe deployment <n> -n <namespace> | grep -A 10 "Conditions:"

# Step 3: Check new pods:
kubectl get pods -n <namespace> -l app=<label> --sort-by='.metadata.creationTimestamp'

# CAUSE 1: New pods not becoming Ready (readiness probe failing)
kubectl describe pod <new-pod> | grep -A 15 "Conditions:"
kubectl logs <new-pod> -n <namespace>
kubectl describe pod <new-pod> | grep -A 10 "Events:"

# Check readiness probe config:
kubectl get pod <new-pod> -o jsonpath='{.spec.containers[0].readinessProbe}'

# CAUSE 2: maxUnavailable=0 and old pods not terminating
kubectl get pods -n <namespace> | grep Terminating
# Force delete stuck terminating pods (LAST RESORT):
kubectl delete pod <stuck-pod> --grace-period=0 --force

# CAUSE 3: PodDisruptionBudget blocking scale-down
kubectl get pdb -n <namespace>
kubectl describe pdb -n <namespace>
# Check if minAvailable is preventing old RS scale-down

# CAUSE 4: Image issue with new version
kubectl describe pod <new-pod> | grep -A 5 "Events:"
# Look for: ErrImagePull, ImagePullBackOff

# Rollback if needed:
kubectl rollout undo deployment/<n> -n <namespace>

# Pause and investigate:
kubectl rollout pause deployment/<n>
# Fix the issue, then resume:
kubectl rollout resume deployment/<n>
```

### Scenario 3: Node NotReady (Pods Affected)

```bash
# Step 1: Check node status:
kubectl get nodes
kubectl describe node <node-name>

# Step 2: Check node conditions:
kubectl get node <node-name> -o jsonpath='{.status.conditions[*]}' | python3 -m json.tool

# Step 3: Check pods on that node:
kubectl get pods --field-selector spec.nodeName=<node-name> -A

# Step 4: SSH to node and check kubelet:
systemctl status kubelet
journalctl -u kubelet -n 100 --no-pager | grep -E "error|Error|failed|Failed"

# Step 5: Check container runtime:
systemctl status containerd
crictl ps

# Step 6: Check disk/memory pressure:
df -h
free -h
kubectl describe node <node-name> | grep -E "DiskPressure|MemoryPressure|PIDPressure"

# Step 7: Force pod rescheduling from failed node:
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data --force
# Or taint to prevent new scheduling:
kubectl taint nodes <node-name> node.kubernetes.io/unreachable:NoExecute

# Step 8: Watch pod rescheduling:
kubectl get pods -A -w | grep -E "Terminating|Pending|Running"
```

### Scenario 4: CrashLoopBackOff

```bash
# Step 1: Get current logs:
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> -c <container-name>  # Multi-container

# Step 2: Get previous container logs (before the crash):
kubectl logs <pod-name> -n <namespace> --previous
kubectl logs <pod-name> -n <namespace> -c <container> --previous

# Step 3: Check exit code:
kubectl get pod <n> -o jsonpath='{.status.containerStatuses[0].lastState.terminated}'
# exit code 137 = OOMKilled (SIGKILL)
# exit code 1 = Application error
# exit code 0 = Clean exit (check restartPolicy)

# Step 4: Check OOMKill:
kubectl describe pod <n> | grep -i "OOMKilled"
dmesg | grep -i "oom" | tail -20  # On node

# Step 5: Check resource limits:
kubectl get pod <n> -o jsonpath='{.spec.containers[0].resources}'

# Step 6: Debug with temporary command override:
# Create debug pod with same image:
kubectl run debug-pod --image=myapp:v1.0 --command -- sleep infinity
kubectl exec -it debug-pod -- /bin/sh

# Or override command in deployment temporarily:
kubectl set env deployment/my-app FORCE_CRASH=false
# Edit deployment to override entrypoint:
kubectl edit deployment my-app
# Change: command: ["sleep", "infinity"]
```

### Scenario 5: Pod Stuck in Terminating

```bash
# Check finalizers blocking deletion:
kubectl get pod <n> -o jsonpath='{.metadata.finalizers}'

# Remove finalizers (if safe):
kubectl patch pod <n> -p '{"metadata":{"finalizers":[]}}' --type=merge

# Check preStop hook hanging:
kubectl describe pod <n> | grep -A 5 "preStop"
kubectl logs <n> --previous  # Check for shutdown issues

# Check volume unmount stuck:
journalctl -u kubelet | grep "unmount\|detach" | tail -20  # On node

# Force delete (LAST RESORT — may cause data inconsistency):
kubectl delete pod <n> --grace-period=0 --force

# Understand: force delete removes the object from API server
# but the container may still be running on the node until kubelet reconciles
```

### Scenario 6: Pod OOMKilled Repeatedly

```bash
# Identify OOMKilled pods:
kubectl get pods -A | grep -v Running

# Check events for OOM:
kubectl get events -A --field-selector reason=OOMKilling

# Check actual memory usage:
kubectl top pod <n> --containers

# VPA recommendation check (if installed):
kubectl describe vpa <deployment-name>

# Check memory usage in cgroup (on node):
POD_UID=$(kubectl get pod <n> -o jsonpath='{.metadata.uid}')
cat /sys/fs/cgroup/kubepods.slice/kubepods-burstable.slice/pod${POD_UID}/memory.current

# Temporary fix: increase memory limit:
kubectl set resources deployment/<n> \
  --containers=<container> \
  --requests=memory=512Mi \
  --limits=memory=1Gi

# Check if app has a memory leak:
# Profile memory usage over time with Prometheus query:
# container_memory_working_set_bytes{pod="<pod>", container="<container>"}
```

---

## 20. Comparison: Pods vs kube-apiserver vs kube-scheduler Perspective

### Role in the Cluster

| Dimension | Pod | kube-apiserver | kube-scheduler |
|---|---|---|---|
| **What it is** | Unit of workload (application) | REST gateway to cluster state | Decision engine for pod placement |
| **Type** | Kubernetes object | Control plane component | Control plane component |
| **Creates it** | Controllers (RS, Deployment, etc.) | It's a component, not an object | It's a component, not an object |
| **Runs where** | Worker nodes (usually) | Control plane nodes | Control plane nodes |
| **Talks to etcd** | Never (no direct access) | Yes (only component) | Never |
| **Restartable** | Yes (by controllers) | Yes (static pod) | Yes (static pod) |
| **Persistent state** | No (ephemeral by design) | No (state is in etcd) | No (reads from API) |
| **Leader election** | No | No (stateless, LB) | Yes (one active) |
| **Networking** | Has Pod IP (from CNI) | Bound to host port 6443 | Bound to host port 10259 |
| **Primary interface** | None (workload) | HTTPS REST/gRPC API | Watches unscheduled pods |
| **Lifecycle controller** | Kubelet (on assigned node) | systemd / kubelet (static) | systemd / kubelet (static) |
| **HA approach** | ReplicaSet (multiple) | Multiple stateless instances | Leader election (one active) |

### The Pod's Relationship to Each Component

```
kube-apiserver:
  → Stores the Pod object in etcd
  → Validates Pod spec via admission controllers
  → Serves Pod status to clients (kubectl get pod)
  → Executes: kubectl exec, kubectl logs (streams via kubelet)
  → Enforces RBAC: who can create/delete/view pods

kube-scheduler:
  → Watches for Pods with empty spec.nodeName
  → Evaluates nodes: resources, affinity, taints, spread
  → Assigns spec.nodeName (binding)
  → Never runs the pod; only decides where it goes

kubelet:
  → Watches for pods assigned to its node
  → Calls container engine (CRI) to create containers
  → Runs health probes
  → Reports pod status back to API server
  → The ONLY component that actually runs the Pod
```

---

## 21. Disaster Recovery Concepts

### Pods Are Stateless Objects (by Design)

```
Pod object in etcd: desired state declaration
Container runtime on node: actual running state

If etcd is backed up and restored:
  → Pod objects restored to backup state
  → Kubelet reconciles: starts missing pods, stops orphaned pods
  → No pod-specific backup needed (pod spec is in etcd)

If a node fails:
  → Pod objects still exist in etcd (state not lost)
  → Controllers (RS, Deployment) detect pods as failed/unknown
  → Controllers create new pods → scheduler assigns to healthy node
  → Data loss only occurs if ephemeral volumes (emptyDir) were in use
```

### etcd Backup for Pod Recovery

```bash
# Backup etcd (includes all pod specs, deployments, configmaps, secrets):
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup:
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-backup.db \
  --write-out=table

# Restore:
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored \
  --initial-cluster=master-1=https://192.168.1.10:2380

# After restore: all pod specs restored
# Kubelet will restart any missing pods automatically
# Controller-manager will reconcile RS/Deployment desired state
```

### Stateful Application Recovery

```
Stateful pods (with PVCs) require:
  1. etcd backup (pod spec, PVC spec, PV reference)
  2. Storage backup (actual data in the PV)
  3. Backup tool: Velero (backs up both etcd objects + volumes)

Recovery with Velero:
  velero backup create prod-backup --include-namespaces=production
  velero restore create --from-backup=prod-backup

Recovery without Velero (manual):
  1. Restore etcd: pod specs + PVC/PV objects restored
  2. PV objects point to existing storage (if backend is durable, like EBS)
  3. PVCs rebind to PVs (by name or storageClass)
  4. Pods recreated → mount PVCs → access existing data
```

### Pod Checkpoint (Kubernetes 1.25+)

```bash
# Checkpoint a running container (for forensics/migration):
curl -X POST \
  https://localhost:10250/checkpoint/default/my-pod/my-container \
  --cert /var/lib/kubelet/pki/kubelet.crt \
  --key /var/lib/kubelet/pki/kubelet.key \
  -k

# Creates an OCI-compatible checkpoint archive
# Can be used to restore container state (experimental)
```

---

## 22. Real-World Production Use Cases

### Use Case 1: Zero-Downtime Blue-Green Deployment via Pods

```yaml
# Blue deployment (current):
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-blue
spec:
  replicas: 5
  selector:
    matchLabels:
      app: web
      slot: blue
  template:
    metadata:
      labels:
        app: web
        slot: blue
        version: v1.0
---
# Green deployment (new version):
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-green
spec:
  replicas: 5
  selector:
    matchLabels:
      app: web
      slot: green
  template:
    metadata:
      labels:
        app: web
        slot: green
        version: v1.1
---
# Service (switch between blue and green by changing selector):
apiVersion: v1
kind: Service
metadata:
  name: web-svc
spec:
  selector:
    app: web
    slot: blue    # Change to 'green' to switch traffic instantly
  ports:
  - port: 80
    targetPort: 8080
```

```bash
# Switch to green (instant, no pod restart):
kubectl patch svc web-svc -p '{"spec":{"selector":{"slot":"green"}}}'

# Rollback (instant):
kubectl patch svc web-svc -p '{"spec":{"selector":{"slot":"blue"}}}'
```

### Use Case 2: Sidecar Pattern for Log Collection

```yaml
spec:
  containers:
  - name: app
    image: myapp:v1.0
    volumeMounts:
    - name: logs
      mountPath: /var/log/app

  - name: log-forwarder             # Traditional sidecar (runs alongside app)
    image: fluent/fluent-bit:2.2
    volumeMounts:
    - name: logs
      mountPath: /var/log/app       # Reads same log directory
    - name: fluent-bit-config
      mountPath: /fluent-bit/etc
    resources:
      requests:
        cpu: "50m"
        memory: "64Mi"
      limits:
        cpu: "200m"
        memory: "128Mi"

  volumes:
  - name: logs
    emptyDir: {}
  - name: fluent-bit-config
    configMap:
      name: fluent-bit-config
```

### Use Case 3: Ambassador Pattern (API Gateway Sidecar)

```yaml
# Ambassador: sidecar handles network complexity
spec:
  containers:
  - name: app
    image: myapp:v1.0
    env:
    - name: BACKEND_URL
      value: "http://localhost:9090"    # App talks to localhost:9090

  - name: ambassador                    # Sidecar handles mTLS, service discovery
    image: envoyproxy/envoy:v1.28
    ports:
    - containerPort: 9090
    # Envoy: localhost:9090 → external service with mTLS, circuit breaking, etc.
    volumeMounts:
    - name: envoy-config
      mountPath: /etc/envoy
```

### Use Case 4: Multi-Region Pod Scheduling

```yaml
# Enforce pods spread across 3 availability zones:
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: critical-service
    minDomains: 3              # Require all 3 AZs to be available

  # Also enforce different nodes within each AZ:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: critical-service
```

### Use Case 5: GPU Workload Pods for ML Training

```yaml
spec:
  tolerations:
  - key: nvidia.com/gpu
    operator: Exists
    effect: NoSchedule

  runtimeClassName: nvidia    # Use NVIDIA container runtime

  containers:
  - name: training
    image: pytorch/pytorch:2.0.0-cuda11.7-cudnn8-runtime
    command: ["python", "/app/train.py"]
    resources:
      limits:
        nvidia.com/gpu: 4      # Request 4 GPUs
        memory: "32Gi"
        cpu: "16"
      requests:
        nvidia.com/gpu: 4
        memory: "16Gi"
        cpu: "8"
    volumeMounts:
    - name: dataset
      mountPath: /data
    - name: checkpoints
      mountPath: /checkpoints

  volumes:
  - name: dataset
    persistentVolumeClaim:
      claimName: training-dataset
  - name: checkpoints
    persistentVolumeClaim:
      claimName: model-checkpoints
```

---

## 23. Best Practices for Production Environments

### Pod Design Principles

- **One process per container** — don't run multiple unrelated processes in a single container; use multiple containers in a Pod for tightly coupled processes.
- **Pods are cattle, not pets** — design pods to be replaceable at any time; store all persistent state in external storage (databases, object storage) or PVCs.
- **Always set resource requests AND limits** — requests for scheduling, limits for isolation. Missing requests = poor scheduling. Missing limits = noisy neighbor problem.
- **Use Guaranteed QoS for critical workloads** — set `requests == limits` for latency-sensitive services to prevent throttling and eviction.
- **Always define readiness and liveness probes** — without them, pods receive traffic before ready and stay in service after becoming unhealthy.
- **Use startupProbe for slow-starting applications** — eliminates the need for fragile `initialDelaySeconds` values.

### Deployment Practices

- **Never use `latest` tag** — use semantic versioning or image digests for reproducibility.
- **Set `terminationGracePeriodSeconds` appropriately** — match to your application's drain time (typically 30-60s for web servers).
- **Use `preStop` hook** — add a `sleep` command (5-10s) to give kube-proxy time to remove the pod from endpoints before SIGTERM is sent.
- **Set `maxUnavailable: 0`** for critical services to guarantee zero downtime during rolling updates.
- **Define PodDisruptionBudgets** for all production Deployments/StatefulSets.
- **Use `podAntiAffinity`** to spread replicas across nodes and AZs.

### Security Practices

- **Run as non-root** — set `runAsNonRoot: true` and a specific `runAsUser`.
- **Read-only root filesystem** — set `readOnlyRootFilesystem: true` and use emptyDir for writable paths.
- **Drop all capabilities** — `capabilities.drop: [ALL]`, add only what's needed.
- **Disable privilege escalation** — `allowPrivilegeEscalation: false`.
- **Use minimal base images** — distroless or scratch reduces attack surface.
- **Apply Network Policies** — default-deny, then allow-list.
- **Use short-lived service account tokens** — projected volumes with `expirationSeconds: 3600`.

### Operational Practices

```bash
# Use labels consistently for every pod:
labels:
  app: web-frontend          # Application name
  version: v1.2.3            # Application version
  environment: production    # Environment
  team: platform             # Owning team
  managed-by: helm           # Management tool

# Always use namespaces for isolation:
kubectl config set-context --current --namespace=production

# Set default resource limits via LimitRange:
kubectl apply -f limitrange.yaml

# Monitor pod density per node:
kubectl get pods -A --field-selector spec.nodeName=<node> | wc -l
```

---

## 24. Common Mistakes and Pitfalls

### Mistake 1: Running Pods Directly (Without a Controller)

```yaml
# WRONG — bare pod, no self-healing:
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
  - name: app
    image: myapp:v1.0
# If this pod crashes or its node fails: pod is GONE FOREVER

# RIGHT — use a Deployment:
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  # ... pod template
# If pod crashes: ReplicaSet creates a replacement
```

### Mistake 2: Missing readinessProbe

```yaml
# WRONG — pod receives traffic immediately on container start:
containers:
- name: app
  image: myapp:v1.0
  # No readinessProbe! App may not be ready to serve!

# RIGHT — wait for app to be truly ready:
containers:
- name: app
  image: myapp:v1.0
  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
    initialDelaySeconds: 10
    periodSeconds: 5
```

### Mistake 3: Setting Only Limits Without Requests

```yaml
# WRONG — limits without requests:
resources:
  limits:
    cpu: "1"
    memory: "512Mi"
  # No requests!
  # → requests default to limits (Guaranteed QoS — maybe OK)
  # → Or requests default to 0 (BestEffort — first evicted!)
  # → Scheduler doesn't know resource needs → poor placement

# RIGHT — always set both:
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1000m"
    memory: "512Mi"
```

### Mistake 4: Using `latest` Tag

```yaml
# WRONG:
image: myapp:latest
imagePullPolicy: IfNotPresent   # Even worse! Won't pull new 'latest'!

# The combination above means: use whatever 'latest' was when first pulled
# Different nodes may run different versions!

# RIGHT:
image: myapp:v1.2.3
imagePullPolicy: IfNotPresent   # Cache specific version → fast restarts
```

### Mistake 5: Ignoring terminationGracePeriodSeconds

```yaml
# WRONG — default 30s may not be enough for your app:
# No terminationGracePeriodSeconds set
# App takes 45s to drain connections → SIGKILL at 30s → dropped requests

# RIGHT — set appropriate grace period:
spec:
  terminationGracePeriodSeconds: 60   # Match to app drain time
  containers:
  - name: app
    lifecycle:
      preStop:
        exec:
          command: ["sleep", "10"]   # Buffer for endpoint removal propagation
```

### Mistake 6: hostPath Volumes in Production

```yaml
# WRONG — hostPath: security risk, node-affinity dependency:
volumes:
- name: data
  hostPath:
    path: /var/app-data
# Problems:
# - Pod can only run on nodes with that path
# - Pod can access/modify node filesystem
# - Data not portable (pod can't move to another node)

# RIGHT — use PVC:
volumes:
- name: data
  persistentVolumeClaim:
    claimName: app-data
```

### Mistake 7: Not Setting podAntiAffinity

```yaml
# WRONG — all replicas may schedule on the same node:
# No affinity rules → scheduler may co-locate all replicas
# If that node fails: service down!

# RIGHT — spread across nodes:
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          topologyKey: kubernetes.io/hostname
          labelSelector:
            matchLabels:
              app: my-app
```

### Mistake 8: No Resource Limits → Node OOM

```yaml
# WRONG — unbounded pod can consume all node memory:
# No resource limits → pod can grow until node OOM killer kills other pods

# Consequence: other pods (even critical system pods) get OOMKilled
# Solution: ALWAYS set limits, and configure LimitRange for defaults:

kubectl apply -f - <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
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
EOF
```

---

## 25. Hands-On Labs & Mini Practical Exercises

### Lab 1: Pod Anatomy Exploration

```bash
# Exercise: Explore every field of a running pod

# 1. Deploy a test pod:
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: anatomy-lab
  labels:
    lab: anatomy
spec:
  containers:
  - name: web
    image: nginx:alpine
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "200m"
        memory: "128Mi"
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
EOF

# 2. Watch it start:
kubectl get pod anatomy-lab -w

# 3. Explore the status:
kubectl get pod anatomy-lab -o json | jq '{
  phase: .status.phase,
  podIP: .status.podIP,
  hostIP: .status.hostIP,
  nodeName: .spec.nodeName,
  conditions: .status.conditions,
  containerState: .status.containerStatuses[0].state
}'

# 4. Check QoS class:
kubectl get pod anatomy-lab -o jsonpath='{.status.qosClass}'

# 5. View resource usage:
kubectl top pod anatomy-lab --containers

# 6. Exec into the pod:
kubectl exec -it anatomy-lab -- sh

# 7. Inside the pod: explore the network:
ip addr show    # Pod IP
ip route show   # Default route
cat /etc/resolv.conf  # DNS config
hostname        # Pod hostname

# 8. Check mounted volumes:
ls /var/run/secrets/kubernetes.io/serviceaccount/  # SA token, ca.crt

# Cleanup:
kubectl delete pod anatomy-lab
```

### Lab 2: Pod Lifecycle Observation

```bash
# Exercise: Watch the complete pod lifecycle

# Terminal 1: Watch events:
kubectl get events -w --sort-by='.lastTimestamp'

# Terminal 2: Watch pod status:
kubectl get pods -w

# Terminal 3: Run lifecycle test:

# 1. Create a pod that runs briefly then exits:
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-lab
spec:
  restartPolicy: Never
  containers:
  - name: app
    image: busybox:1.36
    command: ['sh', '-c', 'echo "Starting..."; sleep 30; echo "Done"']
EOF

# 2. Observe: Pending → Running → Succeeded

# 3. Create a crashing pod:
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: crasher-lab
spec:
  restartPolicy: OnFailure
  containers:
  - name: app
    image: busybox:1.36
    command: ['sh', '-c', 'echo "Crashing!"; exit 1']
EOF

# 4. Observe: CrashLoopBackOff, restart count increasing
kubectl get pod crasher-lab -w
kubectl describe pod crasher-lab

# 5. View backoff timing:
kubectl get events | grep "BackOff"

# 6. Get logs from crashed container:
kubectl logs crasher-lab --previous

# Cleanup:
kubectl delete pod lifecycle-lab crasher-lab
```

### Lab 3: Init Container Dependency Chain

```bash
# Exercise: Use init containers to enforce startup order

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: init-lab
spec:
  initContainers:
  - name: wait-for-service
    image: busybox:1.36
    command: ['sh', '-c',
      'until nslookup kubernetes.default; do
         echo "Waiting for DNS...";
         sleep 2;
       done;
       echo "DNS ready!"']

  - name: setup-config
    image: busybox:1.36
    command: ['sh', '-c',
      'echo "server_mode=production" > /shared/config.ini;
       echo "Config written"']
    volumeMounts:
    - name: shared
      mountPath: /shared

  containers:
  - name: app
    image: busybox:1.36
    command: ['sh', '-c',
      'cat /shared/config.ini;
       echo "App running with above config";
       sleep 3600']
    volumeMounts:
    - name: shared
      mountPath: /shared

  volumes:
  - name: shared
    emptyDir: {}
EOF

# Watch init containers run sequentially:
kubectl get pod init-lab -w

# Check init container status:
kubectl describe pod init-lab | grep -A 20 "Init Containers:"

# View init container logs:
kubectl logs init-lab -c wait-for-service
kubectl logs init-lab -c setup-config

# View app container accessing shared config:
kubectl exec init-lab -- cat /shared/config.ini

# Cleanup:
kubectl delete pod init-lab
```

### Lab 4: Resource Limits and OOM Behavior

```bash
# Exercise: Trigger and observe OOMKill

kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: memory-lab
spec:
  containers:
  - name: memory-hog
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
    resources:
      requests:
        memory: "50Mi"
      limits:
        memory: "100Mi"   # Will be exceeded → OOMKill
EOF

# Watch the pod:
kubectl get pod memory-lab -w
# Should show: Running → OOMKilled → CrashLoopBackOff

# Check OOM reason:
kubectl describe pod memory-lab | grep -A 5 "Last State:"
# Reason: OOMKilled

# Check events:
kubectl get events | grep -i "oom\|kill"

# Observe restart count increasing:
kubectl get pod memory-lab

# Cleanup:
kubectl delete pod memory-lab
```

### Lab 5: Affinity and Scheduling

```bash
# Exercise: Control pod placement with affinity rules

# 1. Label a node:
NODE=$(kubectl get nodes --no-headers | awk 'NR==1{print $1}')
kubectl label node $NODE disk=ssd

# 2. Deploy with nodeAffinity:
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: affinity-lab
spec:
  replicas: 3
  selector:
    matchLabels:
      app: affinity-lab
  template:
    metadata:
      labels:
        app: affinity-lab
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disk
                operator: In
                values: [ssd]
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              topologyKey: kubernetes.io/hostname
              labelSelector:
                matchLabels:
                  app: affinity-lab
      containers:
      - name: app
        image: nginx:alpine
EOF

# 3. Check where pods are scheduled:
kubectl get pods -l app=affinity-lab -o wide

# 4. Try with impossible affinity (should stay Pending):
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: impossible-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: gpu
            operator: In
            values: [tesla-v100]  # No node has this label
  containers:
  - name: app
    image: nginx:alpine
EOF

kubectl get pod impossible-pod   # Should be: Pending
kubectl describe pod impossible-pod | grep "Events:" -A 5

# Cleanup:
kubectl delete deployment affinity-lab
kubectl delete pod impossible-pod
kubectl label node $NODE disk-
```

### Lab 6: Debug with Ephemeral Containers

```bash
# Exercise: Debug a minimal production container

# 1. Deploy a distroless container (no shell):
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: distroless-app
spec:
  containers:
  - name: app
    image: gcr.io/distroless/static:nonroot
    command: ["/bin/sleep"]  # This won't work; showing the concept
EOF

# 2. Try exec (fails):
kubectl exec distroless-app -- sh   # Error: no shell

# 3. Add ephemeral debug container:
kubectl debug -it distroless-app \
  --image=busybox:1.36 \
  --target=app \
  -- sh

# Inside debug container, you can inspect:
ls /proc/1/root     # See app container filesystem
cat /proc/1/cmdline # See app's command
ls /proc/1/fd       # Open file descriptors

# 4. Debug a running pod with network tools:
kubectl debug -it distroless-app \
  --image=nicolaka/netshoot \
  -- bash

# Inside netshoot: full network debugging toolkit
nslookup kubernetes
curl -v http://some-service
ss -tlnp     # Active connections
tcpdump      # Packet capture

# Cleanup:
kubectl delete pod distroless-app
```

---

## 26. Interview Questions: Beginner to Advanced

### Beginner Level

**Q1: What is a Pod in Kubernetes and how does it differ from a container?**

**A:** A Pod is the smallest deployable unit in Kubernetes — a logical wrapper that contains one or more containers. While a container is an isolated process with its own filesystem (via Linux namespaces and cgroups), a Pod is a Kubernetes abstraction that groups containers that need to work together. Containers in the same Pod share the same network namespace (same IP address, same localhost), can share volumes, and are always co-scheduled on the same node. The Pod provides the networking and storage primitives that containers alone don't have. Think of a Pod as a tiny host machine: the containers inside it are like processes on that machine, sharing its network and storage.

---

**Q2: Why are Pods ephemeral? What does that mean in practice?**

**A:** Pods are ephemeral because they can be terminated and replaced at any time — by node failures, scaling events, rolling updates, eviction due to resource pressure, or operator intervention. They are not "pets" to be nursed back to health; they are "cattle" to be replaced. In practice this means: (1) Never store important data inside a Pod's container filesystem — it disappears when the Pod is replaced. Use PersistentVolumeClaims for data that must survive Pod restarts. (2) Never rely on a Pod's IP address being stable — it changes each time the Pod is replaced. Use Services for stable networking. (3) Always run Pods via controllers (Deployment, ReplicaSet) so replacement Pods are created automatically. (4) Design applications to start and stop cleanly at any time.

---

**Q3: What is the difference between `restartPolicy: Always`, `OnFailure`, and `Never`?**

**A:** These control how the Kubelet responds when a container exits: **Always**: Always restart the container when it exits, regardless of exit code (0 = success or non-zero = failure). Used for long-running services (web servers, daemons) via Deployments/StatefulSets. **OnFailure**: Restart only when the container exits with a non-zero code (failure). If exit code is 0 (success), don't restart. Used for Jobs that should complete successfully once. **Never**: Never restart. Container exits = pod is done. Used for one-off jobs or debugging. Note: Deployment and DaemonSet controllers require `restartPolicy: Always`. Job controllers require `OnFailure` or `Never`.

---

**Q4: What happens when a Pod is deleted?**

**A:** Pod deletion triggers a graceful termination sequence: (1) `metadata.deletionTimestamp` is set on the pod object. (2) The EndpointSlice controller removes the Pod's IP from the service endpoints — no new traffic is sent. (3) The Kubelet detects the deletion via watch. (4) If a `preStop` lifecycle hook is defined, it's executed. (5) SIGTERM is sent to all containers (PID 1 in each). (6) The Kubelet waits `terminationGracePeriodSeconds` (default 30s) for containers to exit gracefully. (7) If containers are still running after the grace period, SIGKILL is sent. (8) The Pod object is removed from the API server. If the Pod is owned by a ReplicaSet, the controller immediately creates a replacement Pod.

---

### Intermediate Level

**Q5: Explain the difference between liveness, readiness, and startup probes. When would you use each?**

**A:** **Liveness probe**: "Is this container alive?" — If it fails (after `failureThreshold` consecutive failures), the Kubelet **restarts** the container. Use it to recover from deadlocks or corrupted state that prevents the application from functioning but doesn't cause it to exit. **Readiness probe**: "Is this container ready to serve traffic?" — If it fails, the Pod is **removed from Service Endpoints** but NOT restarted. No new requests are sent to it. Use it during startup (before app is ready), during temporary overload, or when an external dependency is unavailable. **Startup probe**: "Has this container finished starting?" — While it's running (and failing), liveness and readiness probes are **disabled**. Once it passes once, it stops running. Use it for slow-starting applications to avoid liveness probe killing the container during initialization. The startup probe replaces the fragile `initialDelaySeconds` pattern.

---

**Q6: What is the pause container and why does every Pod have one?**

**A:** The pause container (also called the "infra container") is a tiny container (a few KB) that runs as PID 1 in the Pod's network namespace. It serves as the anchor for the Pod's network namespace. When `RunPodSandbox()` is called on the container engine, the pause container is created first, establishing the network namespace with an IP address. All application containers then join this existing network namespace. This design has an important benefit: if an application container dies and is restarted, it rejoins the same network namespace — so the Pod's IP address doesn't change. Without the pause container, restarting any container would tear down and rebuild the network namespace, changing the Pod IP. The pause container just runs `pause()` syscall indefinitely and holds the namespace.

---

**Q7: How does topology-aware scheduling work, and what is `topologySpreadConstraints`?**

**A:** `topologySpreadConstraints` is a scheduling directive that instructs the scheduler to spread Pods across defined topology domains (zones, nodes, etc.) to improve availability. `maxSkew` defines the maximum allowed difference in Pod count between any two topology domains. `topologyKey` defines the domain (e.g., `topology.kubernetes.io/zone`). `whenUnsatisfiable: DoNotSchedule` makes it a hard requirement; `ScheduleAnyway` makes it a soft preference. Example: with `maxSkew: 1` across 3 zones (us-east-1a, 1b, 1c), if zone 1a has 3 pods and zone 1b has 2, zone 1c must have at least 2 (to keep skew ≤ 1). The scheduler will refuse to schedule new pods on zone 1a until the others catch up. This complements `podAntiAffinity` by giving precise control over distribution rather than just "not the same node."

---

**Q8: Explain how a Pod gets a network identity and can communicate with other Pods without NAT.**

**A:** When the Kubelet creates a Pod, it calls `RunPodSandbox()` on the container engine (containerd). The container engine creates the pause container and its network namespace. The CRI then calls the CNI (Container Network Interface) plugin, passing the network namespace path. The CNI plugin: (1) Allocates an IP from the node's Pod CIDR (e.g., 10.244.1.5/32). (2) Creates a veth pair: one end in the Pod namespace (eth0), one end on the node (called cali1234, veth1234, etc.). (3) Sets up routing: packets to 10.244.1.5 come from the node's veth end into the Pod. The key to cross-node Pod communication: CNI implementations like Calico, Cilium, or Flannel set up overlay networks or BGP routing so that each node knows how to reach all other nodes' Pod CIDRs. Traffic from Pod A (10.244.1.5) to Pod B (10.244.2.3) goes: Pod A → node A's kernel → overlay network (VXLAN/BGP/etc.) → node B's kernel → Pod B. No NAT is needed because the CNI ensures each Pod IP is directly routable within the cluster.

---

### Advanced Level

**Q9: A Pod is stuck in `Pending`. Walk me through the systematic diagnosis.**

**A:** A Pod stuck in Pending has been accepted by the API server but not yet scheduled or running. Diagnosis flow: (1) `kubectl describe pod <name>` — examine the Events section. The scheduler writes events when it can't schedule. (2) **Insufficient resources**: "0/3 nodes are available: insufficient memory" → nodes don't have enough CPU/memory to fit the pod's requests. Fix: scale up nodes, reduce requests, or check if nodes are cordoned. (3) **No matching nodes (affinity)**: "didn't match pod's node affinity/selector" → `nodeSelector` or `nodeAffinity` has no matching nodes. Check node labels with `kubectl get nodes --show-labels`. (4) **Taints**: "had taint ... that the pod didn't tolerate" → node is tainted and pod has no matching toleration. (5) **PVC not bound**: Pod won't schedule if it references a PVC that isn't Bound. Check `kubectl get pvc`. StorageClass may not have available capacity. (6) **ResourceQuota exceeded**: "exceeded quota: ... hard limit..." → namespace quota exhausted. Check `kubectl describe quota -n <namespace>`. (7) **PodSecurityAdmission**: Pod spec violates the namespace's security policy. Check with `kubectl create pod --dry-run=server`. (8) **No nodes in cluster**: `kubectl get nodes` shows no Ready nodes.

---

**Q10: How does Pod garbage collection work? What happens to completed and failed pods?**

**A:** Kubernetes has a pod garbage collector (part of kube-controller-manager) that cleans up terminated pods. Terminated pods (Succeeded or Failed phase) are not automatically deleted immediately — they're kept for log access. The garbage collector limits the number of terminated pods per node: `--terminated-pod-gc-threshold` (default: 12,500). When this threshold is exceeded, oldest terminated pods are deleted. For Job pods, `ttlSecondsAfterFinished` controls automatic cleanup. The Garbage Collector controller also handles ownerReference-based cascade deletion: when a ReplicaSet is deleted, the GC deletes all pods with `ownerReferences` pointing to that RS. This uses the finalization mechanism: the RS is first marked with a deletion timestamp, the GC deletes dependent pods, then the RS object is fully removed. Orphan deletion (`--cascade=orphan`) removes the owner object but leaves pods intact — useful for adopting pods by a new controller. Background deletion removes the owner immediately and the GC cleans up dependents asynchronously.

---

**Q11: Explain how Kubernetes achieves zero-downtime deployments at the Pod level. What can go wrong?**

**A:** Zero-downtime relies on the choreography of multiple components: (1) **readinessProbe** gates endpoint addition — new pods receive no traffic until they pass the probe. (2) **preStop hook + sleep** provides a buffer between endpoint removal and SIGTERM — ensures kube-proxy has time to remove the pod from iptables rules (kube-proxy sync period = 1s by default; but with propagation delays, 5-10s sleep is safe). (3) **terminationGracePeriodSeconds** allows in-flight requests to complete. (4) **maxUnavailable: 0** ensures the old pod isn't removed from the load balancer until the new pod is ready. What can go wrong: (a) **Missing readinessProbe**: new pod receives traffic before it's ready → 503 errors during startup. (b) **Missing preStop sleep**: endpoint removed + SIGTERM arrives simultaneously; but kube-proxy might not have updated iptables yet → brief window of traffic to a SIGTERM'd pod → connection reset. (c) **Too short terminationGracePeriodSeconds**: long-running requests get SIGKILL mid-processing. (d) **Application doesn't handle SIGTERM**: if PID 1 doesn't propagate SIGTERM to the actual application (common with shell scripts), the application keeps running but Kubelet doesn't know it should drain. (e) **PodDisruptionBudget conflict**: if minAvailable = replicas, the Deployment controller can never scale down the old ReplicaSet → rollout hangs.

---

**Q12: What is the difference between Pod eviction, Pod disruption, and Pod deletion? How do PodDisruptionBudgets protect against each?**

**A:** **Pod deletion**: Deliberate removal via API (`kubectl delete pod` or controller). Respects `terminationGracePeriodSeconds`. Can be forced with `--grace-period=0`. **Pod eviction**: Resource-pressure-driven removal by the Kubelet's eviction manager when node resources (memory, disk) are critically low. Evicts by QoS class: BestEffort first, then Burstable, then Guaranteed. Also includes API-driven eviction (`kubectl drain` uses the Eviction API). **Pod disruption**: A broader term covering any involuntary removal — node failures (Unknown phase), taint-based eviction (NoExecute), and voluntary disruptions (drain, rolling updates). **PodDisruptionBudgets (PDBs)** protect against *voluntary* disruptions only: `minAvailable` or `maxUnavailable` defines how many pods must remain available. The Kubernetes Eviction API (used by `kubectl drain`, cluster autoscaler) checks PDBs before proceeding. If evicting a pod would violate the PDB, the eviction is rejected (409 Conflict) and the drain waits. PDBs do NOT protect against: involuntary disruptions (node failure), resource-pressure eviction (that's the eviction manager, not the Eviction API), or forced pod deletion (`kubectl delete pod --force --grace-period=0`).

---

## 27. Cheat Sheet: Commands & Flags

### Essential kubectl Pod Commands

```bash
# ── CREATE & MANAGE ────────────────────────────────────────────────────
# Create pod from YAML:
kubectl apply -f pod.yaml

# Create pod quickly (for testing):
kubectl run my-pod --image=nginx:alpine --restart=Never

# Create pod with resource limits:
kubectl run my-pod --image=nginx:alpine \
  --requests='cpu=100m,memory=128Mi' \
  --limits='cpu=500m,memory=256Mi'

# Create pod and exec immediately:
kubectl run debug --image=busybox --rm -it --restart=Never -- sh

# Delete pod:
kubectl delete pod my-pod
kubectl delete pod my-pod --grace-period=0 --force    # Force delete
kubectl delete pods -l app=my-app -n production       # Delete by label

# ── VIEW & INSPECT ─────────────────────────────────────────────────────
# List pods:
kubectl get pods
kubectl get pods -A                                    # All namespaces
kubectl get pods -n production -o wide                 # Show node, IP
kubectl get pods --show-labels                         # Show labels
kubectl get pods -l app=my-app                         # Filter by label
kubectl get pods --field-selector status.phase=Running # Filter by phase

# Describe pod (full details):
kubectl describe pod my-pod -n production

# Get pod YAML:
kubectl get pod my-pod -o yaml

# Get specific fields:
kubectl get pod my-pod -o jsonpath='{.status.podIP}'
kubectl get pod my-pod -o jsonpath='{.spec.nodeName}'
kubectl get pod my-pod -o jsonpath='{.status.phase}'
kubectl get pod my-pod -o jsonpath='{.status.qosClass}'
kubectl get pod my-pod -o jsonpath='{.status.containerStatuses[0].restartCount}'
kubectl get pod my-pod -o jsonpath='{.status.containerStatuses[0].state}'
kubectl get pod my-pod -o jsonpath='{.metadata.ownerReferences[0].kind}'

# Custom columns:
kubectl get pods -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.phase,\
NODE:.spec.nodeName,\
IP:.status.podIP,\
RESTARTS:.status.containerStatuses[0].restartCount

# ── LOGS ────────────────────────────────────────────────────────────────
kubectl logs my-pod
kubectl logs my-pod -f                                # Follow
kubectl logs my-pod --previous                        # Previous container
kubectl logs my-pod -c my-container                   # Specific container
kubectl logs my-pod --tail=100                        # Last 100 lines
kubectl logs my-pod --since=1h                        # Last 1 hour
kubectl logs -l app=my-app -n production --max-log-requests=10

# ── EXEC & DEBUG ────────────────────────────────────────────────────────
kubectl exec my-pod -- command
kubectl exec -it my-pod -- /bin/sh
kubectl exec -it my-pod -c sidecar -- bash            # Specific container
kubectl exec my-pod -- env | grep DATABASE            # Check env vars
kubectl exec my-pod -- cat /etc/resolv.conf           # DNS config
kubectl exec my-pod -- wget -qO- http://my-service    # Test service

# Ephemeral debug container:
kubectl debug -it my-pod --image=busybox -- sh
kubectl debug -it my-pod --image=nicolaka/netshoot --target=app -- bash

# ── PORT FORWARD ────────────────────────────────────────────────────────
kubectl port-forward pod/my-pod 8080:80
kubectl port-forward svc/my-service 8080:80

# ── EVENTS ──────────────────────────────────────────────────────────────
kubectl get events -n production
kubectl get events -n production --sort-by='.lastTimestamp'
kubectl get events -n production --field-selector involvedObject.name=my-pod
kubectl get events -n production -w                    # Watch events

# ── RESOURCE USAGE ──────────────────────────────────────────────────────
kubectl top pods
kubectl top pods -n production --containers
kubectl top pods --sort-by=memory
kubectl top nodes

# ── ROLLOUTS ────────────────────────────────────────────────────────────
kubectl rollout status deployment/my-app
kubectl rollout history deployment/my-app
kubectl rollout undo deployment/my-app
kubectl rollout undo deployment/my-app --to-revision=3
kubectl rollout pause deployment/my-app
kubectl rollout resume deployment/my-app

# ── SCALING ─────────────────────────────────────────────────────────────
kubectl scale deployment my-app --replicas=5
kubectl autoscale deployment my-app --min=3 --max=10 --cpu-percent=70

# ── NODE MANAGEMENT ─────────────────────────────────────────────────────
kubectl cordon <node>                              # Prevent new pods
kubectl uncordon <node>                            # Re-enable scheduling
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl taint nodes <node> key=value:NoSchedule
kubectl taint nodes <node> key:NoSchedule-         # Remove taint

# ── LABEL & ANNOTATE ────────────────────────────────────────────────────
kubectl label pod my-pod environment=production
kubectl label pod my-pod environment-               # Remove label
kubectl annotate pod my-pod description="my pod"

# ── PATCHES ─────────────────────────────────────────────────────────────
# Remove finalizers from stuck pod:
kubectl patch pod my-pod -p '{"metadata":{"finalizers":[]}}' --type=merge

# Update image:
kubectl set image deployment/my-app app=myapp:v1.2.3

# Update resources:
kubectl set resources deployment/my-app \
  --containers=app \
  --requests=cpu=250m,memory=256Mi \
  --limits=cpu=1000m,memory=512Mi
```

### Pod Spec Quick Reference

```yaml
# Minimal production pod template:
spec:
  # Scheduling:
  nodeSelector: {}                           # Simple node selection
  affinity: {}                               # Complex affinity/anti-affinity
  tolerations: []                            # Tolerate node taints
  topologySpreadConstraints: []             # Spread across zones/nodes
  priorityClassName: ""                     # Scheduling priority

  # Identity:
  serviceAccountName: my-sa
  automountServiceAccountToken: false

  # Lifecycle:
  restartPolicy: Always                     # Always|OnFailure|Never
  terminationGracePeriodSeconds: 30

  # Security:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault

  # Init containers (run before app):
  initContainers: []

  # Main containers:
  containers:
  - name: app
    image: myapp:v1.0
    imagePullPolicy: IfNotPresent           # Always|IfNotPresent|Never
    ports: []
    env: []
    envFrom: []
    resources:
      requests: {}
      limits: {}
    volumeMounts: []
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: [ALL]
    lifecycle:
      preStop:
        exec:
          command: ["sleep", "10"]
    startupProbe: {}
    livenessProbe: {}
    readinessProbe: {}

  # Volumes:
  volumes: []
  imagePullSecrets: []
  runtimeClassName: ""
```

### Diagnostic One-Liners

```bash
# Find all pods not in Running state:
kubectl get pods -A | grep -v "Running\|Completed"

# Find pods with high restart counts:
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.status.containerStatuses[0].restartCount > 5) |
  "\(.metadata.namespace)/\(.metadata.name): \(.status.containerStatuses[0].restartCount)"'

# Find pods without resource requests:
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.spec.containers[0].resources.requests == null) |
  "\(.metadata.namespace)/\(.metadata.name)"'

# Find pods not using a specific service account:
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.spec.serviceAccountName != "my-sa") | .metadata.name'

# Show pod-to-node mapping:
kubectl get pods -A -o wide | awk '{print $8, $2, $4}'

# Watch pod restart counts:
watch -n 5 'kubectl get pods -A -o json | jq -r ".items[] |
  select(.status.containerStatuses[0].restartCount > 0) |
  \"\(.metadata.namespace)/\(.metadata.name): \(.status.containerStatuses[0].restartCount)\""'

# Find which pods are on a specific node:
kubectl get pods -A --field-selector spec.nodeName=worker-1 -o wide

# Check all pod QoS classes:
kubectl get pods -A -o json | \
  jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name): \(.status.qosClass)"'

# Export all pod YAML from a namespace (backup):
kubectl get pods -n production -o yaml > production-pods-backup.yaml
```

---

## 28. Key Takeaways & Summary

### The Essential Mental Model

The Pod is the **atom of Kubernetes** — everything else is built on top of it. Controllers manage groups of Pods to achieve desired states. Services provide stable networking to dynamic Pod sets. PersistentVolumeClaims provide durable storage. Network policies control Pod communication. SecurityContexts enforce isolation. Understanding Pods completely is understanding the foundation of every Kubernetes concept.

### The 15 Most Critical Things to Know About Pods

1. **A Pod is NOT a container** — it's a wrapper for one or more containers sharing a network namespace, storage, and lifecycle. The pause container holds the network namespace.

2. **Pods are ephemeral** — their IPs change, they can be killed at any time. Never rely on a Pod IP or store critical data in a Pod's local filesystem.

3. **Always use controllers** — bare Pods have no self-healing. Use Deployment, StatefulSet, DaemonSet, or Job. The controller creates the Pod; the Kubelet runs it.

4. **The scheduling pipeline**: controller creates Pod (no nodeName) → scheduler assigns node (writes nodeName) → kubelet starts containers → CRI + CNI + CSI do the physical work.

5. **Containers within a Pod share network** — same IP, same localhost port space. They can't both bind port 80. They communicate via `localhost`.

6. **Init containers enforce startup ordering** — they run sequentially before the main app. Sidecar containers (init + restartPolicy:Always) run alongside the app.

7. **Three probes serve distinct purposes**: liveness = restart on failure; readiness = remove from load balancer; startup = block liveness/readiness during startup.

8. **Resources: requests for scheduling, limits for isolation** — CPU limits throttle (don't kill); memory limits OOMKill. Always set both. Match requests = limits for Guaranteed QoS.

9. **Pod termination is a graceful sequence**: endpoints removed → preStop hook → SIGTERM → grace period → SIGKILL. Add `preStop: sleep 5-10` for kube-proxy propagation time.

10. **SecurityContext is your security posture** — `runAsNonRoot`, `readOnlyRootFilesystem`, `capabilities.drop: ALL`, `allowPrivilegeEscalation: false`. These should be default.

11. **Pods never talk to etcd** — all cluster state goes through kube-apiserver. Pods use ServiceAccount tokens to authenticate to the API server if they need cluster access.

12. **PodDisruptionBudgets protect availability** — define them for all production workloads so node drains and upgrades don't take down your service.

13. **Ephemeral containers enable production debugging** — add a debug container to a running Pod without restarting it, even if the production image has no shell.

14. **TopologySpreadConstraints and podAntiAffinity prevent availability zones from becoming single points of failure** — always spread stateless services.

15. **Events are your first debugging tool** — `kubectl describe pod` Events section tells you exactly what happened at every phase of the Pod's lifecycle.

### Quick Diagnosis Decision Tree

```
Pod not starting?
  → Pending:   scheduler problem (resources, affinity, taints, PVC, quota)
  → ContainerCreating: CRI/CNI problem (image pull, volume mount, sandbox)
  → CrashLoopBackOff: application problem (check logs --previous, exit code)
  → OOMKilled (137): memory limit too low, increase it
  → ImagePullBackOff: registry auth, image name/tag wrong

Pod not serving traffic?
  → Not in endpoints? → readinessProbe failing (check logs, probe config)
  → Endpoints exist? → kube-proxy/network issue (not a Pod problem)

Pod disappearing?
  → Node NotReady? → kubelet/node problem, controller reschedules
  → Evicted? → node resource pressure (disk/memory), check QoS class
  → Completed? → restartPolicy: Never with Job (expected)

Pod hanging on delete?
  → Finalizers blocking? → kubectl get pod -o jsonpath='{.metadata.finalizers}'
  → preStop hook hung? → kubectl logs --previous
  → Volume unmount stuck? → check kubelet logs on node
```

---

*This document represents production-grade Kubernetes Pod knowledge compiled for DevOps engineers, SREs, and platform engineers. All configurations and commands should be validated in non-production environments before applying to production systems.*

---

**Document Version:** 1.0
**Last Updated:** April 2026
**Kubernetes Reference Version:** v1.28+

---
*End of Document*
