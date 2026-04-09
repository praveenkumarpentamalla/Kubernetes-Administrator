## Table of Contents

1. [Introduction & Why Cluster Installation Matters](#1-introduction--why-cluster-installation-matters)
2. [Core Identity Table — Kubernetes Control Plane Components](#2-core-identity-table--kubernetes-control-plane-components)
3. [Cluster Architecture Overview](#3-cluster-architecture-overview)
4. [The Controller Pattern — Watch → Compare → Act → Loop](#4-the-controller-pattern--watch--compare--act--loop)
5. [Pre-Installation: Hardware & OS Requirements](#5-pre-installation-hardware--os-requirements)
6. [Pre-Installation: Network Planning](#6-pre-installation-network-planning)
7. [OS Hardening & System Preparation](#7-os-hardening--system-preparation)
8. [Container Runtime Installation (containerd)](#8-container-runtime-installation-containerd)
9. [Installing kubeadm, kubelet & kubectl](#9-installing-kubeadm-kubelet--kubectl)
10. [Bootstrapping the Control Plane with kubeadm init](#10-bootstrapping-the-control-plane-with-kubeadm-init)
11. [Installing the CNI Plugin (Calico / Flannel / Cilium)](#11-installing-the-cni-plugin-calico--flannel--cilium)
12. [Joining Worker Nodes](#12-joining-worker-nodes)
13. [Deep Dive: Built-in Controllers](#13-deep-dive-built-in-controllers)
14. [Internal Working Concepts: Informers, Work Queues & Reconciliation](#14-internal-working-concepts-informers-work-queues--reconciliation)
15. [Interaction with API Server and etcd](#15-interaction-with-api-server-and-etcd)
16. [Leader Election for HA Control Planes](#16-leader-election-for-ha-control-planes)
17. [High-Availability Control Plane Setup](#17-high-availability-control-plane-setup)
18. [Performance Tuning](#18-performance-tuning)
19. [Security Hardening Practices](#19-security-hardening-practices)
20. [Monitoring & Observability](#20-monitoring--observability)
21. [Troubleshooting — Real kubectl Commands](#21-troubleshooting--real-kubectl-commands)
22. [Disaster Recovery Concepts](#22-disaster-recovery-concepts)
23. [Cluster Upgrade Strategy](#23-cluster-upgrade-strategy)
24. [Comparison: kubeadm vs Other Install Methods](#24-comparison-kubeadm-vs-other-install-methods)
25. [Comparison: kube-apiserver vs kube-scheduler vs kube-controller-manager](#25-comparison-kube-apiserver-vs-kube-scheduler-vs-kube-controller-manager)
26. [ASCII Architecture Diagram](#26-ascii-architecture-diagram)
27. [Real-World Production Use Cases](#27-real-world-production-use-cases)
28. [Best Practices for Production Environments](#28-best-practices-for-production-environments)
29. [Common Mistakes and Pitfalls](#29-common-mistakes-and-pitfalls)
30. [Hands-On Labs & Practical Exercises](#30-hands-on-labs--practical-exercises)
31. [Interview Questions — Beginner to Advanced](#31-interview-questions--beginner-to-advanced)
32. [Cheat Sheet — Commands, Flags & Manifests](#32-cheat-sheet--commands-flags--manifests)
33. [Key Takeaways & Summary](#33-key-takeaways--summary)

---

## 1. Introduction & Why Cluster Installation Matters

Installing a Kubernetes cluster is not just a technical exercise — it is the foundational act that determines the **security posture**, **performance envelope**, **operational resilience**, and **upgrade path** of your entire container platform. Every architectural decision made during installation echoes forward: the CNI plugin choice affects network policy capabilities years later; the etcd topology determines your blast radius in a failure; the initial CIDR ranges constrain how large your cluster can ever grow.

### Why This Skill Is Critical

In an era where managed Kubernetes services (EKS, GKE, AKS) abstract away the control plane, understanding cluster installation from first principles remains essential for several reasons:

- **On-premises and air-gapped environments** require self-managed clusters (government, banking, manufacturing)
- **Cost optimization** — self-managed clusters on bare metal or spot instances can be 60–80% cheaper than managed services at scale
- **Compliance and data sovereignty** — some regulatory frameworks require full control over the control plane
- **Troubleshooting managed clusters** — understanding installation gives you the mental model to debug even managed services
- **Platform engineering roles** — senior DevOps/platform engineers are expected to understand what's running under the hood
- **Custom configurations** — feature gates, admission controllers, audit policies, and encryption at rest require direct control

### What This Guide Covers

This guide takes you through a **production-grade cluster installation** using `kubeadm` — the official Kubernetes cluster bootstrapping tool — from bare OS to a fully functioning, secured, observable cluster. Every command is explained, every configuration option justified, and every production consideration documented.

---

## 2. Core Identity Table — Kubernetes Control Plane Components

| Component | Binary Name | Default Port | Protocol | Role | Runs On | Managed By |
|---|---|---|---|---|---|---|
| **API Server** | `kube-apiserver` | 6443 | HTTPS | Central REST gateway; only component that reads/writes etcd | Control Plane | Static Pod / systemd |
| **etcd** | `etcd` | 2379 (client), 2380 (peer) | HTTPS/gRPC | Distributed KV store; single source of truth | Control Plane (dedicated or co-located) | Static Pod / systemd |
| **Controller Manager** | `kube-controller-manager` | 10257 | HTTPS | Runs all built-in controllers; reconciliation loop engine | Control Plane | Static Pod |
| **Scheduler** | `kube-scheduler` | 10259 | HTTPS | Assigns Pods to Nodes based on resource/policy constraints | Control Plane | Static Pod |
| **kubelet** | `kubelet` | 10250 | HTTPS | Node agent; manages Pod lifecycle; talks to CRI | All Nodes | systemd |
| **kube-proxy** | `kube-proxy` | 10249 (metrics) | HTTP | Programs iptables/IPVS for Service routing | All Nodes | DaemonSet |
| **CoreDNS** | `coredns` | 53 (DNS), 9153 (metrics) | UDP/TCP | Cluster DNS; resolves Service/Pod names | Control Plane (initially) | Deployment |
| **Container Runtime** | `containerd` / `cri-o` | N/A (Unix socket) | gRPC/CRI | Pulls images, runs containers | All Nodes | systemd |
| **CNI Plugin** | `calico-node`, `cilium`, etc. | Varies | N/A | Pod networking, NetworkPolicy enforcement | All Nodes | DaemonSet |
| **kubeadm** | `kubeadm` | N/A | N/A | Bootstrap/upgrade/join tool; not a running daemon | Installer machine | One-shot CLI |

### Component Certificate Matrix

| Component | Serves TLS On | Client Certificate Used For |
|---|---|---|
| kube-apiserver | 6443 | Receives connections from all clients |
| etcd | 2379, 2380 | kube-apiserver authenticates to etcd |
| kubelet | 10250 | kube-apiserver calls kubelet for exec/logs |
| kube-controller-manager | — | Authenticates to kube-apiserver |
| kube-scheduler | — | Authenticates to kube-apiserver |
| kube-proxy | — | Authenticates to kube-apiserver |

---

## 3. Cluster Architecture Overview

### 3.1 Logical Architecture

A Kubernetes cluster consists of two logical planes:

**Control Plane:** The brain of the cluster. Manages cluster state, schedules workloads, and runs reconciliation loops. Consists of kube-apiserver, etcd, kube-controller-manager, and kube-scheduler.

**Data Plane (Worker Nodes):** The muscle. Runs actual application workloads. Each node has kubelet, kube-proxy, a container runtime, and a CNI plugin.

### 3.2 Minimum vs Production Topology

| Topology | Control Plane Nodes | Worker Nodes | etcd | Use Case |
|---|---|---|---|---|
| **Single-node** (dev) | 1 (all-in-one) | 0 (tainted) | Embedded | Local dev, CI |
| **Single control plane** | 1 | 2+ | Co-located | Small non-prod |
| **HA stacked** | 3 | 3+ | Co-located on CP nodes | Standard production |
| **HA external etcd** | 3 | 3+ | Dedicated 3–5 node cluster | Large-scale production |
| **Multi-region** | 3+ per region | Many | Dedicated per region | Enterprise HA |

### 3.3 Component Communication Map

```
kubectl → kube-apiserver → etcd (read/write state)
kube-controller-manager → kube-apiserver (watch + write)
kube-scheduler → kube-apiserver (watch Pods, write nodeName)
kubelet → kube-apiserver (register node, report status, get Pod specs)
kube-proxy → kube-apiserver (watch Services + EndpointSlices)
CoreDNS → kube-apiserver (watch Services + Pods)
```

**The invariant:** Every component communicates exclusively through the kube-apiserver. No component ever accesses etcd directly except the kube-apiserver itself.

---

## 4. The Controller Pattern — Watch → Compare → Act → Loop

Every built-in controller in Kubernetes follows the same fundamental pattern. Understanding this before installation helps you reason about what's happening at runtime.

### 4.1 The Reconciliation Loop

```
┌─────────────────────────────────────────────────────────┐
│                 CONTROLLER RECONCILIATION LOOP           │
│                                                         │
│   ┌──────────┐    ┌──────────────┐    ┌─────────────┐  │
│   │  WATCH   │───►│   COMPARE    │───►│     ACT     │  │
│   │          │    │              │    │             │  │
│   │ Observe  │    │ Desired State│    │ Create /    │  │
│   │ current  │    │     vs       │    │ Update /    │  │
│   │ state    │    │ Current State│    │ Delete      │  │
│   └──────────┘    └──────────────┘    └─────────────┘  │
│        ▲                                     │          │
│        └─────────────────────────────────────┘          │
│                   Loop forever                          │
└─────────────────────────────────────────────────────────┘
```

### 4.2 Key Properties of the Loop

- **Level-triggered, not edge-triggered:** The controller always reconciles to the full desired state, not just the delta. This means a controller that missed 10 events still produces the correct final state.
- **Idempotent:** Running the reconciliation twice produces the same result. Safe to re-run after crash.
- **Optimistic concurrency:** Uses `resourceVersion` to detect conflicts; retries on conflict.
- **No direct etcd access:** All reads/writes go through the kube-apiserver.

### 4.3 Applied to Cluster Installation

During `kubeadm init`, you can observe the controller pattern being bootstrapped:

1. Static Pods for etcd and kube-apiserver are created on disk
2. kubelet (already running) **watches** the static pod manifests directory
3. kubelet **compares**: no containers running for these manifests
4. kubelet **acts**: creates containers via CRI
5. kube-apiserver comes up, kube-controller-manager registers its controllers
6. Each controller begins its own watch loop

---

## 5. Pre-Installation: Hardware & OS Requirements

### 5.1 Minimum Hardware Requirements

| Role | CPU | RAM | Disk | Network |
|---|---|---|---|---|
| Control Plane (single) | 2 vCPU | 2 GB | 50 GB SSD | 1 Gbps |
| Control Plane (HA, per node) | 4 vCPU | 8 GB | 100 GB SSD | 1 Gbps |
| etcd (dedicated, per node) | 4 vCPU | 8 GB | 100 GB SSD NVMe | 1 Gbps |
| Worker Node (general) | 4 vCPU | 16 GB | 100–200 GB SSD | 1 Gbps |
| Worker Node (data-intensive) | 16+ vCPU | 64+ GB | 500+ GB NVMe | 10 Gbps |

**Production recommendation:** Control plane nodes should be sized for **controller manager load**, not workloads. etcd is extremely sensitive to disk latency — always use SSD or NVMe for etcd data directories.

### 5.2 Supported Operating Systems

| OS | Version | Notes |
|---|---|---|
| Ubuntu | 20.04, 22.04 LTS | Recommended; most widely tested |
| Debian | 11, 12 | Fully supported |
| RHEL / Rocky Linux | 8, 9 | Requires SELinux consideration |
| CentOS Stream | 8, 9 | Community supported |
| Fedora CoreOS | Latest | Immutable; different toolchain |
| Flatcar Container Linux | Latest | Container-optimized |

### 5.3 Network Port Requirements

| Port | Protocol | Component | Direction | Purpose |
|---|---|---|---|---|
| 6443 | TCP | kube-apiserver | Inbound | API server (all clients) |
| 2379 | TCP | etcd | CP only | etcd client requests |
| 2380 | TCP | etcd | CP only | etcd peer communication |
| 10250 | TCP | kubelet | Inbound | API server to kubelet |
| 10257 | TCP | kube-controller-manager | CP only | Metrics/health |
| 10259 | TCP | kube-scheduler | CP only | Metrics/health |
| 10249 | TCP | kube-proxy | Node | Metrics |
| 30000-32767 | TCP/UDP | NodePort | Inbound | NodePort Services |
| 179 | TCP | Calico BGP | Node-to-node | BGP (if Calico BGP mode) |
| 4789 | UDP | Flannel VXLAN | Node-to-node | VXLAN overlay |
| 8472 | UDP | Cilium VXLAN | Node-to-node | eBPF overlay |
| 4240 | TCP | Cilium health | Node-to-node | Health checks |

---

## 6. Pre-Installation: Network Planning

Network planning before installation is **irreversible** after cluster creation. CIDRs cannot be changed without rebuilding the cluster.

### 6.1 CIDR Planning

| Range | Purpose | Example | Notes |
|---|---|---|---|
| **Node Network CIDR** | Physical/VM IP addresses | `10.0.0.0/16` | Provided by your infrastructure |
| **Pod CIDR** | IP addresses for Pods | `10.244.0.0/16` | Must not overlap with Node or Service CIDRs |
| **Service CIDR** | Virtual IPs for Services | `10.96.0.0/12` | Must not overlap with Pod or Node CIDRs |
| **Cluster DNS IP** | CoreDNS Service IP | `10.96.0.10` | Must be within Service CIDR |

### 6.2 CIDR Sizing Guidelines

```
Pod CIDR: /16 = 65,534 IPs
  Per node allocation: /24 = 254 Pods per node
  Max nodes with /16 Pod CIDR + /24 per node = 256 nodes

For larger clusters:
  Pod CIDR: /14 = 262,144 IPs
  Per node: /24 = up to 1,024 nodes

Service CIDR: /12 = 1,048,574 virtual IPs (more than enough)
```

### 6.3 CNI Plugin Selection

| CNI | Overlay | NetworkPolicy | eBPF | Complexity | Best For |
|---|---|---|---|---|---|
| **Flannel** | VXLAN | No (needs Calico) | No | Low | Simple clusters, learning |
| **Calico** | IPIP/BGP/VXLAN | Yes | Optional | Medium | Most production clusters |
| **Cilium** | VXLAN/Geneve | Yes (L7) | Yes | High | Performance, observability |
| **Weave** | Mesh | Yes | No | Medium | Legacy, mesh networking |
| **Antrea** | VXLAN/Geneve | Yes | Yes | Medium | VMware environments |

**Production recommendation:** Use **Calico** for simplicity + full NetworkPolicy support, or **Cilium** for maximum performance and observability (Hubble). Avoid Flannel in production as it has no NetworkPolicy support.

---

## 7. OS Hardening & System Preparation

Run all commands as `root` or with `sudo` on **every node** (control plane and workers) unless otherwise noted.

### 7.1 Set Hostname (Per Node)

```bash
# On control-plane-1
hostnamectl set-hostname k8s-cp-1

# On control-plane-2
hostnamectl set-hostname k8s-cp-2

# On control-plane-3
hostnamectl set-hostname k8s-cp-3

# On worker-1
hostnamectl set-hostname k8s-worker-1
```

### 7.2 Configure /etc/hosts (All Nodes)

```bash
cat >> /etc/hosts << 'EOF'
# Kubernetes Cluster Nodes
10.0.0.10  k8s-cp-1
10.0.0.11  k8s-cp-2
10.0.0.12  k8s-cp-3
10.0.0.20  k8s-worker-1
10.0.0.21  k8s-worker-2
10.0.0.22  k8s-worker-3
10.0.0.100 k8s-api-vip  # Virtual IP for HA load balancer
EOF
```

### 7.3 Disable Swap (Required by kubelet)

```bash
# Disable swap immediately
swapoff -a

# Permanently disable swap (comment out swap entries)
sed -i '/swap/s/^/#/' /etc/fstab

# Verify swap is disabled
free -h
# Swap line should show 0B total
```

**Why:** kubelet refuses to start if swap is enabled by default. Swap causes non-deterministic memory behavior that violates Kubernetes' memory management model.

### 7.4 Load Required Kernel Modules

```bash
# Create module load config
cat > /etc/modules-load.d/k8s.conf << 'EOF'
overlay
br_netfilter
EOF

# Load modules immediately
modprobe overlay
modprobe br_netfilter

# Verify modules are loaded
lsmod | grep overlay
lsmod | grep br_netfilter
```

**Why:**
- `overlay`: Required by containerd for OverlayFS storage driver
- `br_netfilter`: Allows iptables to see bridged traffic (required for kube-proxy)

### 7.5 Configure Kernel Parameters (sysctl)

```bash
cat > /etc/sysctl.d/k8s.conf << 'EOF'
# Allow iptables to see bridged traffic
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1

# Enable IP forwarding (required for Pod networking)
net.ipv4.ip_forward                 = 1

# Increase connection tracking table (for busy clusters)
net.netfilter.nf_conntrack_max      = 1000000

# Increase inotify watches (for large clusters with many informers)
fs.inotify.max_user_watches         = 1048576
fs.inotify.max_user_instances       = 512

# Increase file descriptor limits
fs.file-max                         = 1000000

# TCP tuning for API server connections
net.core.somaxconn                  = 32768
net.ipv4.tcp_max_syn_backlog        = 8096
EOF

# Apply sysctl settings immediately
sysctl --system

# Verify key settings
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
```

### 7.6 Disable Firewalld / Configure UFW

```bash
# Ubuntu (UFW)
# Allow essential ports
ufw allow 6443/tcp       # API server
ufw allow 2379:2380/tcp  # etcd (CP nodes only)
ufw allow 10250/tcp      # kubelet
ufw allow 10257/tcp      # controller-manager
ufw allow 10259/tcp      # scheduler
ufw allow 30000:32767/tcp # NodePort range

# For pod network communication (allow all inter-node)
ufw allow from 10.0.0.0/16   # Node network
ufw allow from 10.244.0.0/16 # Pod CIDR

# RHEL/Rocky (firewalld)
systemctl stop firewalld
systemctl disable firewalld
# OR configure specific zones/ports
```

### 7.7 Update System & Install Prerequisites

```bash
apt-get update && apt-get upgrade -y

# Install essential tools
apt-get install -y \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg \
  lsb-release \
  socat \
  conntrack \
  ipset \
  ipvsadm \
  nfs-common \
  jq \
  net-tools \
  htop \
  vim

# Set timezone
timedatectl set-timezone UTC

# Sync time (critical for etcd cluster consensus)
apt-get install -y chrony
systemctl enable --now chrony
chronyc tracking
```

**Why chrony/NTP is critical:** etcd requires all cluster nodes to have synchronized clocks within 500ms. Clock skew causes etcd leader election failures and certificate validation errors.

---

## 8. Container Runtime Installation (containerd)

Kubernetes uses the **Container Runtime Interface (CRI)** to communicate with container runtimes. The recommended runtime is `containerd` (used by most managed Kubernetes services).

### 8.1 Install containerd

```bash
# Add Docker's official GPG key (containerd is distributed via Docker's repo)
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  gpg --dearmor -o /etc/apt/keyrings/docker.gpg
chmod a+r /etc/apt/keyrings/docker.gpg

# Add Docker apt repository
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install containerd
apt-get update
apt-get install -y containerd.io

# Verify installation
containerd --version
```

### 8.2 Configure containerd for Kubernetes

```bash
# Generate default config
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml

# CRITICAL: Enable SystemdCgroup driver
# (Must match kubelet's cgroupDriver setting)
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml

# Verify the change
grep 'SystemdCgroup' /etc/containerd/config.toml
# Should output: SystemdCgroup = true

# Restart and enable containerd
systemctl restart containerd
systemctl enable containerd

# Verify containerd is running
systemctl status containerd
```

### 8.3 Full Production containerd Configuration

```toml
# /etc/containerd/config.toml (production-grade)
version = 2

[metrics]
  address = "127.0.0.1:1338"  # Enable containerd metrics

[plugins."io.containerd.grpc.v1.cri"]
  sandbox_image = "registry.k8s.io/pause:3.9"

  [plugins."io.containerd.grpc.v1.cri".containerd]
    snapshotter = "overlayfs"
    default_runtime_name = "runc"

    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
      runtime_type = "io.containerd.runc.v2"

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        SystemdCgroup = true   # CRITICAL: Must be true for Kubernetes

  [plugins."io.containerd.grpc.v1.cri".registry]
    # Mirror configuration for private registry or pull-through cache
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
        endpoint = ["https://mirror.example.com"]
```

### 8.4 Verify CRI Socket

```bash
# Test that containerd CRI socket is functional
crictl --runtime-endpoint unix:///run/containerd/containerd.sock info

# Configure crictl to use containerd by default
cat > /etc/crictl.yaml << 'EOF'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 30
debug: false
EOF

# Test image pull
crictl pull nginx:alpine
```

---

## 9. Installing kubeadm, kubelet & kubectl

### 9.1 Add Kubernetes Repository

```bash
# Add Kubernetes signing key (v1.29+)
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

# Add Kubernetes apt repository
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
  tee /etc/apt/sources.list.d/kubernetes.list

# Update package index
apt-get update
```

### 9.2 Install Kubernetes Components

```bash
# Install specific version (always pin versions in production!)
K8S_VERSION="1.29.0-1.1"

apt-get install -y \
  kubelet=${K8S_VERSION} \
  kubeadm=${K8S_VERSION} \
  kubectl=${K8S_VERSION}

# CRITICAL: Pin versions to prevent accidental auto-upgrade
apt-mark hold kubelet kubeadm kubectl

# Verify installations
kubeadm version
kubelet --version
kubectl version --client
```

### 9.3 Pre-flight Checks with kubeadm

```bash
# Run pre-flight checks before init (detect problems early)
kubeadm init phase preflight \
  --config /etc/kubernetes/kubeadm-config.yaml

# Or quick check with flags
kubeadm init --dry-run \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12
```

### 9.4 Create kubeadm Configuration File

Using a config file is the **production-grade approach** — it documents every decision and enables reproducible cluster creation:

```yaml
# /etc/kubernetes/kubeadm-config.yaml
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "10.0.0.10"   # This control plane node's IP
  bindPort: 6443
nodeRegistration:
  name: "k8s-cp-1"
  criSocket: "unix:///run/containerd/containerd.sock"
  kubeletExtraArgs:
    node-labels: "node-role.kubernetes.io/control-plane="
    # Enable IPVS mode for kube-proxy
    feature-gates: ""
---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
clusterName: "production-cluster"
kubernetesVersion: "v1.29.0"
controlPlaneEndpoint: "k8s-api-vip:6443"  # HA VIP or DNS for multi-CP
networking:
  serviceSubnet: "10.96.0.0/12"
  podSubnet: "10.244.0.0/16"
  dnsDomain: "cluster.local"
etcd:
  local:
    dataDir: "/var/lib/etcd"
    extraArgs:
      # Performance tuning
      heartbeat-interval: "250"
      election-timeout: "2500"
      # Security
      auto-tls: "false"
      peer-auto-tls: "false"
      # Compaction
      auto-compaction-retention: "8"
      auto-compaction-mode: "revision"
      quota-backend-bytes: "8589934592"  # 8 GB
apiServer:
  certSANs:
    - "k8s-api-vip"
    - "10.0.0.10"
    - "10.0.0.11"
    - "10.0.0.12"
    - "10.0.0.100"
    - "127.0.0.1"
    - "kubernetes"
    - "kubernetes.default"
    - "kubernetes.default.svc"
    - "kubernetes.default.svc.cluster.local"
  extraArgs:
    # Audit logging
    audit-log-path: "/var/log/kubernetes/audit.log"
    audit-log-maxage: "30"
    audit-log-maxbackup: "10"
    audit-log-maxsize: "100"
    # Security
    enable-admission-plugins: "NodeRestriction,PodSecurity"
    encryption-provider-config: "/etc/kubernetes/encryption-config.yaml"
    # OIDC (optional, for SSO)
    # oidc-issuer-url: "https://your-oidc-provider.example.com"
  extraVolumes:
    - name: "audit-log"
      hostPath: "/var/log/kubernetes"
      mountPath: "/var/log/kubernetes"
      writable: true
      pathType: DirectoryOrCreate
    - name: "encryption-config"
      hostPath: "/etc/kubernetes/encryption-config.yaml"
      mountPath: "/etc/kubernetes/encryption-config.yaml"
      readOnly: true
      pathType: File
controllerManager:
  extraArgs:
    # Security
    bind-address: "0.0.0.0"
    # Leader election
    leader-elect: "true"
    leader-elect-lease-duration: "30s"
    leader-elect-renew-deadline: "25s"
    # Performance
    concurrent-deployment-syncs: "10"
    concurrent-replicaset-syncs: "10"
    concurrent-service-syncs: "5"
scheduler:
  extraArgs:
    bind-address: "0.0.0.0"
    leader-elect: "true"
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: "systemd"              # Must match containerd SystemdCgroup=true
clusterDNS:
  - "10.96.0.10"
clusterDomain: "cluster.local"
# Resource management
evictionHard:
  memory.available: "200Mi"
  nodefs.available: "10%"
  imagefs.available: "15%"
evictionSoft:
  memory.available: "500Mi"
  nodefs.available: "15%"
evictionSoftGracePeriod:
  memory.available: "1m30s"
  nodefs.available: "1m30s"
maxPods: 110
# Security
readOnlyPort: 0                      # Disable unauthenticated kubelet port
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
authorization:
  mode: Webhook
# TLS
tlsCertFile: "/var/lib/kubelet/pki/kubelet.crt"
tlsPrivateKeyFile: "/var/lib/kubelet/pki/kubelet.key"
rotateCertificates: true
serverTLSBootstrap: true
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  scheduler: "rr"
  syncPeriod: "30s"
  minSyncPeriod: "2s"
```

### 9.5 Prepare Encryption at Rest

```bash
# Generate a 32-byte random key for etcd encryption
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

cat > /etc/kubernetes/encryption-config.yaml << EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      - identity: {}  # Fallback for unencrypted data
EOF

chmod 600 /etc/kubernetes/encryption-config.yaml
```

---

## 10. Bootstrapping the Control Plane with kubeadm init

### 10.1 Run kubeadm init

```bash
# Run on first control plane node ONLY
kubeadm init \
  --config /etc/kubernetes/kubeadm-config.yaml \
  --upload-certs \    # Uploads certificates to cluster secret for HA join
  2>&1 | tee /root/kubeadm-init.log

# The --upload-certs flag:
# - Stores encrypted control plane certificates in a Secret in kube-system
# - Allows other control plane nodes to download them during join
# - Certificate encryption key expires after 2 hours
```

### 10.2 Expected Output & What It Means

```
[init] Using Kubernetes version: v1.29.0
[preflight] Running pre-flight checks
  ✓ swap disabled
  ✓ conntrack module loaded
  ✓ br_netfilter loaded
  ✓ ip_forward enabled
[certs] Generating all PKI certificates
  Generating "ca" certificate and key
  Generating "apiserver" certificate and key
  Generating "apiserver-kubelet-client" certificate and key
  Generating "front-proxy-ca" certificate and key
  Generating "etcd/ca" certificate and key
  Generating "etcd/server" certificate and key
  Generating "etcd/peer" certificate and key
  Generating "etcd/healthcheck-client" certificate and key
  Generating "apiserver-etcd-client" certificate and key
  ...
[kubeconfig] Writing kubeconfig files:
  /etc/kubernetes/admin.conf
  /etc/kubernetes/controller-manager.conf
  /etc/kubernetes/scheduler.conf
  /etc/kubernetes/kubelet.conf
[control-plane] Creating static Pod manifests for control plane:
  /etc/kubernetes/manifests/kube-apiserver.yaml
  /etc/kubernetes/manifests/kube-controller-manager.yaml
  /etc/kubernetes/manifests/kube-scheduler.yaml
  /etc/kubernetes/manifests/etcd.yaml
[etcd] Starting local etcd...
[wait-control-plane] Waiting for kube-apiserver to come online...
[apiclient] API server is accessible
[bootstrap-token] Configuring bootstrap tokens...
[addons] Installing CoreDNS...
[addons] Installing kube-proxy...

Your Kubernetes control-plane has initialized successfully!
```

### 10.3 Configure kubectl Access

```bash
# Set up kubeconfig for the current user
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# Verify cluster connectivity
kubectl cluster-info
kubectl get nodes

# Expected output: one node in NotReady state (no CNI yet)
# NAME      STATUS     ROLES           AGE   VERSION
# k8s-cp-1  NotReady   control-plane   2m    v1.29.0
```

### 10.4 Save the Join Command

```bash
# The join token is printed at the end of kubeadm init output
# Save it! It expires after 24 hours.

# To regenerate the token + join command later:
kubeadm token create --print-join-command

# For HA: To get the worker join command
kubeadm token create --print-join-command 2>/dev/null

# For HA: To get control plane join command (with cert key)
CERT_KEY=$(kubeadm init phase upload-certs --upload-certs 2>/dev/null | tail -1)
echo "Control plane join command:"
echo "$(kubeadm token create --print-join-command) \
  --control-plane \
  --certificate-key ${CERT_KEY}"
```

### 10.5 Understand What kubeadm init Created

```bash
# Certificate files
ls -la /etc/kubernetes/pki/
# ca.crt, ca.key, apiserver.crt, apiserver.key, etc.

# Static pod manifests (kubelet watches this directory)
ls -la /etc/kubernetes/manifests/
# etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml

# Kubeconfig files for component authentication
ls -la /etc/kubernetes/
# admin.conf  controller-manager.conf  kubelet.conf  scheduler.conf

# Verify control plane Pods are running
kubectl get pods -n kube-system
```

### 10.6 Static Pod Deep Dive

Static Pods are a critical concept introduced during cluster installation:

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml (excerpt)
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 10.0.0.10:6443
spec:
  hostNetwork: true          # Uses host network namespace (not Pod network)
  priorityClassName: system-node-critical
  containers:
    - name: kube-apiserver
      image: registry.k8s.io/kube-apiserver:v1.29.0
      command:
        - kube-apiserver
        - --advertise-address=10.0.0.10
        - --secure-port=6443
        - --etcd-servers=https://127.0.0.1:2379
        - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
        # ... (many more flags)
      volumeMounts:
        - mountPath: /etc/kubernetes/pki
          name: k8s-certs
          readOnly: true
  volumes:
    - hostPath:
        path: /etc/kubernetes/pki
        type: DirectoryOrCreate
      name: k8s-certs
```

**Key insight:** Static Pods are not stored in etcd. kubelet manages them directly from files in `/etc/kubernetes/manifests/`. This allows control plane components to run even before the API server is fully initialized.

---

## 11. Installing the CNI Plugin (Calico / Flannel / Cilium)

Without a CNI plugin, Pods cannot communicate. The control plane node remains `NotReady` until a CNI is installed.

### 11.1 Option A: Calico (Recommended for Production)

```bash
# Install Calico operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

# Configure Calico with your Pod CIDR
cat > calico-custom-resources.yaml << 'EOF'
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
      - blockSize: 26
        cidr: 10.244.0.0/16        # Must match --pod-network-cidr
        encapsulation: VXLANCrossSubnet  # VXLAN within subnet, native routing across
        natOutgoing: Enabled
        nodeSelector: all()
    mtu: 0                          # Auto-detect MTU
    nodeAddressAutodetectionV4:
      interface: "eth.*"            # Auto-detect interface by regex
---
apiVersion: operator.tigera.io/v1
kind: APIServer
metadata:
  name: default
spec: {}
EOF

kubectl create -f calico-custom-resources.yaml

# Wait for Calico to be ready
watch kubectl get pods -n calico-system
```

### 11.2 Option B: Flannel (Simple, Dev/Test)

```bash
# Install Flannel (uses --pod-network-cidr from kubeadm init)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# If you used a custom Pod CIDR, edit the ConfigMap first
kubectl get configmap -n kube-flannel kube-flannel-cfg -o yaml
```

### 11.3 Option C: Cilium (eBPF-based, Maximum Performance)

```bash
# Install Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --fail --remote-name-all \
  https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz
tar xzvf cilium-linux-amd64.tar.gz -C /usr/local/bin

# Install Cilium
cilium install \
  --set ipam.mode=kubernetes \
  --set tunnel=vxlan \
  --set kubeProxyReplacement=strict \  # Replace kube-proxy entirely
  --set k8sServiceHost=k8s-api-vip \
  --set k8sServicePort=6443

# Check status
cilium status --wait
cilium connectivity test  # Full connectivity test suite
```

### 11.4 Verify CNI Installation

```bash
# Node should now show Ready
kubectl get nodes
# NAME      STATUS   ROLES           AGE   VERSION
# k8s-cp-1  Ready    control-plane   5m    v1.29.0

# All system Pods should be Running
kubectl get pods -n kube-system

# Test Pod networking
kubectl run test-pod --image=busybox --restart=Never -- sleep 3600
kubectl exec test-pod -- ping -c 3 8.8.8.8
kubectl delete pod test-pod
```

---

## 12. Joining Worker Nodes

### 12.1 Run on Each Worker Node

```bash
# On each worker node, run the join command from kubeadm init output
# (Get fresh token if it expired):
kubeadm token create --print-join-command

# Join command format:
kubeadm join k8s-api-vip:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  2>&1 | tee /root/kubeadm-join.log
```

### 12.2 What Happens During Node Join

```
1. kubeadm on worker fetches cluster info using bootstrap token
2. Verifies CA cert hash (prevents man-in-the-middle)
3. Creates a CertificateSigningRequest (CSR) for the kubelet
4. kube-controller-manager's NodeApproving controller auto-approves the CSR
5. kubelet receives its signed certificate
6. kubelet registers with API server as a new Node object
7. kube-scheduler can now place Pods on this node
8. CNI plugin's DaemonSet Pod starts on the new node
9. kube-proxy DaemonSet Pod starts on the new node
10. Node transitions to Ready status
```

### 12.3 Verify Workers Joined

```bash
# On control plane
kubectl get nodes -o wide
# NAME           STATUS   ROLES           AGE    VERSION   INTERNAL-IP   OS-IMAGE
# k8s-cp-1       Ready    control-plane   10m    v1.29.0   10.0.0.10     Ubuntu 22.04
# k8s-worker-1   Ready    <none>          2m     v1.29.0   10.0.0.20     Ubuntu 22.04
# k8s-worker-2   Ready    <none>          90s    v1.29.0   10.0.0.21     Ubuntu 22.04

# Label workers for better identification
kubectl label node k8s-worker-1 node-role.kubernetes.io/worker=
kubectl label node k8s-worker-2 node-role.kubernetes.io/worker=

# Verify system pods on all nodes
kubectl get pods -n kube-system -o wide
```

### 12.4 Taint Control Plane Nodes (Production Best Practice)

```bash
# Prevent workload Pods from running on control plane nodes
# (This taint is already set by kubeadm by default)
kubectl taint nodes k8s-cp-1 \
  node-role.kubernetes.io/control-plane:NoSchedule

# Verify taint
kubectl describe node k8s-cp-1 | grep -i taint

# To allow workloads on control plane (single-node dev only):
kubectl taint nodes k8s-cp-1 \
  node-role.kubernetes.io/control-plane:NoSchedule-
```

---

## 13. Deep Dive: Built-in Controllers

Once the cluster is running, `kube-controller-manager` starts all built-in controllers. Here is a detailed examination of each major controller and its role in cluster operations.

### 13.1 ReplicaSet Controller

**Purpose:** Ensures a specified number of Pod replicas are running at all times.

**Reconciliation logic:**
```
desired = replicaset.spec.replicas
current = count(pods matching replicaset.spec.selector)

if current < desired:
  create (desired - current) Pods
elif current > desired:
  delete (current - desired) Pods (oldest first)
```

**Production relevance:** This is what provides self-healing. If a worker node crashes and its Pods die, the ReplicaSet controller detects the deficit and creates replacement Pods on healthy nodes.

```bash
# Observe ReplicaSet controller in action
kubectl create deployment demo --image=nginx --replicas=3
kubectl get replicasets

# Simulate failure: delete a pod
kubectl delete pod $(kubectl get pods -l app=demo -o name | head -1)

# Watch ReplicaSet immediately create a replacement
kubectl get pods -l app=demo -w
```

### 13.2 Deployment Controller

**Purpose:** Manages ReplicaSets to implement declarative updates, rollbacks, and rollout strategies.

**What it manages:**
- Creates a new ReplicaSet when Pod template changes
- Scales up new RS while scaling down old RS (rolling update)
- Maintains rollout history for rollback

```yaml
# Rolling update configuration
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%        # Create up to 25% extra Pods during update
    maxUnavailable: 25%  # Allow up to 25% of Pods to be unavailable
```

**Reconciliation:** Deployment controller watches Deployments and ReplicaSets. It creates/updates/deletes ReplicaSets to achieve the desired state.

### 13.3 Node Controller

**Purpose:** Manages Node lifecycle — monitors health, assigns CIDRs, evicts Pods from unhealthy nodes.

**Health monitoring cycle:**
```
Every 5 seconds (--node-monitor-period):
  Check node heartbeat (NodeStatus.Conditions[Ready])

If no heartbeat for 40 seconds (--node-monitor-grace-period):
  Set node Ready=Unknown

If no heartbeat for 5 minutes (--pod-eviction-timeout):
  Evict all Pods from node
  Mark node taint: node.kubernetes.io/unreachable
```

**CIDR assignment:** When `--allocate-node-cidrs=true`, the Node controller assigns a `/24` subnet from the pod CIDR to each node for Pod IP allocation.

### 13.4 Service Controller

**Purpose:** Manages cloud load balancers for `LoadBalancer`-type Services. Does NOT manage ClusterIP Services (that's kube-proxy).

**Reconciliation:**
- Watches for Services with `type: LoadBalancer`
- Calls cloud provider API to create/update/delete external load balancers
- Writes the allocated external IP back to `Service.status.loadBalancer.ingress`

On bare-metal clusters without a cloud provider, this controller is idle unless you use MetalLB or similar.

### 13.5 Endpoints / EndpointSlice Controller

**Purpose:** Populates Endpoints and EndpointSlice objects by watching Services and Pods.

**Reconciliation:**
```
For each Service with a selector:
  Find all Pods matching the selector
  Filter to only Ready Pods (readinessProbe passing)
  Write their IPs to the EndpointSlice object
```

This is the mechanism by which Services automatically route only to healthy Pods.

### 13.6 Namespace Controller

**Purpose:** Manages namespace lifecycle — handles cascading deletion of all resources when a namespace is deleted.

**Deletion flow:**
1. Namespace set to `Terminating` phase
2. Namespace controller deletes all resources in the namespace (Pods, Services, Deployments, etc.)
3. Finalizers on the namespace are removed
4. Namespace is deleted

### 13.7 Job Controller

**Purpose:** Creates Pods to run batch tasks to completion, with retry logic.

**Reconciliation:**
```
desired = job.spec.completions
active = count(running/pending Pods for this Job)
succeeded = count(completed Pods for this Job)

if succeeded < desired:
  if active < job.spec.parallelism:
    create new Pod

if active Pod fails:
  if retries < job.spec.backoffLimit:
    create replacement Pod
  else:
    mark Job as Failed
```

### 13.8 StatefulSet Controller

**Purpose:** Manages stateful applications that need stable network identities and persistent storage.

**Key differences from Deployment:**
- Pods created in order (pod-0, pod-1, pod-2)
- Each Pod gets a stable, predictable DNS name
- Each Pod gets its own PersistentVolumeClaim (not shared)
- Pods updated in reverse order (pod-2, pod-1, pod-0)

**Dependencies during cluster setup:** StatefulSets require:
- A headless Service (created by you)
- A StorageClass (or static PVs) for PersistentVolumeClaims

### 13.9 DaemonSet Controller

**Purpose:** Ensures one copy of a Pod runs on every node (or selected nodes).

**Used by Kubernetes itself for:**
- `kube-proxy` — Service routing rules on every node
- CNI plugin pods (calico-node, cilium, etc.) — Pod networking on every node
- `node-exporter` — Prometheus node metrics from every node
- Log collectors (Fluentd, Filebeat) — Log shipping from every node

**Reconciliation:**
```
For each node in the cluster (matching nodeSelector/tolerations):
  If no DaemonSet Pod exists on this node → Create one
  If node removed → Delete the Pod

When new node joins the cluster:
  DaemonSet controller immediately creates Pod on new node
```

### 13.10 Garbage Collector

**Purpose:** Deletes orphaned objects whose owner has been deleted (using `ownerReferences`).

**Owner references example:**
```
Deployment → owns → ReplicaSet → owns → Pods
```

When a Deployment is deleted, the Garbage Collector follows owner references to delete the ReplicaSet, then the Pods.

**Deletion policies:**
- **Background:** Owner deleted immediately; dependents deleted asynchronously
- **Foreground:** Owner set to "being deleted"; all dependents deleted first; then owner deleted
- **Orphan:** Owner deleted; dependents' ownerReferences removed (not deleted)

### 13.11 PersistentVolume Controller

**Purpose:** Binds PersistentVolumeClaims (PVCs) to PersistentVolumes (PVs).

**Two modes:**
- **Static provisioning:** Admin creates PVs manually; controller binds matching PVCs
- **Dynamic provisioning:** StorageClass triggers provisioner to create PV on-demand; controller binds

**During cluster installation:** You need a StorageClass for dynamic provisioning. Common options:
- **Cloud:** `gp3` (AWS), `pd-ssd` (GCP), `managed-premium` (Azure)
- **On-prem:** Rook/Ceph, OpenEBS, Longhorn, NFS Provisioner

```bash
# Install local-path provisioner (development)
kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

# Set as default StorageClass
kubectl patch storageclass local-path \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

## 14. Internal Working Concepts: Informers, Work Queues & Reconciliation

### 14.1 The Informer Pattern — Deep Dive

Every controller in kube-controller-manager uses **informers** to watch the API server efficiently without overwhelming it with requests.

```
┌─────────────────────────────────────────────────────────┐
│                    INFORMER INTERNALS                   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │              Reflector                          │   │
│  │  Initial LIST → populates local cache           │   │
│  │  Ongoing WATCH → streams events from API server │   │
│  └────────────────────┬────────────────────────────┘   │
│                       │ ADDED / MODIFIED / DELETED      │
│                       ▼                                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │              DeltaFIFO Queue                    │   │
│  │  Thread-safe queue of change events             │   │
│  └────────────────────┬────────────────────────────┘   │
│                       │                                 │
│          ┌────────────┼───────────────┐                │
│          ▼            ▼               ▼                │
│  ┌──────────────┐  ┌──────────┐  ┌──────────────────┐ │
│  │  Local Cache │  │ OnAdd()  │  │  Event Handlers  │ │
│  │  (Indexer)   │  │ OnUpdate │  │  (user-defined)  │ │
│  │  Serviced    │  │ OnDelete │  └────────┬─────────┘ │
│  │  reads from  │  └──────────┘           │           │
│  │  controllers │                          ▼           │
│  └──────────────┘             ┌──────────────────────┐ │
│                               │     Work Queue       │ │
│                               └──────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

**Resync:** Every 10–30 minutes (configurable), the informer re-lists all resources and fires synthetic "update" events for all cached objects. This ensures no state drift goes undetected.

### 14.2 Work Queue — Rate Limiting & Retry Backoff

The work queue between the informer and the reconciliation worker implements critical reliability features:

```
Event arrives → add key to queue → worker picks up key

If reconciliation succeeds:
  workQueue.Forget(key)    → Remove from rate limiter history

If reconciliation fails:
  workQueue.AddRateLimited(key)
  → Retry with exponential backoff:
    Attempt 1:  5ms
    Attempt 2:  10ms
    Attempt 3:  20ms
    ...
    Max:        1000s (default)
```

**Deduplication:** If 100 Pod events arrive for the same Deployment before the worker processes them, they collapse to a single reconciliation. This is critical during cluster upgrades when many objects change simultaneously.

### 14.3 Reconciliation Loop Implementation Detail

```go
// Pseudocode for a typical controller reconciliation loop
func (c *Controller) Run(workers int, stopCh <-chan struct{}) {
    // Start informers
    go c.podInformer.Run(stopCh)
    go c.serviceInformer.Run(stopCh)

    // Wait for cache sync (don't reconcile on stale data!)
    if !cache.WaitForCacheSync(stopCh,
        c.podInformer.HasSynced,
        c.serviceInformer.HasSynced) {
        return
    }

    // Start N worker goroutines
    for i := 0; i < workers; i++ {
        go wait.Until(c.runWorker, time.Second, stopCh)
    }

    <-stopCh
}

func (c *Controller) runWorker() {
    for c.processNextWorkItem() {}
}

func (c *Controller) processNextWorkItem() bool {
    key, quit := c.workQueue.Get()
    if quit { return false }
    defer c.workQueue.Done(key)

    err := c.reconcile(key.(string))
    if err != nil {
        c.workQueue.AddRateLimited(key)  // Retry with backoff
        return true
    }
    c.workQueue.Forget(key)             // Success; reset backoff
    return true
}
```

---

## 15. Interaction with API Server and etcd

### 15.1 The Inviolable Rule

```
╔══════════════════════════════════════════════════════╗
║  CONTROLLERS NEVER COMMUNICATE DIRECTLY WITH ETCD   ║
║                                                      ║
║  ALL communication flows through kube-apiserver:     ║
║                                                      ║
║  Controller → kube-apiserver → etcd                  ║
║                                                      ║
║  This is not a convention — it is an architectural   ║
║  requirement enforced by network policy and TLS      ║
╚══════════════════════════════════════════════════════╝
```

### 15.2 Why This Rule Exists

| Concern | What kube-apiserver Provides | Without It |
|---|---|---|
| Authentication | Verifies client identity | Any process could write cluster state |
| Authorization (RBAC) | Enforces who can do what | Controllers could overwrite each other |
| Admission Control | Validates/mutates objects | Invalid objects enter cluster state |
| Audit Logging | Records every change | No audit trail for compliance |
| Watch Multiplexing | Fans out to all watchers | Each client would maintain etcd watches |
| Object Versioning | ResourceVersion optimistic locking | Race conditions corrupt state |
| Schema Validation | Validates object structure | Malformed data corrupts etcd |

### 15.3 Communication Flow During Cluster Bootstrap

```
Phase 1: Before API server is up
  kubeadm → writes static Pod files to /etc/kubernetes/manifests/
  kubelet → reads manifests → starts etcd container
  kubelet → reads manifests → starts kube-apiserver container

Phase 2: API server bootstrapping
  kube-apiserver → connects to etcd (using etcd client certs)
  kube-apiserver → starts serving HTTPS on :6443

Phase 3: Controllers come online
  kube-controller-manager → connects to kube-apiserver:6443
  kube-controller-manager → authenticates with /etc/kubernetes/controller-manager.conf
  kube-controller-manager → starts List/Watch for all resource types

Phase 4: Nodes join
  kubelet on worker → connects to kube-apiserver:6443
  kubelet → registers Node object
  kube-controller-manager → detects new Node → allocates Pod CIDR
```

### 15.4 etcd Access Architecture

```
                    ┌──────────────────────┐
                    │      etcd cluster    │
                    │   :2379 (client)     │
                    │   :2380 (peer)       │
                    └──────────┬───────────┘
                               │ mTLS
                               │ (apiserver-etcd-client.crt)
                    ┌──────────▼───────────┐
                    │   kube-apiserver     │
                    │      :6443           │
                    └──────────┬───────────┘
                               │ mTLS (various client certs)
              ┌────────────────┼───────────────────┐
              │                │                   │
     ┌────────▼──────┐  ┌──────▼──────┐  ┌────────▼──────┐
     │  controller-  │  │  scheduler  │  │   kubelet     │
     │  manager      │  │             │  │               │
     └───────────────┘  └─────────────┘  └───────────────┘
```

---

## 16. Leader Election for HA Control Planes

### 16.1 Why Leader Election Is Needed

In a 3-node HA control plane:
- 3 instances of `kube-controller-manager` are running
- If all 3 reconciled simultaneously → race conditions
- Example: All 3 try to create a cloud load balancer → 3 LBs created

Solution: **Only one instance is the active leader.** Others watch and take over if the leader fails.

### 16.2 Lease Object Mechanism

```bash
# View the current leader
kubectl get lease kube-controller-manager -n kube-system -o yaml
```

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  acquireTime: "2025-01-01T10:00:00.000000Z"
  holderIdentity: "k8s-cp-1_abc-uuid-def"   # Current leader
  leaseDurationSeconds: 30                   # How long the lease is valid
  leaseTransitions: 3                        # How many times leadership changed
  renewTime: "2025-01-01T12:30:45.000000Z"   # Last renewal time
```

**Election algorithm:**
```
Leader:
  Every renewDeadline seconds:
    UPDATE lease.renewTime = now()
    if update fails → I am no longer leader → become standby

Standby:
  Watch the Lease object
  if (now() - lease.renewTime) > leaseDuration:
    Try to UPDATE lease.holderIdentity = my-identity
    if update succeeds → I am the new leader
    if update fails (409 conflict) → another standby won → stay standby
```

### 16.3 Leader Election Flags

```bash
# In kube-controller-manager (and kube-scheduler — same mechanism)
--leader-elect=true
--leader-elect-lease-duration=30s      # Lease validity period
--leader-elect-renew-deadline=25s      # Must renew before this deadline
--leader-elect-retry-period=5s         # How often standbys poll
--leader-elect-resource-lock=leases    # Resource type (leases recommended)
--leader-elect-resource-namespace=kube-system
--leader-elect-resource-name=kube-controller-manager
```

### 16.4 Failover Timeline

```
T+0s:   Current leader crashes (k8s-cp-1)
T+5s:   Standby instances detect lease not renewed (check every 5s)
T+30s:  Lease expires (leaseDurationSeconds=30)
T+35s:  First standby wins the lease election → becomes new leader
T+40s:  New leader (k8s-cp-2) starts reconciling all controllers
T+45s:  Full control plane functionality restored
```

**Impact during failover window (T+0 to T+40):**
- Existing workloads continue running (kube-proxy rules still in place)
- No new Pods scheduled (scheduler also doing leader election)
- No Endpoint updates (new Pods added/removed won't update LB rules)
- No cloud LB changes (LoadBalancer Services not updated)

---

## 17. High-Availability Control Plane Setup

### 17.1 Add Second Control Plane Node

```bash
# On k8s-cp-2 (after completing all steps through Section 9)
# First, generate a fresh certificate key on the EXISTING control plane:
# (Run on k8s-cp-1)
kubeadm init phase upload-certs --upload-certs
# Output: <certificate-key>  (valid for 2 hours)

# Get join command (run on k8s-cp-1)
kubeadm token create --print-join-command
# Output: kubeadm join k8s-api-vip:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

# On k8s-cp-2: Run control plane join
kubeadm join k8s-api-vip:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <certificate-key> \
  2>&1 | tee /root/kubeadm-join-cp.log

# Set up kubectl on k8s-cp-2
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

### 17.2 Load Balancer Setup for HA API Server

```bash
# Using HAProxy as an L4 load balancer for the API server VIP
# Install HAProxy on a dedicated LB node (or use cloud LB)
apt-get install -y haproxy

cat > /etc/haproxy/haproxy.cfg << 'EOF'
global
    log /dev/log local0 warning
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin
    maxconn 4096
    daemon

defaults
    log     global
    mode    tcp
    timeout connect 10s
    timeout client  86400s
    timeout server  86400s

frontend kubernetes-apiserver
    bind *:6443
    default_backend kubernetes-apiserver

backend kubernetes-apiserver
    balance roundrobin
    option tcp-check
    server k8s-cp-1 10.0.0.10:6443 check check-ssl verify none inter 10000
    server k8s-cp-2 10.0.0.11:6443 check check-ssl verify none inter 10000
    server k8s-cp-3 10.0.0.12:6443 check check-ssl verify none inter 10000

listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
EOF

systemctl restart haproxy
systemctl enable haproxy
```

### 17.3 HA etcd Topology Verification

```bash
# Verify etcd cluster health across all control plane nodes
kubectl exec -n kube-system etcd-k8s-cp-1 -- \
  etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    member list

# Expected output:
# <id>, started, k8s-cp-1, https://10.0.0.10:2380, https://10.0.0.10:2379, false
# <id>, started, k8s-cp-2, https://10.0.0.11:2380, https://10.0.0.11:2379, false
# <id>, started, k8s-cp-3, https://10.0.0.12:2380, https://10.0.0.12:2379, false

# Check etcd cluster health
kubectl exec -n kube-system etcd-k8s-cp-1 -- \
  etcdctl \
    --endpoints=https://10.0.0.10:2379,https://10.0.0.11:2379,https://10.0.0.12:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    endpoint health
```

---

## 18. Performance Tuning

### 18.1 API Server Performance Flags

```bash
# In /etc/kubernetes/manifests/kube-apiserver.yaml
- --max-requests-inflight=800         # Max concurrent non-mutating requests
- --max-mutating-requests-inflight=400 # Max concurrent mutating requests
- --request-timeout=60s               # Default request timeout
- --watch-cache-sizes=pod#5000,node#1000,replicaset#2000  # Cache sizes
- --default-watch-cache-size=100      # Default cache size per resource
- --etcd-compaction-interval=5m       # How often to compact etcd
```

### 18.2 etcd Performance Tuning

```yaml
# In /etc/kubernetes/manifests/etcd.yaml
extraArgs:
  # Disk I/O
  quota-backend-bytes: "8589934592"   # 8 GB database size limit
  auto-compaction-mode: "revision"
  auto-compaction-retention: "1000"   # Keep last 1000 revisions
  
  # Network (for high-latency environments)
  heartbeat-interval: "250"           # ms (default 100ms; increase for unstable networks)
  election-timeout: "2500"            # ms (must be 10x heartbeat)
  
  # Snapshots
  snapshot-count: "10000"             # Snapshot every 10000 transactions
```

**etcd disk performance benchmark:**

```bash
# Run etcd's built-in fio benchmark
fio --rw=write --ioengine=sync --fdatasync=1 \
    --directory=/var/lib/etcd \
    --size=22m \
    --bs=2300 \
    --name=etcd-benchmark

# etcd requires < 10ms fsync latency (p99)
# SSDs typically: 1-3ms
# HDDs typically: 5-15ms (may cause etcd timeouts)
# NVMe: <1ms (ideal)
```

### 18.3 Controller Manager Concurrency

```yaml
# In /etc/kubernetes/manifests/kube-controller-manager.yaml
extraArgs:
  concurrent-deployment-syncs: "10"
  concurrent-replicaset-syncs: "10"
  concurrent-service-syncs: "5"
  concurrent-endpoint-syncs: "10"
  concurrent-endpointslice-syncs: "10"
  concurrent-job-syncs: "5"
  concurrent-namespace-syncs: "10"
  concurrent-statefulset-syncs: "5"
  concurrent-daemonset-syncs: "5"
  concurrent-gc-syncs: "20"
  
  # Resync periods (reduce API server load)
  min-resync-period: "12h"
  
  # Endpoint update batching (prevents thundering herd)
  endpoint-updates-batch-period: "500ms"
  endpointslice-updates-batch-period: "500ms"
```

### 18.4 kubelet Performance

```yaml
# In KubeletConfiguration
maxPods: 110                           # Max Pods per node (default 110)
podsPerCore: 0                         # Disable per-core limit (0 = disabled)
kubeReserved:
  cpu: "200m"
  memory: "200Mi"
  ephemeral-storage: "1Gi"
systemReserved:
  cpu: "200m"
  memory: "200Mi"
  ephemeral-storage: "1Gi"
kubeReservedCgroup: "/kubepods"
enforceNodeAllocatable:
  - "pods"
  - "kube-reserved"
  - "system-reserved"
imageGCHighThresholdPercent: 85        # Start GC when disk 85% full
imageGCLowThresholdPercent: 80         # Stop GC when disk 80% full
```

### 18.5 Resource Limits for Control Plane Pods

```yaml
# Add to static pod manifests
resources:
  requests:
    cpu: "1000m"
    memory: "1Gi"
  limits:
    cpu: "4000m"
    memory: "8Gi"
```

---

## 19. Security Hardening Practices

### 19.1 RBAC Hardening

```bash
# Remove cluster-admin from system:masters group in new clusters
# (Not possible post-install — design carefully pre-install)

# Check who has cluster-admin
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name == "cluster-admin") | .subjects'

# Create a restricted admin role (no secret access)
cat << 'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: restricted-admin
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: []  # No secret access
EOF

# Audit all ClusterRoleBindings quarterly
kubectl get clusterrolebindings -A
```

### 19.2 Pod Security Standards

```bash
# Enable Pod Security Standards at namespace level
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# Policy levels:
# privileged: No restrictions
# baseline:   Prevents known privilege escalation
# restricted: Heavily restricted; requires non-root, no host networking, etc.
```

### 19.3 NetworkPolicy — Default Deny

```yaml
# Apply default-deny to all production namespaces FIRST
# then explicitly allow required traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}    # Applies to ALL pods
  policyTypes:
    - Ingress
    - Egress
---
# Then allow DNS (required for all Pods)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### 19.4 TLS Certificate Rotation

```bash
# Check certificate expiration dates
kubeadm certs check-expiration

# Output:
# CERTIFICATE                EXPIRES                  RESIDUAL TIME  CERTIFICATE AUTHORITY
# admin.conf                 Jan 01, 2026 00:00 UTC   364d           ca
# apiserver                  Jan 01, 2026 00:00 UTC   364d           ca
# etcd-server                Jan 01, 2026 00:00 UTC   364d           etcd-ca

# Renew all certificates (run annually or before 30-day expiry)
kubeadm certs renew all

# Restart control plane components after renewal
for component in kube-apiserver kube-controller-manager kube-scheduler etcd; do
  kubectl -n kube-system delete pod -l component=$component
done
```

### 19.5 Audit Logging Policy

```yaml
# /etc/kubernetes/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log everything at metadata level (who did what, but not request body)
  - level: Metadata
    omitStages:
      - RequestReceived

  # Log request and response for sensitive resources
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets", "configmaps", "serviceaccounts"]
      - group: "rbac.authorization.k8s.io"
        resources: ["*"]

  # Don't log health checks and metrics
  - level: None
    users: ["system:serviceaccount:kube-system:kube-proxy"]
    verbs: ["watch"]
  - level: None
    nonResourceURLs:
      - /healthz
      - /readyz
      - /livez
      - /metrics
```

### 19.6 Secrets Encryption Verification

```bash
# Verify secrets are encrypted in etcd
kubectl create secret generic test-secret \
  --from-literal=key=supersecretvalue

# Check if it's encrypted in etcd (should NOT show plain text)
kubectl exec -n kube-system etcd-k8s-cp-1 -- \
  etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    get /registry/secrets/default/test-secret | \
    strings | grep -i "k8s:enc:aescbc"
# Should output "k8s:enc:aescbc:v1:key1" indicating encryption

# Clean up
kubectl delete secret test-secret
```

### 19.7 Service Account Token Security

```bash
# Disable auto-mounting of service account tokens (cluster-wide default)
# Only enable where needed
kubectl patch serviceaccount default \
  -p '{"automountServiceAccountToken": false}'

# For workloads that need API access, create dedicated service accounts
kubectl create serviceaccount my-app-sa -n production

# Bind minimal required permissions
kubectl create rolebinding my-app-binding \
  --role=my-app-role \
  --serviceaccount=production:my-app-sa \
  -n production
```

---

## 20. Monitoring & Observability

### 20.1 Install Prometheus Stack

```bash
# Add Prometheus Community Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install kube-prometheus-stack (Prometheus + Grafana + AlertManager + node-exporter)
helm install kube-prometheus \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=15d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.storageClassName=local-path \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --set grafana.adminPassword=YourSecurePassword \
  --set alertmanager.alertmanagerSpec.storage.volumeClaimTemplate.spec.storageClassName=local-path

# Verify installation
kubectl get pods -n monitoring
kubectl get svc -n monitoring
```

### 20.2 Key Prometheus Metrics

| Metric | Component | Description |
|---|---|---|
| `apiserver_request_total` | kube-apiserver | Total API requests by verb/resource/code |
| `apiserver_request_duration_seconds` | kube-apiserver | API server request latency histogram |
| `etcd_server_proposals_committed_total` | etcd | etcd consensus proposals committed |
| `etcd_disk_wal_fsync_duration_seconds` | etcd | WAL fsync latency (should be <10ms p99) |
| `etcd_server_leader_changes_seen_total` | etcd | Number of leader changes (should be low) |
| `workqueue_depth{name="deployment"}` | controller-manager | Deployment controller queue depth |
| `workqueue_work_duration_seconds` | controller-manager | Time to process each work item |
| `scheduler_scheduling_attempt_duration_seconds` | kube-scheduler | Pod scheduling latency |
| `scheduler_pending_pods` | kube-scheduler | Pods waiting to be scheduled |
| `kubelet_node_name` | kubelet | Node registration status |
| `kubelet_running_pods` | kubelet | Currently running Pods |
| `kubelet_pod_start_duration_seconds` | kubelet | Pod startup latency |
| `node_cpu_seconds_total` | node-exporter | CPU usage per node |
| `node_memory_MemAvailable_bytes` | node-exporter | Available memory per node |
| `node_disk_io_time_seconds_total` | node-exporter | Disk I/O saturation |

### 20.3 Critical Prometheus Alerts

```yaml
groups:
  - name: kubernetes-cluster-health
    rules:
      # Control plane availability
      - alert: KubeAPIServerDown
        expr: absent(up{job="apiserver"} == 1)
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Kubernetes API server unreachable"

      - alert: EtcdInsufficientMembers
        expr: count(etcd_server_id) < 2
        for: 3m
        labels:
          severity: critical
        annotations:
          summary: "etcd cluster has fewer than 2 healthy members"

      - alert: EtcdHighFsyncDuration
        expr: |
          histogram_quantile(0.99,
            rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])
          ) > 0.01
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "etcd fsync latency p99 > 10ms — disk performance degraded"

      - alert: KubeNodeNotReady
        expr: kube_node_status_condition{condition="Ready",status="true"} == 0
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Node {{ $labels.node }} is NotReady"

      - alert: KubeControllerManagerDown
        expr: absent(up{job="kube-controller-manager"} == 1)
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "kube-controller-manager is down"

      - alert: HighAPIServerLatency
        expr: |
          histogram_quantile(0.99,
            rate(apiserver_request_duration_seconds_bucket{verb!~"WATCH|CONNECT"}[5m])
          ) > 1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "API server p99 latency > 1s"

      # Certificate expiration
      - alert: KubernetesCertExpiringSoon
        expr: |
          apiserver_client_certificate_expiration_seconds_count{job="apiserver"} > 0
          and
          histogram_quantile(0.01,
            sum by (job) (rate(apiserver_client_certificate_expiration_seconds_bucket{job="apiserver"}[5m]))
          ) < 604800  # 7 days
        labels:
          severity: warning
        annotations:
          summary: "Kubernetes client certificate expiring in < 7 days"
```

### 20.4 Metrics Endpoints Summary

| Component | Metrics URL | Port |
|---|---|---|
| kube-apiserver | `https://<cp>:6443/metrics` | 6443 |
| kube-controller-manager | `https://<cp>:10257/metrics` | 10257 |
| kube-scheduler | `https://<cp>:10259/metrics` | 10259 |
| etcd | `https://<cp>:2381/metrics` | 2381 |
| kubelet | `https://<node>:10250/metrics` | 10250 |
| kube-proxy | `http://<node>:10249/metrics` | 10249 |
| CoreDNS | `http://<pod>:9153/metrics` | 9153 |
| containerd | `http://127.0.0.1:1338/metrics` | 1338 |

---

## 21. Troubleshooting — Real kubectl Commands

### 21.1 Cluster-Wide Health Check

```bash
# Quick cluster overview
kubectl get nodes -o wide
kubectl get pods -n kube-system
kubectl get componentstatuses  # (deprecated but still useful)

# Check events cluster-wide (sorted by time)
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -50

# Check control plane component health
kubectl get pods -n kube-system -l tier=control-plane

# API server health
curl -k https://localhost:6443/healthz
curl -k https://localhost:6443/readyz
curl -k https://localhost:6443/livez
```

### 21.2 Pods Not Created

```bash
# Check namespace events
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Check if Deployment exists
kubectl get deployment <name> -n <namespace>
kubectl describe deployment <name> -n <namespace>

# Check if ReplicaSet was created
kubectl get replicasets -n <namespace>
kubectl describe replicaset -n <namespace> <rs-name>

# Check for resource quota preventing pod creation
kubectl describe resourcequota -n <namespace>
kubectl describe limitrange -n <namespace>

# Check for pod scheduling failures
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pending-pod> -n <namespace>
# Look for: "0/3 nodes are available: 3 Insufficient memory"
# Look for: "0/3 nodes are available: 3 node(s) had taint"

# Check scheduler logs
kubectl logs -n kube-system -l component=kube-scheduler --tail=100

# Check if node has sufficient resources
kubectl describe node <node-name> | grep -A 10 "Allocated resources"
```

### 21.3 Deployment Stuck

```bash
# Check rollout status
kubectl rollout status deployment/<name> -n <namespace>

# Get detailed rollout history
kubectl rollout history deployment/<name> -n <namespace>

# Compare old and new ReplicaSets
kubectl get replicasets -n <namespace> -l app=<app-name>

# Describe the new (stuck) ReplicaSet
kubectl describe replicaset -n <namespace> <new-rs-name>

# Check why new Pods aren't becoming Ready
kubectl get pods -n <namespace> -l app=<app-name>
kubectl describe pod <new-pod-name> -n <namespace>

# Check for image pull errors
kubectl get events -n <namespace> | grep -i "failed\|error\|pull"

# Check readiness probe configuration
kubectl get deployment <name> -n <namespace> \
  -o jsonpath='{.spec.template.spec.containers[*].readinessProbe}'

# Rollback if stuck
kubectl rollout undo deployment/<name> -n <namespace>

# Force rollout (trigger new rollout without changing image)
kubectl rollout restart deployment/<name> -n <namespace>
```

### 21.4 Node NotReady

```bash
# Identify NotReady nodes
kubectl get nodes | grep NotReady

# Get node details
kubectl describe node <node-name>
# Look for:
#   Conditions: Ready=False/Unknown
#   Events: NodeNotReady, NetworkPlugin not ready
#   Allocated resources: check for over-allocation

# Check kubelet status on the node (SSH into node)
ssh <node-ip>
systemctl status kubelet
journalctl -u kubelet -n 100 --no-pager | grep -i "error\|fail"

# Common kubelet error patterns:
# "network plugin is not ready": CNI not installed/crashing
# "failed to connect to apiserver": network issue or cert problem
# "cgroup driver mismatch": containerd/kubelet cgroupDriver mismatch
# "no such file or directory": containerd socket missing

# Check containerd
systemctl status containerd
journalctl -u containerd -n 50 --no-pager

# Check CNI pods on the problem node
kubectl get pods -n kube-system -o wide | grep <node-name>

# Cordon node (prevent new scheduling while investigating)
kubectl cordon <node-name>

# Drain node (safely evict all pods)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# After fixing, uncordon the node
kubectl uncordon <node-name>
```

### 21.5 etcd Health Issues

```bash
# Check etcd pod logs
kubectl logs -n kube-system etcd-<node-name> --tail=100

# Access etcdctl from within the cluster
kubectl exec -n kube-system etcd-k8s-cp-1 -- \
  etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    endpoint status

# Check etcd database size
kubectl exec -n kube-system etcd-k8s-cp-1 -- \
  etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    endpoint status --write-out=table

# If etcd database is fragmented, defragment
kubectl exec -n kube-system etcd-k8s-cp-1 -- \
  etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    defrag
```

### 21.6 Certificate Problems

```bash
# Check all certificate expiration dates
kubeadm certs check-expiration

# Test API server certificate
openssl s_client -connect <apiserver-ip>:6443 </dev/null 2>/dev/null | \
  openssl x509 -noout -dates

# Decode a specific certificate
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | \
  grep -A 3 "Validity\|Subject Alt"

# Check if kubelet certificate is expired
openssl x509 -in /var/lib/kubelet/pki/kubelet.crt -noout -dates
```

---

## 22. Disaster Recovery Concepts

### 22.1 Stateless Controller Manager

All controllers in `kube-controller-manager` are completely stateless. Their "memory" is in etcd, accessed through the API server. This means:

- Crashing and restarting `kube-controller-manager` is **always safe**
- On restart, it re-Lists all resources and re-reconciles everything
- No data is lost when it crashes
- Multiple restarts in quick succession are idempotent

### 22.2 etcd Backup Strategy

etcd is the **only stateful component**. Everything else can be reconstructed from etcd.

```bash
#!/bin/bash
# /etc/cron.d/etcd-backup — Run hourly
# Production etcd backup script

BACKUP_DIR="/backup/etcd"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
ETCD_ENDPOINTS="https://127.0.0.1:2379"
ETCD_CACERT="/etc/kubernetes/pki/etcd/ca.crt"
ETCD_CERT="/etc/kubernetes/pki/etcd/server.crt"
ETCD_KEY="/etc/kubernetes/pki/etcd/server.key"

mkdir -p ${BACKUP_DIR}

# Take snapshot
ETCDCTL_API=3 etcdctl snapshot save \
  --endpoints=${ETCD_ENDPOINTS} \
  --cacert=${ETCD_CACERT} \
  --cert=${ETCD_CERT} \
  --key=${ETCD_KEY} \
  ${BACKUP_DIR}/etcd-snapshot-${TIMESTAMP}.db

# Verify snapshot integrity
ETCDCTL_API=3 etcdctl snapshot status \
  ${BACKUP_DIR}/etcd-snapshot-${TIMESTAMP}.db \
  --write-out=table

# Compress and upload to object storage
gzip ${BACKUP_DIR}/etcd-snapshot-${TIMESTAMP}.db
aws s3 cp ${BACKUP_DIR}/etcd-snapshot-${TIMESTAMP}.db.gz \
  s3://my-cluster-backups/etcd/

# Prune old local backups (keep 48 hours)
find ${BACKUP_DIR} -name "*.db.gz" -mmin +2880 -delete

echo "Backup completed: etcd-snapshot-${TIMESTAMP}.db.gz"
```

### 22.3 etcd Restore Procedure

```bash
# SCENARIO: All control plane nodes lost, etcd data lost
# Restore from latest snapshot

# Step 1: Stop kube-apiserver (move manifests out)
mv /etc/kubernetes/manifests /etc/kubernetes/manifests.bak

# Step 2: Restore etcd from snapshot
ETCDCTL_API=3 etcdctl snapshot restore \
  /backup/etcd/etcd-snapshot-latest.db \
  --data-dir=/var/lib/etcd-restore \
  --name=k8s-cp-1 \
  --initial-cluster="k8s-cp-1=https://10.0.0.10:2380" \
  --initial-cluster-token=etcd-cluster-restore-1 \
  --initial-advertise-peer-urls=https://10.0.0.10:2380

# Step 3: Move restored data to etcd data directory
mv /var/lib/etcd /var/lib/etcd.old
mv /var/lib/etcd-restore /var/lib/etcd
chown -R etcd:etcd /var/lib/etcd

# Step 4: Restore API server
mv /etc/kubernetes/manifests.bak /etc/kubernetes/manifests

# Step 5: Verify cluster comes back
kubectl get nodes
kubectl get pods -n kube-system
```

### 22.4 Cluster Backup Beyond etcd

```bash
# Velero: Comprehensive cluster backup (namespaces, PVCs, etc.)
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm install velero vmware-tanzu/velero \
  --namespace velero \
  --create-namespace \
  --set configuration.provider=aws \
  --set configuration.backupStorageLocation.bucket=my-velero-backups \
  --set configuration.volumeSnapshotLocation.config.region=us-east-1

# Create scheduled backup
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces production,staging \
  --ttl 240h  # Keep 10 days

# Manual backup
velero backup create pre-upgrade-backup --wait

# Restore
velero restore create --from-backup pre-upgrade-backup
```

---

## 23. Cluster Upgrade Strategy

### 23.1 Upgrade Workflow (kubeadm)

```bash
# Rule: Never skip minor versions (1.27 → 1.29 requires 1.27 → 1.28 → 1.29)

# Step 1: Check available versions
apt-cache madison kubeadm | head -5

# Step 2: Unhold and upgrade kubeadm
apt-mark unhold kubeadm
apt-get install -y kubeadm=1.30.0-1.1
apt-mark hold kubeadm

# Step 3: Check upgrade plan
kubeadm upgrade plan

# Step 4: Apply upgrade on first control plane node
kubeadm upgrade apply v1.30.0

# Step 5: Drain and upgrade each control plane node
kubectl drain k8s-cp-1 --ignore-daemonsets --delete-emptydir-data

apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.30.0-1.1 kubectl=1.30.0-1.1
apt-mark hold kubelet kubectl

systemctl daemon-reload
systemctl restart kubelet

kubectl uncordon k8s-cp-1

# Step 6: Upgrade additional control plane nodes
# kubeadm upgrade node (instead of apply)
kubeadm upgrade node

# Step 7: Upgrade worker nodes (one at a time)
kubectl drain k8s-worker-1 --ignore-daemonsets --delete-emptydir-data

# On the worker node:
apt-mark unhold kubelet kubectl kubeadm
apt-get install -y kubelet=1.30.0-1.1 kubectl=1.30.0-1.1 kubeadm=1.30.0-1.1
apt-mark hold kubelet kubectl kubeadm
kubeadm upgrade node
systemctl daemon-reload
systemctl restart kubelet

# On control plane:
kubectl uncordon k8s-worker-1
```

---

## 24. Comparison: kubeadm vs Other Install Methods

| Tool | Complexity | Production Ready | Features | Best For |
|---|---|---|---|---|
| **kubeadm** | Medium | Yes | Full control, HA support | General production |
| **k3s** | Low | Limited | Lightweight, bundles everything | Edge, IoT, dev |
| **k0s** | Low-Medium | Yes | Zero-friction, single binary | Air-gapped, minimal |
| **minikube** | Very Low | No | Local only | Local development |
| **kind** | Low | No | Docker-based | CI testing |
| **Rancher RKE/RKE2** | Low-Medium | Yes | Opinionated, enterprise features | Rancher ecosystems |
| **Kubespray (Ansible)** | High | Yes | Very flexible | Large enterprises |
| **Talos Linux** | Medium | Yes | Immutable OS, GitOps-first | Security-focused |
| **EKS/GKE/AKS (managed)** | Low | Yes (managed) | No control plane mgmt | Cloud-native orgs |
| **OpenShift** | High | Yes (enterprise) | Full platform | Enterprise RHEL shops |

### kubeadm Advantages

- **Official Kubernetes tooling** — maintained by sig-cluster-lifecycle
- **Maximum flexibility** — supports all Kubernetes configuration options
- **Industry standard** — most tutorials, documentation, and operators assume kubeadm
- **HA support** — built-in support for stacked and external etcd HA topologies
- **Upgrade path** — `kubeadm upgrade` handles the upgrade ceremony

### kubeadm Disadvantages

- Manual node OS preparation required
- No built-in storage provisioner
- No built-in load balancer for bare metal
- More operational overhead vs managed solutions

---

## 25. Comparison: kube-apiserver vs kube-scheduler vs kube-controller-manager

| Dimension | kube-apiserver | kube-scheduler | kube-controller-manager |
|---|---|---|---|
| **Primary Function** | REST API gateway to cluster state | Assign Pods to Nodes | Run reconciliation loops for all resources |
| **State Storage** | Owns etcd (only component to write etcd) | Stateless — reads from API server | Stateless — reads/writes via API server |
| **HA Mode** | Active-Active (all instances serve requests) | Active-Passive (leader election) | Active-Passive (leader election) |
| **Failure Impact** | **Critical**: Cluster management stops | Pending Pods not scheduled | Resources not reconciled |
| **Performance Bottleneck** | Request throughput, watch fan-out | Scheduling throughput (100s of Pods/sec) | Controller concurrency, queue depth |
| **Key Config File** | `kube-apiserver.yaml` static pod | `kube-scheduler.yaml` static pod | `kube-controller-manager.yaml` static pod |
| **Port** | 6443 | 10259 | 10257 |
| **Certificate** | `apiserver.crt` | `scheduler.conf` kubeconfig | `controller-manager.conf` kubeconfig |
| **etcd Access** | Yes — directly | No | No |
| **Cloud Provider Integration** | No | No (uses NodeSelector) | Yes (LoadBalancer, PV provisioning) |
| **Leader Election Lock** | N/A | `kube-scheduler` Lease | `kube-controller-manager` Lease |
| **Worker Goroutines** | Managed by Go HTTP | Scheduling threads (--parallelism) | Per-controller concurrency flags |

---

## 26. ASCII Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║             KUBERNETES CLUSTER — COMPLETE INSTALLATION ARCHITECTURE             ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║                                                                                  ║
║  ┌──────────────────────────────────────────────────────────────────────────┐   ║
║  │                   CONTROL PLANE (3-node HA)                              │   ║
║  │                                                                          │   ║
║  │  ┌────────────────────────────────────────────────────────────────────┐  │   ║
║  │  │  HAProxy / Cloud LB (VIP: 10.0.0.100:6443)                        │  │   ║
║  │  └───────┬──────────────────────┬─────────────────────┬──────────────┘  │   ║
║  │          │                      │                     │                 │   ║
║  │  ┌───────▼────────┐  ┌──────────▼─────────┐  ┌───────▼────────┐        │   ║
║  │  │  k8s-cp-1      │  │  k8s-cp-2          │  │  k8s-cp-3      │        │   ║
║  │  │  10.0.0.10     │  │  10.0.0.11         │  │  10.0.0.12     │        │   ║
║  │  │                │  │                    │  │                │        │   ║
║  │  │ ┌────────────┐ │  │ ┌────────────────┐ │  │ ┌────────────┐ │        │   ║
║  │  │ │kube-api    │ │  │ │ kube-apiserver │ │  │ │kube-api    │ │        │   ║
║  │  │ │server:6443 │ │  │ │    :6443       │ │  │ │server:6443 │ │        │   ║
║  │  │ └─────┬──────┘ │  │ └───────┬────────┘ │  │ └─────┬──────┘ │        │   ║
║  │  │       │        │  │         │           │  │       │        │        │   ║
║  │  │ ┌─────▼──────┐ │  │ ┌───────▼────────┐ │  │ ┌─────▼──────┐ │        │   ║
║  │  │ │   etcd     │◄├──┼─►  etcd (leader) │◄├──┼─►   etcd     │ │        │   ║
║  │  │ │  :2379     │ │  │ │   :2379        │ │  │ │  :2379     │ │        │   ║
║  │  │ │  :2380     │◄├──┼─►   :2380       │◄├──┼─►   :2380    │ │        │   ║
║  │  │ └────────────┘ │  │ └────────────────┘ │  │ └────────────┘ │        │   ║
║  │  │                │  │                    │  │                │        │   ║
║  │  │ ┌────────────┐ │  │ ┌────────────────┐ │  │ ┌────────────┐ │        │   ║
║  │  │ │controller- │ │  │ │  controller-   │ │  │ │controller- │ │        │   ║
║  │  │ │manager     │ │  │ │  manager       │ │  │ │manager     │ │        │   ║
║  │  │ │(standby)   │ │  │ │  (LEADER)      │ │  │ │(standby)   │ │        │   ║
║  │  │ └────────────┘ │  │ └────────────────┘ │  │ └────────────┘ │        │   ║
║  │  │                │  │                    │  │                │        │   ║
║  │  │ ┌────────────┐ │  │ ┌────────────────┐ │  │ ┌────────────┐ │        │   ║
║  │  │ │ scheduler  │ │  │ │   scheduler    │ │  │ │ scheduler  │ │        │   ║
║  │  │ │ (standby)  │ │  │ │  (LEADER)      │ │  │ │ (standby)  │ │        │   ║
║  │  │ └────────────┘ │  │ └────────────────┘ │  │ └────────────┘ │        │   ║
║  │  │                │  │                    │  │                │        │   ║
║  │  │ kubelet        │  │  kubelet           │  │  kubelet       │        │   ║
║  │  │ CoreDNS        │  │                    │  │  CoreDNS       │        │   ║
║  │  └────────────────┘  └────────────────────┘  └────────────────┘        │   ║
║  └──────────────────────────────────────────────────────────────────────────┘   ║
║                                                                                  ║
║  ┌──────────────────────────────────────────────────────────────────────────┐   ║
║  │                        WORKER NODES                                     │   ║
║  │                                                                          │   ║
║  │  ┌────────────────────┐  ┌────────────────────┐  ┌────────────────────┐ │   ║
║  │  │  k8s-worker-1      │  │  k8s-worker-2      │  │  k8s-worker-3      │ │   ║
║  │  │  10.0.0.20         │  │  10.0.0.21         │  │  10.0.0.22         │ │   ║
║  │  │                    │  │                    │  │                    │ │   ║
║  │  │  ┌──────────────┐  │  │  ┌──────────────┐  │  │  ┌──────────────┐  │ │   ║
║  │  │  │    kubelet   │  │  │  │    kubelet   │  │  │  │    kubelet   │  │ │   ║
║  │  │  │   :10250     │  │  │  │   :10250     │  │  │  │   :10250     │  │ │   ║
║  │  │  └──────────────┘  │  │  └──────────────┘  │  │  └──────────────┘  │ │   ║
║  │  │                    │  │                    │  │                    │ │   ║
║  │  │  ┌──────────────┐  │  │  ┌──────────────┐  │  │  ┌──────────────┐  │ │   ║
║  │  │  │  kube-proxy  │  │  │  │  kube-proxy  │  │  │  │  kube-proxy  │  │ │   ║
║  │  │  │ (iptables/   │  │  │  │ (iptables/   │  │  │  │ (iptables/   │  │ │   ║
║  │  │  │  IPVS rules) │  │  │  │  IPVS rules) │  │  │  │  IPVS rules) │  │ │   ║
║  │  │  └──────────────┘  │  │  └──────────────┘  │  │  └──────────────┘  │ │   ║
║  │  │                    │  │                    │  │                    │ │   ║
║  │  │  ┌──────────────┐  │  │  ┌──────────────┐  │  │  ┌──────────────┐  │ │   ║
║  │  │  │  calico-node │  │  │  │  calico-node │  │  │  │  calico-node │  │ │   ║
║  │  │  │  (CNI pod)   │  │  │  │  (CNI pod)   │  │  │  │  (CNI pod)   │  │ │   ║
║  │  │  └──────────────┘  │  │  └──────────────┘  │  │  └──────────────┘  │ │   ║
║  │  │                    │  │                    │  │                    │ │   ║
║  │  │  ┌──────────────┐  │  │  ┌──────────────┐  │  │  ┌──────────────┐  │ │   ║
║  │  │  │  App Pod A   │  │  │  │  App Pod B   │  │  │  │  App Pod C   │  │ │   ║
║  │  │  │  (nginx)     │  │  │  │  (nginx)     │  │  │  │  (nginx)     │  │ │   ║
║  │  │  └──────────────┘  │  │  └──────────────┘  │  │  └──────────────┘  │ │   ║
║  │  │                    │  │                    │  │                    │ │   ║
║  │  │  containerd        │  │  containerd        │  │  containerd        │ │   ║
║  │  └────────────────────┘  └────────────────────┘  └────────────────────┘ │   ║
║  └──────────────────────────────────────────────────────────────────────────┘   ║
║                                                                                  ║
║  ┌──────────────────────────────────────────────────────────────────────────┐   ║
║  │                    STORAGE & NETWORKING                                  │   ║
║  │                                                                          │   ║
║  │  Pod Network: 10.244.0.0/16 (Calico VXLAN overlay)                      │   ║
║  │  Service Network: 10.96.0.0/12 (virtual IPs via iptables/IPVS)          │   ║
║  │  Node Network: 10.0.0.0/16 (physical/VM network)                        │   ║
║  │  StorageClass: local-path / Rook-Ceph / NFS                             │   ║
║  └──────────────────────────────────────────────────────────────────────────┘   ║
╚══════════════════════════════════════════════════════════════════════════════════╝
```

---

## 27. Real-World Production Use Cases

### 27.1 Financial Services — On-Premises HA Cluster

**Requirements:** PCI DSS compliance, no public cloud, sub-100ms latency, full audit trails.

**Installation choices:**
- Bare-metal nodes with IPMI for out-of-band management
- External etcd on dedicated NVMe nodes (not co-located with API server)
- Calico with BGP (no overlay — native routing for minimum latency)
- Encryption at rest for all Secrets
- Full audit logging to SIEM (Splunk/ELK)
- Certificate management with HashiCorp Vault PKI
- Air-gapped cluster with private registry (Harbor)

### 27.2 Healthcare — HIPAA-Compliant Cluster

**Requirements:** PHI data protection, network isolation, regular backups, DR capability.

**Installation choices:**
- AWS EKS (managed control plane for reduced operational burden)
- EBS encryption enabled for all PVCs
- Namespace-level NetworkPolicy (default-deny)
- Velero backups to encrypted S3
- OPA Gatekeeper for policy enforcement (prevent privileged containers)
- Audit logs shipped to CloudTrail

### 27.3 E-commerce — High-Traffic Multi-Region

**Requirements:** 99.99% availability, auto-scaling, global distribution.

**Installation choices:**
- 3 clusters across regions (us-east, eu-west, ap-south)
- Cluster API for infrastructure-as-code cluster management
- Cilium with cluster mesh for cross-cluster service discovery
- ExternalDNS + Route53 for global DNS failover
- KEDA for event-driven autoscaling
- Spot instances for worker nodes with overprovisioning

### 27.4 AI/ML Platform

**Requirements:** GPU support, large ephemeral volumes, long-running jobs.

**Installation choices:**
- GPU nodes with NVIDIA device plugin DaemonSet
- High-bandwidth networking (Infiniband or SR-IOV) for GPU communication
- Rook/Ceph for distributed storage (ML datasets)
- Argo Workflows for ML pipeline orchestration
- Resource quotas and PriorityClasses for job scheduling fairness

---

## 28. Best Practices for Production Environments

### 28.1 Infrastructure

- **Use infrastructure-as-code** (Terraform/Pulumi) to provision nodes. Cluster installation scripts should be in version control.
- **Provision dedicated etcd nodes** for clusters > 200 nodes or high write-throughput workloads.
- **Use NVMe SSDs for etcd** — disk latency directly impacts API server response times.
- **Ensure time synchronization** (chrony/NTP) across all nodes. Clock skew > 500ms causes etcd failures.
- **Size control plane nodes generously** — under-provisioned control planes cause cascading failures under load.

### 28.2 Networking

- **Plan your CIDRs before installation** — they cannot be changed without rebuilding.
- **Use a routable Pod CIDR** (BGP mode with Calico) in environments where VXLAN overhead is unacceptable.
- **Implement default-deny NetworkPolicy** in all production namespaces from day one.
- **Use internal load balancers** for the API server endpoint — never expose API server to the public internet.

### 28.3 Security

- **Enable encryption at rest** for Secrets before your first Secret is created.
- **Rotate certificates annually** and automate the rotation with kubeadm certs renew.
- **Disable anonymous authentication** on kubelet (--anonymous-auth=false).
- **Use dedicated service accounts** with minimal RBAC for every workload.
- **Enable Pod Security Standards** at the namespace level (restricted for production).
- **Implement audit logging** from day one — retrofitting it is painful.

### 28.4 Operations

- **Pin Kubernetes versions** with `apt-mark hold` — auto-upgrades break production clusters.
- **Test upgrades in a staging cluster** that mirrors production topology.
- **Automate etcd backups** every 30 minutes minimum; store in a different AZ/region.
- **Document your installation parameters** — kubeadm config file in Git.
- **Use GitOps (ArgoCD/Flux)** for all workload deployments from day one.
- **Set resource requests and limits on all workloads** — missing requests cause unpredictable scheduling.

---

## 29. Common Mistakes and Pitfalls

### 29.1 Swap Not Disabled

**Mistake:** Leaving swap enabled.
**Symptom:** `kubelet` fails to start with error: `failed to run Kubelet: running with swap on is not supported`.
**Fix:**
```bash
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab
```

### 29.2 Container Runtime cgroup Driver Mismatch

**Mistake:** containerd uses `cgroupfs` but kubelet is configured for `systemd` (or vice versa).
**Symptom:** Pods fail to start; kubelet logs show `Failed to create pod sandbox`.
**Fix:** Ensure both use `systemd`:
```bash
# containerd: SystemdCgroup = true
# kubelet: cgroupDriver: systemd
```

### 29.3 Overlapping CIDRs

**Mistake:** Pod CIDR overlaps with node network.
**Symptom:** Routing conflicts; cannot reach Pods from nodes; traffic loops.
**Fix:** Rebuild the cluster with non-overlapping CIDRs.

### 29.4 kubeadm join Token Expiry

**Mistake:** Trying to use an expired bootstrap token (default TTL: 24 hours).
**Symptom:** `kubeadm join` fails with authentication error.
**Fix:**
```bash
kubeadm token create --print-join-command
```

### 29.5 Not Setting ResourceRequests on System Components

**Mistake:** No resource limits on kube-system Pods.
**Symptom:** Noisy-neighbor workloads starve CoreDNS or kube-proxy.
**Fix:** Set `--system-reserved` in kubelet config.

### 29.6 Forgetting to Taint Control Plane Nodes

**Mistake:** Not tainting control plane nodes, allowing workload Pods to schedule there.
**Symptom:** etcd/API server competes for CPU/memory with application Pods.
**Fix:**
```bash
kubectl taint nodes k8s-cp-1 node-role.kubernetes.io/control-plane:NoSchedule
```

### 29.7 Single etcd Node in "HA" Setup

**Mistake:** Running 2 control plane nodes but forgetting that 2-node etcd has no fault tolerance.
**Symptom:** One node failure takes down the entire cluster.
**Fix:** Always run etcd in odd numbers: 3 or 5 nodes.

### 29.8 Large etcd Database Without Compaction

**Mistake:** Not configuring etcd auto-compaction.
**Symptom:** etcd database grows unboundedly; eventually hits quota limit; API server returns errors.
**Fix:**
```bash
# In etcd extraArgs
auto-compaction-mode: "revision"
auto-compaction-retention: "1000"
```

### 29.9 Not Validating kubeadm Config Before Apply

**Mistake:** Running `kubeadm init` with a misconfigured YAML.
**Symptom:** Mid-init failures that leave the cluster in a partially initialized state.
**Fix:**
```bash
kubeadm init --config config.yaml --dry-run
kubeadm config validate --config config.yaml
```

### 29.10 Skipping Pre-flight Checks

**Mistake:** Ignoring `kubeadm init` pre-flight warnings.
**Symptom:** Silent failures during node operation.
**Fix:** Always resolve ALL pre-flight check warnings before proceeding.

---

## 30. Hands-On Labs & Practical Exercises

### Lab 1: Single-Node Cluster (Local Dev with kubeadm)

**Objective:** Install a single-node cluster on a local VM (Ubuntu 22.04).

```bash
# Prerequisite: 2 vCPU, 4GB RAM Ubuntu VM

# Part 1: System prep (condensed)
swapoff -a
modprobe overlay && modprobe br_netfilter
echo "net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables=1" > /etc/sysctl.d/k8s.conf
sysctl --system

# Part 2: Install containerd
apt-get install -y containerd.io
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd

# Part 3: Install kubeadm/kubelet/kubectl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  gpg --dearmor -o /etc/apt/keyrings/k8s.gpg
echo 'deb [signed-by=/etc/apt/keyrings/k8s.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
  tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

# Part 4: Init cluster
kubeadm init --pod-network-cidr=10.244.0.0/16

# Part 5: Configure kubectl
mkdir -p $HOME/.kube
cp /etc/kubernetes/admin.conf $HOME/.kube/config

# Part 6: Allow scheduling on control plane (single-node only!)
kubectl taint nodes --all node-role.kubernetes.io/control-plane-

# Part 7: Install Flannel CNI
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Part 8: Verify
kubectl get nodes
kubectl get pods -n kube-system

# Part 9: Deploy a test application
kubectl create deployment hello --image=nginx --replicas=2
kubectl expose deployment hello --type=NodePort --port=80
kubectl get svc hello

# Expected: Working cluster with 2 nginx pods accessible via NodePort
```

### Lab 2: Explore Static Pod Manifests

```bash
# Objective: Understand how control plane components start

# View static pod manifests
ls /etc/kubernetes/manifests/

# Read the API server manifest
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Identify: image, command, flags, volumeMounts, hostNetwork

# Simulate a "config change":
# Add a flag to the API server and watch kubelet restart it
cp /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/kube-apiserver.yaml.bak

# Watch the kube-apiserver pod (in another terminal)
kubectl get pods -n kube-system -w

# Make a harmless change (add a comment or change --v flag)
# kubelet will detect the file change and restart the pod
```

### Lab 3: Observe Leader Election

```bash
# Objective: Watch leader election in action

# Find the current leader
kubectl get lease kube-controller-manager -n kube-system \
  -o jsonpath='{.spec.holderIdentity}' ; echo

# Watch the lease in real-time
kubectl get lease kube-controller-manager -n kube-system -w

# In a 3-node HA cluster:
# Restart the current leader's kube-controller-manager
kubectl -n kube-system delete pod \
  -l component=kube-controller-manager \
  --field-selector spec.nodeName=$(kubectl get lease kube-controller-manager \
    -n kube-system -o jsonpath='{.spec.holderIdentity}' | cut -d_ -f1)

# Observe: new leader elected within ~30 seconds
kubectl get lease kube-controller-manager -n kube-system \
  -o jsonpath='{.spec.holderIdentity}' ; echo
```

### Lab 4: etcd Operations

```bash
# Objective: Learn etcd backup/restore and data inspection

# Take a manual backup
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    snapshot save /tmp/etcd-backup.db

# Verify backup
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl snapshot status /tmp/etcd-backup.db --write-out=table

# Inspect a key in etcd (raw data)
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    get /registry/namespaces/default --print-value-only | \
    strings | head -20

# Count all resources in etcd
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl \
    --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key \
    get /registry --prefix --keys-only | wc -l
```

### Lab 5: Certificate Inspection

```bash
# Objective: Understand the PKI structure

# List all certificates
ls /etc/kubernetes/pki/

# Inspect API server certificate
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | \
  grep -A 20 "Subject Alternative Name\|Validity"

# Check expiration of all certs
kubeadm certs check-expiration

# See what's in the admin.conf (kubeconfig)
kubectl config view --raw=false
cat /etc/kubernetes/admin.conf | grep server:
```

---

## 31. Interview Questions — Beginner to Advanced

### Beginner Level

**Q1: What is kubeadm and what is its role in Kubernetes?**

**A:** `kubeadm` is the official Kubernetes cluster bootstrapping tool. It automates the tasks of:
- Generating all TLS certificates for cluster components
- Writing kubeconfig files for component authentication
- Creating static Pod manifests for control plane components (kube-apiserver, etcd, kube-controller-manager, kube-scheduler)
- Configuring the kubelet to join the cluster
- Installing CoreDNS and kube-proxy as the first add-ons

Importantly, kubeadm is a **one-shot tool** — it runs, sets up the cluster, and exits. It is not a running daemon.

---

**Q2: Why must swap be disabled on Kubernetes nodes?**

**A:** Kubernetes manages resource allocation based on declared requests and limits. With swap enabled, a container's memory can overflow onto disk, causing wildly unpredictable performance — the container might run at full speed one moment and be thousands of times slower the next as it hits swap. This violates Kubernetes' resource model. Furthermore, the OOM killer (which Kubernetes uses to enforce memory limits) doesn't work reliably with swap. Since Kubernetes 1.22, there is alpha/beta support for swap with `SwapBehavior: LimitedSwap`, but production clusters should still disable swap unless this feature has been carefully evaluated.

---

**Q3: What is the difference between a control plane node and a worker node?**

**A:** A **control plane node** runs the cluster management components: kube-apiserver (the REST gateway), etcd (the state store), kube-controller-manager (reconciliation loops), and kube-scheduler (Pod placement). Control plane nodes do not typically run workload Pods in production (enforced via taints). A **worker node** runs the actual application workloads. Each worker has kubelet (the node agent that manages Pods), kube-proxy (networking rules), a container runtime (containerd), and a CNI plugin. Worker nodes do not have etcd or control plane components.

---

**Q4: What is a static Pod?**

**A:** A static Pod is a Pod whose manifest is placed as a YAML file in a directory on the node (default: `/etc/kubernetes/manifests/`). The kubelet watches this directory and directly creates/manages the container without going through the API server. Static Pods are how kubeadm runs control plane components — the kube-apiserver, etcd, kube-controller-manager, and kube-scheduler are all static Pods. They appear in the API server (kubelet registers mirror Pods) but are managed by kubelet directly. This allows control plane components to start even before the API server is available.

---

### Intermediate Level

**Q5: Explain the bootstrap token mechanism used by kubeadm join.**

**A:** When `kubeadm init` runs, it creates a `BootstrapToken` in the `kube-system` namespace. This token has a 24-hour TTL by default. During `kubeadm join`, the new node uses this token to authenticate to the API server and fetch the cluster's CA certificate fingerprint. The new node then creates a `CertificateSigningRequest` (CSR) on behalf of its kubelet. The `kube-controller-manager` (NodeApproving controller) automatically approves the CSR if it matches the expected format and the bootstrap credentials check out. The approved CSR results in a signed kubelet certificate, which kubelet then uses for all future API server communication. The bootstrap token is only used once for the initial join and then discarded.

---

**Q6: What happens to running workloads if the kube-controller-manager crashes?**

**A:** Existing Pods and Services continue running normally — kube-proxy rules are already programmed on all nodes and don't require the controller manager to keep working. However, **no new reconciliation happens**:
- Pods that crash won't be replaced by the ReplicaSet controller
- New Deployments won't create Pods
- Scaling operations won't take effect
- LoadBalancer Services won't update cloud LBs
- New nodes that join won't get Pod CIDRs assigned

Because of leader election, a standby kube-controller-manager instance takes over within `leaseDurationSeconds` (default: 15–30 seconds). The new leader re-Lists all resources and reconciles everything, so any changes that occurred during the outage window are caught and processed.

---

**Q7: Why does kubeadm require --upload-certs for HA control plane joins?**

**A:** Control plane components use certificates for mutual TLS authentication. In an HA cluster, all control plane nodes need the same CA certificates and key material. The `--upload-certs` flag causes kubeadm to encrypt the certificate key material and store it in a Secret in the `kube-system` namespace, secured by a random certificate key. When additional control plane nodes join with `--control-plane --certificate-key <key>`, they download and decrypt this Secret to obtain their certificates, rather than requiring manual certificate copying between nodes. The certificate key expires after 2 hours for security.

---

**Q8: What does the reconciliation loop guarantee, and what does it NOT guarantee?**

**A:** The reconciliation loop guarantees **eventual consistency** — given enough time and no bugs, the cluster state will eventually match the desired state. It achieves this through level-triggered reconciliation (always reconciling the full state, not just the delta), idempotent operations (same result if run multiple times), and retry with backoff (temporary failures will be retried).

It does NOT guarantee:
- **Instant convergence** — there are delays (queue wait time, API call latency, cloud API latency)
- **Exact ordering** of operations across different controllers
- **Transactionality** — a partial update (Pod A created, Pod B creation fails) is possible; the next reconciliation will create Pod B
- **Infinite retries without intervention** — some controllers have backoff limits (Job) after which they mark the resource as failed

---

### Advanced Level

**Q9: How does Kubernetes handle a split-brain scenario in an HA etcd cluster?**

**A:** etcd uses the Raft consensus protocol which requires a **quorum** (majority) for any write to succeed. In a 3-node cluster, quorum is 2. If the cluster splits into a 1-node partition and a 2-node partition, only the 2-node partition can achieve quorum and continue accepting writes. The 1-node partition goes into read-only mode. When the partition heals, the 1-node partition catches up from the 2-node partition's log. This prevents split-brain (two partitions diverging) because neither minority partition can write. The implication: a 3-node cluster can tolerate 1 node failure; a 5-node cluster can tolerate 2 node failures. Running 2 nodes provides NO fault tolerance — losing 1 node loses quorum entirely.

---

**Q10: Explain the complete certificate chain in a kubeadm cluster and what happens when any certificate expires.**

**A:** kubeadm creates two CA hierarchies:
1. **Kubernetes CA** (`ca.crt`) → signs: `apiserver.crt`, `apiserver-kubelet-client.crt`, `front-proxy-ca.crt` (front-proxy CA for aggregation layer), kubelet client certs
2. **etcd CA** (`etcd/ca.crt`) → signs: `etcd/server.crt`, `etcd/peer.crt`, `apiserver-etcd-client.crt`

When a **leaf certificate** expires (e.g., `apiserver.crt`): the API server rejects TLS handshakes from clients; kubectl commands fail with certificate validation errors; the fix is `kubeadm certs renew apiserver` + restart the API server static Pod.

When the **CA expires** (10-year lifespan): all leaf certs become invalid simultaneously; requires generating a new CA and rotating all derived certificates — a major operation. Best practice: track CA expiration and plan replacement years in advance.

The kubelet certificates are special — they auto-rotate if `rotateCertificates: true` is set in KubeletConfiguration (default since 1.19).

---

**Q11: Design an air-gapped Kubernetes cluster installation. What are the key challenges?**

**A:** An air-gapped cluster has no internet access. Key challenges and solutions:

**Container images:** All Kubernetes component images (`registry.k8s.io/kube-apiserver`, `coredns`, etc.) must be mirrored to a private registry (Harbor). Configure containerd to use the private registry in the mirror configuration. Pre-load all images before running kubeadm.

**Package repository:** Mirror apt/yum packages for kubelet/kubeadm/kubectl to an internal package server (Nexus, Artifactory). Configure apt sources to point to the internal mirror.

**CNI plugin images:** CNI plugins pull images from public registries. Configure the CNI installation to use your mirrored images.

**kubeadm image pre-load:**
```bash
# On internet-connected machine
kubeadm config images list --kubernetes-version v1.29.0
kubeadm config images pull --kubernetes-version v1.29.0

# Tag and push to private registry
for img in $(kubeadm config images list --kubernetes-version v1.29.0); do
  docker pull $img
  new_tag="private-registry.example.com/${img#registry.k8s.io/}"
  docker tag $img $new_tag
  docker push $new_tag
done

# Use private registry with kubeadm
kubeadm init --image-repository private-registry.example.com
```

---

**Q12: How would you troubleshoot a situation where pods are scheduled but never start, and the node shows as Ready?**

**A:** This is a nuanced failure mode. Systematic investigation:

1. Check Pod status and events:
   ```bash
   kubectl describe pod <pod> | tail -30  # Look for Events section
   ```

2. If `ContainerCreating`:
   - Check if image can be pulled: `Events: Failed to pull image`
   - Check CNI: `network: failed to set up sandbox network`
   - Check PVC: `unable to mount volumes: PVC not found`

3. If `Pending` (not ContainerCreating):
   - Check scheduler: `kubectl describe pod | grep "Node:"`
   - If no node assigned: scheduler can't find a node (resource/taint/affinity)

4. If `Terminating` repeatedly:
   - Application crash: check `kubectl logs <pod> --previous`
   - Resource OOM kill: check `kubectl describe pod | grep -i "OOM\|killed"`

5. If stuck in `Init:0/1`:
   - Init container failing: `kubectl logs <pod> -c <init-container-name>`

6. If containerd is the issue:
   - `journalctl -u containerd -n 50` on the target node
   - `crictl ps -a` to see container states outside kubelet's view

7. If kubelet is the issue:
   - `journalctl -u kubelet -n 100 | grep -i "error\|fail"`
   - `crictl inspect <container-id>` for container-level details

---

## 32. Cheat Sheet — Commands, Flags & Manifests

### 32.1 Installation Phase Commands

```bash
# === PRE-INSTALLATION ===
swapoff -a && sed -i '/swap/s/^/#/' /etc/fstab
modprobe overlay && modprobe br_netfilter
sysctl --system

# === CONTAINERD ===
apt-get install -y containerd.io
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd && systemctl enable containerd

# === KUBERNETES COMPONENTS ===
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  gpg --dearmor -o /etc/apt/keyrings/k8s.gpg
echo 'deb [signed-by=/etc/apt/keyrings/k8s.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
  tee /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubelet=1.29.0-1.1 kubeadm=1.29.0-1.1 kubectl=1.29.0-1.1
apt-mark hold kubelet kubeadm kubectl

# === CLUSTER BOOTSTRAP ===
kubeadm init --config /etc/kubernetes/kubeadm-config.yaml --upload-certs 2>&1 | tee /root/kubeadm-init.log
mkdir -p $HOME/.kube && cp /etc/kubernetes/admin.conf $HOME/.kube/config

# === CNI ===
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

# === JOIN COMMANDS ===
kubeadm token create --print-join-command  # Worker join
CERT_KEY=$(kubeadm init phase upload-certs --upload-certs 2>/dev/null | tail -1)  # CP cert key

# === VERIFY ===
kubectl get nodes
kubectl get pods -n kube-system
```

### 32.2 Day-2 Operations Commands

```bash
# === CERTIFICATE MANAGEMENT ===
kubeadm certs check-expiration
kubeadm certs renew all
kubeadm certs renew apiserver

# === ETCD OPERATIONS ===
ETCDCTL="etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key"

kubectl exec -n kube-system etcd-$(hostname) -- $ETCDCTL endpoint health
kubectl exec -n kube-system etcd-$(hostname) -- $ETCDCTL member list
kubectl exec -n kube-system etcd-$(hostname) -- $ETCDCTL snapshot save /tmp/backup.db
kubectl exec -n kube-system etcd-$(hostname) -- $ETCDCTL defrag

# === NODE MANAGEMENT ===
kubectl cordon <node>             # Prevent new scheduling
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>           # Re-enable scheduling
kubectl taint nodes <node> key=value:NoSchedule
kubectl label node <node> role=worker

# === CLUSTER UPGRADE ===
kubeadm upgrade plan
kubeadm upgrade apply v1.30.0    # On first CP node
kubeadm upgrade node             # On additional CP/worker nodes

# === TROUBLESHOOTING ===
kubectl cluster-info
kubectl cluster-info dump --output-directory=/tmp/cluster-dump
kubectl get events --all-namespaces --sort-by='.lastTimestamp'
kubectl describe node <node> | grep -A 10 "Conditions\|Allocated"
journalctl -u kubelet -f                        # Follow kubelet logs
journalctl -u containerd -n 100 --no-pager     # containerd logs
crictl ps -a                                    # All containers (kubelet's view bypassed)
crictl logs <container-id>                      # Container logs via CRI
```

### 32.3 Key kubeadm Flags Reference

| Command | Important Flag | Description |
|---|---|---|
| `kubeadm init` | `--config` | Path to configuration file |
| `kubeadm init` | `--upload-certs` | Upload certs to cluster for HA joins |
| `kubeadm init` | `--pod-network-cidr` | Pod network CIDR (required for most CNIs) |
| `kubeadm init` | `--service-cidr` | Service network CIDR |
| `kubeadm init` | `--control-plane-endpoint` | VIP/DNS for HA API server |
| `kubeadm init` | `--dry-run` | Simulate without making changes |
| `kubeadm join` | `--control-plane` | Join as additional control plane node |
| `kubeadm join` | `--certificate-key` | Key to decrypt uploaded certs |
| `kubeadm join` | `--node-name` | Override node name |
| `kubeadm reset` | `--force` | Reset cluster without confirmation |
| `kubeadm certs` | `renew all` | Renew all certificates |
| `kubeadm upgrade` | `plan` | Show available upgrades |
| `kubeadm upgrade` | `apply v1.X.Y` | Apply upgrade on control plane |
| `kubeadm token` | `create --print-join-command` | Generate new join command |

### 32.4 Critical File Locations

| File/Directory | Purpose |
|---|---|
| `/etc/kubernetes/manifests/` | Static Pod manifests for control plane |
| `/etc/kubernetes/pki/` | All TLS certificates and keys |
| `/etc/kubernetes/admin.conf` | admin kubeconfig (full cluster access) |
| `/etc/kubernetes/kubelet.conf` | kubelet kubeconfig |
| `/var/lib/etcd/` | etcd data directory |
| `/var/lib/kubelet/` | kubelet state, Pod volumes |
| `/etc/containerd/config.toml` | containerd configuration |
| `/etc/cni/net.d/` | CNI configuration files |
| `/opt/cni/bin/` | CNI plugin binaries |
| `/etc/sysctl.d/k8s.conf` | Kubernetes kernel parameters |
| `/etc/modules-load.d/k8s.conf` | Kubernetes kernel modules |

---

## 33. Key Takeaways & Summary

### The Installation Sequence in One View

```
1. OS Prep → 2. Container Runtime → 3. Kubernetes Binaries
     ↓
4. kubeadm init → Certs → Static Pods → etcd → API Server → Controllers → Scheduler
     ↓
5. kubectl configured → CNI installed → Node turns Ready
     ↓
6. Worker nodes join → CoreDNS operational → Cluster functional
     ↓
7. Hardening → Monitoring → Backup → GitOps
```

### The 10 Golden Rules of Kubernetes Installation

1. **Plan CIDRs before you start** — they are permanent decisions.
2. **Disable swap** — no exceptions in production.
3. **Match cgroup drivers** — containerd and kubelet must use the same driver (`systemd`).
4. **Synchronize clocks** — etcd consensus requires NTP. Clock skew kills clusters.
5. **Use odd numbers for etcd** — 1, 3, or 5 nodes. Never 2 or 4.
6. **Controllers never touch etcd** — everything goes through the API server.
7. **Pin versions with hold** — uncontrolled upgrades break production.
8. **Back up etcd regularly** — it is the only stateful component. Everything else is reconstructable.
9. **Certificates expire** — automate renewal and monitoring before day one.
10. **Document every installation decision** — the kubeadm config YAML should live in Git.

### Why Understanding Installation Makes You a Better Operator

The mental model you build during cluster installation explains almost every Kubernetes operational concept:
- Why Pods are ephemeral (no persistent IP, no guaranteed node)
- Why Services need to exist (stable endpoint over ephemeral Pods)
- Why controller-manager is stateless (all state in etcd via API server)
- Why leader election exists (multiple controllers can't reconcile simultaneously)
- Why static Pods exist (chicken-and-egg: control plane needs to start before API server)
- Why CNI must be installed before workloads (Pods need network before they can run)

Every operational question traces back to these first principles established during installation.

---

> **This guide covers Kubernetes v1.29+ with kubeadm on Ubuntu 22.04 LTS. Production environments should always consult the official Kubernetes documentation at https://kubernetes.io/docs/setup/ and the kubeadm reference at https://kubernetes.io/docs/reference/setup-tools/kubeadm/ for the most current information.**

---

*End of: Lab: Installing a Complete Kubernetes Cluster — Production-Grade Guide*
