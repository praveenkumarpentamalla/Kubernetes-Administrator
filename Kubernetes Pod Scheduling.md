## Table of Contents

1. [Introduction — Pod Scheduling Fundamentals](#1-introduction--pod-scheduling-fundamentals)
2. [Core Identity Table — Scheduling Components](#2-core-identity-table--scheduling-components)
3. [Containers vs Pods — Architecture Deep Dive](#3-containers-vs-pods--architecture-deep-dive)
4. [The Controller Pattern — Watch → Compare → Act → Loop](#4-the-controller-pattern--watch--compare--act--loop)
5. [Manual Pod Scheduling — How It Works Internally](#5-manual-pod-scheduling--how-it-works-internally)
6. [Lab: Creating Pods Using kubectl CLI](#6-lab-creating-pods-using-kubectl-cli)
7. [Lab: Creating Pods Using YAML Manifests](#7-lab-creating-pods-using-yaml-manifests)
8. [kubectl create vs kubectl apply — The Complete Comparison](#8-kubectl-create-vs-kubectl-apply--the-complete-comparison)
9. [Lab: Versioned Resources with kubectl apply](#9-lab-versioned-resources-with-kubectl-apply)
10. [Labels and Selectors in Pods — Deep Dive](#10-labels-and-selectors-in-pods--deep-dive)
11. [Lab: Setting Labels and Selectors on Pods](#11-lab-setting-labels-and-selectors-on-pods)
12. [Resource Requests, Limits, and Quality of Service](#12-resource-requests-limits-and-quality-of-service)
13. [Lab: Setting Resource Requests and Limits on Pods](#13-lab-setting-resource-requests-and-limits-on-pods)
14. [Lab: Troubleshooting Resource Limit Issues](#14-lab-troubleshooting-resource-limit-issues)
15. [Built-in Controllers Deep Dive](#15-built-in-controllers-deep-dive)
16. [Internal Working Concepts — Informers, Work Queues & Reconciliation](#16-internal-working-concepts--informers-work-queues--reconciliation)
17. [API Server and etcd Interaction](#17-api-server-and-etcd-interaction)
18. [Leader Election — HA Scheduling Infrastructure](#18-leader-election--ha-scheduling-infrastructure)
19. [Capturing Resource Metrics](#19-capturing-resource-metrics)
20. [Lab: Installing the Metrics Server](#20-lab-installing-the-metrics-server)
21. [Lab: Filtering Resources Based on Compute Usage](#21-lab-filtering-resources-based-on-compute-usage)
22. [DaemonSets — Node-Level Pod Management](#22-daemonsets--node-level-pod-management)
23. [Static Pods — kubelet-Managed Workloads](#23-static-pods--kubelet-managed-workloads)
24. [Multiple Schedulers — Custom Scheduling Logic](#24-multiple-schedulers--custom-scheduling-logic)
25. [Port Mapping on Pods](#25-port-mapping-on-pods)
26. [Lab: Port Mapping to Access Containers in a Pod](#26-lab-port-mapping-to-access-containers-in-a-pod)
27. [Performance Tuning for Scheduling](#27-performance-tuning-for-scheduling)
28. [Security Hardening Practices](#28-security-hardening-practices)
29. [Monitoring and Observability](#29-monitoring-and-observability)
30. [Troubleshooting — Real kubectl Commands](#30-troubleshooting--real-kubectl-commands)
31. [Disaster Recovery Concepts](#31-disaster-recovery-concepts)
32. [Comparison — kube-apiserver vs kube-scheduler vs kube-controller-manager](#32-comparison--kube-apiserver-vs-kube-scheduler-vs-kube-controller-manager)
33. [ASCII Architecture Diagram](#33-ascii-architecture-diagram)
34. [Real-World Production Use Cases](#34-real-world-production-use-cases)
35. [Best Practices for Production Environments](#35-best-practices-for-production-environments)
36. [Common Mistakes and Pitfalls](#36-common-mistakes-and-pitfalls)
37. [Interview Questions — Beginner to Advanced](#37-interview-questions--beginner-to-advanced)
38. [Cheat Sheet — Commands, Flags & Manifests](#38-cheat-sheet--commands-flags--manifests)
39. [Key Takeaways & Summary](#39-key-takeaways--summary)

---

## 1. Introduction — Pod Scheduling Fundamentals

Kubernetes does not directly run containers — it runs **Pods**. And those Pods don't appear on nodes by magic — they are placed there by a sophisticated scheduling system that considers resources, constraints, taints, tolerations, affinity rules, and topology. Understanding Pod scheduling is the foundation of everything else in Kubernetes operations.

This guide is a comprehensive, production-grade examination of the complete Pod scheduling pipeline: from how containers and Pods relate, through the mechanics of scheduling, resource management, custom schedulers, and DaemonSets, all the way to troubleshooting performance and access issues.

### Why Pod Scheduling Matters

| Concern | Without Understanding Scheduling | With Understanding |
|---|---|---|
| **Resource efficiency** | Over-provisioned nodes, wasted cost | Right-sized Pods, optimal bin-packing |
| **Reliability** | Pods starved of resources, OOMKilled | QoS classes ensure critical workloads get resources |
| **Performance** | Hot spots, uneven node utilization | Labels + selectors route workloads to right nodes |
| **Operations** | Can't explain why Pod is Pending | Can diagnose scheduling failures in minutes |
| **Security** | Workloads land anywhere | node affinity + taints enforce placement policies |
| **Cost** | VM instance sizes guessed randomly | Resource requests drive Kubernetes scheduler to pack optimally |

### The Scheduling Pipeline at a Glance

```
User: kubectl apply -f pod.yaml
        │
        ▼
kube-apiserver  →  stores Pod (with spec.nodeName="") in etcd
        │
        ▼
kube-scheduler  →  WATCHES for unscheduled Pods
                →  FILTERS nodes (resources, taints, affinity)
                →  SCORES nodes (most free resources wins)
                →  BINDS Pod to winning node (writes spec.nodeName)
        │
        ▼
kubelet on node →  WATCHES for Pods bound to its node
                →  Pulls image, creates containers via CRI
                →  Reports readiness back to API server
```

---

## 2. Core Identity Table — Scheduling Components

| Component | Binary | Port | Protocol | Role in Scheduling |
|---|---|---|---|---|
| **kube-apiserver** | `kube-apiserver` | 6443 | HTTPS | Receives Pod manifests; stores in etcd; source of truth |
| **kube-scheduler** | `kube-scheduler` | 10259 | HTTPS | Assigns `spec.nodeName` to pending Pods |
| **kube-controller-manager** | `kube-controller-manager` | 10257 | HTTPS | Manages ReplicaSet/Deployment/DaemonSet/Job controllers |
| **kubelet** | `kubelet` | 10250 | HTTPS | Node agent; starts Pods assigned to its node via CRI |
| **etcd** | `etcd` | 2379, 2380 | HTTPS/gRPC | Stores all Pod specs, bindings, node status |
| **containerd / cri-o** | CRI socket | Unix socket | gRPC/CRI | Container runtime: pulls images, creates containers |
| **kube-proxy** | `kube-proxy` | 10249 | HTTP | Programs iptables/IPVS for Service port routing |
| **CoreDNS** | `coredns` | 53, 9153 | UDP/TCP | DNS for Service discovery inside Pods |
| **metrics-server** | `metrics-server` | 443 | HTTPS | Aggregates CPU/memory from kubelets for `kubectl top` |
| **Static Pod manifests** | File path | N/A | N/A | YAML files in `/etc/kubernetes/manifests/` for kubelet-managed Pods |

---

## 3. Containers vs Pods — Architecture Deep Dive

### 3.1 What Is a Container?

A container is a Linux process running in an isolated environment using:
- **namespaces** (pid, net, uts, ipc, mnt, user): process isolation
- **cgroups**: CPU, memory, I/O resource limits
- **overlay filesystem**: copy-on-write layer over a base image

Containers by themselves have no networking concept shared with other containers, no shared filesystem policy, and no lifecycle management in Kubernetes.

### 3.2 What Is a Pod?

A Pod is the **smallest deployable unit in Kubernetes**. It is a logical wrapper around one or more containers that:
- Share the **same network namespace** (same IP, same port space)
- Optionally share **volumes** (filesystem paths)
- Share the **same lifecycle** (created/destroyed together)
- Run in the **same cgroup hierarchy**

```
┌─────────────────────────────────────────────────────────┐
│                         POD                             │
│  Network Namespace: 10.244.1.5                          │
│  Volumes: /shared-data, /config                         │
│                                                         │
│  ┌───────────────────────────┐  ┌──────────────────┐   │
│  │    Main Container         │  │   Sidecar        │   │
│  │    app:8080               │  │   nginx-proxy:80 │   │
│  │    mounts: /app, /config  │  │   mounts: /logs  │   │
│  └───────────────────────────┘  └──────────────────┘   │
│                                                         │
│  ┌───────────────────────────┐                          │
│  │   Init Container          │                          │
│  │   (runs first, must exit) │                          │
│  └───────────────────────────┘                          │
│                                                         │
│  Pause Container (infra/sandbox) — holds network NS     │
└─────────────────────────────────────────────────────────┘
```

### 3.3 The Pause Container

The **pause** (or infra/sandbox) container is created first for every Pod. It:
- Holds the network namespace open (prevents it from being destroyed if app containers restart)
- Acts as PID 1 and reaps zombie processes
- Has no application code — just `/pause` binary

```bash
# See pause containers running on a node
sudo crictl ps | grep pause
# You'll see one per running Pod

# The pause container image
kubectl get pod nginx -o jsonpath='{.spec.containers[*].image}' 
# Shows app containers; pause is hidden

# API server flag that sets pause image:
# --pod-infra-container-image=registry.k8s.io/pause:3.9
```

### 3.4 Container vs Pod Comparison

| Aspect | Container | Pod |
|---|---|---|
| **Kubernetes visibility** | Not directly visible (via CRI) | First-class API object |
| **IP address** | Shares Pod IP | One IP per Pod |
| **Port space** | Shares Pod port space | Containers in same Pod can't reuse ports |
| **Filesystem** | Isolated unless volumeMount | Can share via volumes |
| **Network** | Loopback to other containers in same Pod | Full IP connectivity to all other Pods |
| **Lifecycle** | Can restart independently | All restart when Pod restarts |
| **Init sequence** | N/A | Init containers run first, then app containers |
| **Scheduling** | Not scheduled directly | Scheduled as a unit to one node |

### 3.5 Multi-Container Pod Patterns

| Pattern | Description | Use Case |
|---|---|---|
| **Sidecar** | Helper runs alongside main app | Log shippers, service mesh proxies, config reloaders |
| **Ambassador** | Proxy external connections | Local DB proxy handling connection pooling |
| **Adapter** | Standardizes app output | Convert proprietary metrics format to Prometheus |
| **Init container** | Runs before app containers | DB migration, wait for dependency, create files |

```yaml
# Example: Multi-container Pod with init container and sidecar
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-demo
spec:
  initContainers:
    - name: init-wait-for-db
      image: busybox:1.36
      command: ['sh', '-c', 
        'until nc -z postgres-svc 5432; do echo waiting; sleep 2; done']
  containers:
    - name: main-app
      image: my-app:v1.0
      ports:
        - containerPort: 8080
      volumeMounts:
        - name: shared-logs
          mountPath: /app/logs
    - name: log-shipper
      image: fluentd:v1.14
      volumeMounts:
        - name: shared-logs
          mountPath: /logs
  volumes:
    - name: shared-logs
      emptyDir: {}
```

---

## 4. The Controller Pattern — Watch → Compare → Act → Loop

Understanding the controller pattern is essential for understanding how every scheduling-related resource works. When you create a Deployment, DaemonSet, or StatefulSet, a controller watches your intent and continuously works to make reality match it.

### 4.1 The Four-Phase Reconciliation Loop

```
┌─────────────────────────────────────────────────────────────────────┐
│                   CONTROLLER RECONCILIATION LOOP                    │
│                                                                     │
│   1. WATCH                                                          │
│   ─────────                                                         │
│   Informer streams ADDED/MODIFIED/DELETED events from API server    │
│   via a persistent HTTP watch connection.                           │
│   Events are placed into a work queue (deduplicated).               │
│                    │                                                │
│                    ▼                                                │
│   2. COMPARE                                                        │
│   ──────────                                                        │
│   Worker goroutine picks key from work queue.                       │
│   Reads DESIRED state from object's .spec field.                    │
│   Reads CURRENT state from cluster (via informer cache).            │
│   Computes the delta.                                               │
│                    │                                                │
│                    ▼                                                │
│   3. ACT                                                            │
│   ───────                                                           │
│   Make API calls to create/update/delete resources.                 │
│   Update .status field with current state.                          │
│   If error: workQueue.AddRateLimited(key)  [backoff retry]          │
│                    │                                                │
│                    ▼                                                │
│   4. LOOP                                                           │
│   ───────                                                           │
│   workQueue.Forget(key) on success.                                 │
│   Return to WATCH for next event.                                   │
│   Periodic resync (10-30 min) triggers full reconciliation.         │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 Applying the Pattern to Pod Scheduling

```bash
# You submit: kubectl apply -f deployment.yaml (replicas: 3)

# WATCH:
#   Deployment controller informer detects new Deployment
#   Event: ADDED deployment/my-app

# COMPARE:
#   Desired: 3 running Pods
#   Current: 0 running Pods
#   Delta: need 3 more Pods

# ACT:
#   Create ReplicaSet with same Pod template
#   ReplicaSet controller creates 3 Pods (nodeName="")
#   kube-scheduler detects 3 unscheduled Pods
#   Scheduler assigns nodeName to each Pod

# LOOP:
#   kubelet on target nodes detect their new Pods
#   kubelet starts containers via containerd
#   kubelet updates Pod.status.phase = Running
#   Deployment controller updates deployment.status.readyReplicas = 3
```

---

## 5. Manual Pod Scheduling — How It Works Internally

### 5.1 What "Manual Scheduling" Means

Normally, the kube-scheduler decides which node a Pod runs on. **Manual scheduling** means you bypass the scheduler by directly specifying `spec.nodeName` in the Pod manifest.

```yaml
# Manual scheduling: Skip the scheduler entirely
apiVersion: v1
kind: Pod
metadata:
  name: manually-scheduled-pod
spec:
  nodeName: k8s-worker-1    # Bypass scheduler; assign directly to this node
  containers:
    - name: nginx
      image: nginx:1.25
```

### 5.2 The Normal Scheduling Pipeline vs Manual

| Aspect | Normal Scheduling | Manual Scheduling |
|---|---|---|
| **Who assigns node** | kube-scheduler | You (in manifest) |
| **Resource validation** | Scheduler checks node capacity | No check — Pod may fail if node full |
| **Taint/toleration check** | Scheduler respects them | Pod is placed regardless of taints |
| **Affinity/anti-affinity** | Scheduler evaluates | Ignored |
| **Binding API call** | Scheduler calls Binding API | Not needed (nodeName already set) |
| **If node doesn't exist** | Pod stays Pending (scheduler retries) | Pod stays Pending indefinitely |
| **Use case** | Normal operations | Control plane bootstrapping, debugging, testing |

### 5.3 The Binding Process

When the scheduler selects a node, it creates a **Binding** object (or directly patches `spec.nodeName`):

```bash
# What the scheduler does internally (you can also do this manually):
curl -X POST \
  -H "Content-Type: application/json" \
  https://<api-server>:6443/api/v1/namespaces/default/pods/my-pod/binding \
  -d '{
    "apiVersion": "v1",
    "kind": "Binding",
    "metadata": {"name": "my-pod"},
    "target": {"apiVersion": "v1", "kind": "Node", "name": "k8s-worker-1"}
  }'

# Or simpler: patch spec.nodeName directly
kubectl patch pod my-pending-pod \
  -p '{"spec":{"nodeName":"k8s-worker-1"}}'
```

### 5.4 Observing the Scheduling Process

```bash
# Watch a Pod get scheduled (see nodeName get assigned)
kubectl get pod my-pod -w
# Initially: NOMINATED NODE = <empty>
# After scheduling: NODE = k8s-worker-1

# See scheduling events
kubectl describe pod my-pod | grep -A 5 "Events:"
# Events:
#   Normal   Scheduled  10s   default-scheduler  Successfully assigned
#              default/my-pod to k8s-worker-1

# See scheduler decisions in scheduler logs
kubectl logs -n kube-system kube-scheduler-$(hostname) \
  --tail=50 | grep "Attempting to bind"
```

---

## 6. Lab: Creating Pods Using kubectl CLI

### 6.1 Imperative Pod Creation

```bash
# ── BASIC POD CREATION ──────────────────────────────────────────────────

# Create a pod with a single container
kubectl run nginx-pod \
  --image=nginx:1.25 \
  --port=80

# Verify it's running
kubectl get pod nginx-pod
kubectl get pod nginx-pod -o wide  # Shows node assignment

# Create pod with environment variables
kubectl run env-pod \
  --image=nginx:1.25 \
  --env="ENV=production" \
  --env="LOG_LEVEL=info"

# Create pod with resource limits
kubectl run limited-pod \
  --image=nginx:1.25 \
  --requests="cpu=100m,memory=128Mi" \
  --limits="cpu=200m,memory=256Mi"

# Create pod with specific labels
kubectl run labeled-pod \
  --image=nginx:1.25 \
  --labels="app=web,env=production,team=platform"

# Create pod and open interactive terminal immediately
kubectl run debug-pod \
  --image=busybox:1.36 \
  --restart=Never \
  -it \
  -- sh

# Run a one-off command in a pod (auto-delete after)
kubectl run temp-curl \
  --image=curlimages/curl:7.87.0 \
  --restart=Never \
  --rm \
  -it \
  -- curl http://nginx-service

# ── GENERATE YAML WITHOUT CREATING ─────────────────────────────────────
# VERY USEFUL: Generate YAML for a pod without creating it
kubectl run nginx-pod \
  --image=nginx:1.25 \
  --dry-run=client \
  -o yaml

# Save the generated YAML to a file
kubectl run nginx-pod \
  --image=nginx:1.25 \
  --port=80 \
  --dry-run=client \
  -o yaml > nginx-pod.yaml

# View what would be created
cat nginx-pod.yaml
```

### 6.2 Inspecting Pods via CLI

```bash
# ── STATUS AND DETAILS ──────────────────────────────────────────────────

# Basic pod listing
kubectl get pods
kubectl get pods -o wide     # Include node, IP
kubectl get pods -A          # All namespaces
kubectl get pods -n kube-system  # Specific namespace

# Detailed pod information
kubectl describe pod nginx-pod
# Shows: Events, node, IP, containers, resource limits, volumes

# Pod YAML (current state + status)
kubectl get pod nginx-pod -o yaml

# Pod JSON format for jq processing
kubectl get pod nginx-pod -o json | jq '.status.phase'

# ── LOGS ──────────────────────────────────────────────────────────────
kubectl logs nginx-pod               # Current container logs
kubectl logs nginx-pod --tail=50     # Last 50 lines
kubectl logs nginx-pod -f            # Follow (stream) logs
kubectl logs nginx-pod --previous    # Previous container (after crash)
kubectl logs nginx-pod -c sidecar    # Specific container in multi-container pod

# ── EXEC INTO POD ────────────────────────────────────────────────────
kubectl exec -it nginx-pod -- bash        # Interactive bash
kubectl exec -it nginx-pod -- sh          # If bash not available
kubectl exec nginx-pod -- nginx -v        # Run single command
kubectl exec nginx-pod -c sidecar -- cat /etc/config  # Specific container

# ── DELETION ─────────────────────────────────────────────────────────
kubectl delete pod nginx-pod
kubectl delete pod nginx-pod --grace-period=0 --force  # Immediate (dangerous)
kubectl delete pods --all                               # All pods in current namespace
kubectl delete pods -l app=nginx                        # By label selector
```

### 6.3 Pod Lifecycle from CLI

```bash
# Watch pod lifecycle stages
kubectl get pod nginx-pod -w
# Pending → ContainerCreating → Running

# Wait for pod to be ready
kubectl wait pod/nginx-pod --for=condition=Ready --timeout=60s

# Wait for multiple pods
kubectl wait pods -l app=nginx --for=condition=Ready --timeout=120s

# Force-delete a stuck terminating pod
kubectl delete pod stuck-pod \
  --grace-period=0 \
  --force \
  --ignore-not-found=true
```

---

## 7. Lab: Creating Pods Using YAML Manifests

### 7.1 Minimal Pod YAML

```yaml
# File: minimal-pod.yaml
# The absolute minimum required fields for a pod
apiVersion: v1
kind: Pod
metadata:
  name: minimal-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.25
```

```bash
kubectl apply -f minimal-pod.yaml
kubectl get pod minimal-pod
```

### 7.2 Production-Grade Pod YAML (Annotated)

```yaml
# File: production-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: production-app
  namespace: production
  # Labels: used for selection, filtering, and routing
  labels:
    app: my-api
    version: "2.1.0"
    environment: production
    tier: backend
    team: platform
  # Annotations: non-identifying metadata (for tools, CI/CD, etc.)
  annotations:
    kubernetes.io/change-cause: "Deploying v2.1.0 with performance fixes"
    deployment.company.com/deployed-by: "github-actions"
    deployment.company.com/jira-ticket: "PLAT-1234"

spec:
  # ServiceAccount to use (creates RBAC identity for this Pod)
  serviceAccountName: my-api-sa
  automountServiceAccountToken: false  # Disable if not needed

  # Pod-level security context
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault

  # Init containers run first, sequentially, to completion
  initContainers:
    - name: init-db-migration
      image: my-migrator:v1.0
      command: ["./run-migrations.sh"]
      env:
        - name: DB_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: connection-string
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 500m
          memory: 256Mi

  # Main application containers
  containers:
    - name: api-server
      image: my-api:v2.1.0
      imagePullPolicy: IfNotPresent   # Always, Never, IfNotPresent

      # Port declarations (informational + used by Services)
      ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        - name: metrics
          containerPort: 9090
          protocol: TCP

      # Environment variables
      env:
        - name: LOG_LEVEL
          value: "info"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        - name: CONFIG_PATH
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: config-path
        # Downward API: access Pod metadata from inside container
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP

      # Resource management (CRITICAL for scheduling)
      resources:
        requests:        # Guaranteed minimum; used by scheduler
          cpu: 250m      # 0.25 CPU cores
          memory: 256Mi  # 256 MiB RAM
        limits:          # Hard maximum; enforced by cgroups
          cpu: 500m      # 0.5 CPU cores
          memory: 512Mi  # 512 MiB RAM

      # Health probes
      startupProbe:      # Delays other probes until app is started
        httpGet:
          path: /health/startup
          port: 8080
        failureThreshold: 30
        periodSeconds: 10

      livenessProbe:     # Restart container if this fails
        httpGet:
          path: /health/live
          port: 8080
        initialDelaySeconds: 0  # startupProbe handles the delay
        periodSeconds: 10
        failureThreshold: 3
        timeoutSeconds: 5

      readinessProbe:    # Remove from Service endpoints if this fails
        httpGet:
          path: /health/ready
          port: 8080
        initialDelaySeconds: 0
        periodSeconds: 5
        failureThreshold: 3
        successThreshold: 1

      # Volume mounts
      volumeMounts:
        - name: app-config
          mountPath: /etc/app
          readOnly: true
        - name: tmp-dir
          mountPath: /tmp

      # Container-level security context
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL

      # Lifecycle hooks
      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]  # Drain connections before SIGTERM

  # Volumes
  volumes:
    - name: app-config
      configMap:
        name: app-config
    - name: tmp-dir
      emptyDir: {}

  # Scheduling configuration
  restartPolicy: Always    # Always, OnFailure, Never

  # Grace period before force-killing during termination
  terminationGracePeriodSeconds: 60

  # DNS configuration
  dnsPolicy: ClusterFirst   # ClusterFirst, ClusterFirstWithHostNet, Default, None

  # Node selection (optional)
  nodeSelector:
    kubernetes.io/arch: amd64
    node-role: application

  # Image pull secrets (for private registries)
  imagePullSecrets:
    - name: registry-credentials

  # Topology constraints (spread Pods across AZs)
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: my-api
```

### 7.3 YAML Validation and Dry-Run

```bash
# Validate YAML without applying
kubectl apply -f production-pod.yaml --dry-run=client
kubectl apply -f production-pod.yaml --dry-run=server  # Includes admission webhook validation

# Explain any YAML field interactively
kubectl explain pod.spec.containers.resources
kubectl explain pod.spec.securityContext
kubectl explain pod.spec.containers.livenessProbe

# Diff: see what would change
kubectl diff -f production-pod.yaml
```

---

## 8. kubectl create vs kubectl apply — The Complete Comparison

### 8.1 The Core Difference

| Aspect | `kubectl create` | `kubectl apply` |
|---|---|---|
| **Operation** | Imperative: "Do this now" | Declarative: "Make it so" |
| **If resource exists** | Error: `AlreadyExists` | Updates to match desired state |
| **If resource doesn't exist** | Creates it | Creates it |
| **Tracking history** | No annotation tracking | Stores `last-applied-configuration` annotation |
| **Three-way merge** | No | Yes (desired vs current vs last-applied) |
| **Use in GitOps** | Not recommended | Standard approach |
| **Partial updates** | Not supported | Yes (only changed fields updated) |
| **Idempotent** | No (fails on re-run) | Yes (safe to re-run) |
| **Rolling updates** | No | Yes (Deployment updates) |

### 8.2 How kubectl apply Works Internally

```
kubectl apply -f deployment.yaml
         │
         │ 1. Read local file
         ▼
         Compute desired state from YAML
         │
         │ 2. GET current state from API server
         ▼
         Current state (from etcd via API server)
         │
         │ 3. Read last-applied annotation
         ▼
         Previous desired state (from annotation)
         │
         │ 4. Three-way merge
         ▼
         Compute PATCH:
         - fields in desired but not in current → ADD
         - fields in last-applied but removed from desired → DELETE
         - fields changed in desired vs current → UPDATE
         - fields not in desired or last-applied → LEAVE ALONE
         │
         │ 5. PATCH request to API server
         ▼
         Resource updated
```

### 8.3 When to Use Each

```bash
# USE kubectl create:
# - Quick one-off tasks (namespaces, configmaps, secrets)
# - Jobs that should fail loudly if already exist
# - Interactive/exploratory work

kubectl create namespace production
kubectl create configmap app-config --from-file=config.yaml
kubectl create secret generic db-pass --from-literal=password=s3cr3t

# USE kubectl apply:
# - Any resource managed long-term
# - GitOps workflows
# - CI/CD pipelines
# - Anything you want to update without deleting

kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -k overlays/production/   # Kustomize

# NEVER mix: once you apply, always apply (don't switch to create)
# The last-applied annotation gets out of sync if you mix them
```

### 8.4 The last-applied-configuration Annotation

```bash
# See the last-applied configuration stored in the annotation
kubectl get deployment my-app \
  -o jsonpath='{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}'

# This annotation is the "previous desired state" used in three-way merge
# It's stored as JSON and updated on every kubectl apply

# IMPORTANT: If you edit with kubectl edit (not apply), 
# the annotation won't be updated → apply may revert your changes!
```

### 8.5 Server-Side Apply (SSA) — The Modern Approach

```bash
# Server-Side Apply: the server manages field ownership
kubectl apply \
  --server-side \
  --field-manager=my-pipeline \
  -f deployment.yaml

# Advantages of SSA:
# - Handles conflicts between multiple controllers managing same resource
# - More reliable for GitOps (ArgoCD uses SSA internally)
# - Works well with CRDs and custom controllers

# Check field manager ownership
kubectl get deployment my-app \
  -o yaml | grep -A 20 'managedFields'
```

---

## 9. Lab: Versioned Resources with kubectl apply

### 9.1 Understanding Resource Versions

```bash
# Every Kubernetes resource has a resourceVersion (from etcd)
kubectl get pod nginx-pod -o yaml | grep resourceVersion
# resourceVersion: "12345"  ← monotonically increasing

# The resourceVersion is used for:
# 1. Optimistic concurrency (prevent lost updates)
# 2. Watch resumption (GET /pods?watch=true&resourceVersion=12345)
# 3. Change detection

# NEVER store resourceVersion in your manifests
# It changes on every update and will cause apply to fail
```

### 9.2 Versioning Your Application Manifests

```bash
# Step 1: Create initial deployment with version annotation
cat > v1-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: versioned-app
  annotations:
    kubernetes.io/change-cause: "Initial deployment of v1.0.0"
    app.kubernetes.io/version: "1.0.0"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: versioned-app
  template:
    metadata:
      labels:
        app: versioned-app
        version: "1.0.0"
    spec:
      containers:
        - name: app
          image: nginx:1.24
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
EOF

kubectl apply -f v1-deployment.yaml
kubectl rollout history deployment/versioned-app

# Step 2: Update to v2 with change cause
cat > v2-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: versioned-app
  annotations:
    kubernetes.io/change-cause: "Upgraded nginx from 1.24 to 1.25, added memory limit increase"
    app.kubernetes.io/version: "1.1.0"
spec:
  replicas: 3    # Scaled up
  selector:
    matchLabels:
      app: versioned-app
  template:
    metadata:
      labels:
        app: versioned-app
        version: "1.1.0"   # Updated version label
    spec:
      containers:
        - name: app
          image: nginx:1.25  # Updated image
          resources:
            requests:
              cpu: 100m
              memory: 256Mi   # Increased
            limits:
              cpu: 200m
              memory: 512Mi   # Increased
EOF

kubectl apply -f v2-deployment.yaml

# View rollout history
kubectl rollout history deployment/versioned-app
# REVISION  CHANGE-CAUSE
# 1         Initial deployment of v1.0.0
# 2         Upgraded nginx from 1.24 to 1.25, added memory limit increase

# Rollback to specific revision
kubectl rollout undo deployment/versioned-app --to-revision=1
kubectl rollout status deployment/versioned-app
kubectl rollout history deployment/versioned-app
```

### 9.3 apply with pruning (Managing Entire Directory)

```bash
# Create directory structure for an application
mkdir -p k8s/app

cat > k8s/app/deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app.kubernetes.io/managed-by: kubectl
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: nginx:1.25
EOF

cat > k8s/app/service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: my-app
  labels:
    app.kubernetes.io/managed-by: kubectl
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 80
EOF

# Apply entire directory
kubectl apply -f k8s/app/

# Apply with pruning: removes resources that were applied before 
# but are no longer in the directory
kubectl apply -f k8s/app/ \
  --prune \
  -l app.kubernetes.io/managed-by=kubectl
```

---

## 10. Labels and Selectors in Pods — Deep Dive

### 10.1 What Are Labels?

Labels are **key-value pairs attached to Kubernetes objects**. They are the primary mechanism for grouping, selecting, and routing to objects.

```yaml
metadata:
  labels:
    # Recommended standard labels
    app.kubernetes.io/name: "my-api"
    app.kubernetes.io/instance: "my-api-production"
    app.kubernetes.io/version: "2.1.0"
    app.kubernetes.io/component: "backend"
    app.kubernetes.io/part-of: "payment-platform"
    app.kubernetes.io/managed-by: "helm"

    # Custom operational labels
    environment: "production"
    team: "platform"
    tier: "backend"
    cost-center: "engineering"
    criticality: "high"
```

### 10.2 Label Constraints

| Rule | Detail |
|---|---|
| **Key format** | `prefix/name` where prefix is optional DNS subdomain |
| **Key max length** | Prefix: 253 chars, name: 63 chars |
| **Value format** | Must begin/end with alphanumeric; may contain `-_.` |
| **Value max length** | 63 characters |
| **Value can be empty** | Yes (`key: ""` is valid) |
| **System prefix** | `kubernetes.io/` and `k8s.io/` are reserved |
| **Custom prefix** | Use your domain: `company.io/env=prod` |

### 10.3 Label Selectors

Labels are useless without selectors. Selectors allow objects to find other objects.

```yaml
# Equality-based selectors (used by Services, ReplicationControllers)
selector:
  app: my-api
  environment: production
# Matches: objects with BOTH labels matching exactly

# Set-based selectors (used by Deployments, ReplicaSets, DaemonSets, Jobs)
selector:
  matchLabels:
    app: my-api
  matchExpressions:
    - key: environment
      operator: In
      values: [production, staging]
    - key: tier
      operator: NotIn
      values: [frontend]
    - key: maintenance-mode
      operator: DoesNotExist   # Label must NOT exist
    - key: team
      operator: Exists         # Label must exist (any value)
```

### 10.4 How Labels Drive Kubernetes Behavior

| Object | Uses Selector To... |
|---|---|
| **Service** | Select Pods to route traffic to (via Endpoints) |
| **Deployment** | Own its ReplicaSets (matchLabels.app=X) |
| **ReplicaSet** | Own its Pods (matchLabels.app=X) |
| **DaemonSet** | Apply to selected nodes (nodeSelector) |
| **NetworkPolicy** | Select Pods to apply firewall rules to |
| **HPA** | Target specific Deployment/ReplicaSet |
| **PodDisruptionBudget** | Protect selected Pods from eviction |
| **kubectl get -l** | Filter objects for display/operations |

### 10.5 Annotations vs Labels

| Feature | Labels | Annotations |
|---|---|---|
| **Identifying** | Yes (used for selection) | No (metadata only) |
| **Used in selectors** | Yes | No |
| **Size limit** | 63 chars | 256 KB total |
| **Use case** | Selection, grouping, routing | Notes, build info, tool config |
| **Example** | `app: nginx` | `deployment.kubernetes.io/revision: "3"` |

---

## 11. Lab: Setting Labels and Selectors on Pods

### 11.1 Adding and Modifying Labels

```bash
# ── CREATING PODS WITH LABELS ────────────────────────────────────────────

# Via CLI
kubectl run web-pod \
  --image=nginx:1.25 \
  --labels="app=web,env=production,tier=frontend"

# Via YAML
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: api-pod
  labels:
    app: api
    env: production
    team: backend
    version: "2.0"
EOF

# ── MANAGING LABELS ─────────────────────────────────────────────────────

# Add a label to existing pod
kubectl label pod api-pod criticality=high

# Update existing label (requires --overwrite)
kubectl label pod api-pod env=staging --overwrite

# Remove a label (use key- syntax)
kubectl label pod api-pod criticality-

# Label multiple pods at once
kubectl label pods -l app=api version=3.0 --overwrite

# Label all pods in a namespace
kubectl label pods --all tier=backend -n production

# ── VIEWING LABELS ───────────────────────────────────────────────────────

# Show labels in wide output
kubectl get pods --show-labels

# Show specific labels as columns
kubectl get pods \
  -L app,env,tier
# NAME       READY   APP   ENV         TIER
# api-pod    1/1     api   production  backend
# web-pod    1/1     web   production  frontend

# ── SELECTING PODS BY LABELS ─────────────────────────────────────────────

# Equality selector
kubectl get pods -l app=web
kubectl get pods -l app=web,env=production  # AND condition

# Set-based selectors
kubectl get pods -l 'env in (production, staging)'
kubectl get pods -l 'app notin (test, dev)'
kubectl get pods -l '!maintenance'   # Label does not exist
kubectl get pods -l 'team'           # Label exists (any value)

# Complex selectors
kubectl get pods \
  -l 'app in (api, web),env=production,!maintenance'

# ── SERVICE SELECTOR DEMO ────────────────────────────────────────────────

# Create pods with labels
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: backend-v1
  labels:
    app: backend
    version: v1
spec:
  containers:
    - name: app
      image: nginx:1.24
      ports:
        - containerPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: backend-v2
  labels:
    app: backend
    version: v2
spec:
  containers:
    - name: app
      image: nginx:1.25
      ports:
        - containerPort: 80
EOF

# Create a service that selects BOTH pods (all versions)
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: backend-all-versions
spec:
  selector:
    app: backend   # Matches both v1 and v2
  ports:
    - port: 80
      targetPort: 80
---
# Service for only v2
apiVersion: v1
kind: Service
metadata:
  name: backend-v2-only
spec:
  selector:
    app: backend
    version: v2    # Only matches v2
  ports:
    - port: 80
      targetPort: 80
EOF

# Verify endpoint selection
kubectl get endpoints backend-all-versions
kubectl get endpoints backend-v2-only

# Cleanup
kubectl delete pods backend-v1 backend-v2
kubectl delete services backend-all-versions backend-v2-only
```

### 11.2 Labels for Pod Affinity and Anti-Affinity

```yaml
# Use labels to schedule Pods near each other
apiVersion: v1
kind: Pod
metadata:
  name: cache-pod
  labels:
    app: redis-cache
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: api-server   # Schedule near api-server pods
          topologyKey: kubernetes.io/hostname
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: redis-cache  # Spread cache pods across nodes
            topologyKey: kubernetes.io/hostname
  containers:
    - name: redis
      image: redis:7
```

---

## 12. Resource Requests, Limits, and Quality of Service

### 12.1 What Are Resource Requests and Limits?

| Concept | Definition | Who Uses It |
|---|---|---|
| **Request** | Minimum guaranteed resource for a container | kube-scheduler (for bin-packing decisions) |
| **Limit** | Hard maximum a container can use | kubelet (enforced via cgroups) |
| **Allocatable** | Node capacity minus reserved system/kubelet resources | kube-scheduler (actual usable capacity) |

```
NODE RESOURCE MODEL:
─────────────────────────────────────────────────────────────
Total Node Capacity:  4 CPU, 16Gi RAM
  - OS / System:      0.5 CPU, 1Gi   (--system-reserved)
  - kubelet:          0.1 CPU, 200Mi  (--kube-reserved)
  ─────────────────────────────────────────────────────────
Allocatable:          3.4 CPU, 14.8Gi  ← Used by scheduler

Pod 1 requests:  0.5 CPU, 2Gi  → scheduler reserves this
Pod 2 requests:  1 CPU,  4Gi   → scheduler reserves this
Pod 3 requests:  1 CPU,  3Gi   → scheduler reserves this
                 ──────────────
Total requested: 2.5 CPU, 9Gi  → Still fits in 3.4 CPU, 14.8Gi
```

### 12.2 CPU and Memory Units

```yaml
resources:
  requests:
    # CPU units:
    cpu: "1"        # 1 full CPU core
    cpu: "0.5"      # 0.5 CPU cores
    cpu: 500m       # 500 millicores = 0.5 cores (most common notation)
    cpu: 100m       # 0.1 cores (minimum meaningful value)

    # Memory units:
    memory: "256Mi"   # 256 Mebibytes (MiB = 1024^2 bytes) ← USE THIS
    memory: "256M"    # 256 Megabytes (MB = 1000^2 bytes) ← AVOID (ambiguous)
    memory: "1Gi"     # 1 Gibibyte = 1024 MiB
    memory: "2G"      # 2 Gigabytes
    memory: "512Ki"   # 512 Kibibytes

  limits:
    cpu: "2"          # 2 full cores (can burst but capped here)
    memory: "512Mi"   # OOMKilled if exceeded
```

### 12.3 Quality of Service (QoS) Classes

Kubernetes assigns a **QoS class** to each Pod based on its resource configuration. This class determines eviction priority when a node runs out of resources.

```
QOS CLASSES (from most protected to least):
───────────────────────────────────────────────────────────────

1. GUARANTEED (most protected — evicted last)
   ┌─────────────────────────────────────────────────────────┐
   │  EVERY container in the Pod has BOTH:                  │
   │  - requests.cpu == limits.cpu                          │
   │  - requests.memory == limits.memory                    │
   │  AND both values are non-zero                          │
   └─────────────────────────────────────────────────────────┘
   Use for: Critical production workloads, databases

2. BURSTABLE (middle tier)
   ┌─────────────────────────────────────────────────────────┐
   │  At least ONE container has a request OR limit set     │
   │  but NOT all containers have equal requests and limits │
   │  OR requests != limits for at least one container      │
   └─────────────────────────────────────────────────────────┘
   Use for: Most production services with known resource needs

3. BESTEFFORT (least protected — evicted first)
   ┌─────────────────────────────────────────────────────────┐
   │  NO containers have ANY requests or limits set         │
   └─────────────────────────────────────────────────────────┘
   Use for: Non-critical batch work, development
```

### 12.4 QoS Examples

```yaml
# GUARANTEED QoS
apiVersion: v1
kind: Pod
metadata:
  name: guaranteed-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 500m     # requests == limits
          memory: 256Mi
        limits:
          cpu: 500m     # SAME as request
          memory: 256Mi # SAME as request
---
# BURSTABLE QoS
apiVersion: v1
kind: Pod
metadata:
  name: burstable-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 100m      # Can use UP TO 500m when node has spare CPU
          memory: 128Mi
        limits:
          cpu: 500m      # Limit != request → BURSTABLE
          memory: 256Mi
---
# BESTEFFORT QoS (NO resources specified)
apiVersion: v1
kind: Pod
metadata:
  name: besteffort-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      # No resources: section at all → BestEffort
```

### 12.5 Resource Enforcement Mechanics

```
CPU (compressible — can be throttled):
  - If Pod uses more than limit → CPU throttled (not killed)
  - Container gets less CPU time via CFS bandwidth control
  - No Pod termination for CPU limit breach
  - Side effect: latency increases, not failure

Memory (incompressible — must be killed):
  - If Pod uses more than limit → OOMKilled
  - Container terminated immediately, restarts based on restartPolicy
  - OOMKill is logged in pod events and container state
  - reason: OOMKilled, exit code: 137
```

### 12.6 LimitRange — Default Resource Limits

```yaml
# LimitRange sets defaults and min/max for a namespace
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    - type: Container
      default:           # Applied when no limits set
        cpu: 200m
        memory: 256Mi
      defaultRequest:    # Applied when no requests set
        cpu: 100m
        memory: 128Mi
      min:               # Minimum allowed
        cpu: 50m
        memory: 64Mi
      max:               # Maximum allowed
        cpu: "4"
        memory: 4Gi
    - type: Pod
      max:
        cpu: "8"
        memory: 8Gi
```

---

## 13. Lab: Setting Resource Requests and Limits on Pods

### 13.1 Basic Resource Configuration

```bash
# ── CREATING PODS WITH RESOURCES ────────────────────────────────────────

# Via CLI
kubectl run resource-pod \
  --image=nginx:1.25 \
  --requests="cpu=100m,memory=128Mi" \
  --limits="cpu=200m,memory=256Mi"

# Via YAML
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 500m
          memory: 256Mi
EOF

# Check QoS class assigned
kubectl get pod resource-demo \
  -o jsonpath='{.status.qosClass}'
# Burstable (requests != limits)

# Check actual resource allocation
kubectl describe pod resource-demo | grep -A 10 "Requests:\|Limits:"
```

### 13.2 Verifying Scheduler Uses Requests for Placement

```bash
# See node capacity and what's allocated
kubectl describe node k8s-worker-1 | grep -A 20 "Allocated resources:"
# Allocated resources:
#   (Total limits may be over 100 percent, i.e., overcommitted.)
#   Resource           Requests      Limits
#   --------           --------      ------
#   cpu                900m (22%)    2 (50%)
#   memory             1280Mi (8%)   2304Mi (15%)
#   ephemeral-storage  0 (0%)        0 (0%)
#   hugepages-1Gi      0 (0%)        0 (0%)
#   hugepages-2Mi      0 (0%)        0 (0%)

# Create a pod that won't fit
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: too-big-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: "100"  # 100 CPU cores — won't fit anywhere!
          memory: 100Gi
EOF

kubectl get pod too-big-pod
# STATUS: Pending

kubectl describe pod too-big-pod | grep "Events:" -A 5
# 0/3 nodes are available: 3 Insufficient cpu, 3 Insufficient memory.
# preemption: 0/3 nodes are available: 3 No preemption victims found...

# Cleanup
kubectl delete pod too-big-pod
```

### 13.3 Setting Guaranteed QoS for Critical Pods

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: critical-db-pod
  annotations:
    description: "Guaranteed QoS - evicted last under pressure"
spec:
  containers:
    - name: postgres
      image: postgres:14
      env:
        - name: POSTGRES_PASSWORD
          value: "example"  # Use Secret in production!
      resources:
        requests:
          cpu: "1"           # Same as limit
          memory: "2Gi"      # Same as limit
        limits:
          cpu: "1"           # GUARANTEED QoS
          memory: "2Gi"
EOF

kubectl get pod critical-db-pod \
  -o jsonpath='{.status.qosClass}'
# Guaranteed

kubectl delete pod critical-db-pod
```

---

## 14. Lab: Troubleshooting Resource Limit Issues

### 14.1 OOMKilled — Memory Limit Exceeded

```bash
# Reproduce OOMKill
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: oom-demo
spec:
  containers:
    - name: memory-hog
      image: polinux/stress:latest
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "256M", "--vm-hang", "1"]
      resources:
        requests:
          memory: 50Mi
        limits:
          memory: 100Mi   # Limit is 100Mi but we're trying to use 256Mi
EOF

# Watch for OOMKill
kubectl get pod oom-demo -w
# memory-hog  0/1  OOMKilled  ...

# Diagnose
kubectl describe pod oom-demo | grep -A 20 "Last State:"
# Last State:  Terminated
#   Reason:    OOMKilled      ← Memory limit exceeded
#   Exit Code: 137            ← SIGKILL (128 + 9)

# Check container state
kubectl get pod oom-demo \
  -o jsonpath='{.status.containerStatuses[0].lastState}' | jq

# Look at node-level OOM events
kubectl describe node k8s-worker-1 | grep -i "oom"

# Fix: Increase memory limit or reduce application memory usage
kubectl delete pod oom-demo
```

### 14.2 CPU Throttling Diagnosis

```bash
# Deploy CPU-intensive workload
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cpu-demo
spec:
  containers:
    - name: cpu-hog
      image: polinux/stress:latest
      command: ["stress"]
      args: ["--cpu", "4", "--timeout", "60"]
      resources:
        requests:
          cpu: 100m
        limits:
          cpu: 200m   # Very low limit relative to what app needs
EOF

# Check CPU throttling
kubectl exec -it cpu-demo -- cat /sys/fs/cgroup/cpu/cpu.stat
# nr_periods 1000
# nr_throttled 750    ← 75% of periods throttled!
# throttled_time 15000000000

# The container is being throttled because it's hitting the 200m limit
# while trying to use 4 CPU cores

# Monitor with metrics-server (after installation)
kubectl top pod cpu-demo
# NAME       CPU(cores)   MEMORY(bytes)
# cpu-demo   200m         5Mi    ← Pinned at 200m = throttled at limit

kubectl delete pod cpu-demo
```

### 14.3 Pod Stuck in Pending Due to Resources

```bash
# Simulate resource pressure
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pending-resource-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: "64"      # Requesting 64 CPU cores — unrealistic
          memory: 128Gi  # Requesting 128Gi RAM
EOF

# Diagnose pending pod
kubectl get pod pending-resource-pod
# STATUS: Pending

kubectl describe pod pending-resource-pod
# Events:
#   0/3 nodes are available:
#     3 Insufficient cpu.    ← CPU requests too high
#     3 Insufficient memory. ← Memory requests too high
#
# You can also see preemption information

# Check what's available on each node
kubectl get nodes \
  -o custom-columns="NAME:.metadata.name,CPU:.status.allocatable.cpu,MEMORY:.status.allocatable.memory"

# Check what's already allocated
kubectl describe nodes | grep -A 5 "Allocated resources:"

# Fix: Reduce requests
kubectl delete pod pending-resource-pod
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pending-resource-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 500m
          memory: 256Mi
EOF

kubectl get pod pending-resource-pod
# Running

kubectl delete pod pending-resource-pod
```

### 14.4 ResourceQuota — Namespace-Level Resource Control

```bash
# Create a ResourceQuota for a namespace
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: default
spec:
  hard:
    # Compute resources
    requests.cpu: "10"
    requests.memory: 20Gi
    limits.cpu: "20"
    limits.memory: 40Gi
    # Object counts
    pods: "50"
    services: "20"
    persistentvolumeclaims: "10"
    # Storage
    requests.storage: 100Gi
EOF

# Check quota usage
kubectl describe resourcequota production-quota
# Resource              Used    Hard
# --------              ----    ----
# limits.cpu            200m    20
# limits.memory         256Mi   40Gi
# pods                  1       50
# requests.cpu          100m    10
# requests.memory       128Mi   20Gi
```

---

## 15. Built-in Controllers Deep Dive

### 15.1 ReplicaSet Controller

**Purpose:** Ensures exactly N Pod replicas run at all times. Primary self-healing mechanism.

```
Reconciliation:
  desired  = replicaset.spec.replicas
  current  = count(pods matching selector, not terminating)
  if current < desired → create pods
  if current > desired → delete pods (oldest first)
```

```bash
kubectl get replicasets --all-namespaces
kubectl describe replicaset <name>
kubectl get replicaset <name> -o yaml | grep replicas
```

### 15.2 Deployment Controller

**Purpose:** Manages ReplicaSets to implement versioned, rolling updates.

```bash
# Watch rolling update in progress
kubectl rollout status deployment/my-app
kubectl get replicasets -l app=my-app -w
# Old RS scales down as new RS scales up

# Rolling update controls
kubectl rollout pause deployment/my-app   # Pause mid-rollout
kubectl rollout resume deployment/my-app  # Resume
kubectl rollout undo deployment/my-app    # Rollback
```

### 15.3 Node Controller

**Purpose:** Monitors node health; assigns CIDRs; evicts Pods from unhealthy nodes.

```
Node heartbeat missing for 40s (node-monitor-grace-period):
  → Set condition Ready=Unknown
Node missing heartbeat for 5m (pod-eviction-timeout):
  → Apply taint node.kubernetes.io/unreachable:NoExecute
  → Evict Pods with tolerationSeconds=0
```

### 15.4 Service Controller

**Purpose:** Provisions cloud load balancers for `LoadBalancer`-type Services.

```bash
kubectl get services --all-namespaces --field-selector=spec.type=LoadBalancer
kubectl describe service my-lb-service | grep "LoadBalancer Ingress"
```

### 15.5 Namespace Controller

**Purpose:** Manages namespace lifecycle; cascades deletion of all namespaced resources.

```bash
# Watch namespace deletion cascade
kubectl delete namespace test-namespace &
kubectl get all -n test-namespace -w
```

### 15.6 Job Controller

**Purpose:** Creates Pods to complete batch tasks; tracks success/failure.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-processor
spec:
  completions: 10      # Run until 10 completions
  parallelism: 3       # Run 3 at a time
  backoffLimit: 6      # Retry 6 times before marking failed
  activeDeadlineSeconds: 600   # Timeout after 10 minutes
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: processor
          image: batch-processor:v1
```

### 15.7 StatefulSet Controller

**Purpose:** Manages stateful applications with stable network identities and persistent storage.

```bash
# StatefulSet pods: ordered, named, stable DNS
# postgres-0.postgres-headless.default.svc.cluster.local
# postgres-1.postgres-headless.default.svc.cluster.local
kubectl get statefulsets --all-namespaces
kubectl get pods -l app=postgres -o wide
```

### 15.8 DaemonSet Controller

**Purpose:** Ensures one Pod per node (or selected nodes). Used for monitoring agents, log shippers, CNI plugins.

```bash
kubectl get daemonsets --all-namespaces
kubectl describe daemonset kube-proxy -n kube-system
# DESIRED: 3 (one per node)
# CURRENT: 3
# READY: 3
```

### 15.9 Garbage Collector

**Purpose:** Deletes orphaned resources when their owner is deleted (uses `ownerReferences`).

```bash
# See ownerReferences chain
kubectl get pod <name> -o jsonpath='{.metadata.ownerReferences}'
# → ReplicaSet → Deployment → garbage collected when Deployment deleted

# Find orphaned ReplicaSets
kubectl get rs --all-namespaces -o json | \
  jq '.items[] | select(.spec.replicas == 0) | .metadata.name'
```

### 15.10 PersistentVolume Controller

**Purpose:** Binds PVCs to PVs; manages volume lifecycle.

```bash
kubectl get pv,pvc --all-namespaces
kubectl describe pvc my-data -n production
# Check: STATUS (Bound/Pending), VOLUME (bound PV name), STORAGECLASS
```

---

## 16. Internal Working Concepts — Informers, Work Queues & Reconciliation

### 16.1 The Informer Pattern

The informer is the mechanism that allows controllers to receive object change events without polling the API server constantly.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        INFORMER INTERNALS                           │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Reflector                                 │  │
│  │  Initial LIST  →  populates local cache (all objects)       │  │
│  │  Ongoing WATCH →  streams deltas from API server            │  │
│  └───────────────────────────┬──────────────────────────────────┘  │
│                              │ ADDED/MODIFIED/DELETED events        │
│                              ▼                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                  Local Cache (Indexer)                       │  │
│  │  Thread-safe in-memory store of all watched objects         │  │
│  │  Controllers READ from this cache (not API server)          │  │
│  │  Periodic resync: fires synthetic UPDATE events             │  │
│  └───────────────────────────┬──────────────────────────────────┘  │
│                              │                                      │
│                 ┌────────────┼─────────────────┐                   │
│                 ▼            ▼                  ▼                   │
│           OnAdd()        OnUpdate()        OnDelete()              │
│                 │            │                  │                   │
│                 └────────────┴─────────── ───────┘                 │
│                              │                                      │
│                              ▼                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │              Work Queue (rate-limited, deduped)              │  │
│  │  Keys only (namespace/name) — multiple events = one item    │  │
│  │  Exponential backoff on errors                               │  │
│  └───────────────────────────┬──────────────────────────────────┘  │
│                              │                                      │
│                 ┌────────────┴─────────────┐                        │
│                 ▼                          ▼                        │
│          Worker goroutine 1      Worker goroutine 2   (N workers)  │
│                 │                          │                        │
│          reconcile(key)           reconcile(key)                   │
└─────────────────────────────────────────────────────────────────────┘
```

### 16.2 Work Queue Properties

```go
// Simplified pseudocode of work queue behavior

workQueue.Add("default/my-deployment")
// If already in queue: DEDUPED (only one item added)

// Worker processes:
func runWorker() {
    for {
        key, quit := workQueue.Get()
        if err := reconcile(key); err != nil {
            // Retry with exponential backoff
            // Attempt 1: 5ms
            // Attempt 2: 10ms
            // ...
            // Attempt N: up to 1000s
            workQueue.AddRateLimited(key)
        } else {
            workQueue.Forget(key)
        }
        workQueue.Done(key)
    }
}
```

### 16.3 Observing Reconciliation From Outside

```bash
# Watch work queue depth for controllers (from metrics)
curl -sk https://127.0.0.1:10257/metrics | \
  grep 'workqueue_depth' | grep -v "^#"

# Watch controller events
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | \
  grep -i "scaled\|created\|deleted\|reconcil"

# See how fast controllers respond to changes
time kubectl scale deployment my-app --replicas=5
# Pod creation happens within milliseconds after command returns
```

---

## 17. API Server and etcd Interaction

### 17.1 The Golden Rule

```
╔══════════════════════════════════════════════════════════════════╗
║                                                                  ║
║  CONTROLLERS NEVER TALK DIRECTLY TO ETCD.                       ║
║                                                                  ║
║  ALL reads and writes go through kube-apiserver.                ║
║                                                                  ║
║  Controller → kube-apiserver → etcd                             ║
║                                                                  ║
║  The API server provides:                                        ║
║  1. Authentication + Authorization (RBAC)                        ║
║  2. Admission control (validation + mutation)                    ║
║  3. Audit logging                                               ║
║  4. Object versioning (resourceVersion)                          ║
║  5. Watch cache (efficient fan-out to all watchers)              ║
║                                                                  ║
╚══════════════════════════════════════════════════════════════════╝
```

### 17.2 What Controllers Actually Call on the API Server

```bash
# The Deployment controller makes these types of calls:
# LIST   apps/v1/deployments?watch=true     (informer watch)
# GET    apps/v1/deployments/my-app          (read current state)
# PATCH  apps/v1/deployments/my-app/status  (update status)
# CREATE apps/v1/namespaces/.../replicasets  (create RS for new deploy)
# PATCH  apps/v1/replicasets/my-rs          (scale RS up/down)

# See what kubectl actually calls (high verbosity)
kubectl get pods -v=9 2>&1 | grep "GET\|POST\|PATCH"
```

---

## 18. Leader Election — HA Scheduling Infrastructure

### 18.1 Why Leader Election Is Needed

In a 3-node HA control plane, multiple kube-schedulers and kube-controller-managers run simultaneously. Without coordination, multiple scheduler instances would race to assign the same Pod to a node, causing conflicts and duplicate work.

**Leader election ensures only one instance of each control plane component actively reconciles at a time.**

### 18.2 How the Lease Object Works

```bash
# View the current scheduler leader
kubectl get lease kube-scheduler -n kube-system -o yaml
# spec:
#   acquireTime: "2025-01-01T10:00:00.000000Z"
#   holderIdentity: "k8s-cp-1_abc123"   ← Current leader
#   leaseDurationSeconds: 15
#   renewTime: "2025-01-01T12:30:45.000000Z"

# View controller manager leader
kubectl get lease kube-controller-manager -n kube-system -o yaml

# Watch for leader changes in real time
kubectl get lease kube-scheduler -n kube-system -w
```

### 18.3 Leader Election Flags

| Flag | Default | Component | Description |
|---|---|---|---|
| `--leader-elect` | `true` | Both | Enable leader election |
| `--leader-elect-lease-duration` | `15s` | Both | How long lease is valid |
| `--leader-elect-renew-deadline` | `10s` | Both | Must renew before this |
| `--leader-elect-retry-period` | `2s` | Both | Standby retry interval |
| `--leader-elect-resource-lock` | `leases` | Both | Lock resource type |

### 18.4 HA Best Practices

```bash
# Run 3 scheduler + 3 controller-manager replicas (odd number)
kubectl get pods -n kube-system | grep -E "scheduler|controller-manager"

# Monitor leader transitions (high count = instability)
kubectl get lease -n kube-system -o yaml | \
  grep 'leaseTransitions'

# Tuning for larger clusters (slower elections = less instability)
# --leader-elect-lease-duration=30s
# --leader-elect-renew-deadline=25s
# --leader-elect-retry-period=5s
```

---

## 19. Capturing Resource Metrics

### 19.1 The Metrics Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    KUBERNETES METRICS PIPELINE                  │
│                                                                 │
│  ┌─────────────┐                                               │
│  │  containerd │ → container CPU/memory stats                 │
│  └──────┬──────┘                                               │
│         │                                                       │
│  ┌──────▼──────┐                                               │
│  │   kubelet   │ → /metrics/cadvisor endpoint (per node)      │
│  │   :10250    │   Aggregates container stats for all Pods    │
│  └──────┬──────┘                                               │
│         │ Scrapes /metrics/resource                            │
│  ┌──────▼──────┐                                               │
│  │  metrics-   │ → Aggregates CPU/memory from ALL nodes       │
│  │  server     │   Stores recent data only (in-memory)        │
│  │  :443       │   Serves: /apis/metrics.k8s.io/v1beta1/...   │
│  └──────┬──────┘                                               │
│         │                                                       │
│  ┌──────▼──────┐                                               │
│  │ API server  │ → kubectl top nodes/pods                     │
│  │ aggregation │   HPA reads CPU/memory for autoscaling       │
│  └─────────────┘                                               │
│                                                                 │
│  Prometheus (separate stack):                                   │
│  kube-state-metrics → cluster state metrics                    │
│  node-exporter → OS-level metrics                              │
└─────────────────────────────────────────────────────────────────┘
```

### 19.2 Metrics API vs Full Observability

| Tool | What It Provides | Retention | Use Case |
|---|---|---|---|
| `kubectl top` (metrics-server) | Current CPU/memory | ~1-2 minutes | Quick checks, HPA |
| Prometheus + kube-state-metrics | Historical metrics, all resources | Days/weeks | Full observability |
| cAdvisor (kubelet) | Container resource stats | Ephemeral | Debugging |
| node-exporter | OS-level node stats | Via Prometheus | Infrastructure |

---

## 20. Lab: Installing the Metrics Server

### 20.1 Installation

```bash
# ── OPTION 1: Official manifest ─────────────────────────────────────────

# Download the manifest
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify installation
kubectl get pods -n kube-system -l k8s-app=metrics-server
kubectl get apiservice v1beta1.metrics.k8s.io
# Should show: AVAILABLE = True

# Wait for it to be ready
kubectl wait deployment metrics-server \
  -n kube-system \
  --for=condition=Available \
  --timeout=120s

# ── OPTION 2: With custom flags (for self-signed certificates) ──────────

cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    k8s-app: metrics-server
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      serviceAccountName: metrics-server
      containers:
        - name: metrics-server
          image: registry.k8s.io/metrics-server/metrics-server:v0.6.4
          args:
            - --cert-dir=/tmp
            - --secure-port=4443
            - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
            - --kubelet-use-node-status-port
            - --metric-resolution=15s
            # Only for lab/dev with self-signed certs — REMOVE IN PRODUCTION
            - --kubelet-insecure-tls
          resources:
            requests:
              cpu: 100m
              memory: 200Mi
          volumeMounts:
            - name: tmp-dir
              mountPath: /tmp
      volumes:
        - name: tmp-dir
          emptyDir: {}
      priorityClassName: system-cluster-critical
EOF

# ── OPTION 3: Via Helm ────────────────────────────────────────────────
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm repo update

helm install metrics-server metrics-server/metrics-server \
  --namespace kube-system \
  --set args[0]="--kubelet-insecure-tls"  # Dev only

# ── VERIFICATION ─────────────────────────────────────────────────────────
# Wait 30-60 seconds after installation

# Check metrics-server is working
kubectl top nodes
# NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# k8s-cp-1       215m         5%     1234Mi          8%
# k8s-worker-1   87m          2%     890Mi           6%

kubectl top pods --all-namespaces
# NAMESPACE     NAME                        CPU(cores)   MEMORY(bytes)
# kube-system   coredns-xxx                 3m           19Mi
# kube-system   kube-proxy-xxx              1m           18Mi

# Access the metrics API directly
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes | jq '.items[].metadata.name'
kubectl get --raw /apis/metrics.k8s.io/v1beta1/namespaces/default/pods | jq '.items[].metadata.name'
```

---

## 21. Lab: Filtering Resources Based on Compute Usage

### 21.1 Using kubectl top

```bash
# ── NODE METRICS ─────────────────────────────────────────────────────────

# All nodes sorted by CPU usage
kubectl top nodes --sort-by=cpu

# All nodes sorted by memory usage
kubectl top nodes --sort-by=memory

# ── POD METRICS ──────────────────────────────────────────────────────────

# All pods in all namespaces
kubectl top pods --all-namespaces

# Sort by CPU usage (most hungry first)
kubectl top pods --all-namespaces --sort-by=cpu

# Sort by memory usage
kubectl top pods --all-namespaces --sort-by=memory

# Top pods in a specific namespace
kubectl top pods -n production --sort-by=memory

# Top pods with specific label
kubectl top pods -l app=my-api

# ── COMBINING WITH OTHER TOOLS ────────────────────────────────────────────

# Top 5 CPU consumers
kubectl top pods --all-namespaces --sort-by=cpu | head -6

# Top 5 memory consumers
kubectl top pods --all-namespaces --sort-by=memory | head -6

# Find pods using more than 500m CPU (requires parsing output)
kubectl top pods --all-namespaces --no-headers | \
  awk '{if ($3 > 500) print $0}' | sort -k3 -rn

# Find pods close to their memory limit
# (requires cross-referencing with resource limits)
kubectl get pods --all-namespaces -o json | \
  jq '.items[] | 
      select(.spec.containers[].resources.limits.memory != null) |
      {namespace: .metadata.namespace,
       name: .metadata.name,
       memLimit: .spec.containers[0].resources.limits.memory}'
```

### 21.2 Creating a CPU-Intensive Workload for Testing

```bash
# Deploy a workload to generate CPU load
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: cpu-burner
  labels:
    app: cpu-test
spec:
  containers:
    - name: burner
      image: polinux/stress:latest
      command: ["stress"]
      args: ["--cpu", "2", "--timeout", "120"]
      resources:
        requests:
          cpu: 100m
          memory: 64Mi
        limits:
          cpu: 500m
          memory: 128Mi
EOF

# Watch CPU usage increase
watch -n 2 kubectl top pods -l app=cpu-test

# Cleanup after test
kubectl delete pod cpu-burner
```

### 21.3 Custom Metric Queries via the API

```bash
# Get detailed node metrics
kubectl get --raw /apis/metrics.k8s.io/v1beta1/nodes | \
  jq '.items[] | {name: .metadata.name, 
                   cpu: .usage.cpu, 
                   memory: .usage.memory}'

# Get pod metrics with structured output
kubectl get --raw \
  /apis/metrics.k8s.io/v1beta1/namespaces/default/pods | \
  jq '.items[] | 
      {pod: .metadata.name,
       containers: [.containers[] | 
         {name: .name, cpu: .usage.cpu, memory: .usage.memory}]}'
```

---

## 22. DaemonSets — Node-Level Pod Management

### 22.1 What Is a DaemonSet?

A DaemonSet ensures that **exactly one copy of a Pod runs on every node** in the cluster (or a subset of nodes matching a `nodeSelector`). When new nodes join the cluster, the DaemonSet controller automatically creates a Pod on the new node. When nodes are removed, the DaemonSet Pod is garbage collected.

### 22.2 DaemonSet Use Cases

| Use Case | Examples |
|---|---|
| **Node monitoring** | prometheus/node-exporter, datadog-agent |
| **Log collection** | fluentd, filebeat, fluent-bit |
| **Storage** | ceph-osd, glusterfs daemon |
| **Networking** | calico-node, cilium-agent, kube-proxy |
| **Security** | falco, NeuVector, Sysdig |
| **GPU drivers** | nvidia-device-plugin |

### 22.3 DaemonSet YAML

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-log-collector
  namespace: monitoring
  labels:
    app: log-collector
    tier: infrastructure
spec:
  selector:
    matchLabels:
      app: log-collector

  # Update strategy
  updateStrategy:
    type: RollingUpdate    # RollingUpdate or OnDelete
    rollingUpdate:
      maxSurge: 0          # No extra pods during update
      maxUnavailable: 1    # Update one node at a time

  template:
    metadata:
      labels:
        app: log-collector
    spec:
      # Run on ALL nodes including control-plane
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
        - key: node.kubernetes.io/not-ready
          operator: Exists
          effect: NoExecute
        - key: node.kubernetes.io/unreachable
          operator: Exists
          effect: NoExecute

      # Use host PID namespace for log collection
      hostPID: false
      hostNetwork: false

      serviceAccountName: log-collector-sa

      containers:
        - name: log-collector
          image: fluent/fluentd:v1.16
          resources:
            requests:
              cpu: 100m
              memory: 200Mi
            limits:
              cpu: 500m
              memory: 500Mi

          # Mount host filesystem to access node logs
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: config
              mountPath: /fluentd/etc
              readOnly: true

      # DaemonSet-specific: run BEFORE other pods (higher priority)
      priorityClassName: system-node-critical

      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: config
          configMap:
            name: fluentd-config

      terminationGracePeriodSeconds: 30
```

### 22.4 DaemonSet Operations

```bash
# ── MANAGEMENT ───────────────────────────────────────────────────────────

# Create DaemonSet
kubectl apply -f daemonset.yaml

# Check deployment status
kubectl get daemonset node-log-collector -n monitoring
# NAME                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
# node-log-collector   3         3         3       3            3

# See pods on each node
kubectl get pods -n monitoring \
  -l app=log-collector \
  -o wide

# Rolling update a DaemonSet
kubectl set image daemonset/node-log-collector \
  log-collector=fluent/fluentd:v1.17 \
  -n monitoring

kubectl rollout status daemonset/node-log-collector -n monitoring
kubectl rollout history daemonset/node-log-collector -n monitoring

# Rollback
kubectl rollout undo daemonset/node-log-collector -n monitoring

# ── NODE SELECTION ────────────────────────────────────────────────────────

# DaemonSet only on worker nodes (not control-plane)
# Add to spec.template.spec:
nodeSelector:
  node-role: worker

# DaemonSet only on GPU nodes
nodeSelector:
  accelerator: nvidia-tesla-t4

# DaemonSet on nodes with label
nodeSelector:
  disk: ssd
```

### 22.5 DaemonSet vs Deployment

| Feature | DaemonSet | Deployment |
|---|---|---|
| **Scheduling model** | One per node | N replicas, any node |
| **Node joins** | Auto-creates Pod | No automatic action |
| **Scaling** | Scales with cluster | Manual/HPA |
| **Primary use** | Infrastructure agents | Applications |
| **Pod count control** | Nodes control count | `spec.replicas` |
| **Update strategy** | RollingUpdate, OnDelete | RollingUpdate, Recreate |

---

## 23. Static Pods — kubelet-Managed Workloads

### 23.1 What Are Static Pods?

Static Pods are Pods **managed directly by kubelet** on a specific node, without going through the API server for creation. The kubelet watches a local directory for YAML manifests and creates/restarts/deletes Pods based on what's in that directory.

**Key characteristics:**
- Created by kubelet, not by any controller
- API server only has a read-only **mirror Pod** (for visibility only — you cannot delete it via kubectl)
- Cannot be managed by Deployments, ReplicaSets, or DaemonSets
- Used for control plane bootstrapping (etcd, kube-apiserver, kube-controller-manager, kube-scheduler)

### 23.2 Where Static Pod Manifests Live

```bash
# Default static pod manifest directory
ls /etc/kubernetes/manifests/
# etcd.yaml
# kube-apiserver.yaml
# kube-controller-manager.yaml
# kube-scheduler.yaml

# kubelet flag that sets the path
cat /var/lib/kubelet/config.yaml | grep staticPodPath
# staticPodPath: /etc/kubernetes/manifests

# Or in kubelet systemd unit
sudo systemctl cat kubelet | grep 'pod-manifest-path\|staticPodPath'
```

### 23.3 Creating a Static Pod

```bash
# Step 1: SSH to the target node
ssh ubuntu@k8s-worker-1

# Step 2: Create the manifest in the static pod directory
sudo cat > /etc/kubernetes/manifests/static-nginx.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: static-nginx
  labels:
    app: static-nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: 100m
          memory: 64Mi
        limits:
          cpu: 200m
          memory: 128Mi
EOF

# Step 3: kubelet detects the new file automatically (no restart needed)
# Verify from the control plane:
kubectl get pods --all-namespaces | grep static-nginx
# default   static-nginx-k8s-worker-1  1/1  Running  ...
#           ^^^^^^^^^^^^^^^^^^^^^^^^
#           Note: node name appended to pod name (mirror pod naming)

# Delete the static pod (you can't delete from kubectl — need to remove file)
sudo rm /etc/kubernetes/manifests/static-nginx.yaml
# kubelet removes the container

# Verify it's gone
kubectl get pods | grep static-nginx
```

### 23.4 Static Pods vs Regular Pods

| Feature | Static Pod | Regular Pod |
|---|---|---|
| **Created by** | kubelet watching directory | API server (via controller) |
| **Managed by** | kubelet (node-local) | Controllers (cluster-wide) |
| **Deleted by** | Removing manifest file | kubectl delete or controller |
| **API server visibility** | Mirror Pod (read-only) | Full API object |
| **High availability** | Single node; not rescheduled | Rescheduled on failure |
| **Primary use** | Control plane components | All user workloads |
| **Namespace** | Can be any namespace | Any namespace |

---

## 24. Multiple Schedulers — Custom Scheduling Logic

### 24.1 Why Multiple Schedulers?

The default `kube-scheduler` covers 95% of use cases. However, some workloads need custom placement logic:
- ML/AI workloads needing GPU co-location
- High-frequency trading requiring specific NUMA topology
- Custom hardware placement (FPGA, InfiniBand)
- Business rules (tenant isolation, cost allocation)

### 24.2 Deploying a Custom Scheduler

```yaml
# Deploy a second scheduler (using the same binary with different config)
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-scheduler-config
  namespace: kube-system
data:
  config.yaml: |
    apiVersion: kubescheduler.config.k8s.io/v1
    kind: KubeSchedulerConfiguration
    profiles:
      - schedulerName: my-custom-scheduler   # UNIQUE NAME
        plugins:
          score:
            enabled:
              - name: NodeResourcesFit
              - name: NodeAffinity
            disabled:
              - name: TaintToleration  # Disable some default plugins
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-custom-scheduler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      component: my-custom-scheduler
  template:
    metadata:
      labels:
        component: my-custom-scheduler
    spec:
      serviceAccountName: my-scheduler-sa
      containers:
        - name: kube-second-scheduler
          image: registry.k8s.io/kube-scheduler:v1.29.0
          command:
            - /usr/local/bin/kube-scheduler
            - --config=/etc/kubernetes/my-scheduler/config.yaml
          volumeMounts:
            - name: config
              mountPath: /etc/kubernetes/my-scheduler
      volumes:
        - name: config
          configMap:
            name: my-scheduler-config
```

### 24.3 Using the Custom Scheduler for a Pod

```yaml
# Assign a Pod to use the custom scheduler
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-custom-scheduler
spec:
  schedulerName: my-custom-scheduler   # Use custom scheduler
  containers:
    - name: app
      image: nginx:1.25
```

### 24.4 Verifying Which Scheduler Was Used

```bash
# Check which scheduler bound the pod
kubectl describe pod pod-with-custom-scheduler | grep "Events:" -A 10
# Normal  Scheduled  5s   my-custom-scheduler  Successfully assigned
#                         ^^^^^^^^^^^^^^^^^^^^^
#                         Custom scheduler made this decision!

# Pods with default scheduler
kubectl get pods -o json | \
  jq '.items[] | 
      {name: .metadata.name, 
       scheduler: .spec.schedulerName}'
# schedulerName: null (uses default-scheduler)
# schedulerName: my-custom-scheduler (uses custom)

# Check default scheduler name
kubectl get pod <name> -o yaml | grep schedulerName
# spec.schedulerName: default-scheduler  (if not specified)
```

### 24.5 Scheduler Framework Plugins

```
kube-scheduler plugin pipeline:
┌────────────────────────────────────────────────────────────────────┐
│  1. Sort         → Prioritize pods in scheduling queue            │
│  2. PreFilter    → Pre-compute info; reject pods early            │
│  3. Filter       → Remove nodes that can't run the pod            │
│     - NodeUnschedulable                                            │
│     - NodeResourcesFit (check requests vs allocatable)            │
│     - NodeAffinity                                                 │
│     - TaintToleration                                              │
│     - InterPodAffinity                                             │
│  4. PostFilter   → Handle all nodes filtered (e.g., preemption)  │
│  5. PreScore     → Pre-compute info for scoring                   │
│  6. Score        → Rank remaining nodes (0-100)                   │
│     - NodeResourcesFit (least waste)                               │
│     - NodeAffinity (preferred)                                     │
│     - InterPodAffinity (preferred)                                 │
│     - ImageLocality (image already on node)                        │
│  7. NormalizeScore → Normalize scores to [0, 100]                 │
│  8. Reserve      → Reserve resources on selected node             │
│  9. Permit       → Allow/wait/deny binding                        │
│  10. Bind        → Write spec.nodeName to API server              │
└────────────────────────────────────────────────────────────────────┘
```

---

## 25. Port Mapping on Pods

### 25.1 How Container Ports Work in Kubernetes

Container ports in Kubernetes are **informational declarations** (unlike Docker's port binding). Simply declaring a `containerPort` does NOT expose the container to external traffic. It is documentation that helps:
- Kubernetes tooling know what ports containers listen on
- Service objects default to using named ports
- Network policy evaluation

```yaml
spec:
  containers:
    - name: web
      image: nginx:1.25
      ports:
        - name: http          # Logical name for the port
          containerPort: 80   # What the container actually listens on
          protocol: TCP       # TCP (default), UDP, SCTP
        - name: https
          containerPort: 443
          protocol: TCP
        - name: metrics
          containerPort: 9090
          protocol: TCP
```

### 25.2 Port Access Methods

| Method | Access From | Use Case |
|---|---|---|
| **Pod IP + port** | Inside cluster only | Service-to-service (ephemeral IP) |
| **ClusterIP Service** | Inside cluster only | Stable internal DNS name |
| **NodePort Service** | Cluster + external (via node IP) | Dev/staging external access |
| **LoadBalancer Service** | External internet | Production external access |
| **kubectl port-forward** | Local machine | Debugging, development |
| **Ingress** | External internet | HTTP/HTTPS routing, TLS |

### 25.3 hostPort vs containerPort

```yaml
# containerPort: Just a declaration (most common)
containers:
  - name: web
    image: nginx:1.25
    ports:
      - containerPort: 80   # Container listens on 80
                            # Accessible via Pod IP:80

# hostPort: Binds container to a specific port on the NODE
containers:
  - name: web
    image: nginx:1.25
    ports:
      - containerPort: 80
        hostPort: 8080      # Accessible via NodeIP:8080
                            # Only one pod per node can use same hostPort!
                            # WARNING: Limits scheduling flexibility
```

---

## 26. Lab: Port Mapping to Access Containers in a Pod

### 26.1 kubectl port-forward — The Primary Lab Tool

```bash
# ── BASIC PORT FORWARDING ────────────────────────────────────────────────

# Create a test pod
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: port-forward-demo
  labels:
    app: demo
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - name: http
          containerPort: 80
        - name: metrics
          containerPort: 9090
EOF

# Wait for pod to be ready
kubectl wait pod/port-forward-demo --for=condition=Ready --timeout=30s

# Forward local port 8080 to pod port 80
kubectl port-forward pod/port-forward-demo 8080:80 &
PF_PID=$!

# Access the container
curl http://localhost:8080
# Returns nginx welcome page

# Stop port-forward
kill $PF_PID

# ── MULTIPLE PORTS ───────────────────────────────────────────────────────
kubectl port-forward pod/port-forward-demo 8080:80 9090:9090 &

# ── BIND TO ALL INTERFACES (access from other machines) ─────────────────
kubectl port-forward \
  --address 0.0.0.0 \
  pod/port-forward-demo \
  8080:80 &
# WARNING: Accessible from all network interfaces on your machine!

# ── PORT-FORWARD TO SERVICE ──────────────────────────────────────────────
# Create a service
kubectl expose pod port-forward-demo \
  --port=80 \
  --target-port=80 \
  --name=demo-service

# Forward to service (load balances across all matching pods)
kubectl port-forward service/demo-service 8080:80 &

# ── PORT-FORWARD TO DEPLOYMENT ───────────────────────────────────────────
kubectl create deployment web-deploy --image=nginx:1.25 --replicas=3
kubectl expose deployment web-deploy --port=80

kubectl port-forward deployment/web-deploy 8080:80 &
# Connects to ONE of the 3 pods

# ── KUBECTL PROXY (API server proxy) ─────────────────────────────────────
kubectl proxy --port=8001 &

# Access API via proxy (no TLS/auth needed through proxy)
curl http://localhost:8001/api/v1/namespaces/default/pods

# Access pod logs via proxy
curl http://localhost:8001/api/v1/namespaces/default/pods/port-forward-demo/log

# Kill proxy
kill %3  # or find PID with: jobs -l

# Cleanup
kubectl delete pod port-forward-demo
kubectl delete service demo-service
kubectl delete deployment web-deploy
```

### 26.2 NodePort Service for External Access

```bash
# Create a pod with an app
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nodeport-demo
  labels:
    app: nodeport-demo
spec:
  containers:
    - name: web
      image: nginx:1.25
      ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nodeport-demo-svc
spec:
  type: NodePort
  selector:
    app: nodeport-demo
  ports:
    - name: http
      port: 80         # Service port (ClusterIP)
      targetPort: 80   # Container port
      nodePort: 31080  # External port on every node (30000-32767)
      protocol: TCP
EOF

# Access via node IP + NodePort
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
curl http://${NODE_IP}:31080

# Clean up
kubectl delete pod nodeport-demo
kubectl delete service nodeport-demo-svc
```

### 26.3 Comprehensive Port Testing Lab

```bash
# ── COMPLETE LAB: Deploy App with Multiple Ports ─────────────────────────

cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-port-app
  labels:
    app: multi-port
spec:
  containers:
    - name: web
      image: nginx:1.25
      ports:
        - name: http
          containerPort: 80
    - name: metrics
      image: prom/pushgateway:v1.6.0
      ports:
        - name: metrics
          containerPort: 9091
EOF

kubectl wait pod/multi-port-app --for=condition=Ready --timeout=60s

# Forward both ports simultaneously
kubectl port-forward pod/multi-port-app 8080:80 9091:9091 &

# Test web container
curl http://localhost:8080

# Test metrics container (different container, same pod, same network namespace)
curl http://localhost:9091/metrics

# Key point: Both containers share the same IP in the pod
# They communicate via localhost within the pod
kubectl exec multi-port-app -c web -- \
  wget -qO- http://localhost:9091/metrics | head -5
# Containers communicate via localhost:9091 inside the pod!

# Kill port-forwards
kill $(jobs -p)

# Cleanup
kubectl delete pod multi-port-app
```

---

## 27. Performance Tuning for Scheduling

### 27.1 kube-scheduler Performance Flags

```yaml
# KubeSchedulerConfiguration for performance tuning
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
  - schedulerName: default-scheduler
    plugins:
      score:
        disabled:
          - name: SelectorSpread     # Disable if not using replication controllers
    pluginConfig:
      - name: NodeResourcesFit
        args:
          scoringStrategy:
            type: MostAllocated      # vs LeastAllocated (default)
            # MostAllocated: bin-packs pods (efficient, fewer idle nodes)
            # LeastAllocated: spreads pods (redundant, less efficient)

# CLI flags
percentageOfNodesToScore: 50    # Only score 50% of nodes for large clusters
# Default: auto (100% for ≤100 nodes, scales down for larger clusters)
```

### 27.2 kube-controller-manager Performance Flags

| Flag | Default | Recommendation for Large Clusters |
|---|---|---|
| `--concurrent-deployment-syncs` | 5 | 10-20 |
| `--concurrent-replicaset-syncs` | 5 | 10-20 |
| `--concurrent-daemonset-syncs` | 2 | 5-10 |
| `--concurrent-job-syncs` | 5 | 10 |
| `--concurrent-statefulset-syncs` | 5 | 10 |
| `--concurrent-gc-syncs` | 20 | 30-50 |
| `--endpoint-updates-batch-period` | 500ms | 1s (batch more updates) |

### 27.3 Resource Limits for Control Plane Pods

```yaml
# Set resource limits on scheduler and controller-manager
# (In static pod manifests)
resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 2000m
    memory: 2Gi
```

---

## 28. Security Hardening Practices

### 28.1 RBAC for Pod Operations

```yaml
# Developer role: can manage pods in their namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-manager
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/exec", "pods/portforward"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "update", "patch"]
---
# CI/CD role: can only update deployments
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "update", "patch"]
    resourceNames: ["my-app"]  # Only specific deployment
  - apiGroups: ["apps"]
    resources: ["deployments/scale"]
    verbs: ["update"]
```

### 28.2 Pod Security Standards

```bash
# Apply restricted Pod Security Standard to namespace
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# Test what PSS allows
kubectl apply -f privileged-pod.yaml -n production
# Error: violates PodSecurity "restricted:latest"
```

### 28.3 Securing Static Pods and DaemonSets

```yaml
# Static Pod and DaemonSet containers should run as non-root
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534    # nobody user
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: agent
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
              add: ["NET_BIND_SERVICE"]  # Only if needed
```

---

## 29. Monitoring and Observability

### 29.1 Key Prometheus Metrics

| Metric | Description | Alert Condition |
|---|---|---|
| `scheduler_pending_pods` | Pods waiting to be scheduled | > 0 for > 5 minutes |
| `scheduler_scheduling_attempt_duration_seconds` | How long scheduling takes | p99 > 100ms |
| `scheduler_preemption_attempts_total` | Preemption events | High rate = resource pressure |
| `kube_pod_status_phase` | Pod phase distribution | Failed > 0 for long time |
| `kube_pod_container_status_restarts_total` | Container restart count | Rate > 1/hour |
| `container_cpu_cfs_throttled_seconds_total` | CPU throttling | Significant throttling rate |
| `container_memory_usage_bytes` | Memory usage | Near limit |
| `container_oom_events_total` | OOM kills | > 0 |
| `kube_daemonset_status_desired_number_scheduled` | DaemonSet desired | Not equal to current |
| `kube_node_status_condition` | Node conditions | NotReady > 0 |

### 29.2 Monitoring Setup

```bash
# Check what metrics endpoints exist
kubectl get --raw /metrics | head -20                    # API server metrics
curl -sk https://127.0.0.1:10259/metrics | head -20     # Scheduler metrics
curl -sk https://127.0.0.1:10257/metrics | head -20     # Controller manager metrics
curl -sk https://127.0.0.1:10250/metrics | head -20     # Kubelet metrics (per node)

# Key scheduler metric
kubectl get --raw /metrics 2>/dev/null | \
  grep 'scheduler_pending_pods' | grep -v '^#'

# View all DaemonSet issues
kubectl get daemonsets --all-namespaces -o wide
# Check DESIRED vs CURRENT vs READY columns
```

---

## 30. Troubleshooting — Real kubectl Commands

### 30.1 Pod Not Created / Stuck in Pending

```bash
# ── STEP 1: Check pod status ─────────────────────────────────────────────
kubectl get pod <name> -n <namespace>
kubectl get events -n <namespace> --sort-by='.lastTimestamp' | tail -20

# ── STEP 2: Describe the pod ─────────────────────────────────────────────
kubectl describe pod <name> -n <namespace>
# Focus on: Events section, Conditions, Node, Status

# ── STEP 3: Common Pending causes ────────────────────────────────────────

# Cause: Insufficient resources
# "0/3 nodes are available: 3 Insufficient cpu, 3 Insufficient memory"
kubectl top nodes                       # Check node utilization
kubectl describe nodes | grep -A 10 "Allocated resources:"
kubectl get pods -A --sort-by='.spec.resources.requests.cpu'

# Cause: No nodes match nodeSelector
# "0/3 nodes are available: 3 node(s) didn't match node selector"
kubectl get nodes --show-labels | grep <your-label>
kubectl get pod <name> -o yaml | grep nodeSelector

# Cause: Taint/Toleration mismatch
# "0/3 nodes are available: 3 node(s) had taint {key:value}, not tolerated"
kubectl describe nodes | grep Taint
kubectl get pod <name> -o yaml | grep tolerations

# Cause: PVC not bound
# "0/3 nodes are available: 3 pod has unbound immediate PersistentVolumeClaims"
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc-name> -n <namespace>
```

### 30.2 Deployment Stuck

```bash
# Check rollout status
kubectl rollout status deployment/<name> -n <namespace> --timeout=2m

# See current ReplicaSets
kubectl get replicasets -l app=<name> -n <namespace>
kubectl describe replicaset <rs-name> -n <namespace>

# New pods not becoming ready
kubectl get pods -l app=<name> -n <namespace>
kubectl logs <new-pod-name> -n <namespace>
kubectl describe pod <new-pod-name> -n <namespace>

# Common causes:
# - readinessProbe failing (check probe path, port, app startup time)
# - Image pull issue (check registry credentials)
# - OOMKilled (increase memory limits)
# - CrashLoopBackOff (app crashing, check logs)

# Rollback if needed
kubectl rollout undo deployment/<name> -n <namespace>
```

### 30.3 Node NotReady

```bash
# Identify NotReady nodes
kubectl get nodes | grep NotReady
kubectl describe node <node-name> | grep -A 20 "Conditions:"

# Check kubelet on the node (SSH required)
ssh <node-ip>
systemctl status kubelet
journalctl -u kubelet -n 100 --no-pager | grep -i "error\|fail"

# Common causes:
# - Network plugin not ready (CNI issue)
kubectl get pods -n kube-system -o wide | grep <node-name>
# - Disk pressure
df -h
# - Memory pressure
free -m
# - PID pressure
cat /proc/sys/kernel/pid_max

# Cordon (prevent new scheduling)
kubectl cordon <node-name>

# Drain (evict pods safely)
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data

# After fixing, uncordon
kubectl uncordon <node-name>
```

### 30.4 OOMKilled Troubleshooting

```bash
# Find all OOMKilled containers
kubectl get pods --all-namespaces \
  -o json | \
  jq '.items[] | 
      select(.status.containerStatuses[]?.lastState.terminated.reason == "OOMKilled") |
      {namespace: .metadata.namespace, name: .metadata.name}'

# Check current limits vs usage
kubectl top pod <name> -n <namespace>
kubectl get pod <name> -n <namespace> \
  -o jsonpath='{.spec.containers[0].resources}'

# Compare limit vs actual usage
# If usage ≈ limit → increase limit
# If usage < limit → something else is wrong

# Increase memory limit
kubectl patch deployment <name> -n <namespace> \
  --type='json' \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/resources/limits/memory","value":"512Mi"}]'
```

---

## 31. Disaster Recovery Concepts

### 31.1 Stateless Nature of Controller Manager and Scheduler

```
KEY INSIGHT:
kube-controller-manager and kube-scheduler are COMPLETELY STATELESS.
All state lives in etcd, accessed via the API server.

If both crash simultaneously:
  - Running Pods CONTINUE RUNNING (kubelet manages them locally)
  - No new Pods will be scheduled (scheduler down)
  - No self-healing (controller manager down)
  - Services continue working (kube-proxy rules intact)

When they restart:
  - They re-LIST all resources from API server
  - Re-run reconciliation on everything
  - Any missed events are caught by the re-list
  - Cluster converges back to desired state
  - Typically takes < 60 seconds to be fully operational
```

### 31.2 etcd and Pod State

```bash
# All Pod specs, DaemonSet configs, ResourceQuotas, etc. live in etcd
# etcd backup = complete backup of all scheduling state

# Backup etcd (includes all pod specs, labels, resource configs)
ETCDCTL_CMD="etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key"

kubectl exec -n kube-system etcd-k8s-cp-1 -- \
  $ETCDCTL_CMD snapshot save /tmp/etcd-backup.db

# After etcd restore, ALL pod specs are restored
# Controllers will reconcile and pods will be rescheduled

# GitOps as DR: Store all manifests in Git
# If cluster is lost: kubectl apply -f . (restores all configurations)
```

---

## 32. Comparison — kube-apiserver vs kube-scheduler vs kube-controller-manager

| Dimension | kube-apiserver | kube-scheduler | kube-controller-manager |
|---|---|---|---|
| **Primary function** | REST API gateway; only etcd accessor | Assigns Pods to Nodes | Runs all reconciliation controllers |
| **HA model** | Active-Active | Active-Passive (leader) | Active-Passive (leader) |
| **Failure impact** | Catastrophic: cluster can't be managed | New Pods not scheduled; running OK | Self-healing stops; running Pods OK |
| **etcd access** | YES — direct access | NO — via API server | NO — via API server |
| **Port** | 6443 | 10259 | 10257 |
| **State** | Stateless (state in etcd) | Stateless | Stateless |
| **Handles scheduling for** | N/A | All workload types | DaemonSet (bypasses scheduler) |
| **Leader election** | None needed (Active-Active) | Lease in kube-system | Lease in kube-system |
| **Recovery time** | Immediate (behind LB) | ~30s (leader election) | ~30s (leader election) |

---

## 33. ASCII Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║                  KUBERNETES POD SCHEDULING — COMPLETE ARCHITECTURE               ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║                                                                                  ║
║  YOU (kubectl)                                                                   ║
║  ┌────────────────────────────────────────────────────────────────────────────┐ ║
║  │  kubectl apply -f deployment.yaml  │  kubectl top pods  │  kubectl run ... │ ║
║  └───────────────────────────────────────────────────────────────────────────┘ ║
║                                │ HTTPS :6443                                     ║
║                                ▼                                                 ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐ ║
║  │                    CONTROL PLANE                                            │ ║
║  │                                                                             │ ║
║  │  ┌─────────────────────────────────────────────────────────────────────┐   │ ║
║  │  │           kube-apiserver :6443                                      │   │ ║
║  │  │    AuthN → AuthZ → Admission → Audit → READ/WRITE etcd             │   │ ║
║  │  └──────────────┬─────────────────────────┬───────────────────────────┘   │ ║
║  │                 │                          │                               │ ║
║  │  ┌──────────────▼──────────┐  ┌───────────▼───────────────────────────┐   │ ║
║  │  │  etcd :2379             │  │  kube-controller-manager :10257       │   │ ║
║  │  │  /registry/pods/...     │  │  ┌──────────────────────────────────┐ │   │ ║
║  │  │  /registry/deployments  │  │  │ Deployment  │ ReplicaSet         │ │   │ ║
║  │  │  /registry/daemonsets   │  │  │ DaemonSet   │ Job / StatefulSet  │ │   │ ║
║  │  │  /registry/nodes/...    │  │  │ Node        │ Garbage Collector  │ │   │ ║
║  │  │  (ALL state lives here) │  │  │ PV / NS     │ Service            │ │   │ ║
║  │  └─────────────────────────┘  │  └──────────────────────────────────┘ │   │ ║
║  │                                │  Leader Election: Lease object        │   │ ║
║  │                                └───────────────────────────────────────┘   │ ║
║  │                                                                             │ ║
║  │  ┌─────────────────────────────────────────────────────────────────────┐   │ ║
║  │  │  kube-scheduler :10259                                              │   │ ║
║  │  │  1. WATCH for unscheduled Pods (nodeName=="")                       │   │ ║
║  │  │  2. FILTER nodes: resources, taints, affinity, nodeSelector         │   │ ║
║  │  │  3. SCORE remaining nodes: LeastAllocated, ImageLocality, etc.      │   │ ║
║  │  │  4. BIND: write spec.nodeName to chosen node                        │   │ ║
║  │  │  Custom schedulers: different schedulerName, same API               │   │ ║
║  │  │  Static Pods: bypass scheduler entirely (kubelet manages)           │   │ ║
║  │  └─────────────────────────────────────────────────────────────────────┘   │ ║
║  │                                                                             │ ║
║  │  ┌─────────────────────────────────────────────────────────────────────┐   │ ║
║  │  │  metrics-server :443                                                │   │ ║
║  │  │  Aggregates kubelet cadvisor → kubectl top nodes/pods → HPA        │   │ ║
║  │  └─────────────────────────────────────────────────────────────────────┘   │ ║
║  └─────────────────────────────────────────────────────────────────────────────┘ ║
║                     │ kubelet watches API for Pods bound to this node            ║
║                     ▼                                                            ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐ ║
║  │                    WORKER NODES                                             │ ║
║  │                                                                             │ ║
║  │  ┌──────────────────────────────────────────────────────────────────────┐  │ ║
║  │  │  kubelet :10250                                                      │  │ ║
║  │  │  - Manages regular Pods (API server assigned)                        │  │ ║
║  │  │  - Manages Static Pods (/etc/kubernetes/manifests/*.yaml)            │  │ ║
║  │  │  - Reports resource metrics to metrics-server                        │  │ ║
║  │  │  - Enforces resource limits via cgroups                              │  │ ║
║  │  └──────────────┬───────────────────────────────────────────────────────┘  │ ║
║  │                 │ CRI (gRPC)                                                │ ║
║  │  ┌──────────────▼───────────────────────────────────────────────────────┐  │ ║
║  │  │  containerd / cri-o                                                  │  │ ║
║  │  │  Creates: pause container (network NS) + app containers              │  │ ║
║  │  └──────────────────────────────────────────────────────────────────────┘  │ ║
║  │                                                                             │ ║
║  │  ┌──────────────────────────────────────────────────────────────────────┐  │ ║
║  │  │  POD (unit of scheduling)                                            │  │ ║
║  │  │  ┌────────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │  │ ║
║  │  │  │ init container │→ │ app container│  │ sidecar container        │  │  │ ║
║  │  │  │ (runs first)   │  │ port: 8080   │  │ shares: network + volume │  │  │ ║
║  │  │  └────────────────┘  └──────────────┘  └──────────────────────────┘  │  │ ║
║  │  │  Shared: IP address, ports, network NS, volumes                      │  │  │ ║
║  │  │  QoS: Guaranteed/Burstable/BestEffort based on requests/limits       │  │  │ ║
║  │  └──────────────────────────────────────────────────────────────────────┘  │ ║
║  │                                                                             │ ║
║  │  DaemonSet Pods run on EVERY node (CNI, log agents, monitoring)            │ ║
║  └─────────────────────────────────────────────────────────────────────────────┘ ║
╚══════════════════════════════════════════════════════════════════════════════════╝
```

---

## 34. Real-World Production Use Cases

### 34.1 Resource Management in a Multi-Tenant Platform

```yaml
# Pattern: LimitRange + ResourceQuota per team namespace
# Prevents "noisy neighbor" problem

apiVersion: v1
kind: Namespace
metadata:
  name: team-analytics
---
# Hard limit on namespace resources
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-analytics-quota
  namespace: team-analytics
spec:
  hard:
    requests.cpu: "8"
    requests.memory: 16Gi
    limits.cpu: "16"
    limits.memory: 32Gi
    pods: "50"
---
# Defaults for containers that don't specify resources
apiVersion: v1
kind: LimitRange
metadata:
  name: team-analytics-defaults
  namespace: team-analytics
spec:
  limits:
    - type: Container
      default:
        cpu: 200m
        memory: 256Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
```

### 34.2 DaemonSet for Distributed Security Scanning

```yaml
# Deploy Falco on every production node for runtime security
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
  namespace: security
spec:
  selector:
    matchLabels:
      app: falco
  template:
    metadata:
      labels:
        app: falco
    spec:
      tolerations:
        - effect: NoSchedule
          operator: Exists  # Tolerate all taints
      hostNetwork: true
      hostPID: true
      serviceAccountName: falco-sa
      containers:
        - name: falco
          image: falcosecurity/falco:0.36
          securityContext:
            privileged: true  # Needs kernel access
          volumeMounts:
            - name: dev
              mountPath: /host/dev
            - name: proc
              mountPath: /host/proc
            - name: boot
              mountPath: /host/boot
            - name: lib-modules
              mountPath: /host/lib/modules
            - name: kernel-src
              mountPath: /host/usr/src
      volumes:
        - hostPath:
            path: /dev
          name: dev
        - hostPath:
            path: /proc
          name: proc
        - hostPath:
            path: /boot
          name: boot
        - hostPath:
            path: /lib/modules
          name: lib-modules
        - hostPath:
            path: /usr/src
          name: kernel-src
```

---

## 35. Best Practices for Production Environments

### 35.1 Pod Scheduling

- **Always set resource requests** — without them, pods are BestEffort and evicted first; scheduler can't make good decisions
- **Always set resource limits** — unbounded memory can OOMKill other processes; unbounded CPU creates noisy neighbors  
- **Match requests to actual usage** — over-requesting wastes capacity; under-requesting causes evictions; measure with `kubectl top`
- **Use Guaranteed QoS for critical workloads** — set requests == limits for databases, stateful services
- **Avoid hostPort** — it limits scheduling flexibility and creates port conflicts; use Services instead
- **Use anti-affinity for HA** — spread replicas across nodes/zones to avoid single points of failure

### 35.2 Labels and RBAC

- **Label everything** — namespace, team, version, environment, tier; makes operations much easier
- **Use standard labels** (`app.kubernetes.io/*`) — tooling like Helm and ArgoCD recognize them
- **Annotate with operational metadata** — who deployed it, what commit, what ticket

### 35.3 DaemonSets

- **Always set tolerations broadly** — DaemonSets should run on all nodes including tainted ones (unless intentionally excluded)
- **Set priorityClassName: system-node-critical** — ensure infrastructure DaemonSets aren't evicted
- **Use RollingUpdate with maxUnavailable: 1** — update one node at a time for safety

### 35.4 Resource Governance

- **Use LimitRange for defaults** — prevent unmanaged namespaces from running unbounded containers
- **Use ResourceQuota for team boundaries** — prevent one team from consuming all cluster resources
- **Use VPA (Vertical Pod Autoscaler)** — automatically right-size resource requests based on actual usage

---

## 36. Common Mistakes and Pitfalls

### 36.1 No Resource Requests

**Mistake:** Deploying pods without resource requests.
**Impact:** Scheduler makes random node assignment; pods become BestEffort QoS and are evicted first under pressure.
**Fix:** Always set `resources.requests.cpu` and `resources.requests.memory`.

---

### 36.2 Memory Limit Too Low

**Mistake:** Setting memory limit close to the application's normal usage without headroom.
**Impact:** Intermittent OOMKills under load, causing pod restarts and service disruption.
**Fix:** Set limit at 2x the normal usage; monitor with `kubectl top`; increase if OOMKills occur.

---

### 36.3 Confusing containerPort with Service Exposure

**Mistake:** Declaring `containerPort: 80` and expecting it to be accessible externally.
**Impact:** Container is accessible within the cluster via Pod IP, but not from outside.
**Fix:** Create a Service (ClusterIP for internal, NodePort or LoadBalancer for external).

---

### 36.4 Using kubectl create in CI/CD Pipelines

**Mistake:** Using `kubectl create -f` in CI/CD.
**Impact:** Pipeline fails on re-run with "already exists" error.
**Fix:** Use `kubectl apply -f` for idempotent deployments.

---

### 36.5 Static Pod Scheduling Limitations

**Mistake:** Using Static Pods for general application workloads.
**Impact:** No auto-rescheduling if node fails; no rolling updates via Deployment.
**Fix:** Static Pods are for control plane components only; use Deployments for applications.

---

### 36.6 Missing Tolerations in DaemonSet

**Mistake:** DaemonSet missing tolerations for control-plane taints.
**Impact:** DaemonSet doesn't run on control-plane nodes — gaps in monitoring/logging coverage.
**Fix:** Add `tolerations` for `node-role.kubernetes.io/control-plane:NoSchedule`.

---

## 37. Interview Questions — Beginner to Advanced

### Beginner Level

**Q1: What is the difference between a Container and a Pod in Kubernetes?**

**A:** A Container is a Linux process running in an isolated environment (using namespaces and cgroups). In Kubernetes, you don't schedule containers directly — you schedule Pods. A Pod is a group of one or more containers that share:
- The same network namespace (same IP address and port space)
- The same storage volumes
- The same lifecycle (they're created and destroyed together)

The key insight is that containers in the same Pod communicate via `localhost`, while containers in different Pods communicate via Pod IPs. The Pod is the smallest deployable unit in Kubernetes — you can't schedule a container directly.

---

**Q2: What is the difference between `kubectl create` and `kubectl apply`?**

**A:** `kubectl create` is **imperative** — it creates a resource once. If the resource already exists, it throws an error (`AlreadyExists`). It's idempotent only if you're creating from scratch.

`kubectl apply` is **declarative** — it creates if not exists, updates if exists. It stores a `last-applied-configuration` annotation and performs a **three-way merge**: desired state vs current state vs last-applied state. This allows partial updates and is safe to run multiple times.

For CI/CD and production environments, always use `kubectl apply`.

---

**Q3: What are QoS classes in Kubernetes and why do they matter?**

**A:** Kubernetes assigns one of three Quality of Service classes to each Pod:
- **Guaranteed**: All containers have equal requests and limits (e.g., CPU request=200m, limit=200m). Evicted last.
- **Burstable**: At least one container has a request or limit, but requests ≠ limits. Evicted when node is under pressure.
- **BestEffort**: No containers have any resource requests or limits. Evicted first.

QoS classes matter because when a node runs out of memory, Kubernetes must evict Pods. BestEffort Pods are evicted first, then Burstable (ordered by usage vs request ratio), then Guaranteed last. For critical production workloads (databases, core services), set Guaranteed QoS.

---

### Intermediate Level

**Q4: Explain how a DaemonSet is different from a Deployment and when you'd use each.**

**A:** A Deployment manages N replicas distributed across available nodes. You control the count via `spec.replicas`, and the scheduler decides placement.

A DaemonSet ensures ONE Pod per node (or per selected nodes). The count is determined by the number of nodes — you don't set `replicas`. When nodes join the cluster, the DaemonSet controller automatically creates a Pod on the new node.

Use DaemonSet for: infrastructure agents that need to run everywhere (log collectors like Fluentd, monitoring agents like node-exporter, CNI plugins like calico-node, security agents like Falco).

Use Deployment for: application workloads where you want N copies but don't care which nodes they run on, and you want to scale up/down independently of node count.

---

**Q5: What happens when you set `spec.nodeName` in a Pod manifest? What's the downside?**

**A:** Setting `spec.nodeName` **bypasses the kube-scheduler entirely**. The kubelet on the specified node directly picks up the Pod and starts it, without any scheduling evaluation.

The downsides are:
1. **No resource validation**: If the node doesn't have enough CPU/memory for your requests, the Pod will fail to start or degrade the node — the scheduler's capacity checks don't run
2. **No taint/toleration evaluation**: The Pod is placed on the node regardless of taints
3. **No affinity rules**: Node/pod affinity and anti-affinity are ignored
4. **Inflexible**: If the specified node is down, the Pod stays Pending indefinitely — it won't be rescheduled

Manual scheduling is used for: control plane bootstrapping, debugging specific node issues, or temporary testing.

---

**Q6: What is a Static Pod and how is it different from a regular Pod?**

**A:** A Static Pod is managed directly by the kubelet on a specific node — it reads YAML manifests from a local directory (typically `/etc/kubernetes/manifests/`) and creates Pods based on those files. The API server doesn't control Static Pods; it only has a read-only "mirror Pod" for visibility.

Differences from regular Pods:
- Created/managed by kubelet, not the API server/controllers
- Cannot be deleted via `kubectl delete pod` — you must remove the manifest file
- Not rescheduled if the node fails — they're node-local
- Used for control plane components (etcd, kube-apiserver, kube-controller-manager, kube-scheduler) in kubeadm clusters

---

### Advanced Level

**Q7: Explain the complete scheduling pipeline when you submit a Deployment manifest.**

**A:** The complete flow:
1. `kubectl apply` sends PATCH/POST to kube-apiserver (6443)
2. API server authenticates, authorizes (RBAC), runs admission controllers
3. API server writes Deployment to etcd
4. kube-controller-manager's Deployment controller informer detects new Deployment
5. Deployment controller creates a ReplicaSet with same pod template
6. ReplicaSet controller creates 3 Pod objects with `spec.nodeName = ""`
7. kube-scheduler informer detects 3 unscheduled Pods
8. For each Pod, scheduler: Filters nodes (resources, taints, affinity) → Scores nodes → Selects winner → Calls Bind API (writes `spec.nodeName`)
9. kubelet on each selected node detects its new Pod via watch
10. kubelet pulls image via containerd CRI
11. containerd creates: pause container (network NS) → init containers → app containers
12. kubelet runs readinessProbe; when passing, adds Pod to Endpoints via EndpointSlice controller
13. kube-proxy on all nodes updates iptables/IPVS rules for the new endpoints

---

**Q8: How does the metrics-server work, and why does `kubectl top` not give historical data?**

**A:** The metrics-server:
1. Deploys as a Deployment in kube-system
2. Periodically scrapes each kubelet's `/metrics/resource` endpoint (every 15s by default)
3. Aggregates CPU/memory across all nodes and Pods
4. Stores ONLY the most recent data point (~1-2 minutes) in memory — no historical storage
5. Serves via the `metrics.k8s.io/v1beta1` API (API server aggregation layer)
6. `kubectl top` reads from this API; HPA also reads from this API for autoscaling decisions

Because metrics-server stores only the current snapshot (not time-series), it can't provide historical data. For historical metrics, you need Prometheus (with kube-state-metrics and node-exporter), which stores time-series data in TSDB.

The metrics-server is a lightweight component for operational use (`kubectl top`, HPA). Prometheus is for full observability with dashboards, alerting, and history.

---

## 38. Cheat Sheet — Commands, Flags & Manifests

### 38.1 Pod Operations

```bash
# Create
kubectl run <name> --image=<image>
kubectl run <name> --image=<image> --dry-run=client -o yaml > pod.yaml
kubectl apply -f pod.yaml
kubectl create -f pod.yaml  # fails if exists

# Inspect
kubectl get pods [-n namespace] [-o wide] [-o yaml] [--show-labels]
kubectl describe pod <name>
kubectl get pod <name> -o jsonpath='{.status.phase}'
kubectl get pod <name> -o jsonpath='{.status.qosClass}'

# Debug
kubectl logs <name> [-c container] [--tail=N] [-f] [--previous]
kubectl exec -it <name> -- bash
kubectl exec <name> -- <command>
kubectl cp <namespace>/<pod>:/path/to/file /local/path

# Lifecycle
kubectl delete pod <name> [--grace-period=0 --force]
kubectl wait pod/<name> --for=condition=Ready --timeout=60s

# Port access
kubectl port-forward pod/<name> 8080:80
kubectl port-forward service/<name> 8080:80
kubectl port-forward --address 0.0.0.0 pod/<name> 8080:80
```

### 38.2 Labels and Selectors

```bash
kubectl label pod <name> key=value
kubectl label pod <name> key=value --overwrite
kubectl label pod <name> key-          # Remove label
kubectl get pods -l 'key=value'
kubectl get pods -l 'key in (v1,v2)'
kubectl get pods --show-labels
kubectl get pods -L key1,key2         # Show labels as columns
```

### 38.3 Resource Management

```bash
kubectl top nodes [--sort-by=cpu|memory]
kubectl top pods [-n ns] [--sort-by=cpu|memory] [-l selector]
kubectl get resourcequota --all-namespaces
kubectl get limitrange --all-namespaces
kubectl describe nodes | grep -A 5 "Allocated resources:"
```

### 38.4 Rollouts

```bash
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name> [--to-revision=N]
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>
kubectl rollout restart deployment/<name>
```

### 38.5 DaemonSet Operations

```bash
kubectl get daemonsets [-n namespace]
kubectl describe daemonset <name>
kubectl rollout status daemonset/<name>
kubectl rollout history daemonset/<name>
kubectl rollout undo daemonset/<name>
kubectl set image daemonset/<name> container=image:tag
```

### 38.6 Static Pod Management

```bash
# On node — manage static pod manifests
ls /etc/kubernetes/manifests/
cat /var/lib/kubelet/config.yaml | grep staticPodPath

# Create static pod
sudo cp pod.yaml /etc/kubernetes/manifests/

# Delete static pod
sudo rm /etc/kubernetes/manifests/pod.yaml

# Check mirror pods (from control plane)
kubectl get pods -n kube-system | grep -v controller | grep -v scheduler
```

### 38.7 Port Mapping

```bash
# Declare ports in pod spec (informational)
kubectl run web --image=nginx --port=80

# Port-forward
kubectl port-forward pod/<name> localPort:containerPort
kubectl port-forward svc/<name> localPort:servicePort
kubectl port-forward deploy/<name> localPort:containerPort

# Check port accessibility
kubectl get svc <name>   # See ClusterIP, NodePort, LoadBalancer IP
kubectl get endpoints <svc-name>   # See pod IPs backing the service
```

### 38.8 Scheduling Controls

```bash
# Manual scheduling
kubectl patch pod <name> -p '{"spec":{"nodeName":"<node>"}}'

# Node management
kubectl cordon <node>      # Prevent scheduling
kubectl uncordon <node>    # Allow scheduling
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data

# See scheduling events
kubectl describe pod <name> | grep -A 10 "Events:"
```

---

## 39. Key Takeaways & Summary

### The Pod Scheduling Mental Model

```
1. CONTAINERS are processes.
   PODS are groups of containers that share a network namespace and volumes.
   
2. The SCHEDULER assigns Pods to Nodes based on:
   - Available resources (CPU, memory, storage)
   - Labels and node selectors
   - Taints and tolerations
   - Affinity and anti-affinity rules

3. RESOURCE REQUESTS drive the scheduler.
   RESOURCE LIMITS are enforced by cgroups on the node.
   
4. QoS CLASSES determine eviction priority:
   BestEffort (no resources) → evicted first
   Burstable (partial resources) → evicted next
   Guaranteed (equal requests/limits) → evicted last

5. LABELS are the universal selection mechanism.
   Services, Deployments, DaemonSets all use label selectors.
   
6. DAEMONSETS ensure one Pod per node.
   Use for infrastructure agents, not applications.
   
7. STATIC PODS are managed by kubelet directly.
   Used for control plane components only in production.
   
8. MULTIPLE SCHEDULERS allow custom placement logic.
   Set spec.schedulerName to use a custom scheduler.
   
9. PORT MAPPING in Kubernetes ≠ Docker port mapping.
   containerPort is informational.
   Use Services for actual traffic routing.
   Use kubectl port-forward for local access.
   
10. METRICS SERVER enables kubectl top and HPA.
    For production observability, add Prometheus.
```

### The 5 Rules Every Kubernetes Engineer Must Know

1. **Always set resource requests and limits** — they drive scheduling AND protect nodes
2. **Use kubectl apply for everything in CI/CD** — it's idempotent
3. **Label everything** — `app`, `version`, `env`, `team` at minimum
4. **Controllers never talk to etcd directly** — everything goes through the API server
5. **Pods are ephemeral** — design applications to handle Pod death gracefully

---

> **This guide covers Kubernetes v1.29+ scheduling concepts. Always consult the official Kubernetes documentation at https://kubernetes.io/docs/concepts/scheduling-eviction/ for the most current information.**

---

*End of: Kubernetes Pod Scheduling — Part 1: Complete Production-Grade Guide*
