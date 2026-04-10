## Table of Contents

1. [Introduction — What Is "Exploring a Cluster" and Why It Matters](#1-introduction--what-is-exploring-a-cluster-and-why-it-matters)
2. [Core Identity Table — Every Component You Will Encounter](#2-core-identity-table--every-component-you-will-encounter)
3. [The Mental Model: Cluster Topology from the Inside](#3-the-mental-model-cluster-topology-from-the-inside)
4. [The Controller Pattern — Watch → Compare → Act → Loop](#4-the-controller-pattern--watch--compare--act--loop)
5. [Exploring the Control Plane](#5-exploring-the-control-plane)
6. [Exploring Nodes — Data Plane Inspection](#6-exploring-nodes--data-plane-inspection)
7. [Exploring Namespaces and Workloads](#7-exploring-namespaces-and-workloads)
8. [Deep Dive: Built-in Controllers and How to Observe Them](#8-deep-dive-built-in-controllers-and-how-to-observe-them)
9. [Internal Working Concepts: Informers, Work Queues & Reconciliation](#9-internal-working-concepts-informers-work-queues--reconciliation)
10. [Interaction with API Server and etcd During Exploration](#10-interaction-with-api-server-and-etcd-during-exploration)
11. [Leader Election — Observing and Understanding](#11-leader-election--observing-and-understanding)
12. [Exploring Networking — Services, DNS, and CNI](#12-exploring-networking--services-dns-and-cni)
13. [Exploring Storage — PersistentVolumes and StorageClasses](#13-exploring-storage--persistentvolumes-and-storageclasses)
14. [Exploring RBAC, Security, and Service Accounts](#14-exploring-rbac-security-and-service-accounts)
15. [Exploring Cluster Events and Audit Logs](#15-exploring-cluster-events-and-audit-logs)
16. [Performance Tuning — What to Look For During Exploration](#16-performance-tuning--what-to-look-for-during-exploration)
17. [Monitoring & Observability — Metrics, Dashboards, and Alerting](#17-monitoring--observability--metrics-dashboards-and-alerting)
18. [Troubleshooting with kubectl — Real Commands for Real Problems](#18-troubleshooting-with-kubectl--real-commands-for-real-problems)
19. [Disaster Recovery Concepts Through the Exploration Lens](#19-disaster-recovery-concepts-through-the-exploration-lens)
20. [Comparing Cluster Components: kube-apiserver vs kube-scheduler vs kube-controller-manager](#20-comparing-cluster-components-kube-apiserver-vs-kube-scheduler-vs-kube-controller-manager)
21. [ASCII Architecture Diagram — The Full Cluster Map](#21-ascii-architecture-diagram--the-full-cluster-map)
22. [Real-World Production Use Cases for Cluster Exploration](#22-real-world-production-use-cases-for-cluster-exploration)
23. [Security Hardening — Practices Revealed Through Exploration](#23-security-hardening--practices-revealed-through-exploration)
24. [Best Practices for Production Cluster Exploration](#24-best-practices-for-production-cluster-exploration)
25. [Common Mistakes and Pitfalls](#25-common-mistakes-and-pitfalls)
26. [Hands-On Labs — Guided Exploration Exercises](#26-hands-on-labs--guided-exploration-exercises)
27. [Interview Questions — Beginner to Advanced](#27-interview-questions--beginner-to-advanced)
28. [Cheat Sheet — Essential Exploration Commands and Flags](#28-cheat-sheet--essential-exploration-commands-and-flags)
29. [Key Takeaways & Summary](#29-key-takeaways--summary)

---

## 1. Introduction — What Is "Exploring a Cluster" and Why It Matters

**Exploring a Kubernetes cluster** is the disciplined practice of systematically inspecting every layer of a running cluster — its control plane, worker nodes, workloads, networking, storage, security, and observability stack — to build an accurate mental model of what is running, how it is configured, and whether it is healthy.

This is not merely running `kubectl get pods`. It is a methodical skill that distinguishes a Kubernetes operator from a Kubernetes expert. Cluster exploration is performed in multiple contexts:

- **Day-1 onboarding:** A new engineer joining a team needs to understand an existing cluster's topology, conventions, and operational state.
- **Incident response:** An on-call SRE needs to quickly map the blast radius of a failing component across interconnected workloads.
- **Security audit:** A security engineer needs to enumerate all service accounts, RBAC bindings, exposed secrets, and network policies to identify risk.
- **Capacity planning:** A platform engineer needs to understand resource utilization, node headroom, and pod density before scaling decisions.
- **Cluster migration:** A team migrating from one cluster to another must first fully understand what is running in the source cluster.
- **Compliance review:** Regulators or internal auditors require proof of what is deployed, how it is secured, and how data flows.

### Why This Skill Is Non-Negotiable for Senior Engineers

Kubernetes clusters are **living systems**. Unlike a static application whose code you can read, a cluster's state is constantly changing — controllers are reconciling, Pods are scheduling and terminating, certificates are rotating, and cloud resources are being provisioned. The only way to understand a cluster is to observe it directly and know which tools and APIs to use.

The exploration skill also reveals a deep truth about Kubernetes architecture: **everything is an API object**. Nodes, Pods, Services, NetworkPolicies, RBAC bindings, certificates, and even leadership elections are represented as resources in the API server's object store. If you know how to query the API server, you can understand everything about a cluster.

---

## 2. Core Identity Table — Every Component You Will Encounter

### 2.1 Control Plane Components

| Component | Binary | Port | Protocol | Role | Config Location |
|---|---|---|---|---|---|
| **API Server** | `kube-apiserver` | 6443 | HTTPS | Single gateway to all cluster state | `/etc/kubernetes/manifests/kube-apiserver.yaml` |
| **etcd** | `etcd` | 2379 (client), 2380 (peer) | HTTPS/gRPC | Distributed KV store; source of truth | `/etc/kubernetes/manifests/etcd.yaml` |
| **Controller Manager** | `kube-controller-manager` | 10257 | HTTPS | Runs all built-in reconciliation loops | `/etc/kubernetes/manifests/kube-controller-manager.yaml` |
| **Scheduler** | `kube-scheduler` | 10259 | HTTPS | Assigns Pods to Nodes | `/etc/kubernetes/manifests/kube-scheduler.yaml` |
| **CoreDNS** | `coredns` | 53 (DNS), 9153 (metrics) | UDP/TCP | Cluster-internal DNS resolution | ConfigMap: `coredns` in `kube-system` |
| **Cloud Controller** | `cloud-controller-manager` | 10258 | HTTPS | Cloud provider resource management | Varies by provider |

### 2.2 Node Components

| Component | Binary | Port | Protocol | Role | Config Location |
|---|---|---|---|---|---|
| **kubelet** | `kubelet` | 10250 | HTTPS | Node agent; Pod lifecycle manager | `/var/lib/kubelet/config.yaml` |
| **kube-proxy** | `kube-proxy` | 10249 (metrics) | HTTP | Programs iptables/IPVS for Service routing | ConfigMap: `kube-proxy` in `kube-system` |
| **Container Runtime** | `containerd` / `cri-o` | CRI socket | gRPC | Pulls images, creates containers | `/etc/containerd/config.toml` |
| **CNI Plugin** | `calico-node`, `cilium`, etc. | Varies | N/A | Pod-to-Pod networking, NetworkPolicy | `/etc/cni/net.d/` |

### 2.3 Exploration Tools

| Tool | Purpose | Install Location |
|---|---|---|
| `kubectl` | Primary cluster interaction CLI | Anywhere with kubeconfig |
| `crictl` | CRI-level container inspection | On each node |
| `etcdctl` | Direct etcd operations | On control plane nodes |
| `kubeadm` | Cluster info, cert inspection | Control plane |
| `helm` | Chart inspection and management | Anywhere |
| `stern` | Multi-pod log tailing | Anywhere |
| `k9s` | Terminal UI cluster explorer | Anywhere |
| `kubectx/kubens` | Context and namespace switching | Anywhere |
| `kube-capacity` | Node resource usage overview | Anywhere |

---

## 3. The Mental Model: Cluster Topology from the Inside

### 3.1 The Layers of a Cluster

When exploring a cluster, think in layers from infrastructure upward:

```
Layer 7 — Applications (your workloads: Deployments, StatefulSets, Jobs)
Layer 6 — Platform Services (Ingress controllers, cert-manager, Prometheus)
Layer 5 — Kubernetes Abstractions (Services, Ingress, NetworkPolicy, RBAC)
Layer 4 — Kubernetes Core (Pods, ReplicaSets, PVCs, ConfigMaps, Secrets)
Layer 3 — Kubernetes Control Plane (API server, controllers, scheduler, etcd)
Layer 2 — Node Operating System (Linux, kernel modules, sysctl, cgroups)
Layer 1 — Infrastructure (VMs, bare metal, cloud instances, networking)
```

A skilled cluster explorer moves through these layers systematically, understanding how each layer depends on and exposes the one below it.

### 3.2 The API as the Universal Explorer Interface

Every layer is reachable through the API server. This is the most important mental model for cluster exploration:

```bash
# The API server exposes all cluster knowledge
kubectl api-resources         # What resource types exist?
kubectl api-versions          # What API versions are registered?
kubectl get --raw /            # Raw API root
kubectl get --raw /apis        # All API groups
kubectl get --raw /api/v1      # Core API resources
```

### 3.3 Namespaced vs Cluster-Scoped Resources

Not all resources are namespaced. Understanding this prevents "missing" resources during exploration.

| Namespaced Resources | Cluster-Scoped Resources |
|---|---|
| Pod, Deployment, Service | Node, Namespace |
| ConfigMap, Secret | PersistentVolume, StorageClass |
| PersistentVolumeClaim | ClusterRole, ClusterRoleBinding |
| ServiceAccount, Role, RoleBinding | IngressClass |
| Ingress, NetworkPolicy | CustomResourceDefinition (CRD) |
| HorizontalPodAutoscaler | APIService |
| Job, CronJob | RuntimeClass |

```bash
# Discover which resources are namespaced vs cluster-scoped
kubectl api-resources --namespaced=true   # All namespaced resources
kubectl api-resources --namespaced=false  # All cluster-scoped resources
```

---

## 4. The Controller Pattern — Watch → Compare → Act → Loop

Understanding the controller pattern is essential when exploring a cluster because it explains **why** the cluster looks the way it does at any given moment. Every object you see is either a **desired state declaration** or **the outcome of a controller acting on that declaration**.

### 4.1 The Four-Phase Loop

```
┌──────────────────────────────────────────────────────────────┐
│               THE KUBERNETES CONTROLLER LOOP                 │
│                                                              │
│  ┌───────────────┐                                           │
│  │  1. WATCH     │  Subscribe to resource events via         │
│  │               │  API server informer (not polling)        │
│  └───────┬───────┘                                           │
│          │ Event: ADDED / MODIFIED / DELETED                 │
│          ▼                                                   │
│  ┌───────────────┐                                           │
│  │  2. COMPARE   │  Fetch desired state from spec            │
│  │               │  Fetch current state from status/world    │
│  │               │  Compute delta                            │
│  └───────┬───────┘                                           │
│          │ Delta found                                        │
│          ▼                                                   │
│  ┌───────────────┐                                           │
│  │  3. ACT       │  Make API calls to create/update/delete   │
│  │               │  resources to close the gap               │
│  └───────┬───────┘                                           │
│          │ Write result back to status                        │
│          ▼                                                   │
│  ┌───────────────┐                                           │
│  │  4. LOOP      │  Return to watching; triggered by next    │
│  │               │  event or periodic resync                 │
│  └───────────────┘                                           │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 Observing the Controller Loop in Action

```bash
# Watch a Deployment's reconciliation in real time
kubectl get deployment nginx-deploy -w

# Watch Pods being created/deleted during a rolling update
kubectl rollout status deployment/nginx-deploy -w

# See what the Deployment controller is doing in its events
kubectl describe deployment nginx-deploy | grep -A 20 "Events:"

# Watch the ReplicaSet controller create Pods
kubectl get pods -l app=nginx -w

# See the reconciliation decisions as events
kubectl get events --field-selector involvedObject.name=nginx-deploy \
  --sort-by='.lastTimestamp'
```

### 4.3 How the Loop Explains Cluster Exploration

When you explore a cluster and see:
- A Deployment with 3 desired but 2 actual replicas → the ReplicaSet controller is mid-reconciliation or blocked
- A Node in NotReady state → the Node controller has detected missed heartbeats and will start evicting Pods
- A Service with no Endpoints → either no Pods match the selector, or no Pods have a passing readiness probe
- A PVC stuck in Pending → the PV controller found no matching PersistentVolume or StorageClass

The controller loop is the diagnostic framework that explains every discrepancy between desired and actual state.

---

## 5. Exploring the Control Plane

### 5.1 Verifying Control Plane Health

```bash
# === CLUSTER INFO ===
kubectl cluster-info
# Output shows API server URL and CoreDNS URL

# === COMPONENT STATUS ===
# Method 1: Static pod inspection
kubectl get pods -n kube-system -l tier=control-plane
kubectl get pods -n kube-system -l component=kube-apiserver
kubectl get pods -n kube-system -l component=kube-controller-manager
kubectl get pods -n kube-system -l component=kube-scheduler
kubectl get pods -n kube-system -l component=etcd

# Method 2: Health check endpoints (from control plane node)
curl -k https://localhost:6443/healthz    # API server
curl -k https://localhost:6443/readyz     # API server readiness
curl -k https://localhost:6443/livez      # API server liveness
# Each returns "ok" if healthy

# Method 3: API version check
kubectl version --short
```

### 5.2 Exploring kube-apiserver Configuration

```bash
# Read the static pod manifest for API server flags
kubectl get pod kube-apiserver-$(hostname) -n kube-system -o yaml

# Or read directly from filesystem (on control plane node)
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Discover what admission plugins are enabled
kubectl get pod kube-apiserver-$(hostname) -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | tr ' ' '\n' | \
  grep 'enable-admission'

# Find all API server flags
kubectl get pod kube-apiserver-$(hostname) -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | tr ',' '\n'

# Check API server certificate SANs (what hostnames it serves)
kubectl get pod kube-apiserver-$(hostname) -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | tr ' ' '\n' | \
  grep 'tls-cert-file'
# Then inspect the cert
openssl x509 -in /etc/kubernetes/pki/apiserver.crt \
  -noout -text | grep -A 10 "Subject Alternative Name"
```

### 5.3 Exploring etcd

```bash
# Check etcd pod health
kubectl get pod etcd-$(hostname) -n kube-system

# Read etcd configuration
kubectl get pod etcd-$(hostname) -n kube-system -o yaml

# Explore etcd using etcdctl (on control plane node)
alias ETCDCTL='etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key'

# Cluster membership
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --write-out=table

# etcd cluster health
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Database size and status
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl endpoint status \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --write-out=table

# List all keys in etcd (top-level exploration)
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl get / --prefix --keys-only \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  2>/dev/null | head -50

# Count total objects in etcd
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl get /registry --prefix --keys-only \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  2>/dev/null | wc -l
```

### 5.4 Exploring the Controller Manager

```bash
# View controller manager pod
kubectl get pod kube-controller-manager-$(hostname) -n kube-system -o yaml

# Read all flags being passed to the controller manager
kubectl get pod kube-controller-manager-$(hostname) -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | tr ',' '\n'

# Check controller manager logs for reconciliation activity
kubectl logs -n kube-system kube-controller-manager-$(hostname) \
  --tail=100 | grep -i "controller\|reconcil\|error"

# View which controllers are running
kubectl logs -n kube-system kube-controller-manager-$(hostname) \
  --tail=1000 | grep "Starting controller" | sort -u

# Controller manager metrics (from control plane node)
curl -sk https://127.0.0.1:10257/metrics | \
  grep workqueue_depth | grep -v "^#"
```

### 5.5 Exploring the Scheduler

```bash
# View scheduler pod
kubectl get pod kube-scheduler-$(hostname) -n kube-system -o yaml

# Scheduler logs — see scheduling decisions
kubectl logs -n kube-system kube-scheduler-$(hostname) \
  --tail=100 | grep -i "scheduled\|preempt\|filter\|score"

# Find pending Pods that the scheduler is trying to place
kubectl get pods --all-namespaces --field-selector=status.phase=Pending

# Why is a Pod not scheduled?
kubectl describe pod <pending-pod-name> | grep -A 20 "Events:"
# Look for: "0/3 nodes are available: X Insufficient memory..."

# Scheduler metrics
curl -sk https://127.0.0.1:10259/metrics | \
  grep 'scheduler_pending_pods\|scheduler_scheduling_attempt'
```

---

## 6. Exploring Nodes — Data Plane Inspection

### 6.1 Node Overview

```bash
# List all nodes with full detail
kubectl get nodes -o wide

# Node capacity and allocatable resources
kubectl get nodes -o custom-columns=\
"NAME:.metadata.name,\
CPU:.status.capacity.cpu,\
MEMORY:.status.capacity.memory,\
PODS:.status.capacity.pods,\
ALLOCATABLE-CPU:.status.allocatable.cpu,\
ALLOCATABLE-MEM:.status.allocatable.memory"

# Describe a specific node — the most information-rich single command
kubectl describe node <node-name>
# Read carefully:
# - Conditions (Ready, MemoryPressure, DiskPressure, PIDPressure)
# - Addresses (InternalIP, Hostname, ExternalIP)
# - Capacity vs Allocatable (reserved for kube/system)
# - System Info (OS, kernel, container runtime, kubelet version)
# - Non-terminated Pods (what's actually running)
# - Allocated resources (CPU/memory requests/limits)
# - Events (recent node-level events)
```

### 6.2 Node Conditions Deep Dive

```bash
# Check all node conditions in structured format
kubectl get nodes -o json | jq \
  '.items[] | {name: .metadata.name, conditions: .status.conditions}'

# Find nodes with any non-True conditions
kubectl get nodes -o json | jq \
  '.items[] | select(.status.conditions[] | 
   select(.type != "Ready" and .status == "True")) | 
   .metadata.name'

# Check specific condition
kubectl get node <node-name> -o jsonpath=\
'{.status.conditions[?(@.type=="Ready")].status}'
# Expected: True

# Full condition matrix for all nodes
for node in $(kubectl get nodes -o name); do
  echo "=== $node ==="
  kubectl get $node -o jsonpath=\
'{range .status.conditions[*]}{.type}={.status} ({.reason}){"\n"}{end}'
  echo ""
done
```

### 6.3 Node Resource Utilization

```bash
# Top-level resource usage (requires metrics-server)
kubectl top nodes

# Detailed per-node resource allocation
kubectl describe nodes | grep -A 8 "Allocated resources:"

# Find most resource-hungry nodes
kubectl top nodes --sort-by=cpu
kubectl top nodes --sort-by=memory

# Calculate resource utilization percentage
kubectl get nodes -o json | jq -r '
  .items[] | 
  "\(.metadata.name): CPU capacity=\(.status.capacity.cpu), 
   Allocatable=\(.status.allocatable.cpu)"'

# Pod density per node
kubectl get pods --all-namespaces -o wide | \
  awk '{print $8}' | sort | uniq -c | sort -rn
```

### 6.4 Node Taints and Labels

```bash
# View all node labels
kubectl get nodes --show-labels

# Find nodes by label
kubectl get nodes -l node-role.kubernetes.io/control-plane
kubectl get nodes -l kubernetes.io/arch=amd64
kubectl get nodes -l topology.kubernetes.io/zone

# View all taints
kubectl get nodes -o custom-columns=\
"NAME:.metadata.name,TAINTS:.spec.taints"

# Which workloads tolerate which taints?
kubectl get pods --all-namespaces -o json | jq \
  '.items[] | select(.spec.tolerations != null) | 
   {name: .metadata.name, tolerations: .spec.tolerations}'
```

### 6.5 Node-Level Inspection (SSH Required)

```bash
# On the node itself

# Check kubelet status and configuration
systemctl status kubelet
cat /var/lib/kubelet/config.yaml
journalctl -u kubelet -n 100 --no-pager

# Check running containers via CRI
crictl ps                          # Running containers
crictl ps -a                       # All containers (including stopped)
crictl pods                        # All pods via CRI
crictl stats                       # Container resource stats
crictl images                      # Cached images on this node

# Container runtime
crictl info                        # Runtime info
crictl version                     # Runtime version

# Node networking
ip route show                      # Routing table
ip addr show                       # Network interfaces
iptables -t nat -L KUBE-SERVICES   # Service routing rules
ipvsadm -L -n 2>/dev/null || echo "IPVS not in use"

# Check CNI config
cat /etc/cni/net.d/*.conf* 2>/dev/null | head -50

# Check cgroup allocation
cat /proc/meminfo
df -h
cat /sys/fs/cgroup/memory/kubepods/memory.limit_in_bytes
```

---

## 7. Exploring Namespaces and Workloads

### 7.1 Namespace Inventory

```bash
# List all namespaces with status
kubectl get namespaces

# Namespace labels (often reveal team/environment assignments)
kubectl get namespaces --show-labels

# Describe a namespace for quota and limit range info
kubectl describe namespace production

# Resource quotas per namespace
kubectl get resourcequota --all-namespaces

# Limit ranges per namespace
kubectl get limitrange --all-namespaces

# How many resources exist in each namespace?
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  count=$(kubectl get all -n $ns --ignore-not-found 2>/dev/null | \
    grep -v "^NAME" | wc -l)
  echo "$ns: $count resources"
done
```

### 7.2 Workload Inventory

```bash
# Complete workload overview across all namespaces
kubectl get all --all-namespaces

# Just the key workload types
kubectl get deployments --all-namespaces
kubectl get statefulsets --all-namespaces
kubectl get daemonsets --all-namespaces
kubectl get jobs --all-namespaces
kubectl get cronjobs --all-namespaces
kubectl get replicasets --all-namespaces

# Pod status summary
kubectl get pods --all-namespaces -o wide
kubectl get pods --all-namespaces --field-selector=status.phase=Pending
kubectl get pods --all-namespaces --field-selector=status.phase=Failed
kubectl get pods --all-namespaces | grep -v Running | grep -v Completed

# Find pods that have restarted many times (unstable)
kubectl get pods --all-namespaces -o json | jq \
  '.items[] | select(.status.containerStatuses != null) |
   .status.containerStatuses[] | 
   select(.restartCount > 5) |
   {pod: .name, restarts: .restartCount, reason: .state}'
```

### 7.3 Deep Pod Inspection

```bash
# Full pod specification
kubectl get pod <pod-name> -n <namespace> -o yaml

# Runtime status (what's actually happening)
kubectl describe pod <pod-name> -n <namespace>

# Pod logs (current)
kubectl logs <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace> -c <container-name>  # Multi-container

# Pod logs (previous container — after crash)
kubectl logs <pod-name> -n <namespace> --previous

# Stream logs in real time
kubectl logs -f <pod-name> -n <namespace>

# Last N lines with timestamps
kubectl logs <pod-name> -n <namespace> \
  --tail=100 --timestamps=true

# Execute into a running pod
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Specific container in multi-container pod
kubectl exec -it <pod-name> -n <namespace> \
  -c <container-name> -- /bin/sh

# Check environment variables (reveals ConfigMap/Secret injections)
kubectl exec <pod-name> -n <namespace> -- env | sort

# Inspect filesystem mounts
kubectl exec <pod-name> -n <namespace> -- mount | grep -v tmpfs

# Check /etc/resolv.conf (DNS configuration)
kubectl exec <pod-name> -n <namespace> -- cat /etc/resolv.conf

# Copy files from pod for inspection
kubectl cp <namespace>/<pod-name>:/var/log/app.log /tmp/app.log
```

### 7.4 Deployment Health Analysis

```bash
# Deployment rollout status
kubectl rollout status deployment/<name> -n <namespace>

# Deployment history
kubectl rollout history deployment/<name> -n <namespace>

# Specific revision detail
kubectl rollout history deployment/<name> -n <namespace> --revision=3

# What ReplicaSets exist for this Deployment?
kubectl get replicasets -n <namespace> \
  -l app=<app-label> \
  -o wide
# Multiple RS = rollout in progress or history retained

# Deployment scaling status
kubectl get deployment <name> -n <namespace> -o json | jq \
  '{name: .metadata.name,
    desired: .spec.replicas,
    ready: .status.readyReplicas,
    updated: .status.updatedReplicas,
    available: .status.availableReplicas}'
```

---

## 8. Deep Dive: Built-in Controllers and How to Observe Them

### 8.1 ReplicaSet Controller

The ReplicaSet controller ensures the desired number of Pod replicas are running at all times. It is the self-healing mechanism for stateless workloads.

**What it manages:** Maintains Pod count by creating or deleting Pods based on the delta between `spec.replicas` and the count of Pods matching `spec.selector`.

```bash
# Explore ReplicaSets
kubectl get replicasets -n <namespace> -o wide
kubectl describe replicaset <rs-name> -n <namespace>

# See how ReplicaSet controller manages Pod count
kubectl scale deployment <name> --replicas=5 -n <namespace>
kubectl get pods -n <namespace> -w  # Watch Pods being created

# Orphaned ReplicaSets (from old deployments)
kubectl get replicasets -n <namespace> | awk '{print $3}' | sort | uniq -c
# Column 3 is DESIRED - RSes with 0 desired are orphaned history

# Ownerreferences: who owns this ReplicaSet?
kubectl get replicaset <rs-name> -n <namespace> \
  -o jsonpath='{.metadata.ownerReferences}'
```

**Reconciliation flow:**

```
1. ReplicaSet event (created/updated) enters work queue
2. Controller reads RS.spec.replicas (desired count)
3. Controller counts Pods matching RS.spec.selector that are NOT terminating
4. Delta = desired - current
5. If delta > 0: create |delta| Pods
6. If delta < 0: delete |delta| Pods (preferring unscheduled > failed > older)
7. Update RS.status.readyReplicas, .availableReplicas
```

### 8.2 Deployment Controller

The Deployment controller manages ReplicaSets to implement declarative, versioned application updates.

```bash
# Observe Deployment controller creating new ReplicaSet during update
kubectl set image deployment/nginx nginx=nginx:1.25 -n <namespace>
kubectl get replicasets -n <namespace> -w  # Watch new RS scale up, old scale down

# See the strategy being applied
kubectl get deployment <name> -n <namespace> \
  -o jsonpath='{.spec.strategy}'

# Observe annotations that track rollout state
kubectl get replicaset <rs-name> -n <namespace> \
  -o jsonpath='{.metadata.annotations}' | jq

# Key annotation: deployment.kubernetes.io/revision
```

**ReplicaSet lineage:**

```bash
# Map all ReplicaSets back to their owning Deployment
kubectl get replicasets --all-namespaces -o json | jq \
  '.items[] | {
    namespace: .metadata.namespace,
    rs: .metadata.name,
    owner: .metadata.ownerReferences[0].name,
    desired: .spec.replicas
  }'
```

### 8.3 Node Controller

The Node controller manages the lifecycle of Node objects. It monitors heartbeats from kubelets and takes action when nodes become unreachable.

```bash
# View Node controller behavior through node status
kubectl get nodes
kubectl describe node <node-name> | grep -A 20 "Conditions:"

# Monitor node heartbeat status
kubectl get node <node-name> \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")]}' | jq

# Node controller taints added when node is NotReady
kubectl describe node <node-name> | grep -i "taint"
# node.kubernetes.io/not-ready:NoSchedule
# node.kubernetes.io/unreachable:NoExecute

# Pods evicted from NotReady nodes
kubectl get events --all-namespaces | grep -i "evict\|node"

# Node CIDR allocation by Node controller
kubectl get node <node-name> \
  -o jsonpath='{.spec.podCIDR}'
```

**Node controller timeline:**
```
T+0s:    Node kubelet stops sending heartbeats
T+40s:   node-monitor-grace-period expires → condition set to Unknown
T+5m:    pod-eviction-timeout expires → Pods evicted (taint applied)
T+5m+:   Replacement Pods scheduled on other nodes
```

### 8.4 Service Controller

The Service controller handles cloud load balancer provisioning for `LoadBalancer`-type Services.

```bash
# Find LoadBalancer Services
kubectl get services --all-namespaces \
  --field-selector spec.type=LoadBalancer

# Check if external IP has been assigned
kubectl get svc <name> -n <namespace> -w
# EXTERNAL-IP transitions from <pending> to an actual IP

# View Service controller events
kubectl describe svc <name> -n <namespace> | grep -A 10 "Events:"
# Look for: "Ensuring load balancer", "Created load balancer"

# Inspect load balancer annotations (cloud-specific config)
kubectl get svc <name> -n <namespace> -o yaml | \
  grep -A 20 "annotations:"
```

### 8.5 Endpoints / EndpointSlice Controller

```bash
# View Endpoints for a Service
kubectl get endpoints <service-name> -n <namespace>
kubectl describe endpoints <service-name> -n <namespace>

# View EndpointSlices (v1.21+ default)
kubectl get endpointslices -n <namespace> \
  -l kubernetes.io/service-name=<service-name>

# Trace why a Service has no Endpoints
# Step 1: Check Service selector
kubectl get svc <name> -n <namespace> \
  -o jsonpath='{.spec.selector}'

# Step 2: Find Pods matching that selector
kubectl get pods -n <namespace> -l <key>=<value>

# Step 3: Check Pod readiness
kubectl get pods -n <namespace> -l <key>=<value> \
  -o jsonpath='{range .items[*]}{.metadata.name}: {.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'
```

### 8.6 Namespace Controller

```bash
# Observe namespace deletion cascade
kubectl delete namespace <ns-name>

# Watch the cascade (in another terminal)
kubectl get all -n <ns-name> -w

# Namespace stuck in Terminating?
kubectl get namespace <ns-name> -o yaml | grep finalizers

# Force-remove finalizers (use carefully!)
kubectl patch namespace <ns-name> \
  -p '{"metadata":{"finalizers":[]}}' \
  --type=merge
```

### 8.7 Job Controller

```bash
# Explore all Jobs
kubectl get jobs --all-namespaces

# Job completion status
kubectl get job <name> -n <namespace> -o json | jq \
  '{name: .metadata.name,
    completions: .spec.completions,
    parallelism: .spec.parallelism,
    succeeded: .status.succeeded,
    failed: .status.failed,
    active: .status.active}'

# Job pods (including completed)
kubectl get pods -n <namespace> \
  --selector job-name=<job-name> \
  --show-all  # Shows completed pods

# Job history in CronJob
kubectl get cronjob <name> -n <namespace> -o yaml
kubectl get jobs -n <namespace> \
  -l job-name=<cronjob-name>
```

### 8.8 StatefulSet Controller

```bash
# Explore StatefulSets
kubectl get statefulsets --all-namespaces

# StatefulSet pod ordering (note sequential -0, -1, -2 naming)
kubectl get pods -n <namespace> \
  -l app=<statefulset-app> \
  -o wide

# PVC per StatefulSet pod
kubectl get pvc -n <namespace> \
  -l app=<statefulset-app>

# Headless Service (required by StatefulSet)
kubectl get svc -n <namespace> --field-selector spec.clusterIP=None

# StatefulSet Pod DNS names
kubectl exec <statefulset-pod-0> -n <namespace> -- \
  nslookup <statefulset-name>-0.<headless-svc>.<namespace>.svc.cluster.local

# StatefulSet update strategy
kubectl get statefulset <name> -n <namespace> \
  -o jsonpath='{.spec.updateStrategy}'
```

### 8.9 DaemonSet Controller

```bash
# List all DaemonSets
kubectl get daemonsets --all-namespaces

# DaemonSet rollout status
kubectl rollout status daemonset/<name> -n <namespace>

# Verify one Pod per node
kubectl get pods -n <namespace> \
  -l name=<daemonset-label> \
  -o wide | awk '{print $7}' | sort

# Nodes where DaemonSet is NOT running (investigate why)
kubectl get nodes -o name | while read node; do
  kubectl get pods -n <namespace> \
    --field-selector spec.nodeName=${node#node/} \
    -l name=<daemonset-label> 2>/dev/null | \
    grep -q <daemonset-label> || echo "MISSING: $node"
done

# DaemonSet tolerations (why it runs on all nodes including control-plane)
kubectl get daemonset <name> -n <namespace> \
  -o jsonpath='{.spec.template.spec.tolerations}'
```

### 8.10 Garbage Collector

```bash
# View owner references (what GC will clean up)
kubectl get pod <pod-name> -n <namespace> \
  -o jsonpath='{.metadata.ownerReferences}'

# Find orphaned resources (no owner reference but type usually has one)
kubectl get pods --all-namespaces -o json | jq \
  '.items[] | select(.metadata.ownerReferences == null) | 
   {namespace: .metadata.namespace, name: .metadata.name}'

# Orphaned ReplicaSets (0 desired, relics of rollouts)
kubectl get replicasets --all-namespaces \
  -o json | jq \
  '.items[] | select(.spec.replicas == 0) | 
   {namespace: .metadata.namespace, name: .metadata.name}'

# Completed Jobs (GC will clean these based on TTL)
kubectl get jobs --all-namespaces \
  --field-selector status.successful=1
```

### 8.11 PersistentVolume Controller

```bash
# Full storage inventory
kubectl get persistentvolumes
kubectl get persistentvolumeclaims --all-namespaces
kubectl get storageclasses

# PV binding status
kubectl get pv -o custom-columns=\
"NAME:.metadata.name,\
CAPACITY:.spec.capacity.storage,\
ACCESS:.spec.accessModes,\
RECLAIM:.spec.persistentVolumeReclaimPolicy,\
STATUS:.status.phase,\
CLAIM:.spec.claimRef.name"

# PVC binding status
kubectl get pvc --all-namespaces -o custom-columns=\
"NAMESPACE:.metadata.namespace,\
NAME:.metadata.name,\
STATUS:.status.phase,\
VOLUME:.spec.volumeName,\
CAPACITY:.spec.resources.requests.storage,\
STORAGECLASS:.spec.storageClassName"

# Stuck PVC (Pending)
kubectl describe pvc <pvc-name> -n <namespace>
# Look for: "no persistent volumes available for this claim"
# Or: "waiting for first consumer" (WaitForFirstConsumer binding mode)

# Default StorageClass
kubectl get storageclass | grep "(default)"
```

---

## 9. Internal Working Concepts: Informers, Work Queues & Reconciliation

### 9.1 The Informer Pattern — Observing Its Effects

The informer is the mechanism that allows controllers to watch the API server without polling. Understanding this helps you diagnose performance and correctness issues.

```
┌──────────────────────────────────────────────────────────────────┐
│                    INFORMER ARCHITECTURE                         │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │                    Reflector                             │   │
│  │  ┌─────────────────┐    ┌──────────────────────────────┐ │   │
│  │  │  Initial LIST   │    │   Watch (streaming events)   │ │   │
│  │  │  GET /api/v1/   │    │   GET /api/v1/pods?watch=1   │ │   │
│  │  │  pods           │───►│   &resourceVersion=12345     │ │   │
│  │  └─────────────────┘    └──────────────────────────────┘ │   │
│  └────────────────────────────────┬─────────────────────────┘   │
│                                   │ ADDED / MODIFIED / DELETED   │
│                                   ▼                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Local Cache (Indexer / Store)                          │    │
│  │  Thread-safe in-memory copy of all watched objects      │    │
│  │  Controllers read FROM THIS, not from API server        │    │
│  └────────────────────────┬────────────────────────────────┘    │
│                            │                                     │
│            ┌───────────────┼───────────────────┐                │
│            ▼               ▼                   ▼                │
│       OnAdd()          OnUpdate()          OnDelete()           │
│            │               │                   │                │
│            └───────────────┴───────────────────┘                │
│                            │                                     │
│                            ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Work Queue (Rate-Limited)                   │    │
│  │  Keys only (namespace/name) — deduplicated              │    │
│  │  Exponential backoff on errors                          │    │
│  └────────────────────────┬────────────────────────────────┘    │
│                            │                                     │
│                    ┌───────┴──────┐                              │
│                    ▼              ▼                              │
│             Worker 1          Worker 2    (N concurrent workers) │
│                    │              │                              │
│                    └──────┬───────┘                              │
│                           ▼                                      │
│             reconcile(namespace/name)                            │
│             → Read from cache                                    │
│             → Call API server to make changes                    │
└──────────────────────────────────────────────────────────────────┘
```

### 9.2 Observing Informer Effects

```bash
# See how many objects each resource type has (informer cache size)
kubectl api-resources --verbs=list \
  -o name | while read resource; do
  count=$(kubectl get $resource --all-namespaces \
    --ignore-not-found 2>/dev/null | \
    grep -v "^NAME" | wc -l)
  echo "$resource: $count"
done 2>/dev/null | sort -t: -k2 -rn | head -20

# Observe API server watch connections (from control plane node)
ss -tnp | grep 6443 | wc -l
# Number of active watch connections to API server

# See informer resync happening (bump in API requests)
kubectl get --raw /metrics 2>/dev/null | \
  grep 'apiserver_request_total{.*watch' | head -5
```

### 9.3 Work Queue Metrics

```bash
# Check controller work queue depths (from control plane node)
curl -sk https://127.0.0.1:10257/metrics | \
  grep 'workqueue_depth' | grep -v "^#" | \
  awk '$2 > 0'

# Identify backed-up controllers
curl -sk https://127.0.0.1:10257/metrics | \
  grep 'workqueue_depth' | \
  sort -t= -k2 -rn | head -10

# Work queue retry rates (high = controllers struggling)
curl -sk https://127.0.0.1:10257/metrics | \
  grep 'workqueue_retries_total' | grep -v "^#"

# Processing duration (how long reconciliations take)
curl -sk https://127.0.0.1:10257/metrics | \
  grep 'workqueue_work_duration_seconds_bucket' | \
  head -20
```

### 9.4 Reconciliation Loop Timing

```bash
# Observe a reconciliation loop in action
# Terminal 1: Watch a Deployment
kubectl get deployment nginx -n default -w &

# Terminal 2: Introduce a discrepancy
kubectl delete pod $(kubectl get pods -l app=nginx -o name | head -1)

# Observe: ReplicaSet controller detects deficit, creates new Pod
# Timeline is typically:
# T+0ms: Pod deletion event enters informer
# T+5ms: Work queue item added
# T+10ms: Worker picks up item
# T+50ms: New Pod created via API server
```

---

## 10. Interaction with API Server and etcd During Exploration

### 10.1 The Inviolable Rule

```
╔═══════════════════════════════════════════════════════════════╗
║                                                               ║
║   ALL KUBERNETES COMPONENTS COMMUNICATE EXCLUSIVELY          ║
║   THROUGH THE KUBE-APISERVER.                                 ║
║                                                               ║
║   CONTROLLERS NEVER ACCESS ETCD DIRECTLY.                     ║
║   KUBECTL NEVER ACCESSES ETCD DIRECTLY.                       ║
║   KUBELETS NEVER ACCESS ETCD DIRECTLY.                        ║
║                                                               ║
║   etcd ←→ kube-apiserver ←→ Everything else                  ║
║                                                               ║
╚═══════════════════════════════════════════════════════════════╝
```

### 10.2 Tracing a kubectl Command Through the System

```bash
# When you run:
kubectl get pods -n production

# What actually happens:
# 1. kubectl reads ~/.kube/config for server URL and credentials
# 2. kubectl sends HTTP GET to https://<api-server>:6443/api/v1/namespaces/production/pods
# 3. kube-apiserver authenticates kubectl's certificate
# 4. kube-apiserver authorizes via RBAC (can this user list pods in production?)
# 5. kube-apiserver reads from etcd: GET /registry/pods/production/*
# 6. kube-apiserver serializes and returns the response
# 7. kubectl formats and displays the output

# To see the actual HTTP request being made:
kubectl get pods -n production -v=9 2>&1 | head -50
# -v=9 shows full HTTP request/response

# Just the URL:
kubectl get pods -n production -v=6 2>&1 | grep GET
```

### 10.3 API Exploration Techniques

```bash
# Explore the raw API hierarchy
kubectl get --raw /
kubectl get --raw /apis
kubectl get --raw /api/v1
kubectl get --raw /apis/apps/v1

# Get a specific resource via raw API
kubectl get --raw /api/v1/namespaces/default/pods | jq '.items[].metadata.name'

# Watch via raw API (see streaming events)
kubectl get --raw '/api/v1/namespaces/default/pods?watch=1' &

# See ALL API versions for a resource type
kubectl explain pod --api-version=v1
kubectl explain deployment --api-version=apps/v1

# Deep field exploration
kubectl explain pod.spec.containers
kubectl explain deployment.spec.strategy.rollingUpdate

# API server request statistics
kubectl get --raw /metrics | grep 'apiserver_request_total' | \
  grep -v "^#" | sort -t= -k5 -rn | head -10
```

### 10.4 etcd as the Ground Truth

```bash
# Read a specific resource directly from etcd (bypasses API server cache)
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl get /registry/pods/default/nginx-pod \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  | strings | head -30

# Read all resources of a type from etcd
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl get /registry/pods --prefix --keys-only \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# etcd key structure (always /registry/<resource-type>/<namespace>/<name>)
# Examples:
# /registry/pods/default/my-pod
# /registry/services/production/my-service
# /registry/deployments/production/my-deployment
# /registry/secrets/kube-system/my-secret
# /registry/namespaces/production   (namespace is cluster-scoped)
```

### 10.5 API Server Audit Log Exploration

```bash
# Real-time audit log (if enabled)
tail -f /var/log/kubernetes/audit.log | jq \
  'select(.verb != "watch" and .verb != "get") | 
   {user: .user.username, verb: .verb, 
    resource: .objectRef.resource, 
    name: .objectRef.name}'

# Who made the most API calls recently?
tail -1000 /var/log/kubernetes/audit.log | \
  jq '.user.username' | sort | uniq -c | sort -rn

# What resources were modified in the last hour?
tail -f /var/log/kubernetes/audit.log | \
  jq 'select(.verb == "create" or .verb == "update" or .verb == "delete")'
```

---

## 11. Leader Election — Observing and Understanding

### 11.1 Why Leader Election Exists

In high-availability setups, multiple instances of `kube-controller-manager` and `kube-scheduler` run simultaneously. Without coordination, multiple leaders would:
- Simultaneously reconcile the same Deployment → race conditions
- Create duplicate cloud load balancers → resource waste and routing failures
- Issue contradictory Pod deletion decisions → unexpected behavior

**Leader election ensures exactly one instance is the active "leader" at any time.** The others are warm standbys.

### 11.2 Observing Leader Election via Lease Objects

```bash
# View the current kube-controller-manager leader
kubectl get lease kube-controller-manager -n kube-system -o yaml

# Output reveals:
# spec.holderIdentity: "k8s-cp-1_<uuid>"  — which node is the leader
# spec.renewTime: "2025-01-01T..."          — when it last renewed the lease
# spec.leaseDurationSeconds: 15            — how long the lease is valid
# spec.leaseTransitions: 3                 — how many times leadership has changed

# View the kube-scheduler leader
kubectl get lease kube-scheduler -n kube-system -o yaml

# Watch leader election in real time
kubectl get lease -n kube-system -w

# All leases in the cluster (includes custom operators)
kubectl get lease --all-namespaces

# Identify leader-elected components that are custom
kubectl get lease --all-namespaces | grep -v "kube-system"
```

### 11.3 Lease Object Deep Dive

```yaml
# Structure of a Lease object
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  acquireTime: "2025-01-01T10:00:00.000000Z"   # When current leader acquired
  holderIdentity: "k8s-cp-2_abc-def-uuid"       # Current leader identity
  leaseDurationSeconds: 30                       # Lease validity period
  leaseTransitions: 7                            # Historical leadership changes
  renewTime: "2025-01-01T12:45:30.000000Z"      # Last heartbeat from leader
```

**Election algorithm:**
```
All standby instances race to UPDATE the Lease object with their identity.
Kubernetes uses optimistic concurrency (resourceVersion) — only one UPDATE succeeds.
The winner becomes leader and must renew the lease before leaseDurationSeconds expires.
If the leader fails to renew, the lease expires and standbys race again.
```

### 11.4 Leader Election Flags

| Flag | Default | Component | Purpose |
|---|---|---|---|
| `--leader-elect` | `true` | controller-manager, scheduler | Enable/disable leader election |
| `--leader-elect-lease-duration` | `15s` | Both | How long a non-renewing leader holds the lease |
| `--leader-elect-renew-deadline` | `10s` | Both | How often the leader must renew |
| `--leader-elect-retry-period` | `2s` | Both | How often standbys attempt to acquire |
| `--leader-elect-resource-lock` | `leases` | Both | Resource type used for locking |
| `--leader-elect-resource-namespace` | `kube-system` | Both | Namespace of lock object |
| `--leader-elect-resource-name` | (component name) | Both | Name of lock object |

### 11.5 HA Best Practices for Leader Election

```bash
# For production HA, run 3 replicas of each control plane component
# Verify replica counts
kubectl get pods -n kube-system | grep -E "controller-manager|scheduler"

# For large clusters, tune lease duration (longer = more stable, slower failover)
# Recommended for production:
# --leader-elect-lease-duration=30s
# --leader-elect-renew-deadline=25s
# --leader-elect-retry-period=5s

# Monitor lease transitions (high count = instability)
kubectl get lease kube-controller-manager -n kube-system \
  -o jsonpath='{.spec.leaseTransitions}'
# Should be a low number; high values indicate control plane instability

# Alert rule for leader election instability
# etcd_server_leader_changes_seen_total increasing rapidly = etcd instability
```

---

## 12. Exploring Networking — Services, DNS, and CNI

### 12.1 Service Discovery

```bash
# Full service inventory
kubectl get services --all-namespaces -o wide

# Categorize by type
kubectl get svc --all-namespaces \
  -o custom-columns="NS:.metadata.namespace,\
NAME:.metadata.name,\
TYPE:.spec.type,\
CLUSTER-IP:.spec.clusterIP,\
EXTERNAL-IP:.status.loadBalancer.ingress[0].ip,\
PORT:.spec.ports[*].port"

# Find services with no endpoints (potential misconfiguration)
for svc in $(kubectl get svc --all-namespaces \
  -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}'); do
  ns=$(echo $svc | cut -d/ -f1)
  name=$(echo $svc | cut -d/ -f2)
  ep=$(kubectl get endpoints $name -n $ns \
    -o jsonpath='{.subsets}' 2>/dev/null)
  if [ -z "$ep" ]; then
    echo "NO ENDPOINTS: $svc"
  fi
done

# Explore a Service in detail
kubectl describe svc <name> -n <namespace>

# What Pods does this Service route to?
kubectl get svc <name> -n <namespace> \
  -o jsonpath='{.spec.selector}' | \
  jq 'to_entries[] | "\(.key)=\(.value)"' | \
  xargs -I{} kubectl get pods -n <namespace> -l {}
```

### 12.2 DNS Exploration

```bash
# CoreDNS configuration
kubectl get configmap coredns -n kube-system -o yaml

# CoreDNS health
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Test DNS resolution from inside the cluster
kubectl run dnstest --image=busybox:1.28 \
  --restart=Never -it --rm \
  -- sh -c "
    nslookup kubernetes.default
    nslookup kube-dns.kube-system.svc.cluster.local
    cat /etc/resolv.conf
  "

# Test cross-namespace resolution
kubectl run dnstest --image=busybox:1.28 \
  --restart=Never -it --rm \
  -- nslookup <service>.<namespace>.svc.cluster.local

# DNS lookup timing
kubectl run dnstest --image=busybox:1.28 \
  --restart=Never -it --rm \
  -- time nslookup kubernetes.default

# CoreDNS logs (check for NXDOMAIN or SERVFAIL)
kubectl logs -n kube-system -l k8s-app=kube-dns \
  --tail=100 | grep -i "error\|fail\|refused"
```

### 12.3 CNI Exploration

```bash
# Identify which CNI is installed
kubectl get pods -n kube-system | grep -E "calico|cilium|flannel|weave|antrea"

# CNI configuration files on a node
ls /etc/cni/net.d/
cat /etc/cni/net.d/*.conf* 2>/dev/null

# Calico specific
kubectl get felixconfigurations default -o yaml 2>/dev/null
kubectl get ippool --all-namespaces 2>/dev/null
calicoctl get nodes 2>/dev/null

# Cilium specific
cilium status 2>/dev/null
kubectl get ciliumnetworkpolicies --all-namespaces 2>/dev/null

# NetworkPolicies (security-relevant)
kubectl get networkpolicies --all-namespaces

# Namespaces with no NetworkPolicy (security risk)
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  count=$(kubectl get networkpolicies -n $ns \
    --ignore-not-found 2>/dev/null | grep -v "^NAME" | wc -l)
  echo "$ns: $count network policies"
done
```

### 12.4 kube-proxy Exploration

```bash
# kube-proxy mode (iptables vs IPVS)
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# Check iptables rules on a node
kubectl exec -n kube-system \
  $(kubectl get pods -n kube-system -l k8s-app=kube-proxy -o name | head -1) \
  -- iptables -t nat -L KUBE-SERVICES --line-numbers | head -30

# IPVS virtual server table (if IPVS mode)
kubectl exec -n kube-system \
  $(kubectl get pods -n kube-system -l k8s-app=kube-proxy -o name | head -1) \
  -- ipvsadm -L -n 2>/dev/null | head -30

# kube-proxy last sync time
kubectl get --raw /api/v1/nodes/<node-name>/proxy/metrics 2>/dev/null | \
  grep 'kubeproxy_sync_proxy_rules_last_timestamp'
```

### 12.5 Ingress Exploration

```bash
# All Ingress objects
kubectl get ingress --all-namespaces

# Ingress controllers
kubectl get pods --all-namespaces | grep -i "ingress\|nginx\|traefik\|haproxy"

# Ingress detail
kubectl describe ingress <name> -n <namespace>

# IngressClass definitions
kubectl get ingressclass

# Ingress with TLS
kubectl get ingress --all-namespaces \
  -o json | jq \
  '.items[] | select(.spec.tls != null) | 
   {name: .metadata.name, tls: .spec.tls}'
```

---

## 13. Exploring Storage — PersistentVolumes and StorageClasses

### 13.1 Storage Topology

```bash
# Complete storage picture
echo "=== StorageClasses ==="
kubectl get storageclass

echo "=== PersistentVolumes ==="
kubectl get pv

echo "=== PersistentVolumeClaims ==="
kubectl get pvc --all-namespaces

# Storage usage by namespace
kubectl get pvc --all-namespaces -o json | jq \
  'group_by(.metadata.namespace)[] | 
   {namespace: .[0].metadata.namespace,
    pvc_count: length,
    total_storage: [.[].spec.resources.requests.storage] | 
    map(gsub("Gi";"") | tonumber) | add | tostring + "Gi"}'

# Unbound PVCs (storage not yet provisioned)
kubectl get pvc --all-namespaces \
  --field-selector status.phase=Pending

# Released PVs (not yet available for reuse)
kubectl get pv --field-selector status.phase=Released

# PV reclaim policies
kubectl get pv -o custom-columns=\
"NAME:.metadata.name,\
RECLAIM:.spec.persistentVolumeReclaimPolicy,\
STATUS:.status.phase,\
CLAIM:.spec.claimRef.namespace"
```

### 13.2 Storage Class Inspection

```bash
# Detailed StorageClass inspection
kubectl describe storageclass <name>

# Default StorageClass (must be exactly one)
kubectl get storageclass \
  -o json | jq \
  '.items[] | 
   select(.metadata.annotations["storageclass.kubernetes.io/is-default-class"] == "true") |
   .metadata.name'

# Volume binding modes
kubectl get storageclass -o custom-columns=\
"NAME:.metadata.name,\
PROVISIONER:.provisioner,\
BINDING:.volumeBindingMode,\
EXPAND:.allowVolumeExpansion"
```

---

## 14. Exploring RBAC, Security, and Service Accounts

### 14.1 RBAC Exploration

```bash
# Who has cluster-admin?
kubectl get clusterrolebindings -o json | jq \
  '.items[] | 
   select(.roleRef.name == "cluster-admin") | 
   {binding: .metadata.name, 
    subjects: .subjects}'

# All ClusterRoleBindings
kubectl get clusterrolebindings -o wide

# What can a specific service account do?
kubectl auth can-i --list \
  --as=system:serviceaccount:<namespace>:<sa-name>

# What can a specific user do?
kubectl auth can-i --list --as=<username>

# Can this user perform a specific action?
kubectl auth can-i create pods -n production \
  --as=system:serviceaccount:production:my-app

# Explore Roles in a namespace
kubectl get roles -n <namespace>
kubectl describe role <name> -n <namespace>

# Who has access to Secrets? (sensitive audit)
kubectl get clusterrolebindings,rolebindings --all-namespaces \
  -o json | jq \
  '.items[] | 
   select(.roleRef.name | 
   contains("secret") or . == "view" or . == "edit" or . == "admin")'

# RBAC audit: all users who can get secrets
kubectl auth can-i get secrets \
  --as=system:serviceaccount:default:default
```

### 14.2 Service Account Exploration

```bash
# All service accounts
kubectl get serviceaccounts --all-namespaces

# Service accounts with auto-mounted tokens
kubectl get serviceaccounts --all-namespaces -o json | jq \
  '.items[] | 
   select(.automountServiceAccountToken != false) | 
   {namespace: .metadata.namespace, name: .metadata.name}'

# Secrets associated with a service account
kubectl describe serviceaccount <name> -n <namespace>

# What roles are bound to this service account?
kubectl get rolebindings,clusterrolebindings \
  --all-namespaces -o json | jq \
  '.items[] | 
   select(.subjects != null) |
   select(.subjects[] | 
     select(.kind == "ServiceAccount" and 
            .name == "<sa-name>" and 
            .namespace == "<namespace>")) | 
   {binding: .metadata.name, role: .roleRef.name}'

# Pods using a specific service account
kubectl get pods -n <namespace> \
  -o json | jq \
  '.items[] | 
   select(.spec.serviceAccountName == "<sa-name>") | 
   .metadata.name'
```

### 14.3 Secret Exploration (Security-Sensitive)

```bash
# Count secrets by type
kubectl get secrets --all-namespaces -o json | jq \
  '[.items[] | .type] | group_by(.) | 
   map({type: .[0], count: length})'

# Find secrets by type
kubectl get secrets --all-namespaces \
  --field-selector type=kubernetes.io/tls

# Secrets not used by any Pod (potential cleanup candidates)
# (Requires custom scripting — example approach)
for ns in $(kubectl get ns -o name | cut -d/ -f2); do
  echo "Namespace: $ns"
  kubectl get secrets -n $ns -o name
done

# Check certificate expiry in TLS secrets
for secret in $(kubectl get secrets --all-namespaces \
  --field-selector type=kubernetes.io/tls \
  -o json | jq -r '.items[] | "\(.metadata.namespace)/\(.metadata.name)"'); do
  ns=$(echo $secret | cut -d/ -f1)
  name=$(echo $secret | cut -d/ -f2)
  kubectl get secret $name -n $ns \
    -o jsonpath='{.data.tls\.crt}' | \
    base64 -d | \
    openssl x509 -noout -dates 2>/dev/null && \
    echo "  → $secret"
done
```

### 14.4 Certificate Exploration

```bash
# Control plane certificates
kubeadm certs check-expiration

# API server certificate details
openssl x509 -in /etc/kubernetes/pki/apiserver.crt \
  -noout -text | grep -E "Subject:|Validity|DNS:|IP"

# Check all certificates in PKI directory
for cert in /etc/kubernetes/pki/*.crt; do
  echo "=== $cert ==="
  openssl x509 -in $cert -noout -subject -dates 2>/dev/null
done

# kubelet serving certificate
openssl x509 -in /var/lib/kubelet/pki/kubelet.crt \
  -noout -text | grep -E "Subject:|Validity" 2>/dev/null
```

---

## 15. Exploring Cluster Events and Audit Logs

### 15.1 Events as the Cluster's Log

Events are the primary observability mechanism that doesn't require a dedicated monitoring stack. Every controller action produces an Event.

```bash
# All events cluster-wide (sorted by time)
kubectl get events --all-namespaces \
  --sort-by='.lastTimestamp' | tail -50

# Only Warning events
kubectl get events --all-namespaces \
  --field-selector type=Warning \
  --sort-by='.lastTimestamp'

# Events for a specific resource
kubectl get events -n <namespace> \
  --field-selector involvedObject.name=<pod-or-deployment-name> \
  --sort-by='.lastTimestamp'

# Events from the last 30 minutes
kubectl get events --all-namespaces \
  --sort-by='.lastTimestamp' | \
  awk 'NR==1 || /[0-9]m/'  # Events within minutes

# Event frequency (which resource generates most events?)
kubectl get events --all-namespaces \
  -o json | jq \
  '[.items[] | .involvedObject.name] | 
   group_by(.) | 
   map({resource: .[0], count: length}) | 
   sort_by(.count) | reverse | .[0:10]'

# Watch events live
kubectl get events --all-namespaces \
  --sort-by='.lastTimestamp' -w
```

### 15.2 Audit Log Exploration

```bash
# Check if audit logging is enabled
kubectl get pod kube-apiserver-$(hostname) -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | \
  tr ' ' '\n' | grep audit

# Recent modifications (if audit log path is configured)
tail -100 /var/log/kubernetes/audit.log 2>/dev/null | \
  jq 'select(.verb != "watch" and .verb != "get" and .verb != "list") | 
      {time: .requestReceivedTimestamp, 
       user: .user.username, 
       verb: .verb,
       resource: .objectRef.resource,
       name: .objectRef.name}' 2>/dev/null

# Track Deployment changes
grep '"deployments"' /var/log/kubernetes/audit.log 2>/dev/null | \
  jq 'select(.verb == "create" or .verb == "update" or .verb == "patch")' | \
  head -20
```

---

## 16. Performance Tuning — What to Look For During Exploration

### 16.1 API Server Performance

```bash
# API server request latency histogram
kubectl get --raw /metrics 2>/dev/null | \
  grep 'apiserver_request_duration_seconds_bucket' | \
  grep 'verb="GET"' | tail -10

# Request inflight limits
kubectl get pod kube-apiserver-$(hostname) -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | \
  tr ' ' '\n' | grep 'max-requests-inflight'

# Check if API server is being throttled
kubectl get --raw /metrics | \
  grep 'apiserver_current_inflight_requests' | grep -v "^#"

# Watch cache sizes
kubectl get --raw /metrics | \
  grep 'watch_cache_capacity' | grep -v "^#"
```

### 16.2 etcd Performance

```bash
# etcd latency (critical — should be <10ms p99 for disk fsync)
kubectl exec -n kube-system etcd-$(hostname) -- \
  /bin/sh -c "
    etcdctl endpoint status \
      --endpoints=https://127.0.0.1:2379 \
      --cacert=/etc/kubernetes/pki/etcd/ca.crt \
      --cert=/etc/kubernetes/pki/etcd/server.crt \
      --key=/etc/kubernetes/pki/etcd/server.key \
      --write-out=table
  "

# etcd database size
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl endpoint status \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --write-out=json | jq '.[0].Status.dbSize'

# High fsync latency indicates disk I/O issues
kubectl exec -n kube-system etcd-$(hostname) -- \
  cat /proc/$(pgrep etcd)/io 2>/dev/null
```

### 16.3 Controller Manager Concurrency

```bash
# Current concurrency flags
kubectl get pod kube-controller-manager-$(hostname) -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | \
  tr ',' '\n' | grep 'concurrent'

# Work queue depths indicate if concurrency needs to be increased
curl -sk https://127.0.0.1:10257/metrics | \
  grep 'workqueue_depth' | grep -v "^#" | \
  awk '{print $1": "$2}' | sort -t'"' -k4

# Recommended concurrency settings for different cluster sizes
# < 50 nodes: defaults (5-10 concurrent syncs)
# 50-200 nodes: increase to 10-20 for major controllers
# 200+ nodes: 20-50 concurrent syncs; profile first
```

### 16.4 Node Resource Pressure

```bash
# Nodes with memory pressure
kubectl get nodes -o json | jq \
  '.items[] | 
   select(.status.conditions[] | 
     select(.type == "MemoryPressure" and .status == "True")) | 
   .metadata.name'

# Nodes with disk pressure
kubectl get nodes -o json | jq \
  '.items[] | 
   select(.status.conditions[] | 
     select(.type == "DiskPressure" and .status == "True")) | 
   .metadata.name'

# Node resource headroom
kubectl describe nodes | grep -A 6 "Allocated resources:" | \
  grep -E "cpu|memory"
```

### 16.5 Important Performance Tuning Flags

| Component | Flag | Default | Recommendation |
|---|---|---|---|
| API Server | `--max-requests-inflight` | 400 | 800 for large clusters |
| API Server | `--max-mutating-requests-inflight` | 200 | 400 for large clusters |
| API Server | `--watch-cache-sizes` | auto | `pod#1000,node#500` for large clusters |
| Controller Manager | `--concurrent-deployment-syncs` | 5 | 10-20 for busy clusters |
| Controller Manager | `--concurrent-replicaset-syncs` | 5 | 10-20 |
| Controller Manager | `--concurrent-endpoint-syncs` | 5 | 10-20 |
| Controller Manager | `--concurrent-gc-syncs` | 20 | 20-50 for many objects |
| Scheduler | `--parallelism` | 16 | 32-64 for large clusters |
| kubelet | `--max-pods` | 110 | 200-250 with adequate resources |
| etcd | `--quota-backend-bytes` | 2GB | 8GB for large clusters |

---

## 17. Monitoring & Observability — Metrics, Dashboards, and Alerting

### 17.1 Metrics Endpoints Discovery

```bash
# Discover all metrics endpoints in the cluster
kubectl get svc --all-namespaces | grep -i "prometheus\|metrics\|monitor"

# Control plane metrics (from control plane node — requires auth)
curl -sk https://127.0.0.1:6443/metrics | head -20
curl -sk https://127.0.0.1:10257/metrics | head -20
curl -sk https://127.0.0.1:10259/metrics | head -20

# Node-level metrics (kubelet)
curl -sk https://localhost:10250/metrics/cadvisor | head -20

# kube-proxy metrics (no auth required)
curl -s http://localhost:10249/metrics | head -20
```

### 17.2 Key Prometheus Metrics by Category

**Cluster Health:**

| Metric | Description | Alert Threshold |
|---|---|---|
| `up{job="apiserver"}` | API server reachability | < 1 for > 2 minutes |
| `etcd_server_health_success` | etcd health | < 1 |
| `etcd_server_leader_changes_seen_total` | etcd leader changes | rate > 3/hour |
| `kube_node_status_condition{condition="Ready",status="true"}` | Node ready | < 1 for > 15 min |
| `kube_node_spec_unschedulable` | Cordoned nodes | > 0 |

**API Server:**

| Metric | Description | Alert Threshold |
|---|---|---|
| `apiserver_request_total` | Total requests by code | code="5xx" increasing |
| `apiserver_request_duration_seconds` | Request latency | p99 > 1s |
| `apiserver_current_inflight_requests` | In-flight requests | > 80% of max |
| `etcd_request_duration_seconds` | etcd call latency | p99 > 100ms |

**Controller Manager:**

| Metric | Description | Alert Threshold |
|---|---|---|
| `workqueue_depth` | Controller queue backlog | > 100 for > 5 min |
| `workqueue_retries_total` | Reconciliation retries | increasing rapidly |
| `workqueue_work_duration_seconds` | Reconciliation time | p99 > 10s |
| `controller_runtime_reconcile_errors_total` | Reconcile errors | > 0 sustained |

**Scheduler:**

| Metric | Description | Alert Threshold |
|---|---|---|
| `scheduler_pending_pods` | Unschedulable Pods | > 0 for > 5 min |
| `scheduler_scheduling_attempt_duration_seconds` | Scheduling latency | p99 > 100ms |
| `scheduler_schedule_attempts_total{result="error"}` | Scheduling errors | increasing |

**Workloads:**

| Metric | Description | Alert Threshold |
|---|---|---|
| `kube_deployment_status_replicas_unavailable` | Unavailable replicas | > 0 for > 5 min |
| `kube_pod_container_status_restarts_total` | Container restarts | rate > 1/hour |
| `kube_pod_status_phase{phase="Failed"}` | Failed Pods | > 0 |
| `kube_persistentvolumeclaim_status_phase{phase="Pending"}` | Pending PVCs | > 0 for > 5 min |

### 17.3 Installing and Exploring Prometheus Stack

```bash
# Check if Prometheus is already installed
kubectl get pods --all-namespaces | grep -i "prometheus\|grafana\|alertmanager"

# Install kube-prometheus-stack if needed
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace

# Explore what gets installed
kubectl get pods -n monitoring
kubectl get svc -n monitoring
kubectl get servicemonitors -n monitoring  # Targets for Prometheus scraping
kubectl get prometheusrules -n monitoring  # Alert rules

# Access Grafana
kubectl port-forward svc/kube-prometheus-grafana -n monitoring 3000:80 &
# Open http://localhost:3000 (admin/prom-operator default)

# Access Prometheus UI
kubectl port-forward svc/kube-prometheus-kube-prome-prometheus \
  -n monitoring 9090:9090 &
# Open http://localhost:9090
```

### 17.4 Exploring ServiceMonitors and PodMonitors

```bash
# What is Prometheus scraping?
kubectl get servicemonitors --all-namespaces
kubectl get podmonitors --all-namespaces

# Inspect a ServiceMonitor
kubectl describe servicemonitor <name> -n monitoring

# Check Prometheus scrape targets (via API)
curl -s http://localhost:9090/api/v1/targets | \
  jq '.data.activeTargets[] | 
      {job: .labels.job, 
       health: .health, 
       lastScrape: .lastScrape}' | head -50
```

---

## 18. Troubleshooting with kubectl — Real Commands for Real Problems

### 18.1 Pods Not Created

```bash
# ===== DIAGNOSTIC TREE FOR MISSING PODS =====

# Step 1: Is the controller working?
kubectl get deployment <name> -n <namespace>
kubectl describe deployment <name> -n <namespace> | tail -20

# Step 2: Was a ReplicaSet created?
kubectl get replicasets -n <namespace> -l app=<app-name>

# Step 3: If RS exists, why no Pods?
kubectl describe replicaset <rs-name> -n <namespace>
# Look for: "unable to create pods: pods 'xx' is forbidden"
# Resource quota: "exceeded quota"
# LimitRange: "maximum memory usage per container is"
# Image: "not found"

# Step 4: Check resource quotas
kubectl describe resourcequota -n <namespace>

# Step 5: Check LimitRange
kubectl describe limitrange -n <namespace>

# Step 6: Check node capacity
kubectl describe nodes | grep -A 6 "Allocated resources:"

# Step 7: Is kube-controller-manager running?
kubectl get pod -n kube-system -l component=kube-controller-manager

# Step 8: Controller manager logs
kubectl logs -n kube-system \
  $(kubectl get pod -n kube-system -l component=kube-controller-manager \
  -o name | head -1) | grep -i "error\|fail" | tail -20
```

### 18.2 Deployment Stuck / Rolling Update Frozen

```bash
# ===== DEPLOYMENT STUCK DIAGNOSTICS =====

# Step 1: Check rollout status
kubectl rollout status deployment/<name> -n <namespace> --timeout=30s

# Step 2: Compare old vs new ReplicaSet
kubectl get replicasets -n <namespace> \
  -l app=<app-name> -o wide
# New RS should be scaling up, old scaling down

# Step 3: New Pods not becoming Ready?
kubectl get pods -n <namespace> -l app=<app-name>
# Look for Pods stuck in Init, Pending, ContainerCreating, CrashLoopBackOff

# Step 4: Why are new Pods not Ready?
kubectl describe pod <new-pod-name> -n <namespace>
# Common causes:
# - readinessProbe failing: Check probe configuration and app startup time
# - Image pull failed: Check image name and registry access
# - OOMKilled: Increase memory limits
# - CrashLoopBackOff: App crashing; check logs

# Step 5: Logs from the failing container
kubectl logs <new-pod-name> -n <namespace>
kubectl logs <new-pod-name> -n <namespace> --previous

# Step 6: Check deployment progress deadline
kubectl get deployment <name> -n <namespace> \
  -o jsonpath='{.spec.progressDeadlineSeconds}'
# Default 600s — Deployment marked ProgressDeadlineExceeded after this

# Step 7: Check conditions
kubectl get deployment <name> -n <namespace> \
  -o json | jq '.status.conditions'

# Step 8: Rollback if needed
kubectl rollout undo deployment/<name> -n <namespace>

# Step 9: Force restart
kubectl rollout restart deployment/<name> -n <namespace>
```

### 18.3 Node NotReady

```bash
# ===== NODE NOTREADY DIAGNOSTICS =====

# Step 1: Identify NotReady nodes
kubectl get nodes | grep -v "Ready"
kubectl describe node <node-name> | grep -A 5 "Conditions:"

# Step 2: Get specific condition
kubectl get node <node-name> -o json | jq \
  '.status.conditions[] | select(.type == "Ready")'

# Step 3: Check condition details
kubectl describe node <node-name> | grep -A 3 "Ready.*False\|Ready.*Unknown"

# Step 4: SSH into node and check kubelet
systemctl status kubelet
journalctl -u kubelet -n 100 --no-pager | grep -i "error\|fail\|fatal"

# Common kubelet error messages:
# "failed to connect to apiserver": Network issue or cert expired
# "network plugin not ready": CNI not installed or crashing
# "failed to create pod sandbox": containerd issue
# "node has 0 allocatable cpu": cgroup issue or resource exhaustion

# Step 5: Check containerd
systemctl status containerd
crictl info

# Step 6: Check node's CNI pod
kubectl get pods -n kube-system -o wide | grep <node-name>

# Step 7: Check network connectivity
ping <api-server-ip>
curl -k https://<api-server-ip>:6443/healthz

# Step 8: Check disk space
df -h
# /var and /var/lib/kubelet fill up frequently

# Step 9: Check inotify limits (informer watches)
cat /proc/sys/fs/inotify/max_user_watches
# If too low, kubelet can't watch enough files

# Step 10: Remediation options
kubectl cordon <node-name>              # Prevent new Pods
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data               # Move existing Pods
# Fix the issue on the node
kubectl uncordon <node-name>           # Re-enable scheduling
```

### 18.4 Service Not Routing Traffic

```bash
# Step 1: Does the Service exist?
kubectl get svc <name> -n <namespace>

# Step 2: Does it have Endpoints?
kubectl get endpoints <name> -n <namespace>
# Empty subsets = no ready Pods matching selector

# Step 3: Check selector
kubectl get svc <name> -n <namespace> \
  -o jsonpath='{.spec.selector}'

# Step 4: Do Pods match?
kubectl get pods -n <namespace> \
  -l <key>=<value>

# Step 5: Are matched Pods Ready?
kubectl get pods -n <namespace> \
  -l <key>=<value> -o wide
# READY column must show 1/1 (or N/N)

# Step 6: Test DNS
kubectl run dnstest --image=busybox \
  --restart=Never -it --rm \
  -- nslookup <service-name>.<namespace>.svc.cluster.local

# Step 7: Test direct Pod connectivity
kubectl exec -it <client-pod> -n <namespace> \
  -- wget -O- http://<pod-ip>:<port>

# Step 8: Test via Service ClusterIP
kubectl exec -it <client-pod> -n <namespace> \
  -- wget -O- http://<clusterip>:<port>
```

### 18.5 CrashLoopBackOff

```bash
# Step 1: Current logs
kubectl logs <pod-name> -n <namespace>

# Step 2: Previous container logs (after crash)
kubectl logs <pod-name> -n <namespace> --previous

# Step 3: Container exit code and reason
kubectl get pod <pod-name> -n <namespace> -o json | jq \
  '.status.containerStatuses[] | 
   {name: .name, 
    restarts: .restartCount,
    state: .state,
    lastState: .lastState}'

# Exit code meanings:
# 0: Success (unusual for crash)
# 1: Generic error (application error)
# 137: OOMKilled (out of memory)
# 139: Segmentation fault
# 143: SIGTERM received (graceful shutdown)
# 255: Container exited unexpectedly

# Step 4: Resource limits (OOMKill)
kubectl describe pod <pod-name> -n <namespace> | \
  grep -A 10 "Limits:\|Requests:"

# Step 5: Exec into init container or ephemeral debug container
kubectl debug -it <pod-name> -n <namespace> \
  --image=busybox --target=<container-name>
```

---

## 19. Disaster Recovery Concepts Through the Exploration Lens

### 19.1 What Must Be Protected

During cluster exploration, you identify the critical components whose loss would cause data loss. Understanding this shapes backup strategy.

```bash
# What lives in etcd? (Everything must be backed up from here)
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl get /registry \
  --prefix --keys-only \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  2>/dev/null | \
  sed 's|/registry/||' | \
  cut -d/ -f1 | sort | uniq -c | sort -rn

# Output reveals the resource type distribution in etcd:
# 1543  pods
#  342  deployments
#  287  configmaps
#  201  services
#   98  secrets
#   45  replicasets
#   ...
```

### 19.2 Stateless Controllers — Safe to Restart Anytime

```bash
# The kube-controller-manager is STATELESS
# Its "memory" is entirely in etcd
# Safe to kill and it will reconcile from scratch

# Verify: restart controller manager and observe recovery
kubectl delete pod -n kube-system \
  $(kubectl get pods -n kube-system \
  -l component=kube-controller-manager -o name)

# Controller manager will:
# 1. Restart immediately (static pod, kubelet recreates it)
# 2. Re-list ALL resources from API server
# 3. Compare desired vs actual state for everything
# 4. Reconcile any discrepancies found

# This is why you can restart the controller manager safely at any time
```

### 19.3 etcd — The Only Stateful Component

```bash
# Check etcd backup status
# (This should be automated — if it's not, that's a critical finding)
ls -la /backup/etcd/ 2>/dev/null || echo "No local backup directory found"

# Manual backup
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl snapshot save /tmp/cluster-state-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify backup
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl snapshot status \
  /tmp/cluster-state-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --write-out=table

# Check last backup time (if using automated backup)
kubectl get cronjobs -n kube-system | grep backup
```

### 19.4 Reconstructable vs Non-Reconstructable State

| Component | Reconstructable Without Backup | Notes |
|---|---|---|
| etcd data | ❌ NO — must be backed up | Backup hourly minimum |
| Control plane PKI certs | ❌ NO — back up `/etc/kubernetes/pki/` | Or keep kubeadm config to regenerate |
| Running Pods | ✅ YES — Pods are recreated by controllers | Stateless Pods only |
| Endpoints/EndpointSlices | ✅ YES — rebuilt from Pod/Service state | Ephemeral |
| kube-proxy iptables rules | ✅ YES — rebuilt on kube-proxy restart | Ephemeral |
| Node registrations | ✅ YES — Nodes re-register on kubelet restart | Fast recovery |
| PersistentVolume data | ❌ NO — depends on storage backend | Backup separately |
| ConfigMaps/Secrets | ❌ NO — stored in etcd | Included in etcd backup |

---

## 20. Comparing Cluster Components: kube-apiserver vs kube-scheduler vs kube-controller-manager

### 20.1 Functional Comparison

| Dimension | kube-apiserver | kube-scheduler | kube-controller-manager |
|---|---|---|---|
| **Core Function** | REST API gateway; enforces auth/authz; reads/writes etcd | Assigns Pods to Nodes | Runs all reconciliation loops |
| **Operates On** | All resources | Only unscheduled Pods | Resource-type-specific controllers |
| **State** | Stateless (state in etcd) | Stateless | Stateless |
| **HA Model** | Active-Active (all serve traffic) | Active-Passive (leader election) | Active-Passive (leader election) |
| **Failure Impact** | Total: all kubectl commands fail | Pending Pods not scheduled | Resources not reconciled; no self-healing |
| **Recovery Time** | <30s (static pod restart) | <30s + leader election (~30-60s) | <30s + leader election (~30-60s) |
| **Load Profile** | High (all API traffic flows through it) | Low-Medium (only scheduling events) | Medium (proportional to change rate) |
| **etcd Access** | YES — only component that writes etcd | NO | NO |
| **Cloud Provider** | NO | NO (uses NodeSelector/affinity) | YES (LoadBalancer, PV provisioning) |
| **Metrics Port** | 6443 (`/metrics` with auth) | 10259 | 10257 |
| **Leader Lease** | N/A (Active-Active) | `kube-scheduler` in `kube-system` | `kube-controller-manager` in `kube-system` |

### 20.2 How They Collaborate for Every Pod Lifecycle Event

```
┌─────────────────────────────────────────────────────────────────────┐
│                  POD LIFECYCLE THROUGH ALL COMPONENTS               │
│                                                                     │
│  kubectl apply -f deployment.yaml                                   │
│        │                                                            │
│        ▼                                                            │
│  kube-apiserver ──► validate ──► authorize ──► write to etcd       │
│        │                                                            │
│        ▼ (watch event)                                              │
│  kube-controller-manager (Deployment controller)                   │
│    ──► creates ReplicaSet ──► API server writes to etcd            │
│        │                                                            │
│        ▼ (watch event)                                              │
│  kube-controller-manager (ReplicaSet controller)                   │
│    ──► creates Pod with spec.nodeName="" ──► API server            │
│        │                                                            │
│        ▼ (watch event — unscheduled Pod)                           │
│  kube-scheduler                                                     │
│    ──► runs filter + score plugins                                  │
│    ──► selects best Node                                            │
│    ──► updates Pod.spec.nodeName ──► API server                    │
│        │                                                            │
│        ▼ (watch event — Pod assigned to my node)                   │
│  kubelet (on target Node)                                           │
│    ──► pulls image via containerd                                   │
│    ──► creates Pod sandbox (CNI allocates IP)                      │
│    ──► starts container                                             │
│    ──► runs readinessProbe                                          │
│    ──► updates Pod.status.conditions[Ready]=True ──► API server    │
│        │                                                            │
│        ▼ (watch event — Pod Ready)                                 │
│  kube-controller-manager (EndpointSlice controller)                │
│    ──► adds Pod IP to EndpointSlice ──► API server                 │
│        │                                                            │
│        ▼ (watch event — EndpointSlice updated)                     │
│  kube-proxy (on all Nodes)                                          │
│    ──► updates iptables/IPVS rules                                  │
│    ──► Pod now receives Service traffic                             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 21. ASCII Architecture Diagram — The Full Cluster Map

```
╔════════════════════════════════════════════════════════════════════════════════════╗
║                   KUBERNETES CLUSTER — EXPLORATION MAP                            ║
╠════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                    ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐  ║
║  │                           CONTROL PLANE                                    │  ║
║  │                                                                             │  ║
║  │  ┌──────────────────────────────────────────────────────────────────────┐  │  ║
║  │  │                    kube-apiserver :6443                              │  │  ║
║  │  │                                                                      │  │  ║
║  │  │  ┌───────────────┐  ┌───────────────┐  ┌──────────────────────────┐ │  │  ║
║  │  │  │ Authentication│  │ Authorization │  │  Admission Controllers   │ │  │  ║
║  │  │  │  (certs/OIDC) │  │    (RBAC)     │  │  (PodSecurity, NodeRes.) │ │  │  ║
║  │  │  └───────────────┘  └───────────────┘  └──────────────────────────┘ │  │  ║
║  │  │                              │                                        │  │  ║
║  │  │                              ▼ (only component that touches etcd)    │  │  ║
║  │  │  ┌───────────────────────────────────────────────────────────────┐   │  │  ║
║  │  │  │                    etcd :2379/:2380                           │   │  │  ║
║  │  │  │         /registry/pods  /registry/services  /registry/...    │   │  │  ║
║  │  │  └───────────────────────────────────────────────────────────────┘   │  │  ║
║  │  └──────────────────────────────────────────────────────────────────────┘  │  ║
║  │                      ▲ HTTPS API calls (never etcd direct)                 │  ║
║  │                      │                                                     │  ║
║  │  ┌───────────────────┼────────────────────────────────────────────────┐   │  ║
║  │  │  kube-controller-manager :10257         kube-scheduler :10259      │   │  ║
║  │  │                                                                     │   │  ║
║  │  │  ┌─────────────────────────────────┐  ┌────────────────────────┐  │   │  ║
║  │  │  │     Controllers (Informers)     │  │   Scheduling Plugins   │  │   │  ║
║  │  │  │ ┌──────────┐ ┌──────────────┐  │  │ ┌────────┐ ┌────────┐  │  │   │  ║
║  │  │  │ │ReplicaSet│ │  Deployment  │  │  │ │ Filter │ │ Score  │  │  │   │  ║
║  │  │  │ └──────────┘ └──────────────┘  │  │ └────────┘ └────────┘  │  │   │  ║
║  │  │  │ ┌──────────┐ ┌──────────────┐  │  └────────────────────────┘  │   │  ║
║  │  │  │ │   Node   │ │  StatefulSet │  │                               │   │  ║
║  │  │  │ └──────────┘ └──────────────┘  │  Leader Election:            │   │  ║
║  │  │  │ ┌──────────┐ ┌──────────────┐  │  Lease: kube-scheduler       │   │  ║
║  │  │  │ │ Service  │ │  DaemonSet   │  │                               │   │  ║
║  │  │  │ └──────────┘ └──────────────┘  │                               │   │  ║
║  │  │  │ ┌──────────┐ ┌──────────────┐  │                               │   │  ║
║  │  │  │ │   Job    │ │  GarbageGC   │  │                               │   │  ║
║  │  │  │ └──────────┘ └──────────────┘  │                               │   │  ║
║  │  │  │ Leader Election: kube-ctrl-mgr │                               │   │  ║
║  │  │  └─────────────────────────────────┘                              │   │  ║
║  │  └───────────────────────────────────────────────────────────────────┘   │  ║
║  │                                                                           │  ║
║  │  ┌──────────────────────────────────────────────────────────────────┐    │  ║
║  │  │   CoreDNS :53/:9153  │  cloud-controller-manager (optional)      │    │  ║
║  │  └──────────────────────────────────────────────────────────────────┘    │  ║
║  └─────────────────────────────────────────────────────────────────────────────┘  ║
║                                                                                    ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐  ║
║  │                            WORKER NODES                                    │  ║
║  │                                                                             │  ║
║  │  ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────┐  │  ║
║  │  │     Node 1           │  │     Node 2           │  │     Node 3       │  │  ║
║  │  │                      │  │                      │  │                  │  │  ║
║  │  │  kubelet :10250      │  │  kubelet :10250      │  │  kubelet :10250  │  │  ║
║  │  │  kube-proxy :10249   │  │  kube-proxy :10249   │  │  kube-proxy      │  │  ║
║  │  │  containerd (CRI)    │  │  containerd (CRI)    │  │  containerd      │  │  ║
║  │  │  calico/cilium (CNI) │  │  calico/cilium (CNI) │  │  calico/cilium   │  │  ║
║  │  │                      │  │                      │  │                  │  │  ║
║  │  │  ┌────────────────┐  │  │  ┌────────────────┐  │  │  ┌────────────┐  │  │  ║
║  │  │  │ App Pods       │  │  │  │ App Pods       │  │  │  │ App Pods   │  │  │  ║
║  │  │  │ ┌──┐ ┌──┐ ┌──┐ │  │  │  │ ┌──┐ ┌──┐     │  │  │  │ ┌──┐ ┌──┐ │  │  │  ║
║  │  │  │ │P1│ │P2│ │P3│ │  │  │  │ │P4│ │P5│     │  │  │  │ │P6│ │P7│ │  │  │  ║
║  │  │  │ └──┘ └──┘ └──┘ │  │  │  │ └──┘ └──┘     │  │  │  │ └──┘ └──┘ │  │  │  ║
║  │  │  └────────────────┘  │  │  └────────────────┘  │  │  └────────────┘  │  │  ║
║  │  │                      │  │                      │  │                  │  │  ║
║  │  │  iptables/IPVS rules │  │  iptables/IPVS rules │  │  IPVS rules      │  │  ║
║  │  │  Pod CIDR: 10.244.1.0│  │  Pod CIDR: 10.244.2.0│  │  10.244.3.0     │  │  ║
║  │  └──────────────────────┘  └──────────────────────┘  └──────────────────┘  │  ║
║  └─────────────────────────────────────────────────────────────────────────────┘  ║
║                                                                                    ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐  ║
║  │             EXPLORATION TOOLS & ENTRY POINTS                               │  ║
║  │                                                                             │  ║
║  │  kubectl → :6443 (API server)  │  crictl → CRI socket (on node)            │  ║
║  │  etcdctl → :2379 (control plane only)  │  k9s → API server                 │  ║
║  │  Prometheus → /metrics endpoints  │  Grafana → dashboards                  │  ║
║  │  curl -k https://<api>:6443/healthz  │  journalctl -u kubelet              │  ║
║  └─────────────────────────────────────────────────────────────────────────────┘  ║
╚════════════════════════════════════════════════════════════════════════════════════╝
```

---

## 22. Real-World Production Use Cases for Cluster Exploration

### 22.1 Incident Response — Mapping Blast Radius

**Scenario:** A critical service is down at 2 AM. The on-call engineer needs to understand the scope within minutes.

```bash
# Quick blast radius map
echo "=== Nodes down ==="
kubectl get nodes | grep -v Ready

echo "=== Pods not running ==="
kubectl get pods --all-namespaces | grep -v "Running\|Completed"

echo "=== Services with no endpoints ==="
kubectl get endpoints --all-namespaces | grep -v "<none>" | \
  awk 'NR==1 || $2==""'

echo "=== Recent warning events ==="
kubectl get events --all-namespaces \
  --field-selector type=Warning \
  --sort-by='.lastTimestamp' | tail -30

echo "=== Failing pods ==="
kubectl get pods --all-namespaces | \
  grep -E "CrashLoop|Error|OOMKilled|Evicted"
```

### 22.2 Security Audit — Privilege Escalation Discovery

```bash
# Find over-privileged service accounts
kubectl get clusterrolebindings -o json | jq \
  '.items[] | 
   select(.roleRef.name == "cluster-admin") | 
   {binding: .metadata.name, subjects: .subjects}'

# Find pods running as root
kubectl get pods --all-namespaces -o json | jq \
  '.items[] | 
   select(.spec.containers[].securityContext.runAsUser == 0 or
          .spec.securityContext.runAsUser == 0) |
   {namespace: .metadata.namespace, name: .metadata.name}'

# Find pods with privileged containers
kubectl get pods --all-namespaces -o json | jq \
  '.items[] | 
   select(.spec.containers[] | 
     .securityContext.privileged == true) |
   {namespace: .metadata.namespace, name: .metadata.name}'

# Find pods mounting host paths
kubectl get pods --all-namespaces -o json | jq \
  '.items[] | 
   select(.spec.volumes[] | .hostPath != null) |
   {namespace: .metadata.namespace, 
    name: .metadata.name, 
    hostPaths: [.spec.volumes[] | 
      select(.hostPath != null) | .hostPath.path]}'
```

### 22.3 Capacity Planning

```bash
# Resource utilization across all nodes
kubectl top nodes

# Pods with no resource requests (risk for scheduling instability)
kubectl get pods --all-namespaces -o json | jq \
  '.items[] | 
   select(.spec.containers[] | 
     .resources.requests == null or .resources.requests == {}) |
   {namespace: .metadata.namespace, name: .metadata.name}'

# Namespace resource consumption
kubectl top pods --all-namespaces --sort-by=memory | head -30

# PVC utilization (requires metrics)
kubectl get pvc --all-namespaces -o json | jq \
  '.items[] | 
   {namespace: .metadata.namespace,
    name: .metadata.name,
    requested: .spec.resources.requests.storage,
    storageClass: .spec.storageClassName}'
```

### 22.4 Pre-Migration Inventory

```bash
# Complete cluster inventory for migration
echo "# Cluster Migration Inventory"
echo "## Nodes"
kubectl get nodes -o wide

echo "## Namespaces"
kubectl get namespaces

echo "## Workloads"
kubectl get deployments,statefulsets,daemonsets,jobs,cronjobs \
  --all-namespaces

echo "## Services"
kubectl get services --all-namespaces

echo "## ConfigMaps"
kubectl get configmaps --all-namespaces | grep -v kube-system

echo "## Secrets (names only, not values)"
kubectl get secrets --all-namespaces | grep -v kube-system

echo "## PVCs"
kubectl get pvc --all-namespaces

echo "## Ingresses"
kubectl get ingress --all-namespaces

echo "## Custom Resources"
kubectl get crd

echo "## RBAC"
kubectl get clusterroles,clusterrolebindings --all-namespaces
```

---

## 23. Security Hardening — Practices Revealed Through Exploration

### 23.1 RBAC Hardening

```bash
# Audit: who has write access to all namespaces?
kubectl get clusterrolebindings -o json | jq \
  '.items[] | 
   select(.roleRef.name == "edit" or .roleRef.name == "admin") | 
   {binding: .metadata.name, role: .roleRef.name, subjects: .subjects}'

# Enforce least privilege: create namespace-scoped roles
cat << 'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
EOF

# Test the permission
kubectl auth can-i get pods -n production \
  --as=system:serviceaccount:production:my-app
```

### 23.2 TLS Certificate Hardening

```bash
# Verify all certificates are using strong algorithms
openssl x509 -in /etc/kubernetes/pki/apiserver.crt \
  -noout -text | grep -i "Signature Algorithm"
# Should be: sha256WithRSAEncryption or ecdsa-with-SHA256

# Check minimum TLS version on API server
kubectl get pod kube-apiserver-$(hostname) -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | \
  tr ' ' '\n' | grep 'tls-min-version'
# Should be: --tls-min-version=VersionTLS12 or VersionTLS13

# Check cipher suites
kubectl get pod kube-apiserver-$(hostname) -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | \
  tr ' ' '\n' | grep 'tls-cipher-suites'
```

### 23.3 Service Account Security Audit

```bash
# Find all pods with mounted service account tokens (default behavior)
kubectl get pods --all-namespaces -o json | jq \
  '.items[] | 
   select(.spec.automountServiceAccountToken != false) | 
   {namespace: .metadata.namespace, name: .metadata.name,
    sa: .spec.serviceAccountName}'

# Recommended: disable automount for pods that don't need it
# In pod spec:
# spec:
#   automountServiceAccountToken: false

# Check which service accounts have custom roles vs defaults
kubectl get rolebindings,clusterrolebindings \
  --all-namespaces -o json | jq \
  '.items[] | 
   select(.subjects != null) | 
   select(.subjects[] | .kind == "ServiceAccount") |
   {name: .metadata.name, 
    namespace: .metadata.namespace,
    role: .roleRef.name,
    sa: [.subjects[] | select(.kind == "ServiceAccount") | .name]}'
```

### 23.4 Pod Security Exploration

```bash
# Check namespace Pod Security labels
kubectl get namespaces -o json | jq \
  '.items[] | 
   {name: .metadata.name, 
    enforce: .metadata.labels["pod-security.kubernetes.io/enforce"],
    audit: .metadata.labels["pod-security.kubernetes.io/audit"]}'

# Namespaces without Pod Security enforcement
kubectl get namespaces -o json | jq \
  '.items[] | 
   select(.metadata.labels["pod-security.kubernetes.io/enforce"] == null) | 
   .metadata.name'

# Apply restricted policy to a namespace
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted
```

---

## 24. Best Practices for Production Cluster Exploration

### 24.1 Read-Only First

- **Always start with read-only commands** (`get`, `describe`, `logs`) before any mutation
- Use `--dry-run=client` for any write command during exploration
- In production clusters, use a ServiceAccount with `view` ClusterRole only

### 24.2 Namespace Context

```bash
# Always set the namespace context to avoid cross-namespace accidents
kubectl config set-context --current --namespace=production

# Use kubens for quick switching
kubens production
kubens kube-system

# Always confirm current context before running commands
kubectl config current-context
kubectl config get-contexts
```

### 24.3 Systematic Documentation

```bash
# Capture cluster state snapshot for comparison
kubectl cluster-info dump --output-directory=/tmp/cluster-snapshot-$(date +%Y%m%d)

# Diff two cluster snapshots
diff -r /tmp/cluster-snapshot-20250101 /tmp/cluster-snapshot-20250115

# Export all resources in a namespace
kubectl api-resources --verbs=list --namespaced -o name | \
  while read resource; do
    kubectl get $resource -n production \
      --ignore-not-found -o yaml 2>/dev/null
  done > /tmp/production-namespace-backup.yaml
```

### 24.4 Exploration Efficiency Tools

```bash
# k9s — terminal UI for cluster exploration (highly recommended)
k9s -n production
# Navigation: :pods, :deployments, :nodes, :events

# kubectl aliases
alias k=kubectl
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods --all-namespaces'
alias kgs='kubectl get services'
alias kgn='kubectl get nodes'
alias kdp='kubectl describe pod'
alias kl='kubectl logs'

# kubectx for cluster context switching
kubectx production-cluster
kubectx staging-cluster

# stern for multi-pod log following
stern -n production app=my-app  # Follow all pods with label app=my-app
```

### 24.5 Never Trust Cached Data

```bash
# Disable kubectl caching for critical operations
kubectl get nodes --cache-dir=/dev/null

# Force refresh (bypasses 10-minute default cache)
kubectl get --raw /api/v1/nodes
```

---

## 25. Common Mistakes and Pitfalls

### 25.1 Exploring the Wrong Context

**Mistake:** Running destructive exploration commands against production when staging was intended.

```bash
# ALWAYS verify context first
kubectl config current-context
kubectl config get-contexts

# Set up aliases with context guards
alias k-prod='kubectl --context=production-cluster'
alias k-staging='kubectl --context=staging-cluster'
```

### 25.2 Missing Cluster-Scoped Resources

**Mistake:** Running `kubectl get all -n production` and believing you've seen everything. `get all` omits many important resource types.

```bash
# get all misses: PVCs, ConfigMaps, Secrets, ServiceAccounts, NetworkPolicies, etc.
# Always explore specific resource types for complete coverage

# Better: explicit resource list
kubectl get pods,deployments,statefulsets,services,\
configmaps,secrets,pvc,sa,ingress,networkpolicies \
-n production
```

### 25.3 Not Reading Events First

**Mistake:** Spending time debugging by reading logs before looking at Events, which usually contain the direct error message.

```bash
# Events are the FIRST thing to read for any debugging session
kubectl get events -n <namespace> \
  --sort-by='.lastTimestamp' | tail -20
```

### 25.4 Ignoring kube-system Namespace

Many critical issues originate in `kube-system` (CoreDNS failures, kube-proxy issues, CNI problems) but are only visible there.

```bash
# Include kube-system in all-namespace queries
kubectl get pods --all-namespaces | grep -v Running
```

### 25.5 Not Understanding OwnerReferences

**Mistake:** Manually deleting Pods to "fix" issues, only for them to be immediately recreated by the ReplicaSet controller.

```bash
# Before deleting any resource, understand its owner chain
kubectl get pod <name> -n <namespace> \
  -o jsonpath='{.metadata.ownerReferences}' | jq

# The correct action is usually to delete the highest-level owner (Deployment)
# or scale it to 0
```

### 25.6 Misinterpreting Pending Pods

**Mistake:** Assuming a Pending Pod means a bug, when it may be waiting for a `WaitForFirstConsumer` PVC binding or is correctly respecting pod anti-affinity.

```bash
# Read the scheduling reason carefully
kubectl describe pod <pending-pod> | grep -A 5 "Events:"
```

### 25.7 Over-Relying on kubectl top

`kubectl top` shows **current CPU usage** sampled by metrics-server at a point in time, not average or peak. It requires metrics-server to be running.

```bash
# Verify metrics-server is running
kubectl top nodes || echo "metrics-server not available"

# Use Prometheus for historical/accurate data
```

---

## 26. Hands-On Labs — Guided Exploration Exercises

### Lab 1: Complete Cluster Inventory

```bash
# Objective: Build a complete mental map of a cluster in 10 minutes

# Part 1: Identify cluster version and topology
kubectl version --short
kubectl get nodes -o wide
kubectl cluster-info

# Part 2: Control plane health
kubectl get pods -n kube-system
kubectl get componentstatuses 2>/dev/null || echo "deprecated"

# Part 3: Workload inventory
kubectl get deployments,statefulsets,daemonsets --all-namespaces

# Part 4: Networking
kubectl get services --all-namespaces
kubectl get ingress --all-namespaces 2>/dev/null

# Part 5: Storage
kubectl get pv,pvc --all-namespaces
kubectl get storageclass

# Part 6: Security surface
kubectl get serviceaccounts --all-namespaces | wc -l
kubectl get networkpolicies --all-namespaces | wc -l
kubectl get secrets --all-namespaces | wc -l

# Part 7: Health check
kubectl get pods --all-namespaces | grep -v "Running\|Completed" | \
  grep -v "^NAME"
kubectl get nodes | grep -v " Ready"

# Expected output: A complete picture of what's running, 
# any health issues, and the security posture at a glance
```

### Lab 2: Controller Loop Observation

```bash
# Objective: Watch the controller pattern in real time

# Terminal 1: Watch Pods being managed
kubectl get pods -l app=lab-demo -w &

# Terminal 2: Watch ReplicaSet state
kubectl get replicasets -l app=lab-demo -w &

# Create a Deployment
kubectl create deployment lab-demo --image=nginx --replicas=3

# Observe: 
# - ReplicaSet created instantly
# - 3 Pods created by ReplicaSet controller
# - Pods transition: Pending → ContainerCreating → Running

# Now delete a Pod and watch self-healing
kubectl delete pod \
  $(kubectl get pods -l app=lab-demo -o name | head -1)
# Observe: new Pod immediately created

# Scale up
kubectl scale deployment lab-demo --replicas=5
# Observe: 2 new Pods created

# Perform rolling update
kubectl set image deployment/lab-demo nginx=nginx:1.25
# Observe: new RS created, old RS scaled down, new RS scaled up

# Cleanup
kubectl delete deployment lab-demo
kill %1 %2  # Stop background watches
```

### Lab 3: etcd vs API Server Comparison

```bash
# Objective: Understand the etcd ← API server relationship

# Create a resource
kubectl create configmap exploration-test \
  --from-literal=key1=value1

# Read it via kubectl (goes through API server)
kubectl get configmap exploration-test -o yaml

# Read it directly from etcd (bypasses API server)
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl get \
  /registry/configmaps/default/exploration-test \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  | strings | grep -A 5 "key1"

# Comparison reveals:
# kubectl output: clean, structured YAML
# etcd output: binary-encoded protobuf data with the same underlying information

# Cleanup
kubectl delete configmap exploration-test
```

### Lab 4: Leader Election Observation

```bash
# Objective: Watch leader election happen

# View current leader
kubectl get lease kube-controller-manager -n kube-system \
  -o jsonpath='{.spec.holderIdentity}' && echo

# Watch the lease object
kubectl get lease kube-controller-manager -n kube-system -w &

# (In HA cluster) Delete the current leader pod to trigger failover
# CAUTION: Only do this in a non-production cluster!
LEADER_NODE=$(kubectl get lease kube-controller-manager -n kube-system \
  -o jsonpath='{.spec.holderIdentity}' | cut -d_ -f1)
kubectl delete pod -n kube-system \
  kube-controller-manager-${LEADER_NODE} 2>/dev/null || \
  echo "Cannot delete static pod directly — it will be recreated automatically"

# Observe: lease expires after leaseDurationSeconds
# New leader acquires the lease
# Background watch shows holderIdentity changes

kill %1  # Stop background watch
```

### Lab 5: Security Exploration Audit

```bash
# Objective: Enumerate security surface area of a namespace

TARGET_NS="default"

echo "=== Service Accounts ==="
kubectl get serviceaccounts -n $TARGET_NS

echo "=== RBAC Bindings ==="
kubectl get rolebindings -n $TARGET_NS -o wide

echo "=== Network Policies ==="
kubectl get networkpolicies -n $TARGET_NS
# If empty: all Pod-to-Pod traffic is allowed!

echo "=== Privileged Pods ==="
kubectl get pods -n $TARGET_NS -o json | jq \
  '.items[] | 
   select(.spec.containers[].securityContext.privileged == true) |
   .metadata.name'

echo "=== Pods Without Resource Limits ==="
kubectl get pods -n $TARGET_NS -o json | jq \
  '.items[] | 
   select(.spec.containers[] | .resources.limits == null) |
   .metadata.name'

echo "=== Pods With HostPath Mounts ==="
kubectl get pods -n $TARGET_NS -o json | jq \
  '.items[] | 
   select(.spec.volumes[] | .hostPath != null) |
   {name: .metadata.name, paths: [.spec.volumes[] | .hostPath.path]}'

echo "=== Secrets in Environment Variables ==="
kubectl get pods -n $TARGET_NS -o json | jq \
  '.items[].spec.containers[].env[] | 
   select(.valueFrom.secretKeyRef != null) |
   {secret: .valueFrom.secretKeyRef.name}'
```

---

## 27. Interview Questions — Beginner to Advanced

### Beginner Level

**Q1: What is the first command you run to get an overview of a Kubernetes cluster you've never seen before?**

**A:** `kubectl get nodes -o wide` gives the immediate topology — how many nodes, their roles, versions, IPs, and operating systems. Combined with `kubectl get pods --all-namespaces`, you get a full picture of what's running. A more systematic approach starts with `kubectl cluster-info` for the control plane URL, then `kubectl get nodes`, then `kubectl get pods -n kube-system` to verify control plane health, then `kubectl get all --all-namespaces` for the full workload inventory.

---

**Q2: What does `kubectl describe node <node-name>` tell you that `kubectl get node` doesn't?**

**A:** `describe` reveals the full node state:
- All **conditions** with reason and message (not just the final status)
- **Capacity vs Allocatable** resources (reserved amounts for kube-system and OS)
- **System info** (kernel version, container runtime, kubelet version)
- **Non-terminated Pods** currently running on this node with their resource requests
- **Allocated resources** summary (what % of CPU/memory is committed)
- **Events** — recent events like NodeNotReady, EvictionThresholdMet
- **Taints and annotations** with full values

The `get` command shows only a summary one-liner.

---

**Q3: You run `kubectl get endpoints my-service` and see `<none>`. What does this mean and how do you investigate?**

**A:** `<none>` means the Endpoints object has no backend Pod IPs. The Service selector matches zero Ready Pods. Investigation steps:
1. `kubectl get svc my-service -o jsonpath='{.spec.selector}'` — get the selector labels
2. `kubectl get pods -l <key>=<value>` — find Pods matching those labels
3. If no Pods match: label mismatch between Service selector and Pod labels
4. If Pods exist but aren't in Endpoints: check `kubectl get pods -l <key>=<value>` for READY column — Pods may have failing readiness probes
5. `kubectl describe pod <name>` — check readiness probe events

---

### Intermediate Level

**Q4: Explain how `kubectl get pods` works internally — what happens between the command and the displayed output?**

**A:** 
1. `kubectl` reads `~/.kube/config` to find the API server URL and auth credentials (client certificate or token)
2. `kubectl` constructs an HTTP GET request to `https://<api-server>:6443/api/v1/namespaces/<namespace>/pods`
3. kube-apiserver **authenticates** the request (verifies the certificate or token)
4. kube-apiserver **authorizes** the request via RBAC (can this identity list pods in this namespace?)
5. kube-apiserver reads from etcd at `/registry/pods/<namespace>/` — it may use its watch cache for better performance
6. kube-apiserver **serializes** the response to JSON/YAML
7. `kubectl` receives the response and **formats** it according to the requested output format (table by default)
8. The table output is generated from the resource's `additionalPrinterColumns` definition in the CRD/built-in schema

You can observe all HTTP calls with `kubectl get pods -v=9`.

---

**Q5: How would you use cluster exploration to determine if a cluster is ready for a production workload?**

**A:** A production readiness check via exploration covers:
1. **Control plane health:** All control plane Pods Running, etcd cluster health, certificate expiration dates (`kubeadm certs check-expiration`)
2. **Node readiness:** All nodes Ready, no Pressure conditions, adequate resources
3. **Networking:** CNI running on all nodes, CoreDNS healthy, NetworkPolicy support
4. **Storage:** At least one default StorageClass, PV provisioner running
5. **Security:** No overly-broad ClusterRoleBindings, PodSecurity enforced on namespaces, NetworkPolicy default-deny in place
6. **Monitoring:** metrics-server installed, Prometheus/Grafana deployed, alerts configured
7. **Backup:** etcd backup automation verified
8. **Versions:** Kubernetes version within supported range, no version skew between components

---

**Q6: What is the difference between `kubectl exec` and `crictl exec`? When would you use each?**

**A:** `kubectl exec` routes through the API server → kubelet → container runtime chain. It requires a running Pod and valid kubeconfig with `exec` RBAC permission. It works from anywhere with API server access.

`crictl exec` (run directly on the node) bypasses Kubernetes entirely and talks to the container runtime (containerd/cri-o) via its CRI socket. It works even when the API server is down, the kubelet is unhealthy, or the Pod's Kubernetes metadata is unavailable. Use `crictl` for:
- Debugging node-level container issues when the API server is unreachable
- Inspecting containers that are failing before kubelet can report their status
- Node-level debugging without kubeconfig or RBAC permissions
- Investigating container runtime issues directly

---

### Advanced Level

**Q7: You observe that the kube-controller-manager work queue depth for "deployment" is consistently at 500+. What does this indicate and how would you address it?**

**A:** A consistently high work queue depth means the Deployment controller is receiving new reconciliation requests faster than it can process them. This causes delayed rolling updates, slow scale operations, and slow self-healing.

**Investigation:**
- `curl -sk https://127.0.0.1:10257/metrics | grep 'workqueue_work_duration_seconds'` — check if individual reconciliations are slow (cloud LB timeouts, slow API calls)
- Check if there's a "thundering herd" — many Deployments changing simultaneously (deployment pipeline)
- Check API server latency — if API server is slow, reconciliations take longer
- Check etcd latency — slow etcd cascades to slow API server

**Solutions:**
1. Increase `--concurrent-deployment-syncs` (from 5 to 10-20) in kube-controller-manager flags
2. Profile to find if individual reconciliations are slow vs volume is high
3. If deployment pipeline causes spikes: rate-limit deployments at the CD system level
4. Scale control plane nodes to give kube-controller-manager more CPU
5. If etcd is slow: migrate to faster disk (NVMe), defragment, or reduce object count

---

**Q8: Describe how you would identify all security risks in a Kubernetes cluster through exploration alone (without security scanning tools).**

**A:** Systematic exploration reveals:

**Identity and Access:**
- `cluster-admin` bindings beyond necessary (`kubectl get clusterrolebindings`)
- ServiceAccounts with broad roles (`kubectl auth can-i --list --as=system:serviceaccount:...`)
- Pods using default ServiceAccount with auto-mounted tokens

**Workload Security:**
- Privileged containers (`kubectl get pods --all-namespaces -o json | jq 'select(.spec.containers[].securityContext.privileged == true)'`)
- Containers running as root (UID 0)
- Pods mounting sensitive host paths (`/var/run/docker.sock`, `/etc/kubernetes/`)
- No resource limits (DoS risk)
- No readiness/liveness probes (traffic to unhealthy instances)

**Network Security:**
- Namespaces without NetworkPolicy (unrestricted Pod-to-Pod traffic)
- Services of type LoadBalancer or NodePort exposing internal services
- Ingress without TLS

**Data Security:**
- Secrets in environment variables (vs mounted as files)
- TLS certificates expiring soon
- etcd encryption not enabled

**Configuration:**
- Anonymous authentication not disabled on kubelet
- API server audit logging not enabled
- Pod Security Standards not enforced on production namespaces

---

## 28. Cheat Sheet — Essential Exploration Commands and Flags

### 28.1 Context and Configuration

```bash
kubectl config current-context
kubectl config get-contexts
kubectl config use-context <name>
kubectl config set-context --current --namespace=<ns>
kubectl version --short
```

### 28.2 Cluster Health Quick Check

```bash
# 30-second cluster health check
kubectl get nodes
kubectl get pods -n kube-system
kubectl get pods --all-namespaces | grep -v "Running\|Completed"
kubectl get events --all-namespaces \
  --field-selector type=Warning \
  --sort-by='.lastTimestamp' | tail -20
```

### 28.3 Control Plane Exploration

```bash
kubectl get pods -n kube-system -l tier=control-plane
kubectl logs -n kube-system kube-apiserver-$(hostname) --tail=50
kubectl logs -n kube-system kube-controller-manager-$(hostname) --tail=50
kubectl logs -n kube-system kube-scheduler-$(hostname) --tail=50
kubectl get lease -n kube-system        # Leader election status
kubeadm certs check-expiration          # Certificate expiry
```

### 28.4 Node Exploration

```bash
kubectl get nodes -o wide
kubectl describe node <name>            # Full node detail
kubectl top nodes                       # Resource usage
kubectl cordon <node>                   # Disable scheduling
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>                 # Re-enable scheduling
```

### 28.5 Workload Exploration

```bash
kubectl get all --all-namespaces
kubectl get pods --all-namespaces -o wide
kubectl describe pod <name> -n <ns>
kubectl logs <pod> -n <ns> --tail=100
kubectl logs <pod> -n <ns> --previous    # Crashed container logs
kubectl exec -it <pod> -n <ns> -- /bin/sh
kubectl top pods -n <ns> --sort-by=memory
kubectl rollout status deployment/<name> -n <ns>
kubectl rollout history deployment/<name> -n <ns>
```

### 28.6 Networking Exploration

```bash
kubectl get svc --all-namespaces
kubectl get endpoints --all-namespaces
kubectl get ingress --all-namespaces
kubectl get networkpolicies --all-namespaces
kubectl run dnstest --image=busybox --restart=Never -it --rm -- \
  nslookup <service>.<namespace>.svc.cluster.local
```

### 28.7 Storage Exploration

```bash
kubectl get pv
kubectl get pvc --all-namespaces
kubectl get storageclass
kubectl describe pvc <name> -n <ns>    # Debug pending PVCs
```

### 28.8 RBAC and Security Exploration

```bash
kubectl auth can-i --list --as=<user>
kubectl auth can-i get secrets -n <ns> --as=<user>
kubectl get clusterrolebindings -o wide
kubectl get rolebindings -n <ns> -o wide
kubectl get serviceaccounts --all-namespaces
```

### 28.9 Events and Debugging

```bash
kubectl get events -n <ns> --sort-by='.lastTimestamp'
kubectl get events --field-selector type=Warning --all-namespaces
kubectl describe <resource> <name> -n <ns>
kubectl cluster-info dump --output-directory=/tmp/cluster-dump
```

### 28.10 etcd Operations (Control Plane Node Only)

```bash
# Alias for etcdctl
alias ETCDCTL='etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key'

kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --write-out=table

kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### 28.11 Raw API Exploration

```bash
kubectl get --raw /healthz               # API server health
kubectl get --raw /readyz                # API server readiness
kubectl get --raw /apis                  # All API groups
kubectl get --raw /api/v1               # Core v1 resources
kubectl get pods -v=9 2>&1 | grep GET   # See HTTP calls
kubectl api-resources --verbs=list      # All listable resources
kubectl api-versions                     # All API versions
kubectl explain <resource>.<field>       # Field documentation
```

### 28.12 Important kubectl Flags

| Flag | Description | Example |
|---|---|---|
| `-n <namespace>` | Target namespace | `kubectl get pods -n production` |
| `--all-namespaces` / `-A` | All namespaces | `kubectl get pods -A` |
| `-o wide` | Extended output | `kubectl get nodes -o wide` |
| `-o yaml` | Full YAML output | `kubectl get pod <name> -o yaml` |
| `-o json` | JSON (for jq) | `kubectl get pods -o json \| jq` |
| `-o jsonpath='...'` | Specific field | `-o jsonpath='{.status.phase}'` |
| `-o custom-columns=...` | Custom table | See examples above |
| `-w` / `--watch` | Live updates | `kubectl get pods -w` |
| `--sort-by=` | Sort output | `--sort-by='.lastTimestamp'` |
| `--field-selector` | Filter by field | `--field-selector status.phase=Running` |
| `-l` / `--selector` | Filter by label | `-l app=nginx` |
| `-v=9` | Verbose (HTTP debug) | `kubectl get pods -v=9` |
| `--dry-run=client` | No actual change | `kubectl apply -f ... --dry-run=client` |
| `--show-labels` | Include labels | `kubectl get pods --show-labels` |
| `--as=<user>` | Impersonate user | `kubectl get pods --as=jane` |

---

## 29. Key Takeaways & Summary

### The 10 Principles of Effective Cluster Exploration

1. **Everything is an API object.** If it's in the cluster, it can be read via `kubectl` or the raw API. There are no hidden components.

2. **Start with health, then go deep.** Always confirm the cluster is stable before exploring workloads. A `NotReady` node or a crashing controller manager changes the interpretation of everything you see.

3. **Events are your first diagnostic tool.** `kubectl get events --sort-by='.lastTimestamp'` in the relevant namespace tells you what the controllers are actually doing and why.

4. **The controller pattern explains everything.** Every discrepancy between desired and actual state is the controller loop mid-reconciliation, blocked by an error, or waiting for an external dependency.

5. **Controllers never touch etcd directly.** All cluster communication flows through the API server. Understanding this allows you to trace any operation: who made a change, when, and why.

6. **The API server is the universal lens.** `kubectl get --raw /`, `-v=9` verbose output, and the metrics endpoint reveal everything happening in the cluster's nervous system.

7. **Leader election = one active controller at a time.** When exploring HA clusters, check Lease objects to understand which node is actively reconciling and whether leadership has been stable.

8. **Read before writing.** In production clusters, always use read-only exploration (`get`, `describe`, `logs`) before any mutation. `-v=9` first, then act.

9. **Namespace isolation is not security.** Always verify NetworkPolicies, RBAC bindings, and PodSecurity levels. Namespaces are logical boundaries, not hard security guarantees.

10. **etcd is the only stateful component.** Everything else (controller manager, scheduler, kube-proxy rules, even node registrations) can be reconstructed. Protect etcd with regular backups.

### The Cluster Exploration Mental Framework

```
When something is wrong:
  1. Where is the problem? (kubectl get events)
  2. What should be happening? (kubectl describe <resource>)
  3. What is actually happening? (kubectl logs + crictl on node)
  4. Why is there a gap? (controller loop blocked or resource missing)
  5. How do I close the gap? (fix the root cause, not the symptom)

When onboarding to a new cluster:
  1. Control plane health first
  2. Node inventory and capacity
  3. Namespace structure and workload topology
  4. Networking architecture (CNI, Services, Ingress)
  5. Storage configuration
  6. Security posture (RBAC, PodSecurity, NetworkPolicy)
  7. Monitoring and alerting coverage
  8. Backup and DR verification
```

---

> **This guide covers Kubernetes v1.29+ exploration techniques. Consult the official documentation at https://kubernetes.io/docs and the kubectl reference at https://kubernetes.io/docs/reference/kubectl/ for the most current information.**

---

*End of: Exploring a Kubernetes Cluster — Complete Production-Grade Guide*
