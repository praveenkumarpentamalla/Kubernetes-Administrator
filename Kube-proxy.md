## Table of Contents

1. [Introduction & Strategic Importance](#1-introduction--strategic-importance)
2. [Core Identity Table](#2-core-identity-table)
3. [ASCII Architecture Diagram](#3-ascii-architecture-diagram)
4. [The Controller Pattern: Watch → Compare → Act → Loop](#4-the-controller-pattern-watch--compare--act--loop)
5. [Deep Dive: How Kube-proxy Interacts with Major Controllers](#5-deep-dive-how-kube-proxy-interacts-with-major-controllers)
6. [Kube-proxy Modes: iptables, IPVS, and eBPF](#6-kube-proxy-modes-iptables-ipvs-and-ebpf)
7. [iptables Mode: Deep Internals](#7-iptables-mode-deep-internals)
8. [IPVS Mode: Deep Internals](#8-ipvs-mode-deep-internals)
9. [eBPF Mode and the Future of Kube-proxy](#9-ebpf-mode-and-the-future-of-kube-proxy)
10. [Service Types and Kube-proxy Behavior](#10-service-types-and-kube-proxy-behavior)
11. [Internal Working Concepts: Reconciliation, Informer Pattern & Work Queues](#11-internal-working-concepts-reconciliation-informer-pattern--work-queues)
12. [Leader Election in Kube-proxy Context](#12-leader-election-in-kube-proxy-context)
13. [Interaction with API Server and etcd](#13-interaction-with-api-server-and-etcd)
14. [EndpointSlices: The Modern Backend](#14-endpointslices-the-modern-backend)
15. [DNS and kube-proxy: How They Work Together](#15-dns-and-kube-proxy-how-they-work-together)
16. [Security Hardening Practices](#16-security-hardening-practices)
17. [Performance Tuning & Configuration](#17-performance-tuning--configuration)
18. [Monitoring & Observability](#18-monitoring--observability)
19. [Troubleshooting Guide with Real Commands](#19-troubleshooting-guide-with-real-commands)
20. [Comparison: Kube-proxy vs kube-apiserver vs kube-scheduler](#20-comparison-kube-proxy-vs-kube-apiserver-vs-kube-scheduler)
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

**Kube-proxy** is the network proxy component of Kubernetes that runs on every node in the cluster. It is responsible for implementing the **Service** abstraction — one of Kubernetes's most fundamental concepts. Without kube-proxy (or a replacement like Cilium), Services would be meaningless: their virtual IPs would exist only as objects in etcd with no network routing behind them.

### What Is Kube-proxy?

Kube-proxy is a per-node daemon that:

- Watches the Kubernetes API server for changes to **Service** and **EndpointSlice** (or legacy Endpoints) objects
- Programs the local node's network stack (via iptables, IPVS, or eBPF) to implement the routing rules that make Service virtual IPs (ClusterIPs) functional
- Ensures that traffic sent to a Service's ClusterIP is load-balanced and forwarded to one of the healthy backing Pod IPs
- Handles external traffic via `NodePort` and `LoadBalancer` service types

### The Core Problem Kube-proxy Solves

In Kubernetes, Pods are ephemeral — they come and go, their IP addresses change, and their count scales up and down dynamically. Applications need a stable way to reach a set of Pods providing a service. The **Service** object provides a stable virtual IP (ClusterIP), a stable DNS name, and a stable port. But who actually makes that virtual IP route to the right backend Pods?

That's kube-proxy.

```
Without kube-proxy:
  Client Pod → 10.96.0.100:80 (ClusterIP) → ??? (no rules, traffic dropped)

With kube-proxy:
  Client Pod → 10.96.0.100:80 (ClusterIP)
             → iptables/IPVS DNAT → 10.244.1.5:8080 (Pod A)
                                  → 10.244.2.3:8080 (Pod B)  ← load balanced
                                  → 10.244.3.7:8080 (Pod C)
```

### Why Kube-proxy Matters in Production

- **Every Service call goes through kube-proxy's rules** — slow or broken kube-proxy = broken service mesh for the entire node's workloads.
- **Rule programming latency affects rollout speed** — kube-proxy must update iptables/IPVS rules within seconds of endpoint changes for zero-downtime deployments to work.
- **It scales poorly with naive operation** — iptables mode has O(n) rule complexity; at 10,000+ services, performance degrades significantly without IPVS.
- **It runs as a DaemonSet** — one kube-proxy pod per node, making it a node-level infrastructure concern.
- **Networking CNI plugins like Cilium can replace it** — understanding kube-proxy's role clarifies why eBPF-based replacements are architecturally superior at scale.
- **Security is enforced here** — network policies and firewall rules interact with kube-proxy's iptables chains.

### Evolution of Kube-proxy

```
Kubernetes 1.0:   userspace proxy mode (kube-proxy forwarded packets in userspace)
Kubernetes 1.1:   iptables mode introduced (kernel-space, much faster)
Kubernetes 1.8:   IPVS mode introduced (designed for large-scale)
Kubernetes 1.12:  IPVS mode graduates to stable
Kubernetes 1.19:  EndpointSlices become default (replaces Endpoints)
Kubernetes 1.29:  nftables mode introduced (alpha) — successor to iptables
2019+:            Cilium (eBPF) emerges as kube-proxy replacement
2023+:            Many production clusters running without kube-proxy (Cilium)
```

---

## 2. Core Identity Table

| Field | Value |
|---|---|
| **Binary Name** | `kube-proxy` |
| **Version Command** | `kube-proxy --version` |
| **Deployment Method** | DaemonSet (`kube-proxy` in `kube-system` namespace) |
| **Runs On** | Every node (worker + control plane) |
| **Runs As** | `root` (requires privileged access to iptables/IPVS) |
| **Config File** | `KubeProxyConfiguration` YAML (via ConfigMap) |
| **ConfigMap Name** | `kube-proxy` in `kube-system` namespace |
| **Metrics Port** | `10249` (Prometheus metrics) |
| **Health Check Port** | `10256` (`/healthz` endpoint) |
| **Admin Port** | `10249` (`/configz`, `/metrics`) |
| **Primary Watch Target** | Service, EndpointSlice objects |
| **Talks to API Server** | Yes (HTTPS watch) |
| **Talks to etcd** | Never (only through API server) |
| **Talks to Container Runtime** | Never |
| **Talks to Other Nodes** | Never directly |
| **Programming Target** | Local node's kernel (iptables/IPVS/nftables) |
| **Leader Election** | Not needed (per-node daemon) |
| **Service Account** | `kube-proxy` (in `kube-system`) |
| **RBAC ClusterRole** | `system:node-proxier` |
| **Default Proxy Mode** | `iptables` (most clusters) |
| **Primary Linux Subsystems** | Netfilter (iptables), IPVS, conntrack |
| **Log Location** | `kubectl logs -n kube-system -l k8s-app=kube-proxy` |
| **Key Config Options** | `mode`, `clusterCIDR`, `iptables.masqueradeBit`, `ipvs.scheduler` |

---

## 3. ASCII Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║                     KUBE-PROXY ARCHITECTURE IN KUBERNETES                        ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║                                                                                  ║
║   ┌─────────────────────────── CONTROL PLANE ──────────────────────────────┐   ║
║   │                                                                          │   ║
║   │  ┌──────────────────┐  ┌──────────────┐  ┌────────────────────────┐   │   ║
║   │  │  kube-apiserver  │  │ kube-scheduler│  │ kube-controller-manager│   │   ║
║   │  │    :6443         │  │              │  │                        │   │   ║
║   │  └────────┬─────────┘  └──────────────┘  └────────────────────────┘   │   ║
║   │           │                                                              │   ║
║   │  ┌────────▼──────────────────────────────────────────────────────────┐  │   ║
║   │  │   etcd  ← Service, EndpointSlice objects stored here              │  │   ║
║   │  └───────────────────────────────────────────────────────────────────┘  │   ║
║   └──────────────────────────────────────────────────────────────────────────┘   ║
║                │                                                                  ║
║                │  HTTPS Watch (Services + EndpointSlices)                        ║
║   ┌────────────┼──────────────────────────────────────────────────────────┐     ║
║   │            │          WORKER NODE                                       │     ║
║   │            │                                                            │     ║
║   │   ┌────────▼──────────────────────────────────────────────────────┐   │     ║
║   │   │                      KUBE-PROXY :10249                         │   │     ║
║   │   │                                                                │   │     ║
║   │   │  ┌─────────────┐  ┌───────────────┐  ┌────────────────────┐  │   │     ║
║   │   │  │ ServiceWatch│  │EndpointSlice  │  │  Proxier Engine    │  │   │     ║
║   │   │  │  (Informer) │  │Watch (Informer│  │  (iptables/IPVS/   │  │   │     ║
║   │   │  │             │  │             ) │  │   nftables/eBPF)   │  │   │     ║
║   │   │  └──────┬──────┘  └───────┬───────┘  └────────┬───────────┘  │   │     ║
║   │   │         │                 │                    │               │   │     ║
║   │   │         └─────────────────┴────────────────────┘               │   │     ║
║   │   │                           │                                     │   │     ║
║   │   │              ┌────────────▼────────────┐                        │   │     ║
║   │   │              │  Sync Rules Loop        │                        │   │     ║
║   │   │              │  (reconcile every Xs)   │                        │   │     ║
║   │   │              └────────────┬────────────┘                        │   │     ║
║   │   └───────────────────────────│────────────────────────────────────┘   │     ║
║   │                               │ programs kernel                         │     ║
║   │   ┌───────────────────────────▼────────────────────────────────────┐   │     ║
║   │   │                    LINUX KERNEL                                  │   │     ║
║   │   │                                                                  │   │     ║
║   │   │   ┌─────────────────────────────────────────────────────────┐   │   │     ║
║   │   │   │  Netfilter / iptables chains:                           │   │   │     ║
║   │   │   │                                                         │   │   │     ║
║   │   │   │  PREROUTING → KUBE-SERVICES → KUBE-SVC-XXXXX           │   │   │     ║
║   │   │   │                             → KUBE-SEP-YYYYY (DNAT)    │   │   │     ║
║   │   │   │                             → KUBE-SEP-ZZZZZ (DNAT)    │   │   │     ║
║   │   │   │                                                         │   │   │     ║
║   │   │   │  OUTPUT → KUBE-SERVICES (same chain for local traffic)  │   │   │     ║
║   │   │   │  POSTROUTING → KUBE-POSTROUTING → MASQUERADE           │   │   │     ║
║   │   │   └─────────────────────────────────────────────────────────┘   │   │     ║
║   │   │                    ─────── OR ──────                             │   │     ║
║   │   │   ┌─────────────────────────────────────────────────────────┐   │   │     ║
║   │   │   │  IPVS virtual servers:                                  │   │   │     ║
║   │   │   │  TCP 10.96.0.100:80 → rr                               │   │   │     ║
║   │   │   │    → 10.244.1.5:8080 (weight 1)                        │   │   │     ║
║   │   │   │    → 10.244.2.3:8080 (weight 1)                        │   │   │     ║
║   │   │   └─────────────────────────────────────────────────────────┘   │   │     ║
║   │   └────────────────────────────────────────────────────────────────┘   │     ║
║   │                                                                          │     ║
║   │   ┌──────────────────────────────────────────────────────────────────┐  │     ║
║   │   │  Pod A (10.244.1.5)    Pod B (10.244.2.3)   Pod C (10.244.3.7)  │  │     ║
║   │   │  ← actual backend Pods that receive load-balanced traffic        │  │     ║
║   │   └──────────────────────────────────────────────────────────────────┘  │     ║
║   └──────────────────────────────────────────────────────────────────────────┘     ║
╚══════════════════════════════════════════════════════════════════════════════════╝

Traffic flow for ClusterIP service:
  Client Pod → dst: 10.96.0.100:80
  → Netfilter PREROUTING → KUBE-SERVICES → KUBE-SVC-HASH
  → DNAT: dst becomes 10.244.2.3:8080 (randomly selected endpoint)
  → Packet routed to Pod B's node via CNI overlay
  → Pod B receives packet on port 8080

Traffic flow for NodePort service:
  External Client → dst: <NodeIP>:30080
  → Netfilter PREROUTING → KUBE-NODEPORTS → KUBE-SVC-HASH
  → DNAT: dst becomes 10.244.1.5:8080
  → MASQUERADE: src becomes Node IP (for return traffic)
  → Pod A receives packet
```

---

## 4. The Controller Pattern: Watch → Compare → Act → Loop

Kube-proxy is fundamentally a **controller** — it watches cluster state, compares it to local node state, and acts to close any gap. However, unlike most Kubernetes controllers that create or delete API objects, kube-proxy's "act" phase programs the Linux kernel's networking subsystems.

### The Kube-proxy Control Loop

```
┌─────────────────────────────────────────────────────────────────────┐
│                  KUBE-PROXY RECONCILIATION LOOP                      │
│                                                                       │
│  ┌──────────────┐      ┌──────────────────────┐      ┌───────────┐  │
│  │    WATCH     │─────►│      COMPARE         │─────►│    ACT    │  │
│  │              │      │                      │      │           │  │
│  │  API Server  │      │ Desired rules        │      │ Update:   │  │
│  │  Services +  │      │ (from Services +     │      │ iptables  │  │
│  │  Endpoint-   │      │  EndpointSlices)     │      │ IPVS      │  │
│  │  Slices      │      │      vs              │      │ nftables  │  │
│  │              │      │ Actual kernel rules  │      │ rules     │  │
│  └──────────────┘      └──────────────────────┘      └─────┬─────┘  │
│          ▲                                                   │        │
│          └───────────────────────────────────────────────────┘        │
│                               LOOP (every syncPeriod seconds)         │
└─────────────────────────────────────────────────────────────────────┘
```

### Phase 1: WATCH — Observe Desired State

Kube-proxy maintains persistent **watch connections** to the API server for:

- `Service` objects (all namespaces) — to know what ClusterIPs, ports, and service types exist
- `EndpointSlice` objects (all namespaces) — to know which Pod IPs back each service
- `Node` objects — to determine its own node IP and topology labels

These watches use the **Informer pattern** — a `List + Watch` mechanism with a local in-memory cache.

```
Events that trigger rule updates:
  ├── Service created / updated / deleted
  ├── EndpointSlice created / updated / deleted
  │    (happens when Pods are created, become ready, or terminate)
  └── Node topology changes (for topology-aware routing)
```

### Phase 2: COMPARE — Desired vs Actual

Kube-proxy computes what iptables/IPVS rules *should* exist based on the current Services and EndpointSlices, and compares that to what currently exists in the kernel.

```
Desired state (from API server cache):
  Service: my-svc, ClusterIP: 10.96.0.100, Port: 80→8080
  EndpointSlice: [10.244.1.5:8080 (ready), 10.244.2.3:8080 (ready)]

Actual state (from iptables-save / ipvsadm -ln):
  KUBE-SVC-HASH chain: 2 rules (for 2 endpoints)
  KUBE-SEP-AAA DNAT → 10.244.1.5:8080
  KUBE-SEP-BBB DNAT → 10.244.2.3:8080

Diff: No change needed.

After Pod scaling to 3:
  New EndpointSlice: [10.244.1.5:8080, 10.244.2.3:8080, 10.244.3.7:8080]
  Diff: Missing KUBE-SEP for 10.244.3.7:8080 → ACT
```

### Phase 3: ACT — Program the Kernel

Kube-proxy calls the appropriate kernel API to update rules:

| Mode | Act Method | Kernel Subsystem |
|---|---|---|
| `iptables` | `iptables-restore` (atomic bulk write) | Netfilter |
| `ipvs` | `ipvsadm` / netlink | IP Virtual Server |
| `nftables` | `nft` atomic ruleset update | Netfilter (new) |
| `userspace` (deprecated) | `net.Listen()` + proxy | Userspace Go code |

### Phase 4: LOOP

The reconciliation loop runs on two triggers:

| Trigger | Description | Default |
|---|---|---|
| **Event-driven** | Watch event from API server (Service/EndpointSlice changed) | Immediate (with rate limiting) |
| **Periodic full sync** | Re-sync all rules regardless of events | `syncPeriod: 30s` |
| **Minimum sync period** | Minimum time between syncs (rate limit) | `minSyncPeriod: 1s` |

```yaml
# KubeProxyConfiguration — sync tuning:
iptables:
  syncPeriod: 30s       # Full sync every 30s (safety net)
  minSyncPeriod: 1s     # Don't sync more often than every 1s
```

---

## 5. Deep Dive: How Kube-proxy Interacts with Major Controllers

Kube-proxy does not run controllers itself — it reacts to the outputs of controllers. Here is how each major built-in controller's actions translate into kube-proxy behavior.

### 5.1 ReplicaSet Controller

```
ReplicaSet Controller:
  → Creates/deletes Pods to match .spec.replicas

Impact on kube-proxy:
  → New Pod starts → gets IP assigned by CNI
  → Pod becomes Ready → EndpointSlice controller adds Pod IP to EndpointSlice
  → kube-proxy watches EndpointSlice change
  → kube-proxy adds DNAT rule for new Pod IP to KUBE-SVC chain

Example: Scale from 2 → 3 replicas:
  1. New Pod created: 10.244.3.7
  2. EndpointSlice updated: [10.244.1.5, 10.244.2.3, 10.244.3.7]
  3. kube-proxy receives EndpointSlice watch event
  4. kube-proxy adds KUBE-SEP-NEW chain + DNAT rule → 10.244.3.7:8080
  5. Traffic is now load-balanced across 3 Pods
```

### 5.2 Deployment Controller

```
Deployment Controller (rolling update):
  → Scales up new ReplicaSet, scales down old

Impact on kube-proxy during rolling update:
  Phase 1: New Pod created, becomes Ready
    → EndpointSlice adds new Pod IP
    → kube-proxy adds new DNAT rule
    → Service now routes to BOTH old and new Pods ✅

  Phase 2: Old Pod terminated (SIGTERM → grace period → deleted)
    → Pod removed from EndpointSlice (when not Ready anymore)
    → kube-proxy REMOVES DNAT rule for old Pod
    → No traffic sent to terminating Pod ✅

Zero-downtime achieved when:
  readinessProbe passes before pod is added to endpoints AND
  terminationGracePeriodSeconds is honored before endpoint is removed
```

### 5.3 Node Controller

```
Node Controller:
  → Detects node failure, evicts pods

Impact on kube-proxy:
  → Pods on failed node removed from EndpointSlices
  → kube-proxy removes DNAT rules for those Pod IPs
  → Traffic no longer routed to dead node's Pods
  → New pods scheduled on healthy nodes
  → kube-proxy adds rules for new Pod IPs

Important: kube-proxy on OTHER nodes must also be updated
  → All kube-proxy instances on all nodes update their rules
  → Each watches the same EndpointSlice objects
```

### 5.4 Service Controller (EndpointSlice Controller)

```
EndpointSlice Controller (part of kube-controller-manager):
  → Watches Pods and Services
  → Maintains EndpointSlice objects: maps Service → ready Pod IPs

This is the MOST DIRECT connection to kube-proxy:
  Service created → EndpointSlice created (empty) → kube-proxy creates SVC chain
  Pod becomes Ready → EndpointSlice updated (IP added) → kube-proxy adds DNAT rule
  Pod fails readiness → EndpointSlice updated (IP removed) → kube-proxy removes rule
  Pod terminated → EndpointSlice updated (IP removed) → kube-proxy removes rule
  Service deleted → EndpointSlice deleted → kube-proxy removes entire SVC chain
```

### 5.5 Namespace Controller

```
Namespace Controller:
  → Manages namespace deletion cleanup

Impact on kube-proxy:
  → Namespace deleted → all Services in namespace deleted
  → All EndpointSlices in namespace deleted
  → kube-proxy removes all iptables chains for those Services
  → All KUBE-SVC-* and KUBE-SEP-* chains for that namespace are cleaned up
```

### 5.6 Job Controller

```
Job Controller:
  → Creates one-off or batch Pods

Impact on kube-proxy:
  → If Job Pods are backed by a Service (rare but valid):
    → Job Pod becomes ready → added to EndpointSlice → kube-proxy adds rule
    → Job Pod completes (exit 0) → Pod removed from EndpointSlice → kube-proxy removes rule
  → Most Jobs don't have Services (not client-facing)
  → kube-proxy has minimal interaction with Job Pods
```

### 5.7 StatefulSet Controller

```
StatefulSet Controller:
  → Creates Pods with stable network identities

Impact on kube-proxy:
  StatefulSets use TWO types of Services:

  1. Headless Service (clusterIP: None):
     → No ClusterIP assigned
     → kube-proxy creates NO rules for headless services
     → DNS returns individual Pod IPs directly
     → Traffic goes directly to Pod (no kube-proxy DNAT)

  2. Regular Service (for external access):
     → kube-proxy programs DNAT rules normally
     → All StatefulSet Pods are valid backends

  Ordered startup impact:
     → pod-0 ready → kube-proxy adds rule → traffic can reach pod-0
     → pod-1 ready → kube-proxy adds rule → load balanced to both
     → Each pod's readiness directly controls endpoint membership
```

### 5.8 DaemonSet Controller

```
DaemonSet Controller:
  → Runs one Pod per node

Impact on kube-proxy:
  → kube-proxy ITSELF is a DaemonSet
  → DaemonSet pods for infrastructure (logging, monitoring) may have Services
  → kube-proxy handles these like any other Service

Critical interaction: CNI DaemonSet
  → CNI plugins (Calico, Cilium) run as DaemonSets
  → They set up node networking BEFORE kube-proxy can program rules
  → kube-proxy depends on CNI overlay to route DNAT'd packets to remote Pod IPs
  → kube-proxy: handles Service VIP → Pod IP DNAT (layer 4)
  → CNI: handles routing between nodes for Pod IPs (layer 3)
```

### 5.9 Garbage Collector Controller

```
Garbage Collector:
  → Deletes orphaned resources

Impact on kube-proxy:
  → Orphaned EndpointSlices (owner Service deleted) are garbage collected
  → kube-proxy detects EndpointSlice deletion via watch
  → kube-proxy removes associated iptables/IPVS rules
  → Important: if GC is slow, stale rules may briefly persist
    → This is safe (routes to potentially dead pods)
    → Not dangerous (connection refused quickly)
    → Resolved when GC deletes the EndpointSlice
```

### 5.10 PersistentVolume Controller

```
PersistentVolume Controller:
  → Binds PVCs to PVs, manages storage

Impact on kube-proxy:
  → PV controller has NO direct interaction with kube-proxy
  → Storage is network-transparent to kube-proxy
  → However: StatefulSet pods using PVCs may fail to start if PVC unbound
    → Failed Pod → never becomes Ready → never added to EndpointSlice
    → kube-proxy never adds DNAT rule for that Pod
    → Service traffic routes only to Pods with successfully mounted volumes
```

---

## 6. Kube-proxy Modes: iptables, IPVS, and eBPF

Kube-proxy supports multiple proxy modes, each with different performance characteristics, scalability limits, and operational requirements.

### Mode Comparison Table

| Feature | userspace (deprecated) | iptables | IPVS | nftables (alpha) | eBPF (Cilium) |
|---|---|---|---|---|---|
| **Kernel Subsystem** | None (userspace) | Netfilter | IP Virtual Server | Netfilter (new) | eBPF programs |
| **Performance** | Very Poor | Good | Excellent | Good | Excellent |
| **Scalability** | Very Low | O(n) rules | O(1) hash lookup | Better than iptables | O(1) |
| **Load Balancing Algorithms** | Round-robin | Random (probability) | rr, lc, dh, sh, sed, nq... | Random | Maglev, random, rr |
| **Connection Tracking** | No | Yes (conntrack) | Yes (conntrack) | Yes | Optional (stateless) |
| **Health Check** | Basic | No native | Yes (native) | No native | Yes |
| **Session Affinity** | No | Yes (recent pkts) | Yes (source hash) | Yes | Yes |
| **DSR (Direct Server Return)** | No | No | Yes | No | Yes |
| **Production Ready** | No | Yes | Yes | No (1.29 alpha) | Yes (via Cilium) |
| **Replaces kube-proxy** | No | No | No | No | Yes |
| **Typical Scale** | <10 svcs | <2000 svcs | <10000+ svcs | TBD | Unlimited |

### Checking Current Mode

```bash
# Check kube-proxy mode:
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# Or from a kube-proxy pod:
kubectl logs -n kube-system -l k8s-app=kube-proxy | grep "Using proxy mode"
# Output: "Using proxy mode: iptables" or "Using proxy mode: ipvs"

# Check if running without kube-proxy (Cilium replacement):
kubectl get pods -n kube-system | grep kube-proxy
# No results = kube-proxy replaced by CNI
kubectl get pods -n kube-system | grep cilium
```

---

## 7. iptables Mode: Deep Internals

iptables mode is the default and most widely used mode. It uses the Linux Netfilter framework to program DNAT (Destination Network Address Translation) rules.

### iptables Chain Architecture

```
Netfilter hook points used by kube-proxy:

PREROUTING (incoming packets from network):
  └── KUBE-SERVICES chain
       └── Matches Service ClusterIP:Port
       └── KUBE-SVC-<hash> chain (per service)
            └── KUBE-SEP-<hash> (per endpoint, with probability)
                 └── DNAT to Pod IP:Port

OUTPUT (packets originating from local processes):
  └── KUBE-SERVICES chain (same as above, catches local → ClusterIP)

POSTROUTING:
  └── KUBE-POSTROUTING chain
       └── MASQUERADE (for packets that were DNAT'd and need source NAT)

INPUT:
  └── KUBE-NODEPORTS chain (for NodePort services)
       └── Only matches destination port = NodePort range
```

### Real iptables Rules Example

```bash
# View all kube-proxy created chains:
iptables-save | grep -E "^:KUBE"

# View KUBE-SERVICES chain (entry point for all service traffic):
iptables-save | grep "KUBE-SERVICES"

# Example output for a service "my-svc" with ClusterIP 10.96.0.100:80:
# -A KUBE-SERVICES -d 10.96.0.100/32 -p tcp -m tcp --dport 80 \
#   -j KUBE-SVC-XPGD46QRK7WJZT7O

# View the service chain (load balancing rules):
iptables-save | grep "KUBE-SVC-XPGD46QRK7WJZT7O"
# -A KUBE-SVC-XPGD46QRK7WJZT7O -m statistic --mode random \
#   --probability 0.33333333349 -j KUBE-SEP-ENDPOINT1
# -A KUBE-SVC-XPGD46QRK7WJZT7O -m statistic --mode random \
#   --probability 0.50000000000 -j KUBE-SEP-ENDPOINT2
# -A KUBE-SVC-XPGD46QRK7WJZT7O -j KUBE-SEP-ENDPOINT3

# View endpoint chains (DNAT rules):
iptables-save | grep "KUBE-SEP-ENDPOINT1"
# -A KUBE-SEP-ENDPOINT1 -p tcp -m tcp -j DNAT \
#   --to-destination 10.244.1.5:8080

# View MASQUERADE rule (for return traffic):
iptables-save | grep "KUBE-POSTROUTING"
# -A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic \
#   requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE
```

### Load Balancing Probability Math

iptables mode uses **statistical probability** for load balancing:

```
For N endpoints, each packet is routed with probability 1/N:

3 endpoints:
  Rule 1: probability 1/3 (0.333) → Endpoint 1
  Rule 2: of remaining 2/3, probability 1/2 (0.5) → Endpoint 2  
  Rule 3: catch-all → Endpoint 3

This achieves equal distribution over many connections.
But: it's random, not true round-robin.
Consequence: small number of connections may be unevenly distributed.
```

### iptables Performance Characteristics

```
Rule Count Growth:
  Per service:     ~10-15 iptables rules
  Per endpoint:    ~3-5 iptables rules

Example:
  1,000 services × 10 avg endpoints:
  → 10,000 endpoints × 5 rules = 50,000 rules
  → 1,000 services × 15 rules = 15,000 rules
  → Total: ~65,000 iptables rules

Performance impact of 65,000 rules:
  → Each packet traverses rules linearly (O(n))
  → Rule programming (iptables-restore) takes seconds
  → High rule churn (many endpoints changing) creates CPU spikes
  → Conntrack table fills quickly under high connection rates
```

### iptables-restore: The Atomic Update Mechanism

```bash
# Kube-proxy uses iptables-restore for atomic updates:
# 1. Compute full desired ruleset in memory
# 2. Write to temp file
# 3. Call iptables-restore --noflush (atomic kernel update)

# This prevents partial-state where some rules are updated and others aren't.

# You can observe kube-proxy doing this:
strace -p $(pgrep kube-proxy) -e trace=openat 2>&1 | grep iptables

# Or check with:
watch -n 1 'iptables-save | wc -l'  # Line count should jump atomically
```

### Connection Tracking (conntrack)

iptables mode relies heavily on the Linux **conntrack** module:

```
Connection tracking flow:
  First packet: DNAT applied (10.96.0.100:80 → 10.244.1.5:8080)
  conntrack stores: {src_ip, src_port, dst_ip(original), dst_port} → real_dst

  Subsequent packets: conntrack matches existing entry
  → Same DNAT applied without traversing all iptables rules again
  → Reply packets: Automatic reverse DNAT (10.244.1.5:8080 → 10.96.0.100:80)

This means:
  ✅ Only the FIRST packet of each connection traverses iptables rules
  ✅ Subsequent packets use fast conntrack lookup
  ⚠️ conntrack table has a size limit → if exhausted, connections fail
```

```bash
# Check conntrack table usage:
cat /proc/sys/net/netfilter/nf_conntrack_count   # Current entries
cat /proc/sys/net/netfilter/nf_conntrack_max     # Maximum entries

# If approaching max, increase:
sysctl -w net.netfilter.nf_conntrack_max=1048576
echo "net.netfilter.nf_conntrack_max=1048576" >> /etc/sysctl.conf

# View active connections:
conntrack -L | head -20
conntrack -L | wc -l   # Total tracked connections
```

---

## 8. IPVS Mode: Deep Internals

IPVS (IP Virtual Server) is a Layer 4 load balancer built into the Linux kernel, originally designed for high-performance load balancing. Kube-proxy's IPVS mode uses it as the forwarding engine instead of iptables.

### Why IPVS vs iptables?

```
iptables lookup: O(n) — must traverse rules linearly
IPVS lookup:     O(1) — uses hash tables for service lookup

At 10,000 services:
  iptables: traverse potentially 100,000+ rules per packet
  IPVS:     hash lookup → O(1) regardless of service count

Rule programming:
  iptables-restore with 100,000 rules: several seconds
  IPVS netlink update: milliseconds (incremental)
```

### IPVS Architecture in Kubernetes

```
IPVS Virtual Server per Service:
  ClusterIP:Port → list of Real Servers (Pod IPs)

IPVS uses a hash table:
  {protocol, VIP, port} → Virtual Server entry → Real Server list

Unlike iptables, IPVS has NATIVE health checking and more LB algorithms.
```

### Enabling IPVS Mode

```bash
# Prerequisites: Load required kernel modules:
modprobe ip_vs
modprobe ip_vs_rr
modprobe ip_vs_wrr
modprobe ip_vs_sh
modprobe nf_conntrack

# Verify modules loaded:
lsmod | grep -E "ip_vs|nf_conntrack"

# Make persistent:
cat >> /etc/modules-load.d/ipvs.conf <<EOF
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF
```

```yaml
# KubeProxyConfiguration:
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  scheduler: "rr"              # round-robin (default)
  # Alternatives: wrr, lc, wlc, lblc, dh, sh, sed, nq
  syncPeriod: "30s"
  minSyncPeriod: "1s"
  excludeCIDRs: []             # CIDRs to exclude from IPVS
  strictARP: true              # Required for MetalLB compatibility
  tcpTimeout: "900s"           # TCP session timeout
  tcpFinTimeout: "30s"         # TCP FIN timeout
  udpTimeout: "300s"           # UDP session timeout
```

### IPVS Load Balancing Schedulers

| Scheduler | Name | Best For |
|---|---|---|
| `rr` | Round Robin | Equal load, stateless services |
| `lc` | Least Connection | Long-lived connections, unequal capacity |
| `dh` | Destination Hashing | Cache servers (same client → same server) |
| `sh` | Source Hashing | Session affinity by client IP |
| `wrr` | Weighted Round Robin | Nodes with different capacities |
| `wlc` | Weighted Least Connection | Varied capacity + connection counting |
| `sed` | Shortest Expected Delay | Minimize response time |
| `nq` | Never Queue | Low-latency, never queue if idle server exists |

### Inspecting IPVS Rules

```bash
# Install ipvsadm:
apt-get install ipvsadm   # Debian/Ubuntu
yum install ipvsadm       # RHEL/CentOS

# List all IPVS virtual services:
ipvsadm -ln

# Example output:
# IP Virtual Server version 1.2.1 (size=4096)
# Prot LocalAddress:Port Scheduler Flags
#   -> RemoteAddress:Port  Forward Weight ActiveConn InActConn
# TCP  10.96.0.100:80 rr
#   -> 10.244.1.5:8080     Masq    1      5          0
#   -> 10.244.2.3:8080     Masq    1      3          0
#   -> 10.244.3.7:8080     Masq    1      4          0

# Show stats:
ipvsadm -ln --stats

# Show rates:
ipvsadm -ln --rate

# Show a specific service:
ipvsadm -ln -t 10.96.0.100:80
```

### IPVS + iptables Coexistence

Even in IPVS mode, kube-proxy still creates some iptables rules:

```bash
# IPVS mode still uses iptables for:
iptables-save | grep KUBE | head -20

# 1. MASQUERADE (SNAT for traffic leaving the cluster)
# 2. NodePort handling (IPVS handles the VIP, iptables catches NodePort)
# 3. Network Policy integration hooks
# 4. Hairpin NAT (pod talking to its own service)

# Key: IPVS handles the load balancing (the heavy part)
#      iptables handles edge cases (the light part)
```

### strictARP for IPVS

```bash
# IPVS requires ARP to not respond to VIP queries on other interfaces
# Enable strictARP:
kubectl get configmap kube-proxy -n kube-system -o yaml | \
  sed 's/strictARP: false/strictARP: true/' | \
  kubectl apply -f -

# Or directly in /proc:
sysctl -w net.ipv4.conf.all.arp_ignore=1
sysctl -w net.ipv4.conf.all.arp_announce=2
```

---

## 9. eBPF Mode and the Future of Kube-proxy

eBPF (extended Berkeley Packet Filter) represents the next generation of kernel networking. Cilium, the most popular eBPF-based Kubernetes networking solution, can fully replace kube-proxy.

### Why eBPF Supersedes iptables/IPVS

```
iptables/IPVS flow:
  Packet → NIC → kernel → Netfilter hooks → iptables rules → routing
  (multiple kernel subsystem crossings, conntrack overhead)

eBPF flow:
  Packet → NIC → eBPF program (inline in driver or TC hook) → direct forward
  (single kernel path, no conntrack required for many flows)

Performance advantage:
  → 2-3x faster packet processing at high PPS (packets per second)
  → No conntrack table exhaustion (stateless LB possible)
  → Incremental rule updates (microseconds vs seconds for iptables)
  → DSR (Direct Server Return) without SNAT overhead
```

### Running Without kube-proxy (Cilium)

```bash
# Deploy Cilium with kube-proxy replacement:
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=<API_SERVER_IP> \
  --set k8sServicePort=6443

# Verify kube-proxy replacement:
kubectl -n kube-system exec ds/cilium -- cilium status | grep KubeProxyReplacement
# Output: KubeProxyReplacement: True

# Verify no kube-proxy pods:
kubectl get pods -n kube-system | grep kube-proxy
# No results

# Verify services still work:
kubectl get svc
kubectl exec -it test-pod -- curl http://my-service
```

### nftables Mode (Kubernetes 1.29+, Alpha)

```bash
# nftables is the successor to iptables (same Netfilter, modern API)
# Benefits over iptables:
#   - Atomic ruleset updates (entire ruleset swapped atomically)
#   - Better performance than iptables at scale
#   - Cleaner API
#   - Native sets and maps (more efficient than chains for service lookup)

# Enable nftables mode (alpha):
# /var/lib/kube-proxy/config.conf
mode: nftables

# Verify:
nft list ruleset | grep KUBE
```

---

## 10. Service Types and Kube-proxy Behavior

Different Kubernetes Service types require different kube-proxy behaviors:

### ClusterIP

```
Most common service type. Only accessible within the cluster.

kube-proxy action:
  → Programs KUBE-SERVICES rules matching ClusterIP:Port
  → DNAT to selected endpoint (Pod IP:Port)
  → Works on ALL nodes (every kube-proxy programs the same rules)

iptables rules created:
  KUBE-SERVICES → KUBE-SVC-<hash> → KUBE-SEP-<hash> (DNAT)

Traffic path:
  Pod A → ClusterIP:80 → iptables DNAT → Pod B:8080
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-svc
spec:
  type: ClusterIP           # Default
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

### NodePort

```
Exposes service on each node's IP at a static port (30000-32767 range).

kube-proxy action:
  → ADDITIONALLY programs KUBE-NODEPORTS rules
  → Any traffic to <NodeIP>:NodePort is DNAT'd to a backend Pod
  → MASQUERADE applied so Pod sees Node IP (not original client IP)

iptables rules created:
  KUBE-NODEPORTS → KUBE-SVC-<hash> → KUBE-SEP-<hash> (DNAT)
  KUBE-POSTROUTING → MASQUERADE

Traffic path:
  External client → NodeIP:30080 → iptables DNAT → PodIP:8080 (any node)
  Note: Pod may be on a DIFFERENT node than the one receiving traffic
  (kube-proxy handles cross-node routing via DNAT + CNI overlay)
```

```bash
# View NodePort rules:
iptables-save | grep "KUBE-NODEPORTS"
iptables-save | grep "dport 30080"
```

### LoadBalancer

```
Extends NodePort with an external load balancer (cloud provider).

kube-proxy action:
  → Same as NodePort (programs NodePort rules)
  → Cloud provider's load balancer controller creates external LB
  → LB routes to NodePort on each node
  → kube-proxy handles the NodePort → Pod DNAT

kube-proxy does NOT create the external LB.
The cloud LoadBalancer controller (separate component) does.

ExternalTrafficPolicy:
  Cluster (default): May route to pods on other nodes (extra hop)
  Local:             Only routes to pods on THIS node
    → kube-proxy only creates rules for local pods
    → Benefits: Preserves client IP, avoids extra hop
    → Risk: Uneven load if pods not distributed evenly
```

```yaml
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local   # Preserve source IP, local pods only
```

### ExternalName

```
Maps service to external DNS name. No ClusterIP, no endpoints.

kube-proxy action:
  → Programs NO iptables rules
  → kube-proxy does nothing for ExternalName services
  → DNS resolution handles it: CNAME returned for the service name

kubectl exec pod -- curl http://my-external-svc
→ DNS: my-external-svc.namespace.svc.cluster.local → CNAME → external.example.com
→ Connection goes directly to external service (bypasses kube-proxy entirely)
```

### Headless Service (clusterIP: None)

```
No ClusterIP assigned. Used for StatefulSets and direct Pod addressing.

kube-proxy action:
  → Programs NO iptables rules (no ClusterIP to route)
  → DNS returns individual Pod IPs (A records, not CNAME)
  → Client connects directly to Pod IPs

kube-proxy has ZERO role in headless service traffic.
```

### Service Type Summary

| Service Type | kube-proxy Role | Traffic Entry Point |
|---|---|---|
| ClusterIP | Full DNAT rules | ClusterIP (cluster-internal only) |
| NodePort | ClusterIP + NodePort rules | NodeIP:NodePort (external) |
| LoadBalancer | ClusterIP + NodePort rules | External LB VIP |
| ExternalName | None | External DNS (CNAME) |
| Headless | None | Direct Pod IPs (via DNS) |

---

## 11. Internal Working Concepts: Reconciliation, Informer Pattern & Work Queues

### The Informer Pattern in Kube-proxy

```
┌──────────────────────────────────────────────────────────────────────┐
│               KUBE-PROXY INFORMER ARCHITECTURE                        │
│                                                                        │
│  kube-apiserver                                                        │
│       │                                                                │
│       │  1. List (initial full state of Services + EndpointSlices)   │
│       ▼                                                                │
│  ┌────────────┐                                                        │
│  │  Reflector │◄── 2. Watch (streaming delta events)                  │
│  └─────┬──────┘                                                        │
│         │                                                              │
│  ┌──────▼──────────┐                                                   │
│  │  Delta FIFO     │  ← ADDED, UPDATED, DELETED events                │
│  │    Queue        │                                                    │
│  └──────┬──────────┘                                                   │
│          │                                                              │
│  ┌───────▼──────────┐     ┌───────────────────────────────────────┐   │
│  │   Local Cache    │     │         Event Handler                 │   │
│  │   (Indexer)      │     │  Service added/changed:               │   │
│  │                  │     │    → Mark service as "needs sync"     │   │
│  │  Thread-safe     │     │  EndpointSlice added/changed:         │   │
│  │  in-memory       │     │    → Mark service as "needs sync"     │   │
│  └──────────────────┘     └──────────────────┬────────────────────┘   │
│                                               │                        │
│                                   ┌───────────▼──────────────┐        │
│                                   │ serviceChanges map        │        │
│                                   │ endpointsChanges map      │        │
│                                   │ (pending changes buffer)  │        │
│                                   └───────────┬──────────────┘        │
│                                               │                        │
│                                   ┌───────────▼──────────────┐        │
│                                   │  syncProxyRules()         │        │
│                                   │  (rate-limited)           │        │
│                                   │  → compute full desired   │        │
│                                   │    iptables ruleset       │        │
│                                   │  → iptables-restore       │        │
│                                   └──────────────────────────┘        │
└──────────────────────────────────────────────────────────────────────┘
```

### The Change Tracker (Kube-proxy's Work Queue Equivalent)

Unlike typical controllers that use work queues, kube-proxy uses a **change tracking map** pattern:

```go
// Conceptual: how kube-proxy tracks pending changes

type serviceChange struct {
    previous ServiceMap  // what was there before
    current  ServiceMap  // what is there now
}

serviceChanges := map[types.NamespacedName]*serviceChange{}

// When a Service event arrives:
OnServiceUpdate(oldSvc, newSvc) {
    serviceChanges[key] = &serviceChange{
        previous: oldSvc,
        current:  newSvc,
    }
    // Signal sync is needed (but don't sync immediately — rate limit)
}

// syncProxyRules() (run on timer or trigger):
  1. Consume all pending serviceChanges and endpointsChanges
  2. Compute complete desired state (all services + endpoints)
  3. Diff against current kernel state
  4. Call iptables-restore / ipvsadm with changes
  5. Clear change maps
```

### Rate Limiting and Coalescing

```
Why rate limit?
  → Deployment of 100 pods generates 100 EndpointSlice update events
  → Without rate limiting: 100 iptables-restore calls in rapid succession
  → With rate limiting: coalesce into 1-3 iptables-restore calls

How it works:
  minSyncPeriod: 1s    → Don't sync more than once per second
  syncPeriod: 30s      → Always sync at least every 30s (full reconcile)

  Event arrives → schedule sync after minSyncPeriod delay
  More events arrive during delay → coalesced into same sync
  → Single iptables-restore handles all accumulated changes
```

---

## 12. Leader Election in Kube-proxy Context

### Does Kube-proxy Use Leader Election?

**No.** Kube-proxy does not use leader election. It is a **per-node daemon** — exactly one kube-proxy instance per node. There is no shared state between kube-proxy instances on different nodes; each programs only its own local kernel.

| Component | Leader Election | Reason |
|---|---|---|
| kube-proxy | ❌ No | Per-node; each manages only local kernel rules |
| kube-controller-manager | ✅ Yes | Shared cluster state; only one should act |
| kube-scheduler | ✅ Yes | Shared pod queue; only one should assign |
| etcd | ✅ Yes (Raft) | Distributed consensus |
| kube-apiserver | ❌ No | Stateless, load-balanced |

### Why Each Node Needs Its Own kube-proxy

```
Network rule programming is NODE-LOCAL:

  iptables rules on Node A affect ONLY traffic on Node A
  iptables rules on Node B affect ONLY traffic on Node B

  Node A's kube-proxy:
    → Programs Node A's iptables with ALL service rules
    → Pod on Node A → ClusterIP → hits Node A's iptables → DNAT → any pod IP
    → If backend Pod is on Node B: CNI routes DNAT'd packet to Node B

  Node B's kube-proxy:
    → Programs Node B's iptables with ALL service rules (same rules)
    → Pod on Node B → ClusterIP → hits Node B's iptables → DNAT → any pod IP

All kube-proxy instances program the SAME logical rules (same Services)
but the rules are stored locally in each node's kernel.
```

### Leader Election for Adjacent Components

The **EndpointSlice controller** (in kube-controller-manager) DOES use leader election. If kube-controller-manager's leader dies:

```
Scenario: kube-controller-manager leader crashes during scaling event

1. EndpointSlice update may be delayed (new leader must be elected, ~15s)
2. kube-proxy on all nodes waits for EndpointSlice update
3. During this window: new Pod is Running but NOT in EndpointSlice
4. kube-proxy has NO rule for new Pod → no traffic routed there
5. New leader elected → EndpointSlice updated → kube-proxy updates rules
6. Traffic begins flowing to new Pod

Production impact: Brief period where new pods don't receive traffic during
controller-manager leader election failover.
```

```bash
# Monitor leader election of kube-controller-manager:
kubectl get lease kube-controller-manager -n kube-system -o yaml

# Monitor EndpointSlice controller specifically:
kubectl get lease endpointslicemirror-controller -n kube-system -o yaml
```

---

## 13. Interaction with API Server and etcd

### The Immutable Rule

> **Kube-proxy NEVER communicates with etcd directly. It only talks to the kube-apiserver.**

```
CORRECT:
  etcd ← kube-apiserver ← kube-proxy (watch via HTTPS)

INCORRECT:
  etcd ← kube-proxy   ✗  (never happens)
```

### Kube-proxy ↔ API Server Communication

| Operation | Method | What's Watched/Read |
|---|---|---|
| Watch Services | GET (stream) | `/api/v1/services` |
| Watch EndpointSlices | GET (stream) | `/apis/discovery.k8s.io/v1/endpointslices` |
| Watch Nodes | GET (stream) | `/api/v1/nodes` |
| Get ConfigMap | GET | `kube-proxy` ConfigMap (own config) |
| Post Events | POST | `/api/v1/events` |
| Update lease | PUT | `/apis/coordination.k8s.io/v1/leases` (node lease, if configured) |

### Authentication to API Server

```bash
# Kube-proxy uses a ServiceAccount token:
# ServiceAccount: kube-proxy (in kube-system)
# Token: mounted at /var/run/secrets/kubernetes.io/serviceaccount/token

# OR: uses a kubeconfig file:
# /var/lib/kube-proxy/kubeconfig.conf

# Verify kube-proxy's kubeconfig:
kubectl exec -n kube-system kube-proxy-<node> -- cat /var/lib/kube-proxy/kubeconfig.conf
```

### What Kube-proxy Reads from API Server

```
1. Service objects:
   - .spec.clusterIP (VIP to program)
   - .spec.ports (port mappings)
   - .spec.type (ClusterIP, NodePort, LoadBalancer)
   - .spec.sessionAffinity
   - .spec.externalTrafficPolicy

2. EndpointSlice objects:
   - .endpoints[].addresses (Pod IPs)
   - .endpoints[].conditions.ready (is this endpoint healthy?)
   - .endpoints[].conditions.serving (is this endpoint serving, even if terminating?)
   - .endpoints[].nodeName (for topology-aware routing)
   - .ports[].port (backend port)
   - .ports[].protocol

3. Node objects (own node):
   - .status.addresses (to determine NodePort bind addresses)
   - .metadata.labels (for topology-aware routing labels)
```

### What Kube-proxy Does NOT Read

```
kube-proxy has NO knowledge of:
  - Pod objects directly (learns about pods only via EndpointSlices)
  - Deployment objects
  - ReplicaSet objects
  - ConfigMap/Secret objects (except its own config)
  - PersistentVolume objects
  - Namespace objects (services are watched across all namespaces)
```

### API Server Connection Resilience

```bash
# If API server is temporarily unreachable:
# 1. kube-proxy's watch connection breaks
# 2. Local cache becomes stale (but rules stay in place)
# 3. Existing connections continue working (kernel rules still active)
# 4. New services/endpoints won't be programmed during outage
# 5. kube-proxy reconnects with exponential backoff
# 6. Upon reconnect: Re-list all objects, re-reconcile all rules

# This is safe: kube-proxy rules persist in kernel even if kube-proxy
# process is killed. They only change when kube-proxy actively programs them.
```

---

## 14. EndpointSlices: The Modern Backend

EndpointSlices replaced the legacy Endpoints API in Kubernetes 1.19 and became the default in 1.21. Understanding the difference is critical for troubleshooting.

### Endpoints vs EndpointSlices

| Feature | Endpoints (legacy) | EndpointSlices |
|---|---|---|
| **Max IPs per object** | Unbounded (but etcd 1MB limit) | 100 (configurable) |
| **Scale** | Poor (1 object = all endpoints) | Excellent (sharded) |
| **Update cost** | Writes entire object on any change | Updates only affected slice |
| **Topology info** | No | Yes (nodeName, zone) |
| **Terminating endpoints** | No | Yes (serving but terminating) |
| **Multiple port protocols** | Limited | Full support |
| **Default since** | Kubernetes 1.0 | Kubernetes 1.21 |

### EndpointSlice Sharding

```
Service with 1000 endpoints:
  Legacy Endpoints: 1 object with 1000 IPs (single 1MB etcd write on any change)
  EndpointSlices:   10 objects × 100 IPs each
    → Scale pod from 901 to 902: only 1 EndpointSlice updated
    → API server writes ~1KB instead of ~100KB

kube-proxy receives updates for only changed slices → less work.
```

### Topology-Aware Routing

```yaml
# Service with topology hints:
apiVersion: v1
kind: Service
metadata:
  annotations:
    service.kubernetes.io/topology-mode: Auto  # Enable topology-aware routing
spec:
  selector:
    app: my-app
```

```
How kube-proxy uses topology hints:
  EndpointSlice contains hints:
    endpoint:
      addresses: [10.244.1.5]
      zone: us-east-1a            ← zone from node label
      nodeName: worker-1

  kube-proxy on a node in us-east-1a:
    → Prefers endpoints with hint: us-east-1a
    → Falls back to other zones if no local endpoints

  Benefit: Reduces cross-AZ traffic (cost + latency)
  Requirement: pods distributed evenly across zones
```

```bash
# View EndpointSlices:
kubectl get endpointslices -n <namespace>
kubectl describe endpointslice <name> -n <namespace>

# Check topology hints:
kubectl get endpointslice -o json | jq '.items[].endpoints[].hints'

# Check which pods are "ready" vs "serving" vs "terminating":
kubectl get endpointslice -o json | \
  jq '.items[].endpoints[] | {addr: .addresses, ready: .conditions.ready, serving: .conditions.serving}'
```

### Terminating Endpoints

```
Kubernetes 1.26+: Traffic can be sent to terminating endpoints

Scenario: Pod receives SIGTERM (is terminating):
  .conditions.ready = false    (pod not ready for new connections)
  .conditions.serving = true   (pod can still serve existing connections)
  .conditions.terminating = true

kube-proxy behavior:
  Default: Don't route new traffic to terminating endpoints
  With ProxyTerminatingEndpoints feature gate:
    → If all endpoints are terminating: route to terminating ones (better than error)
    → If healthy endpoints exist: don't route to terminating ones

This enables true graceful shutdown with connection draining.
```

---

## 15. DNS and kube-proxy: How They Work Together

DNS and kube-proxy solve complementary problems in Kubernetes service discovery.

### The Division of Labor

```
DNS (CoreDNS):
  → Resolves service names to ClusterIPs
  → "my-svc.default.svc.cluster.local" → "10.96.0.100"
  → Does NOT handle traffic routing

kube-proxy:
  → Routes traffic from ClusterIP to Pod IPs
  → "10.96.0.100:80" → DNAT → "10.244.1.5:8080"
  → Does NOT handle name resolution

Together:
  Application code: connect("my-svc", port=80)
  DNS: my-svc → 10.96.0.100
  kernel iptables: 10.96.0.100:80 → DNAT → 10.244.1.5:8080
  Application reaches: Pod B's process on port 8080
```

### DNS Resolution Flow

```
Pod A wants to connect to "my-svc":

1. Application calls getaddrinfo("my-svc")
2. /etc/resolv.conf points to CoreDNS:
   nameserver 10.96.0.10   ← CoreDNS ClusterIP
   search default.svc.cluster.local svc.cluster.local cluster.local

3. DNS query: my-svc.default.svc.cluster.local → 10.96.0.100
   Note: CoreDNS traffic itself goes through kube-proxy
   (CoreDNS ClusterIP 10.96.0.10 is itself a Service!)

4. Application: connect(10.96.0.100, 80)
5. iptables DNAT: 10.96.0.100:80 → 10.244.1.5:8080
6. Packet routed to Pod B

Headless service DNS:
  query: my-headless-svc.default.svc.cluster.local
  → Returns: 10.244.1.5, 10.244.2.3, 10.244.3.7 (all pod IPs)
  → Application connects directly to Pod IPs
  → kube-proxy NOT involved
```

### CoreDNS as a Service (Meta-connection)

```
CoreDNS runs as a Deployment backed by a Service:
  Service: kube-dns, ClusterIP: 10.96.0.10, Port: 53

kube-proxy programs rules for kube-dns service!

This creates a circular dependency at bootstrap:
  1. kube-proxy needs API server to get service rules
  2. API server name might need DNS resolution
  3. DNS needs kube-proxy rules to reach CoreDNS pods

Resolution: kube-proxy uses direct IP for API server (not DNS name)
  /var/lib/kube-proxy/kubeconfig.conf:
    server: https://10.0.0.1:6443   ← Direct IP, no DNS
```

---

## 16. Security Hardening Practices

### RBAC Configuration

```yaml
# Kube-proxy's ClusterRole (system:node-proxier):
# View what it can access:
kubectl describe clusterrole system:node-proxier

# Key permissions:
# - get, list, watch: services, endpoints, endpointslices, nodes
# - get, list, watch: events (for posting)
# - get: configmaps (for its own config)
# These are minimal required permissions — no write access to Services/Pods

# Verify ClusterRoleBinding:
kubectl describe clusterrolebinding system:node-proxier
# Should bind system:node-proxier to serviceaccount kube-system:kube-proxy
```

```yaml
# Minimal RBAC for kube-proxy (reference):
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:node-proxier
rules:
- apiGroups: [""]
  resources: ["endpoints", "services", "nodes"]
  verbs: ["list", "watch", "get"]
- apiGroups: ["discovery.k8s.io"]
  resources: ["endpointslices"]
  verbs: ["list", "watch", "get"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch", "update"]
- apiGroups: [""]
  resources: ["configmaps"]
  resourceNames: ["kube-proxy"]
  verbs: ["get"]
```

### TLS Configuration

```bash
# Kube-proxy connects to API server over TLS:
# Verify TLS settings in kubeconfig:
kubectl get configmap kube-proxy -n kube-system -o yaml | grep -A 5 clientConnection

# The kubeconfig specifies:
# - Certificate Authority for API server verification
# - Client certificate/token for kube-proxy's identity

# Verify the kubeconfig kube-proxy uses:
kubectl exec -n kube-system kube-proxy-<id> -- \
  cat /var/lib/kube-proxy/kubeconfig.conf
```

### Securing the Metrics Endpoint

```bash
# kube-proxy exposes metrics on :10249 (default: localhost only)
# In hardened environments, verify metrics aren't exposed externally:

kubectl get configmap kube-proxy -n kube-system -o yaml | grep metricsBindAddress
# Should show: metricsBindAddress: 127.0.0.1:10249  (localhost only)

# NOT: 0.0.0.0:10249  ← exposes to all interfaces
```

### DaemonSet Security Context

```yaml
# kube-proxy DaemonSet should run with these security settings:
spec:
  containers:
  - name: kube-proxy
    securityContext:
      privileged: true    # Required for iptables/IPVS programming
    # Note: kube-proxy MUST be privileged to program kernel rules
    # This is a known security trade-off
    # Mitigation: Use Cilium (eBPF-based) which requires less privilege
```

### Protecting iptables Rules

```bash
# Kube-proxy's iptables rules should not be manually modified:
# If you manually flush iptables on a node, kube-proxy will re-create rules
# But there's a brief window where service routing is broken

# To protect against accidental iptables flushes:
# 1. Never run `iptables -F` on Kubernetes nodes
# 2. Use iptables-legacy vs iptables-nft consistently:
update-alternatives --display iptables
# All tools (iptables, ip6tables, kube-proxy) must use the SAME implementation

# Verify kube-proxy iptables mode matches system:
kubectl logs -n kube-system kube-proxy-<id> | grep "iptables"
iptables --version   # Check mode: legacy or nf_tables
```

### Network Policy Integration

```
Network Policies work ALONGSIDE kube-proxy:
  kube-proxy: handles Service → Pod routing (DNAT)
  NetworkPolicy: handles allow/deny rules for Pod-to-Pod traffic

They use DIFFERENT iptables chains:
  kube-proxy: KUBE-* chains
  Calico/Cilium/WeaveNet: cali-*, cilium-*, WEAVE-* chains

Important: Network policies do NOT restrict kube-proxy's forwarded traffic
without proper CNI integration. The CNI must implement NetworkPolicy.
```

---

## 17. Performance Tuning & Configuration

### Complete KubeProxyConfiguration Reference

```yaml
# Full production KubeProxyConfiguration:
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration

# Proxy mode selection:
mode: "ipvs"         # Recommended for production at scale
                     # Options: iptables, ipvs, nftables (alpha)

# Bind addresses:
bindAddress: "0.0.0.0"
healthzBindAddress: "0.0.0.0:10256"
metricsBindAddress: "127.0.0.1:10249"   # Localhost only for security

# API server connection:
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig.conf"
  contentType: "application/vnd.kubernetes.protobuf"  # Binary for efficiency
  qps: 10              # API server QPS rate
  burst: 20            # API server burst rate
  acceptContentTypes: "application/vnd.kubernetes.protobuf,application/json"

# Cluster network:
clusterCIDR: "10.244.0.0/16"    # Must match Pod network CIDR
                                  # Used to determine if MASQUERADE needed

# iptables mode settings:
iptables:
  masqueradeAll: false           # Only MASQUERADE non-cluster traffic
  masqueradeBit: 14              # iptables mark bit for MASQUERADE
  minSyncPeriod: "1s"            # Minimum sync interval (rate limiter)
  syncPeriod: "30s"              # Maximum sync interval (full reconcile)
  localhostNodePorts: true       # Allow NodePort on localhost

# IPVS mode settings:
ipvs:
  minSyncPeriod: "1s"
  syncPeriod: "30s"
  scheduler: "rr"               # Load balancing algorithm
  excludeCIDRs: []              # Don't touch these CIDRs with IPVS
  strictARP: false              # Set true for MetalLB
  tcpTimeout: "900s"
  tcpFinTimeout: "30s"
  udpTimeout: "300s"

# Node port ranges:
nodePortAddresses: []   # Empty = bind to all interfaces
                        # Or specify: ["192.168.1.0/24"]

# Feature flags:
featureGates:
  TopologyAwareHints: true      # Enable topology-aware routing

# Logging:
logging:
  verbosity: 2          # 0=silent, 2=info, 4=debug, 6=verbose

# Windows-specific (ignore on Linux):
winkernel:
  enableDSR: false
```

### Performance Tuning for Large Clusters

```bash
# Tune conntrack table for high connection rates:
sysctl -w net.netfilter.nf_conntrack_max=1048576
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=86400
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_close_wait=60

# Tune connection tracking hash table:
sysctl -w net.netfilter.nf_conntrack_buckets=262144

# Increase iptables hashlimit for NodePort:
sysctl -w net.ipv4.ip_local_port_range="1024 65535"

# For IPVS mode: tune IPVS connection timeout:
# (already in KubeProxyConfiguration ipvs section)

# Tune kernel networking stack:
sysctl -w net.core.somaxconn=65535
sysctl -w net.ipv4.tcp_max_syn_backlog=65535
```

### Resource Limits for kube-proxy

```yaml
# kube-proxy DaemonSet resource tuning:
resources:
  requests:
    cpu: "100m"       # Baseline for small clusters
    memory: "128Mi"
  limits:
    cpu: "500m"       # Allow bursts during rule syncs
    memory: "512Mi"

# For large clusters (>1000 services):
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"
  limits:
    cpu: "1000m"      # iptables-restore can be CPU-intensive
    memory: "1Gi"
```

### Switching from iptables to IPVS Mode

```bash
# Step 1: Load IPVS kernel modules:
cat >> /etc/modules-load.d/ipvs.conf <<EOF
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
ip_vs_lc
nf_conntrack
EOF
modprobe ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh ip_vs_lc nf_conntrack

# Step 2: Update kube-proxy ConfigMap:
kubectl get configmap kube-proxy -n kube-system -o yaml > kube-proxy-config.yaml
# Edit: mode: "ipvs" and strictARP: true (if using MetalLB)
kubectl apply -f kube-proxy-config.yaml

# Step 3: Restart kube-proxy DaemonSet:
kubectl rollout restart daemonset kube-proxy -n kube-system

# Step 4: Verify IPVS mode active:
kubectl logs -n kube-system -l k8s-app=kube-proxy | grep "Using proxy mode"
# Should show: "Using proxy mode: ipvs"

# Step 5: Verify IPVS rules:
ipvsadm -ln | head -30

# Step 6: Verify services still work:
kubectl get svc
kubectl exec -it test-pod -- curl http://my-service:80
```

---

## 18. Monitoring & Observability

### Kube-proxy Metrics Endpoint

```bash
# Access metrics (from the node):
curl http://localhost:10249/metrics | head -50

# Access from within cluster (via API server proxy):
kubectl get --raw /api/v1/nodes/<node>/proxy/metrics

# Verify metrics endpoint is up:
curl http://localhost:10249/healthz
# Returns: ok

# View current proxy mode config:
curl http://localhost:10249/configz | python3 -m json.tool | grep -i mode
```

### Key Prometheus Metrics

**Sync Performance Metrics:**

| Metric | Type | Description |
|---|---|---|
| `kubeproxy_sync_proxy_rules_duration_seconds` | Histogram | Time to sync iptables/IPVS rules |
| `kubeproxy_sync_proxy_rules_last_queued_timestamp_seconds` | Gauge | When last sync was queued |
| `kubeproxy_sync_proxy_rules_last_timestamp_seconds` | Gauge | When last sync completed |
| `kubeproxy_sync_proxy_rules_no_local_endpoints_total` | Counter | Services with no local endpoints |

**iptables-specific Metrics:**

| Metric | Type | Description |
|---|---|---|
| `kubeproxy_iptables_rules_total` | Gauge | Total iptables rules programmed |
| `kubeproxy_iptables_rules_restore_failures_total` | Counter | iptables-restore failures |
| `kubeproxy_iptables_sync_full_duration_seconds` | Histogram | Full iptables sync duration |
| `kubeproxy_iptables_partial_restore_failures_total` | Counter | Partial restore attempts |

**IPVS-specific Metrics:**

| Metric | Type | Description |
|---|---|---|
| `kubeproxy_ipvs_sync_proxy_rules_duration_seconds` | Histogram | IPVS sync duration |

**Network Metrics (from conntrack):**

| Metric | Type | Description |
|---|---|---|
| `kubeproxy_network_programming_duration_seconds` | Histogram | End-to-end network programming time |

**Error Metrics:**

| Metric | Type | Description |
|---|---|---|
| `kubeproxy_sync_proxy_rules_iptables_restore_failures_total` | Counter | iptables-restore failures |
| `rest_client_requests_total` | Counter | API server requests (watch + get) |
| `rest_client_request_duration_seconds` | Histogram | API server request latency |

### Critical Prometheus Alert Rules

```yaml
groups:
- name: kube-proxy.rules
  rules:

  # kube-proxy pod not running on a node:
  - alert: KubeProxyDown
    expr: absent(up{job="kube-proxy"} == 1)
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "kube-proxy is not running on {{ $labels.node }}"
      description: "Service traffic on this node may be broken"

  # Rule sync taking too long:
  - alert: KubeProxySyncSlow
    expr: |
      histogram_quantile(0.99,
        rate(kubeproxy_sync_proxy_rules_duration_seconds_bucket[5m])
      ) > 10
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "kube-proxy rule sync slow: {{ $value }}s (p99)"
      description: "Service endpoint changes are being applied slowly"

  # iptables restore failures:
  - alert: KubeProxyIptablesRestoreFailures
    expr: increase(kubeproxy_iptables_rules_restore_failures_total[5m]) > 0
    labels:
      severity: critical
    annotations:
      summary: "kube-proxy iptables-restore failing on {{ $labels.node }}"

  # Too many rules (iptables mode warning):
  - alert: KubeProxyTooManyIptablesRules
    expr: kubeproxy_iptables_rules_total > 50000
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "High iptables rule count: {{ $value }} on {{ $labels.node }}"
      description: "Consider switching to IPVS mode for better performance"

  # Conntrack table near full:
  - alert: ConntrackTableFull
    expr: |
      node_nf_conntrack_entries /
      node_nf_conntrack_entries_limit > 0.9
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "conntrack table {{ $value | humanizePercentage }} full"
      description: "New connections will be dropped when table is full"

  # High sync latency (endpoint change not applied quickly):
  - alert: KubeProxyEndpointSyncLatency
    expr: |
      kubeproxy_sync_proxy_rules_last_timestamp_seconds -
      kubeproxy_sync_proxy_rules_last_queued_timestamp_seconds > 30
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "kube-proxy sync lag > 30s on {{ $labels.node }}"
```

### Prometheus Scrape Configuration

```yaml
# prometheus.yml:
scrape_configs:
  - job_name: "kube-proxy"
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - source_labels: [__address__]
        regex: '(.*):10250'
        target_label: __address__
        replacement: '${1}:10249'   # kube-proxy metrics port
      - action: labelmap
        regex: __meta_kubernetes_node_label_(.+)
    scheme: http    # kube-proxy metrics are HTTP (not HTTPS) by default
    # Note: set metricsBindAddress to 0.0.0.0:10249 if scraping from outside node
```

### Observability Stack: Key Dashboards

```bash
# Key metrics to dashboard:

# 1. Rule sync duration over time:
histogram_quantile(0.99, rate(kubeproxy_sync_proxy_rules_duration_seconds_bucket[5m]))

# 2. Number of services tracked:
kubeproxy_iptables_rules_total / 15  # Approximate service count

# 3. Conntrack utilization:
node_nf_conntrack_entries / node_nf_conntrack_entries_limit * 100

# 4. API server watch health:
rate(rest_client_requests_total{job="kube-proxy", code=~"2.."}[5m])

# 5. Rule sync lag:
kubeproxy_sync_proxy_rules_last_timestamp_seconds - kubeproxy_sync_proxy_rules_last_queued_timestamp_seconds
```

---

## 19. Troubleshooting Guide with Real Commands

### Scenario 1: Service Not Reachable (ClusterIP Not Working)

```bash
# Step 1: Verify service exists and has correct selector/ports:
kubectl get svc my-svc -n my-namespace
kubectl describe svc my-svc -n my-namespace
# Check: selector matches pod labels, port/targetPort correct

# Step 2: Verify pods are running AND ready:
kubectl get pods -n my-namespace -l app=my-app
# Check: STATUS=Running, READY=1/1 (not 0/1)

# Step 3: Verify endpoints exist:
kubectl get endpointslices -n my-namespace -l kubernetes.io/service-name=my-svc
# If empty → selector doesn't match pod labels, or pods not ready

# Legacy endpoints (if endpointslices not enabled):
kubectl get endpoints my-svc -n my-namespace
# Should show: my-svc   10.244.1.5:8080,10.244.2.3:8080

# Step 4: Test direct Pod connectivity:
kubectl exec -it test-pod -- curl http://10.244.1.5:8080
# If this fails: problem is with the Pod, not kube-proxy

# Step 5: Test ClusterIP connectivity:
kubectl exec -it test-pod -- curl http://10.96.0.100:80
# If pod-direct works but ClusterIP fails → kube-proxy issue

# Step 6: Check kube-proxy logs on the node where test-pod runs:
# Find node:
kubectl get pod test-pod -o wide
# Check kube-proxy on that node:
kubectl logs -n kube-system -l k8s-app=kube-proxy --field-selector spec.nodeName=<node>

# Step 7: Check iptables rules on the node:
# SSH to node, then:
iptables-save | grep "10.96.0.100"
# Should show KUBE-SERVICES chain matching the ClusterIP

# Step 8: Trace the full chain:
iptables-save | grep "KUBE-SVC-" | head -5
# Get the SVC hash, then:
SVCCHAIN=$(iptables-save | grep "10.96.0.100" | grep -o "KUBE-SVC-[A-Z0-9]*")
iptables-save | grep $SVCCHAIN

# Step 9: For IPVS mode:
ipvsadm -ln | grep -A 5 "10.96.0.100"
# Should show virtual server and real servers
```

### Scenario 2: Pods Not Created / Service Discovery Broken

```bash
# Symptom: New pods added to deployment but Service not routing to them

# Step 1: Check if new pods are in EndpointSlice:
kubectl get endpointslice -n <ns> -l kubernetes.io/service-name=<svc> -o yaml | \
  grep -A 5 "addresses:"

# Step 2: Check pod readiness:
kubectl get pod <new-pod> -o wide
kubectl describe pod <new-pod> | grep -A 10 "Conditions:"

# Step 3: Check kube-proxy sync status:
kubectl logs -n kube-system kube-proxy-<node-id> | tail -20
# Look for: "Syncing iptables rules" or sync errors

# Step 4: Verify kube-proxy is running on all nodes:
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
# All nodes should have a Running kube-proxy pod

# Step 5: Force kube-proxy sync (restart pod):
kubectl delete pod -n kube-system kube-proxy-<node-id>
# DaemonSet will recreate it; kube-proxy will full-sync all rules

# Step 6: Check for kube-proxy crash loops:
kubectl get pod -n kube-system kube-proxy-<id>
# If CrashLoopBackOff:
kubectl logs -n kube-system kube-proxy-<id> --previous
```

### Scenario 3: Node NotReady / kube-proxy Broken on Node

```bash
# Step 1: Check kube-proxy pod status:
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide | grep <node>

# Step 2: Check kube-proxy pod logs:
kubectl logs -n kube-system -l k8s-app=kube-proxy \
  --field-selector spec.nodeName=<node> --tail=100

# Common errors:
# "Failed to execute iptables-restore" → iptables binary issue
# "Failed to list *v1.Service" → API server connectivity issue
# "Error syncing endpoints" → EndpointSlice watch failure
# "ip_vs module not loaded" → IPVS modules missing (if in IPVS mode)

# Step 3: SSH to node and check iptables:
iptables -L KUBE-SERVICES --line-numbers | head -20
# If empty or missing: kube-proxy hasn't programmed rules

# Step 4: Check if iptables is functional:
iptables -L -n | head -5
# If error: iptables installation broken

# Step 5: Verify iptables mode (legacy vs nf_tables):
iptables --version
update-alternatives --list iptables
# Must be consistent with what kube-proxy uses

# Step 6: For IPVS mode — check modules:
lsmod | grep ip_vs
# If empty: modules not loaded
modprobe ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack

# Step 7: Restart kube-proxy:
kubectl delete pod -n kube-system kube-proxy-<id>
# OR on node:
systemctl restart kubelet  # If kube-proxy is managed by kubelet as static pod
```

### Scenario 4: NodePort Not Accessible from External

```bash
# Step 1: Verify NodePort service:
kubectl get svc my-svc
# Check: TYPE=NodePort, PORT(S)=80:30080/TCP

# Step 2: Test from node itself:
curl http://localhost:30080
curl http://<node-ip>:30080

# Step 3: Check iptables NodePort rules:
iptables-save | grep "dport 30080"
# Should show: -A KUBE-NODEPORTS -p tcp --dport 30080 -j KUBE-SVC-...

# Step 4: Check firewall (cloud security groups):
# AWS: Check security group inbound rules allow port 30080
# GCP: Check firewall rules
# On-prem: Check host firewall:
iptables -L INPUT -n | grep 30080
ufw status | grep 30080   # If using ufw

# Step 5: Check externalTrafficPolicy:
kubectl get svc my-svc -o jsonpath='{.spec.externalTrafficPolicy}'
# If "Local": only nodes WITH the pod will respond on NodePort
# Test by hitting a node that runs the pod

# Step 6: Find which nodes have the backing pods:
kubectl get pods -l app=my-app -o wide
# Then test NodePort on those specific nodes

# Step 7: Check kube-proxy nodePortAddresses configuration:
kubectl get configmap kube-proxy -n kube-system -o yaml | grep nodePortAddresses
# If set: kube-proxy only listens on specified IPs for NodePort
```

### Scenario 5: High Latency or Slow Service Calls

```bash
# Step 1: Check sync duration:
curl -s http://localhost:10249/metrics | grep "sync_proxy_rules_duration"
# histogram_quantile to find p99

# Step 2: Count iptables rules (if in iptables mode):
iptables-save | wc -l
# > 50,000 lines → consider switching to IPVS

# Step 3: Check conntrack table:
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max
# If count approaching max: connections being dropped

# Step 4: Check for conntrack errors:
conntrack -S | grep -v "^$"
# Look for: insert_failed, drop, early_drop

# Step 5: Check kube-proxy CPU usage:
kubectl top pods -n kube-system -l k8s-app=kube-proxy
# High CPU → sync is consuming resources (consider IPVS)

# Step 6: Measure endpoint sync latency:
kubectl logs -n kube-system kube-proxy-<id> | grep "Syncing iptables"
# Timestamps show how long syncs take

# Step 7: Check for iptables lock contention:
# Other tools (ufw, firewalld, docker) may hold iptables lock:
fuser /run/xtables.lock
# If kube-proxy is waiting for lock: competing iptables usage
```

### Scenario 6: Session Affinity Not Working

```bash
# Step 1: Verify service has sessionAffinity configured:
kubectl get svc my-svc -o yaml | grep -A 5 "sessionAffinity"
# Should show: sessionAffinity: ClientIP

# Step 2: Check timeout configuration:
kubectl get svc my-svc -o yaml | grep -A 5 "sessionAffinityConfig"
# Default timeout: 10800s (3 hours)

# Step 3: Verify iptables recent module is loaded:
lsmod | grep xt_recent
# kube-proxy uses xt_recent for ClientIP affinity in iptables mode

# Step 4: Check iptables rules for affinity:
iptables-save | grep "recent" | grep <svc-hash>
# Should show --set and --rcheck rules

# Step 5: In IPVS mode, session affinity uses source hashing (sh):
ipvsadm -ln | grep "sh"
# If scheduler is "rr" but you want affinity, change to "sh"
```

---

## 20. Comparison: Kube-proxy vs kube-apiserver vs kube-scheduler

### Architectural Role Comparison

| Dimension | kube-proxy | kube-apiserver | kube-scheduler |
|---|---|---|---|
| **Primary Role** | Service → Pod routing (per node) | Cluster API gateway | Pod → Node assignment |
| **Scope** | Node-local (network rules) | Cluster-wide | Cluster-wide |
| **Instances Per Cluster** | One per node (DaemonSet) | 1-5 (HA, stateless) | 1-5 (HA, leader election) |
| **Leader Election** | ❌ No | ❌ No (stateless) | ✅ Yes |
| **Talks to etcd** | ❌ Never | ✅ Yes (only component) | ❌ Never |
| **Talks to API Server** | ✅ Yes (watch) | Receives requests | ✅ Yes (watch + write) |
| **Programs** | Linux kernel (iptables/IPVS) | etcd | Pod.spec.nodeName |
| **Restart Impact (self)** | Rules persist in kernel; new changes won't apply | Cluster-wide API outage | Pending pods stuck |
| **Restart Impact (cluster)** | Node's services unreachable for new endpoints | All cluster ops fail | No new pod placement |
| **Port** | 10249 (metrics), 10256 (health) | 6443 | 10259 |
| **Authentication** | ServiceAccount / kubeconfig | x509, OIDC, Bearer | TLS cert to API server |
| **State Storage** | None (programs kernel; reads from API) | etcd | Stateless (reads API) |
| **Plugin System** | Mode selection (iptables/IPVS/eBPF) | Admission webhooks | Scheduler plugins |
| **Can be replaced** | ✅ Yes (Cilium, etc.) | ❌ Central to Kubernetes | ✅ Yes (custom schedulers) |
| **Core Dependency** | Must run for Service routing | Must run for everything | Must run for scheduling |

### Interaction Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│              Service Traffic Lifecycle                               │
│                                                                      │
│  1. kubectl apply -f service.yaml                                   │
│             │                                                        │
│             ▼                                                        │
│  2. kube-apiserver: Validate + store Service in etcd               │
│             │                                                        │
│             ▼                                                        │
│  3. kube-proxy on ALL nodes: receives Service watch event           │
│     → All nodes add KUBE-SVC rules for new ClusterIP               │
│             │                                                        │
│  4. kubectl apply -f deployment.yaml                                │
│             │                                                        │
│             ▼                                                        │
│  5. kube-apiserver: Store Deployment → ReplicaSet → Pods (Pending) │
│             │                                                        │
│             ▼                                                        │
│  6. kube-scheduler: Assign Pod → Node                               │
│             │                                                        │
│             ▼                                                        │
│  7. kubelet: Start container, get Pod IP from CNI                   │
│             │                                                        │
│             ▼                                                        │
│  8. Pod passes readinessProbe                                        │
│             │                                                        │
│             ▼                                                        │
│  9. EndpointSlice controller: Add Pod IP to EndpointSlice           │
│             │                                                        │
│             ▼                                                        │
│ 10. kube-proxy on ALL nodes: receives EndpointSlice watch event     │
│     → All nodes add KUBE-SEP DNAT rule for new Pod IP              │
│             │                                                        │
│             ▼                                                        │
│ 11. Client Pod: curl http://my-svc:80                               │
│     DNS: my-svc → ClusterIP 10.96.0.100                            │
│     iptables: 10.96.0.100:80 → DNAT → 10.244.1.5:8080             │
│     → Pod receives request ✅                                       │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 21. Disaster Recovery Concepts

### Kube-proxy is Stateless by Design

Kube-proxy stores NO persistent state of its own. Its kernel rules are derived from:
1. API server (Services + EndpointSlices)
2. Local node kernel (current iptables/IPVS state — programmed by kube-proxy)

```
True source of truth:
  etcd (via API server) → kube-proxy → kernel rules

Kube-proxy recovery:
  1. Restart kube-proxy process
  2. kube-proxy re-lists all Services + EndpointSlices from API server
  3. kube-proxy computes full desired ruleset
  4. kube-proxy calls iptables-restore / ipvsadm with complete ruleset
  5. All rules correctly programmed

No backup of kube-proxy state needed.
```

### What Happens to Rules When kube-proxy Stops?

```
IMPORTANT: When kube-proxy process stops, iptables rules PERSIST.

kube-proxy stops:
  → iptables/IPVS rules remain in kernel
  → Existing connections continue working
  → New connections to ClusterIPs continue working
  → But: endpoint changes are not reflected
    → If a pod dies: old DNAT rule still points to dead IP → connection refused
    → If new pod starts: no DNAT rule added → never receives traffic

Impact severity:
  kube-proxy down briefly (<30s): Minimal impact (existing connections fine)
  kube-proxy down >30s: Endpoint drift accumulates, stale rules cause failures

Recovery: kube-proxy restarts, does full reconcile → rules corrected
```

### Clearing kube-proxy Rules (Manual Recovery)

```bash
# WARNING: This will break all Service traffic briefly

# If kube-proxy rules are corrupted (rare):
# 1. Flush kube-proxy chains:
iptables -t nat -F KUBE-SERVICES
iptables -t nat -F KUBE-NODEPORTS
# (This doesn't break existing connections — conntrack handles those)

# 2. Restart kube-proxy (it will re-create all rules):
kubectl delete pod -n kube-system kube-proxy-<node-id>

# 3. Verify rules are restored:
iptables-save | grep "KUBE-SERVICES" | wc -l
# Should have many rules back within seconds

# For IPVS mode:
ipvsadm --clear    # Clears all IPVS rules (DANGEROUS on live node)
# Then restart kube-proxy to restore
```

### etcd Backup Relationship

```bash
# etcd backup (cluster-level DR):
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# After etcd restore:
# 1. etcd has Services and EndpointSlices from backup time
# 2. kube-proxy watches → receives restored objects
# 3. kube-proxy reconciles kernel rules with restored state
# 4. Service routing reflects state at backup time
# 5. If pods were rescheduled after backup time:
#    → New pod IPs may not be in restored EndpointSlices
#    → Kube-proxy won't route to them
#    → Kubernetes will reconcile: endpoint controller re-syncs EndpointSlices
#    → kube-proxy picks up new EndpointSlices → rules corrected

# kube-proxy does NOT need a separate backup.
```

### Node Failure and kube-proxy Impact

```
Node A fails (kube-proxy on Node A stops):
  1. Pods on Node A become unavailable
  2. EndpointSlice controller removes Pod A IPs from EndpointSlices
  3. kube-proxy on ALL OTHER nodes:
     → Receives EndpointSlice update
     → Removes DNAT rules for Node A's Pod IPs
     → Traffic no longer routed to Node A's pods

kube-proxy on the FAILED node A:
  → Doesn't matter (node is down anyway)
  → When node recovers: kube-proxy restarts, full reconcile
```

---

## 22. Real-World Production Use Cases

### Use Case 1: High-Scale Cluster with IPVS (>1000 Services)

**Scenario**: SaaS platform with 500 microservices, each with 10-50 endpoints.

```bash
# Problem with iptables mode:
# 500 services × 30 avg endpoints × 8 rules = 120,000 iptables rules
# iptables-restore takes 3-4 seconds per sync
# Every endpoint change (pod restart) triggers 3-4 second rule sync
# During sync: CPU spike on all nodes

# Solution: Switch to IPVS mode
# IPVS handles 500 services in microseconds with O(1) lookup
# Endpoint changes: incremental IPVS netlink update (milliseconds)

# Configuration:
mode: "ipvs"
ipvs:
  scheduler: "lc"    # Least connection for uneven pod sizes

# Result:
# Service lookup: O(1) hash vs O(n) linear
# Sync time: <100ms vs 3-4 seconds
# CPU during sync: minimal vs spikes
```

### Use Case 2: Topology-Aware Routing for Multi-AZ Cost Reduction

**Scenario**: Kubernetes on AWS with 3 AZs, cross-AZ traffic costs $0.01/GB.

```yaml
# Enable topology-aware routing:
apiVersion: v1
kind: Service
metadata:
  name: my-api
  annotations:
    service.kubernetes.io/topology-mode: Auto
spec:
  selector:
    app: my-api
  ports:
  - port: 80
    targetPort: 8080
```

```
Result: kube-proxy on nodes in us-east-1a preferentially routes to
pods in us-east-1a. Cross-AZ traffic reduced by ~60%.

Savings: 1TB/day × 60% × $0.01 = $6,000/month cost reduction
```

### Use Case 3: Zero-Downtime Deployments with terminationGracePeriod

```yaml
# Ensure kube-proxy removes endpoints BEFORE pod termination:
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: app
        lifecycle:
          preStop:
            exec:
              command: ["sleep", "10"]  # Give kube-proxy 10s to remove endpoint
```

```
Flow:
  1. Pod marked for deletion (SIGTERM)
  2. Pod condition: Ready=false (immediately)
  3. EndpointSlice controller: removes Pod IP (triggered by Ready=false)
  4. kube-proxy: removes DNAT rule (within minSyncPeriod: 1s)
  5. preStop hook: sleeps 10s (buffer for kube-proxy propagation)
  6. Application processes remaining connections (50s remaining grace)
  7. SIGKILL at 60s

Result: No dropped connections during rolling update
```

### Use Case 4: MetalLB with IPVS Mode (On-Premises LoadBalancer)

```bash
# MetalLB provides LoadBalancer services for bare-metal clusters
# Requires IPVS mode with strictARP: true

# Configure kube-proxy:
kubectl get configmap kube-proxy -n kube-system -o yaml | \
  sed -e "s/strictARP: false/strictARP: true/" | \
  sed -e "s/mode: iptables/mode: ipvs/" | \
  kubectl apply -f -

# Restart kube-proxy:
kubectl rollout restart daemonset kube-proxy -n kube-system

# Install MetalLB:
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.0/config/manifests/metallb-native.yaml

# Configure IP pool:
kubectl apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: production-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.10.100-192.168.10.200
EOF
```

### Use Case 5: Replacing kube-proxy with Cilium at Scale

```bash
# Scenario: Large cluster (>500 nodes) where kube-proxy becomes bottleneck
# iptables rule churn from pod churn is causing high CPU on every node

# Install Cilium as kube-proxy replacement:
helm install cilium cilium/cilium \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=<API_SERVER_IP> \
  --set k8sServicePort=6443 \
  --set loadBalancer.algorithm=maglev   # Consistent hashing LB

# Remove kube-proxy DaemonSet:
kubectl -n kube-system delete ds kube-proxy
kubectl -n kube-system delete cm kube-proxy

# Flush old kube-proxy rules (Cilium will manage them via eBPF):
# (Cilium does this automatically when it starts with kubeProxyReplacement=true)

# Verify:
kubectl -n kube-system exec ds/cilium -- cilium status | grep KubeProxyReplacement
# KubeProxyReplacement: True
```

---

## 23. Best Practices for Production Environments

### Mode Selection

- **< 500 services**: iptables mode is fine, simpler to operate
- **500-2000 services**: IPVS mode strongly recommended
- **> 2000 services or > 200 nodes**: Strongly consider Cilium (eBPF) as kube-proxy replacement
- **Any scale with security requirements**: Cilium provides Network Policy + kube-proxy replacement in one

### Performance

- Set `minSyncPeriod: 1s` and `syncPeriod: 30s` — these defaults are appropriate for most clusters
- In IPVS mode, use `scheduler: lc` (least connection) for services with variable request durations
- Increase conntrack table size before hitting limits: `nf_conntrack_max=1048576`
- Set `serializeImagePulls: false` in kubelet to reduce kube-proxy startup contention

### Reliability

- Monitor `kubeproxy_sync_proxy_rules_duration_seconds` — sync time > 5s is a warning
- Alert on `kubeproxy_iptables_rules_restore_failures_total > 0`
- Alert on `node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 0.8`
- Run kube-proxy with `priorityClassName: system-node-critical` (already default in most distributions)

### Zero-Downtime Deployments

- Always set `readinessProbe` on all pods — endpoints are only added when ready
- Set `preStop: exec: command: ["sleep", "N"]` where N > `minSyncPeriod` (e.g., 5-10s)
- Set `terminationGracePeriodSeconds` > preStop sleep + application drain time
- Use `maxUnavailable: 0` in rolling update strategy for critical services

### Networking

- Keep `clusterCIDR` consistent with your CNI configuration
- Use `nodePortAddresses` to restrict NodePort binding to specific interfaces
- Set `masqueradeAll: false` (default) — only masquerade non-cluster traffic
- For ExternalTrafficPolicy: Local services, ensure pods are evenly distributed

---

## 24. Common Mistakes and Pitfalls

### Mistake 1: iptables vs nftables Mismatch

```bash
# PROBLEM: System uses iptables-nft but kube-proxy uses iptables-legacy
# Symptom: kube-proxy programs rules in iptables-legacy, but packets
#          are processed by nf_tables → rules have no effect

# Check system iptables mode:
iptables --version
# Should show: iptables v1.8.7 (nf_tables) OR v1.8.7 (legacy)

# Check what kube-proxy sees:
kubectl logs -n kube-system kube-proxy-<id> | grep "iptables"

# Fix: Ensure all tools use the same iptables implementation:
update-alternatives --set iptables /usr/sbin/iptables-nft
update-alternatives --set ip6tables /usr/sbin/ip6tables-nft

# Then restart kube-proxy:
kubectl delete pod -n kube-system kube-proxy-<id>
```

### Mistake 2: IPVS Mode Without Kernel Modules

```bash
# PROBLEM: Configure mode: ipvs but don't load kernel modules
# Symptom: kube-proxy falls back to iptables or fails entirely

kubectl logs -n kube-system kube-proxy-<id>
# Error: "can't use ipvs proxier: ipvs required kernel modules are not loaded"

# Fix: Load modules before switching mode:
modprobe ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh ip_vs_lc nf_conntrack

# Make persistent across reboots:
cat > /etc/modules-load.d/ipvs.conf << EOF
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
ip_vs_lc
nf_conntrack
EOF
```

### Mistake 3: Forgetting clusterCIDR Configuration

```yaml
# PROBLEM: clusterCIDR not set in KubeProxyConfiguration
# Symptom: MASQUERADE applied to cluster-internal traffic → source IP lost
#          Intra-cluster communication shows node IP instead of pod IP

# Wrong (missing clusterCIDR):
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
# clusterCIDR: not set!

# Correct:
clusterCIDR: "10.244.0.0/16"    # Must match your Pod CIDR
# kube-proxy uses this to know: "if destination is in clusterCIDR,
# don't MASQUERADE — it can reach it directly via CNI overlay"
```

### Mistake 4: Running iptables Commands on Kubernetes Nodes

```bash
# PROBLEM: Admin runs iptables -F to flush rules during debugging
# Symptom: ALL service traffic broken immediately

# NEVER do this on a Kubernetes node:
iptables -F           # Flushes ALL chains including KUBE-* chains
iptables -t nat -F    # Breaks all Service DNAT

# kube-proxy will eventually fix it (next sync), but there's a gap.

# Safe alternative (just show rules without modifying):
iptables-save | grep KUBE | head -50   # Read-only inspection
```

### Mistake 5: Not Setting readinessProbe (Premature Endpoint Addition)

```yaml
# PROBLEM: Pod added to endpoints before it's actually ready to serve
# Symptom: Connection refused errors during rollouts

# WRONG — no readinessProbe:
spec:
  containers:
  - name: app
    image: myapp:v2
    # No readinessProbe!
    # Pod added to endpoints as soon as container starts (not ready to serve!)

# RIGHT — with readinessProbe:
spec:
  containers:
  - name: app
    image: myapp:v2
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
    # Pod added to endpoints ONLY when /ready returns 200
```

### Mistake 6: ExternalTrafficPolicy: Local with Uneven Pod Distribution

```yaml
# PROBLEM: ExternalTrafficPolicy: Local but pods only on some nodes
# Symptom: NodePort only works when accessing specific nodes

spec:
  externalTrafficPolicy: Local
  # kube-proxy only creates rules for pods LOCAL to each node
  # Nodes without pods: NodePort returns connection refused!

# Fix option 1: Use externalTrafficPolicy: Cluster (default)
# Fix option 2: Use DaemonSet to ensure pod on every node
# Fix option 3: Use topology-aware routing instead
```

### Mistake 7: Conntrack Table Exhaustion

```bash
# PROBLEM: High-connection-rate services exhaust conntrack table
# Symptom: "nf_conntrack: table full, dropping packet" in kernel logs

dmesg | grep -i "conntrack"
# nf_conntrack: nf_conntrack: table full, dropping packet.

# Fix: Increase conntrack table size:
sysctl -w net.netfilter.nf_conntrack_max=1048576
echo "net.netfilter.nf_conntrack_max=1048576" >> /etc/sysctl.conf

# Also tune timeout for short-lived connections:
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=30
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_close_wait=15

# Monitor before it becomes a problem:
# Alert at 80% utilization (see Prometheus alert rules above)
```

### Mistake 8: Using `kubectl exec` Service Tests Without Understanding kube-proxy Scope

```bash
# PROBLEM: Service works from some pods but not others
# Confusion: "kube-proxy should make it work everywhere!"

# Reality check: kube-proxy programs rules on EACH NODE SEPARATELY
# If kube-proxy is broken on ONE node, only pods on that node are affected

# Test from multiple nodes:
kubectl exec -it test-pod-node-a -- curl http://my-svc    # Works?
kubectl exec -it test-pod-node-b -- curl http://my-svc    # Works?

# If one fails: kube-proxy issue on that specific node
# Check: kubectl logs -n kube-system kube-proxy-<node-id>
```

---

## 25. Hands-On Labs & Mini Practical Exercises

### Lab 1: Inspect kube-proxy Rules for a Service

```bash
# Exercise: Trace a service's complete iptables rule chain

# Step 1: Create a test service:
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: lab-svc
spec:
  selector:
    app: lab-app
  ports:
  - port: 80
    targetPort: 8080
EOF

kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lab-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: lab-app
  template:
    metadata:
      labels:
        app: lab-app
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
EOF

# Step 2: Get the ClusterIP:
CLUSTERIP=$(kubectl get svc lab-svc -o jsonpath='{.spec.clusterIP}')
echo "ClusterIP: $CLUSTERIP"

# Step 3: Find the service's iptables chain:
# SSH to any node, then:
SVCCHAIN=$(iptables-save | grep "$CLUSTERIP" | grep -o "KUBE-SVC-[A-Z0-9]*")
echo "Service chain: $SVCCHAIN"

# Step 4: View the service chain (load balancing rules):
iptables-save | grep "$SVCCHAIN"

# Step 5: View endpoint chains (DNAT rules):
iptables-save | grep "KUBE-SEP-" | grep -A 2 "DNAT"

# Step 6: Verify endpoints match pod IPs:
kubectl get endpointslices -l kubernetes.io/service-name=lab-svc
kubectl get pods -l app=lab-app -o wide

# Step 7: Scale and watch rules update:
kubectl scale deployment lab-app --replicas=5
sleep 5
iptables-save | grep "$SVCCHAIN"
# Count: should now show 5 probability rules

# Cleanup:
kubectl delete deployment lab-app
kubectl delete svc lab-svc
```

### Lab 2: Compare iptables vs IPVS Mode

```bash
# Exercise: Switch between modes and compare rule structures

# Step 1: Check current mode:
kubectl get configmap kube-proxy -n kube-system -o yaml | grep "mode:"

# Step 2: View current iptables rules count:
# SSH to a node:
iptables-save | grep "^-A" | wc -l

# Step 3: View current IPVS state (if in IPVS mode):
ipvsadm -ln

# Step 4: Create multiple services and observe scaling:
for i in {1..5}; do
  kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: test-svc-$i
spec:
  selector:
    app: test-app-$i
  ports:
  - port: 80
    targetPort: 8080
EOF
done

# Step 5: Count rules again:
# SSH to node:
iptables-save | grep "^-A" | wc -l
# Or IPVS:
ipvsadm -ln | wc -l

# Step 6: Cleanup:
for i in {1..5}; do kubectl delete svc test-svc-$i; done
```

### Lab 3: Observe Real-Time Rule Updates

```bash
# Exercise: Watch kube-proxy update rules as endpoints change

# Terminal 1: Watch kube-proxy logs:
kubectl logs -n kube-system -l k8s-app=kube-proxy -f | grep -E "sync|endpoint|service"

# Terminal 2: Watch iptables rules (on a node):
# SSH to node:
watch -n 1 'iptables-save | grep "KUBE-SVC-" | wc -l'

# Terminal 3: Create service and scale:
kubectl create deployment nginx-lab --image=nginx:alpine --replicas=2
kubectl expose deployment nginx-lab --port=80 --target-port=80

# Observe:
# - kube-proxy logs show sync triggered
# - iptables rule count increases

# Scale up:
kubectl scale deployment nginx-lab --replicas=5
# Observe: logs show endpoint update, rules count changes

# Scale down:
kubectl scale deployment nginx-lab --replicas=1
# Observe: stale DNAT rules removed

# Cleanup:
kubectl delete deployment nginx-lab
kubectl delete svc nginx-lab
```

### Lab 4: Test Service Types

```bash
# Exercise: Understand behavior of each service type

# 1. ClusterIP (default):
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: clusterip-test
spec:
  type: ClusterIP
  selector:
    app: test-web
  ports:
  - port: 80
EOF

# Test: Only accessible from within cluster
kubectl exec -it test-pod -- curl http://clusterip-test
# Works ✅
curl http://<clusterip>:80   # From outside: FAILS ✗

# 2. NodePort:
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nodeport-test
spec:
  type: NodePort
  selector:
    app: test-web
  ports:
  - port: 80
    nodePort: 30080
EOF

# Test: Accessible from any node IP:
NODEIP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
curl http://$NODEIP:30080  # Works from external ✅

# Inspect NodePort iptables rules:
# SSH to node:
iptables-save | grep "30080"

# 3. Headless (no kube-proxy involvement):
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: headless-test
spec:
  clusterIP: None
  selector:
    app: test-web
  ports:
  - port: 80
EOF

# Verify: No iptables rules for headless service:
# SSH to node:
HEADLESSIP=$(kubectl get svc headless-test -o jsonpath='{.spec.clusterIP}')
echo "ClusterIP: $HEADLESSIP"   # Should show "None"
iptables-save | grep "headless-test"   # No rules

# DNS returns pod IPs directly:
kubectl exec -it test-pod -- nslookup headless-test
# Returns: individual pod IP addresses

# Cleanup:
kubectl delete svc clusterip-test nodeport-test headless-test
```

### Lab 5: Diagnose a Broken Service

```bash
# Exercise: Systematically debug a service that isn't working

# Step 1: Create a "broken" service (wrong selector):
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: broken-app
  template:
    metadata:
      labels:
        app: broken-app   # Label on pod
    spec:
      containers:
      - name: app
        image: nginx:alpine
---
apiVersion: v1
kind: Service
metadata:
  name: broken-svc
spec:
  selector:
    app: wrong-label     # WRONG: doesn't match pod label!
  ports:
  - port: 80
    targetPort: 80
EOF

# Step 2: Try to access it:
kubectl exec -it test-pod -- curl http://broken-svc
# Should fail: connection refused

# Step 3: Debug:
kubectl get svc broken-svc                    # Service exists?
kubectl get endpointslices -l kubernetes.io/service-name=broken-svc
# Should show: No resources (or empty endpoints)

kubectl get pods -l app=broken-app            # Pods exist?
kubectl describe svc broken-svc | grep "Selector:"
kubectl get pods --show-labels               # Check labels

# Step 4: Fix the selector:
kubectl patch svc broken-svc -p '{"spec":{"selector":{"app":"broken-app"}}}'

# Step 5: Verify fixed:
kubectl get endpointslices -l kubernetes.io/service-name=broken-svc
kubectl exec -it test-pod -- curl http://broken-svc   # Should work now

# Cleanup:
kubectl delete deployment broken-app
kubectl delete svc broken-svc
```

### Lab 6: Monitor conntrack Table

```bash
# Exercise: Understand conntrack and its limits

# Step 1: Check current conntrack state:
# SSH to a node:
cat /proc/sys/net/netfilter/nf_conntrack_count   # Current
cat /proc/sys/net/netfilter/nf_conntrack_max     # Max

# Step 2: View active connections:
conntrack -L | head -20
# Shows: proto, state, timeout, src/dst addresses

# Step 3: Count by protocol:
conntrack -L | awk '{print $1}' | sort | uniq -c | sort -rn

# Step 4: Monitor in real time:
watch -n 2 'echo "Current: $(cat /proc/sys/net/netfilter/nf_conntrack_count)"; \
            echo "Max:     $(cat /proc/sys/net/netfilter/nf_conntrack_max)"; \
            echo "Usage:   $(echo "scale=2; \
              $(cat /proc/sys/net/netfilter/nf_conntrack_count) * 100 / \
              $(cat /proc/sys/net/netfilter/nf_conntrack_max)" | bc)%"'

# Step 5: View kube-proxy related connections:
conntrack -L | grep "dport=80" | wc -l   # Connections to port 80
conntrack -L | grep "ESTABLISHED" | wc -l  # Active connections

# Step 6: Check for connection errors:
conntrack -S
# Look for: insert_failed (table full), early_drop, drop
```

---

## 26. Interview Questions: Beginner to Advanced

### Beginner Level

**Q1: What is kube-proxy and what problem does it solve?**

**A:** Kube-proxy is a per-node network proxy that implements the Kubernetes Service abstraction. The core problem it solves is: Pods have ephemeral IPs that change constantly, but clients need a stable way to reach a group of Pods. Services provide stable virtual IPs (ClusterIPs), and kube-proxy makes those VIPs actually work by programming the node's Linux kernel (via iptables or IPVS) to route traffic from the ClusterIP to the actual Pod IPs. Without kube-proxy (or a replacement), Service ClusterIPs would be meaningless — they'd exist in etcd but have no network routing behind them.

---

**Q2: How is kube-proxy deployed in Kubernetes?**

**A:** Kube-proxy is deployed as a **DaemonSet** in the `kube-system` namespace, ensuring exactly one kube-proxy pod runs on every node in the cluster (both worker nodes and control plane nodes). It runs with `privileged: true` security context because it needs root-level access to program iptables/IPVS rules in the Linux kernel. Each kube-proxy instance manages only the network rules on its local node — it does not coordinate with kube-proxy instances on other nodes.

---

**Q3: What is the difference between iptables and IPVS mode in kube-proxy?**

**A:** Both implement Service routing, but they use different Linux kernel subsystems. **iptables mode** uses the Netfilter framework — it programs DNAT rules in iptables chains, using statistical probability for load balancing. It has O(n) complexity because each packet must traverse rules linearly. **IPVS mode** uses the Linux IP Virtual Server subsystem — it creates virtual servers with hash-based lookups for O(1) complexity regardless of service count. IPVS also supports more load balancing algorithms (round-robin, least-connection, source-hash, etc.) and has better performance at scale (>500 services). The tradeoff: IPVS requires loading kernel modules (`ip_vs`, `ip_vs_rr`, etc.) and is slightly more complex to operate.

---

**Q4: Does kube-proxy communicate directly with etcd?**

**A:** No — kube-proxy **never** communicates directly with etcd. It only communicates with the kube-apiserver via HTTPS watch connections. Kube-proxy watches for changes to Service and EndpointSlice objects via the API server. The API server is the only Kubernetes component that reads and writes etcd. This is a fundamental architectural principle — all cluster state access goes through the API server, which provides authentication, authorization, validation, and audit logging.

---

### Intermediate Level

**Q5: What is the role of EndpointSlices in kube-proxy's operation?**

**A:** EndpointSlices are the objects that map Services to Pod IPs. When a Pod becomes ready, the EndpointSlice controller adds that Pod's IP to the relevant EndpointSlice. Kube-proxy watches EndpointSlices and uses the included Pod IPs to program DNAT rules — these are the rules that actually load-balance traffic to Pod IPs. EndpointSlices replaced the legacy Endpoints API to solve scalability: a service with 1000 pods generates 10 EndpointSlice objects (100 endpoints each) instead of one massive Endpoints object. This means a single pod restart only triggers an update to one EndpointSlice shard instead of rewriting the entire object — significantly reducing API server load and kube-proxy update frequency.

---

**Q6: How does kube-proxy achieve zero-downtime during a rolling deployment?**

**A:** Kube-proxy works in conjunction with readiness probes and the EndpointSlice controller. During a rolling deployment: (1) New Pod starts but `readinessProbe` hasn't passed yet — the Pod is NOT added to the EndpointSlice, so kube-proxy does NOT add a DNAT rule for it. (2) `readinessProbe` passes — EndpointSlice is updated, kube-proxy adds DNAT rule, traffic now flows to the new Pod. (3) Old Pod receives SIGTERM — its `ready` condition becomes false immediately, EndpointSlice controller removes it, kube-proxy removes the DNAT rule, no new traffic goes to it. (4) `preStop` hook (if configured) runs, giving the old Pod time to drain existing connections. (5) Pod terminates. The key: kube-proxy's rule updates happen based on endpoint readiness, not container status.

---

**Q7: What happens if kube-proxy crashes on a node?**

**A:** When kube-proxy crashes, the **existing iptables/IPVS rules persist in the kernel** — they are not removed. This means existing services continue to work for connections that match the current (now frozen) rules. However, any endpoint changes that happen while kube-proxy is down are not reflected: new Pods that become ready won't have DNAT rules added (they receive no traffic), and Pods that terminate will still have rules pointing to their dead IPs (connections to them will get connection refused). When kube-proxy restarts, it re-lists all Services and EndpointSlices from the API server, computes the full desired ruleset, and reconciles — fixing all stale/missing rules in a single sync.

---

**Q8: How does kube-proxy handle NodePort services and what is externalTrafficPolicy?**

**A:** For NodePort services, kube-proxy programs an additional set of iptables rules matching any traffic arriving on the NodePort (e.g., 30080) on any node's IP. The `externalTrafficPolicy` setting controls whether traffic is routed across nodes: `Cluster` (default) — kube-proxy routes to any healthy backend Pod regardless of which node it's on, applying MASQUERADE so the Pod sees the node's IP as the source (client IP is lost). `Local` — kube-proxy only creates rules for Pods running on the local node; traffic to a node without local Pods gets connection refused. `Local` preserves the client's source IP (no MASQUERADE needed) and avoids an extra network hop, but risks uneven load distribution if Pods aren't evenly spread across nodes.

---

### Advanced Level

**Q9: Explain the full iptables chain traversal for a ClusterIP service request.**

**A:** When a Pod sends traffic to a ClusterIP (e.g., 10.96.0.100:80), the packet hits Netfilter hooks in this order: (1) **OUTPUT hook** (for locally-originated traffic) → traverses `KUBE-SERVICES` chain — kube-proxy adds a rule matching `-d 10.96.0.100/32 -p tcp --dport 80 -j KUBE-SVC-<hash>`. (2) **KUBE-SVC-<hash> chain** — contains probability-based rules: `-m statistic --mode random --probability 0.333 -j KUBE-SEP-A`, then `-m statistic --mode random --probability 0.5 -j KUBE-SEP-B`, then `-j KUBE-SEP-C`. One KUBE-SEP chain is selected. (3) **KUBE-SEP-<hash> chain** — applies DNAT: `-j DNAT --to-destination 10.244.1.5:8080`. Packet destination is rewritten. (4) **conntrack records** the translation. (5) Packet routes to the backend Pod (possibly via CNI overlay to another node). (6) **Reply packet** arrives, conntrack automatically reverses the DNAT, reply goes back to original client. Subsequent packets from the same connection use the fast conntrack path, bypassing iptables rule traversal entirely.

---

**Q10: How does topology-aware routing work with kube-proxy, and what are its limitations?**

**A:** Topology-aware routing (via `service.kubernetes.io/topology-mode: Auto` annotation) enables kube-proxy to prefer endpoints in the same topology zone (e.g., AWS availability zone) as the node. The EndpointSlice controller populates `hints.forZones` fields in EndpointSlice objects, indicating which zone each endpoint should serve. Kube-proxy reads these hints and, when programming rules, preferentially routes to endpoints with matching zone hints. Limitations: (1) Requires at least 3 endpoints and reasonably even distribution across zones — if one zone has significantly more pods, the controller may not generate hints (falls back to normal routing). (2) Requires all nodes to have a `topology.kubernetes.io/zone` label. (3) Hint generation has a 20% skew tolerance — if zone distribution is too uneven, no hints are generated. (4) It's a preference, not a guarantee — if all hints point to a zone with no ready endpoints, kube-proxy falls back to routing across zones.

---

**Q11: At what scale does iptables mode become problematic and why? Design a migration plan to IPVS.**

**A:** iptables mode becomes problematic at approximately 1,000-2,000 services because: (1) iptables rules grow as O(services × endpoints) — at 1,000 services with avg 10 endpoints = ~65,000 rules; (2) Each packet must traverse rules linearly until a match — with 65,000 rules, high-PPS workloads burn significant CPU; (3) `iptables-restore` for a full ruleset sync is a single-threaded operation that takes 2-10 seconds for large rulesets, during which the iptables lock is held, causing lock contention with other iptables operations; (4) Any endpoint change (single pod restart) triggers a full ruleset rewrite. **Migration plan**: Step 1 — load kernel modules (`ip_vs`, `ip_vs_rr`, `ip_vs_sh`, `nf_conntrack`) on all nodes and make persistent in `/etc/modules-load.d/`. Step 2 — update kube-proxy ConfigMap: set `mode: "ipvs"`, `ipvs.scheduler: "rr"` (or `lc`), and `strictARP: true` if using MetalLB. Step 3 — rolling restart kube-proxy DaemonSet: `kubectl rollout restart daemonset kube-proxy -n kube-system`. Step 4 — verify via `ipvsadm -ln` and `kubectl exec` service tests. Step 5 — monitor sync latency metrics and conntrack table size. Rollback: change ConfigMap back to `mode: "iptables"` and restart; kube-proxy automatically flushes IPVS and re-programs iptables.

---

**Q12: Explain how Cilium replaces kube-proxy and the architectural advantages.**

**A:** Cilium replaces kube-proxy by using eBPF programs loaded directly into the Linux kernel, bypassing the Netfilter layer entirely. When deployed with `kubeProxyReplacement=true`, Cilium: (1) watches Services and EndpointSlices directly from the API server (just like kube-proxy), but instead of programming iptables, it loads eBPF programs at the TC (Traffic Control) hook point — early in the networking stack before Netfilter. (2) Uses BPF maps (hash tables in kernel memory) for O(1) service lookup — similar to IPVS but even faster because it's processed inline in the driver. (3) Supports **Direct Server Return (DSR)** for NodePort — the backend pod can reply directly to the client without going through the ingress node, eliminating a network hop. (4) Supports **stateless load balancing** for certain flows (using Maglev consistent hashing) — eliminating conntrack overhead for protocols that don't need it. (5) Updates are fully incremental — updating one service's endpoints modifies only the relevant BPF map entry, without any global lock or full-ruleset rewrite. Architectural advantages: significantly lower latency (no Netfilter hook traversal), no conntrack table exhaustion risk, faster rule updates, lower CPU overhead at high packet rates, and native integration with Network Policy enforcement.

---

## 27. Cheat Sheet: Commands & Flags

### Essential kubectl Commands

```bash
# === KUBE-PROXY STATUS ===
# List kube-proxy pods (one per node):
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide

# Check kube-proxy pod status on specific node:
kubectl get pods -n kube-system -l k8s-app=kube-proxy \
  --field-selector spec.nodeName=<node-name>

# View kube-proxy logs:
kubectl logs -n kube-system -l k8s-app=kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy -f     # Follow
kubectl logs -n kube-system -l k8s-app=kube-proxy --previous  # Previous crash

# Check proxy mode:
kubectl get configmap kube-proxy -n kube-system -o yaml | grep mode

# Check full kube-proxy config:
kubectl get configmap kube-proxy -n kube-system -o yaml

# Edit kube-proxy config:
kubectl edit configmap kube-proxy -n kube-system

# Restart kube-proxy (picks up config changes):
kubectl rollout restart daemonset kube-proxy -n kube-system

# Check rollout status:
kubectl rollout status daemonset kube-proxy -n kube-system

# === SERVICE DEBUGGING ===
# View all services:
kubectl get svc -A
kubectl get svc -n <namespace>

# Describe service (check selector, ports, endpoints):
kubectl describe svc <service-name> -n <namespace>

# View EndpointSlices for a service:
kubectl get endpointslices -n <namespace> \
  -l kubernetes.io/service-name=<service-name>

# View EndpointSlice details (ready status, pod IPs):
kubectl describe endpointslice <name> -n <namespace>

# Legacy endpoints:
kubectl get endpoints <service-name> -n <namespace>

# Check if pods match service selector:
kubectl get pods -l <selector-key>=<selector-value> -n <namespace>

# View service with all fields:
kubectl get svc <name> -n <namespace> -o yaml

# === NETWORK DEBUGGING ===
# Test service connectivity from a pod:
kubectl exec -it <pod> -n <namespace> -- curl http://<service>:<port>
kubectl exec -it <pod> -n <namespace> -- curl http://<clusterip>:<port>
kubectl exec -it <pod> -n <namespace> -- nslookup <service>

# Run a debug pod:
kubectl run debug --image=nicolaka/netshoot --rm -it -- bash

# Check DNS from pod:
kubectl exec -it <pod> -- cat /etc/resolv.conf
kubectl exec -it <pod> -- nslookup kubernetes.default

# Port forward to bypass kube-proxy (test pod directly):
kubectl port-forward pod/<pod-name> 8080:80

# === LEADER ELECTION (adjacent components) ===
kubectl get lease -n kube-system
kubectl get lease kube-controller-manager -n kube-system -o yaml
kubectl get lease kube-scheduler -n kube-system -o yaml
```

### Node-Level Commands (SSH Required)

```bash
# === IPTABLES MODE ===
# View all kube-proxy chains:
iptables-save | grep "^:KUBE"

# Count kube-proxy rules:
iptables-save | grep "^-A KUBE" | wc -l

# Find chain for a specific service ClusterIP:
iptables-save | grep "<CLUSTERIP>"

# Trace a service's full chain:
SVCIP="10.96.0.100"
SVCCHAIN=$(iptables-save | grep "$SVCIP" | grep -o "KUBE-SVC-[A-Z0-9]*")
echo "Service chain: $SVCCHAIN"
iptables-save | grep "$SVCCHAIN"

# View DNAT rules for endpoints:
iptables -t nat -L KUBE-SEP-<hash> -n -v

# View MASQUERADE rules:
iptables -t nat -L KUBE-POSTROUTING -n -v

# View NodePort rules:
iptables-save | grep "KUBE-NODEPORTS"

# Check rule counts (should grow with services):
iptables-save | grep "^-A" | wc -l

# === IPVS MODE ===
# List all virtual services:
ipvsadm -ln

# List with stats:
ipvsadm -ln --stats

# List with connection rates:
ipvsadm -ln --rate

# Show specific service:
ipvsadm -ln -t <ClusterIP>:<Port>

# Count virtual services:
ipvsadm -ln | grep "^TCP\|^UDP" | wc -l

# === CONNTRACK ===
# View conntrack table usage:
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# List active connections:
conntrack -L | head -20
conntrack -L | wc -l

# Show conntrack statistics:
conntrack -S

# View by protocol:
conntrack -L -p tcp | head
conntrack -L -p udp | head

# Delete specific connection (force re-routing):
conntrack -D -d <pod-ip>

# Increase conntrack table (production tuning):
sysctl -w net.netfilter.nf_conntrack_max=1048576
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=86400

# === IPVS KERNEL MODULES ===
lsmod | grep ip_vs
modprobe ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh ip_vs_lc nf_conntrack

# === METRICS ===
# View kube-proxy metrics:
curl http://localhost:10249/metrics | grep kubeproxy

# Check health:
curl http://localhost:10256/healthz

# View config:
curl http://localhost:10249/configz

# Key metrics to watch:
curl -s http://localhost:10249/metrics | grep \
  "sync_proxy_rules_duration\|iptables_rules_total\|nf_conntrack"
```

### KubeProxyConfiguration Key Fields Reference

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration

# Mode options: iptables, ipvs, nftables (alpha 1.29)
mode: "ipvs"

# Network:
bindAddress: "0.0.0.0"
clusterCIDR: "10.244.0.0/16"        # MUST match your pod CIDR

# Ports:
healthzBindAddress: "0.0.0.0:10256"
metricsBindAddress: "127.0.0.1:10249"

# iptables settings:
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: "1s"
  syncPeriod: "30s"
  localhostNodePorts: true

# IPVS settings:
ipvs:
  minSyncPeriod: "1s"
  syncPeriod: "30s"
  scheduler: "rr"         # rr, lc, dh, sh, wrr, wlc, sed, nq
  strictARP: false        # Set true for MetalLB
  excludeCIDRs: []
  tcpTimeout: "900s"
  tcpFinTimeout: "30s"
  udpTimeout: "300s"

# API server:
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig.conf"
  qps: 10
  burst: 20

# NodePort:
nodePortAddresses: []     # Empty = all interfaces, or e.g. ["192.168.0.0/24"]

# Feature gates:
featureGates:
  TopologyAwareHints: true

# Logging:
logging:
  verbosity: 2            # 0-9; 4 = debug, 6 = trace
```

### Quick Reference: Rule Counts by Mode

```bash
# iptables mode — count service rules:
echo "Total iptables rules: $(iptables-save | grep '^-A' | wc -l)"
echo "KUBE rules: $(iptables-save | grep '^-A KUBE' | wc -l)"
echo "Approx services: $(iptables-save | grep 'KUBE-SVC-' | grep -c 'KUBE-SERVICES')"

# IPVS mode — count virtual services:
echo "IPVS virtual services: $(ipvsadm -ln | grep '^TCP\|^UDP\|^SCTP' | wc -l)"
echo "IPVS real servers: $(ipvsadm -ln | grep '^ ->' | wc -l)"

# conntrack — usage percentage:
CURRENT=$(cat /proc/sys/net/netfilter/nf_conntrack_count)
MAX=$(cat /proc/sys/net/netfilter/nf_conntrack_max)
echo "conntrack: $CURRENT / $MAX ($(echo "scale=1; $CURRENT * 100 / $MAX" | bc)%)"
```

---

## 28. Key Takeaways & Summary

### The Essential Mental Model

Kube-proxy is the **network plumber** of Kubernetes. The control plane (scheduler, controller-manager) decides the *what* and *where* of workloads. Kube-proxy implements the *how* of service networking — it translates the abstract Service objects in etcd into concrete kernel routing rules on every node, making the virtual IP abstraction a physical reality.

### The 12 Most Critical Things to Know About Kube-proxy

1. **kube-proxy is a per-node DaemonSet** — one instance per node, each managing only its own local kernel's networking rules. No coordination between instances.

2. **It never touches etcd** — all communication goes through kube-apiserver via HTTPS watch. Kube-proxy watches Services and EndpointSlices.

3. **It programs the kernel, not pods** — kube-proxy's output is iptables/IPVS rules in the Linux kernel. It doesn't create, modify, or delete any Kubernetes objects (except events).

4. **Existing rules survive kube-proxy crashes** — iptables/IPVS rules persist in the kernel. Services keep working until endpoint drift accumulates.

5. **EndpointSlices are the critical link** — kube-proxy learns about Pod IPs exclusively from EndpointSlices (or legacy Endpoints). Pod readiness gates endpoint inclusion, which gates rule creation.

6. **iptables mode has O(n) complexity** — at large scale (>1,000 services), rule traversal and sync time degrade. Switch to IPVS mode for production at scale.

7. **IPVS mode has O(1) hash-based lookup** — designed for high-scale environments. Requires loading `ip_vs*` kernel modules. More scheduling algorithms available.

8. **Headless services and ExternalName services bypass kube-proxy** — kube-proxy programs zero rules for these types. Their "routing" is handled by DNS.

9. **externalTrafficPolicy: Local preserves source IP** — but only routes to local pods. Use with DaemonSets or evenly-distributed deployments to avoid traffic black-holing.

10. **Zero-downtime deployments require readinessProbe + preStop** — readinessProbe controls endpoint addition, preStop provides drain time between endpoint removal and container termination.

11. **conntrack table exhaustion breaks connections** — at high connection rates, the conntrack table fills. Monitor usage and tune `nf_conntrack_max` proactively.

12. **Cilium (eBPF) can fully replace kube-proxy** — for clusters at scale where iptables/IPVS is a bottleneck, eBPF-based solutions offer O(1) lookup, no conntrack overhead, DSR support, and faster rule updates.

### Quick Decision Framework

```
Service not reachable?
  → Endpoints exist?
      NO:  selector mismatch, pods not ready, readinessProbe failing
      YES: kube-proxy rules missing? → check iptables/ipvsadm on node
           DNS resolving? → check CoreDNS, /etc/resolv.conf in pod
           Network between nodes? → check CNI (Calico, Cilium, etc.)

kube-proxy logs showing errors?
  → "iptables-restore failed" → iptables binary issue, mode mismatch
  → "can't connect to API server" → API server unavailable
  → "ip_vs module not loaded" → run: modprobe ip_vs ip_vs_rr ...
  → "PLEG" errors → wrong component (that's Kubelet)

Performance issues?
  → iptables rules > 50,000? → switch to IPVS mode
  → conntrack table > 80% full? → increase nf_conntrack_max
  → sync time > 5 seconds? → switch to IPVS or Cilium
  → High CPU on sync? → check minSyncPeriod, consider Cilium

Production scale?
  → < 500 services: iptables is fine
  → 500-2000 services: use IPVS mode
  → > 2000 services or > 200 nodes: evaluate Cilium (eBPF)
  → Regulated/security-sensitive: Cilium (better Network Policy + observability)
```

### Service Traffic in One Sentence

> Kube-proxy watches **Services** and **EndpointSlices** in the API server and programs **iptables or IPVS rules** on the local node's kernel so that traffic to a stable virtual **ClusterIP** is **DNAT'd** to a dynamically-selected, healthy backing **Pod IP** — making Kubernetes Services physically real.

---

*This document represents production-grade Kubernetes networking knowledge compiled for DevOps engineers, SREs, and platform engineers managing Kubernetes clusters in production. All configurations and commands should be validated in a non-production environment before applying to production systems.*

---

**Document Version:** 1.0
**Last Updated:** April 2026
**Kubernetes Reference Version:** v1.28+

---
*End of Document*
