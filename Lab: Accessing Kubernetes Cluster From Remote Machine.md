## Table of Contents

1. [Introduction — What Remote Cluster Access Means and Why It Matters](#1-introduction--what-remote-cluster-access-means-and-why-it-matters)
2. [Core Identity Table — Remote Access Components](#2-core-identity-table--remote-access-components)
3. [How Remote Access Works — The Complete Request Pipeline](#3-how-remote-access-works--the-complete-request-pipeline)
4. [The Controller Pattern — Watch → Compare → Act → Loop](#4-the-controller-pattern--watch--compare--act--loop)
5. [Pre-Lab Requirements and Environment Setup](#5-pre-lab-requirements-and-environment-setup)
6. [Step 1 — Installing kubectl on the Remote Machine](#6-step-1--installing-kubectl-on-the-remote-machine)
7. [Step 2 — Obtaining and Configuring the kubeconfig](#7-step-2--obtaining-and-configuring-the-kubeconfig)
8. [Step 3 — Network Connectivity and Firewall Configuration](#8-step-3--network-connectivity-and-firewall-configuration)
9. [Step 4 — Certificate and TLS Verification](#9-step-4--certificate-and-tls-verification)
10. [Step 5 — Creating Dedicated User Credentials for Remote Access](#10-step-5--creating-dedicated-user-credentials-for-remote-access)
11. [Step 6 — RBAC Configuration for Remote Users](#11-step-6--rbac-configuration-for-remote-users)
12. [Step 7 — Multi-Cluster and Multi-Context Access Management](#12-step-7--multi-cluster-and-multi-context-access-management)
13. [Built-in Controllers Deep Dive — Understanding What You're Accessing](#13-built-in-controllers-deep-dive--understanding-what-youre-accessing)
14. [Internal Working Concepts — Informers, Work Queues & Reconciliation](#14-internal-working-concepts--informers-work-queues--reconciliation)
15. [API Server and etcd Interaction — The Access Enforcement Layer](#15-api-server-and-etcd-interaction--the-access-enforcement-layer)
16. [Leader Election — HA Considerations for Remote Access Infrastructure](#16-leader-election--ha-considerations-for-remote-access-infrastructure)
17. [Advanced Remote Access Patterns](#17-advanced-remote-access-patterns)
18. [Performance Tuning for Remote API Access](#18-performance-tuning-for-remote-api-access)
19. [Security Hardening for Remote Access](#19-security-hardening-for-remote-access)
20. [Monitoring and Observability for Remote Access Sessions](#20-monitoring-and-observability-for-remote-access-sessions)
21. [Troubleshooting Remote Access Issues](#21-troubleshooting-remote-access-issues)
22. [Disaster Recovery — Maintaining Access During Failures](#22-disaster-recovery--maintaining-access-during-failures)
23. [Comparison — kubectl vs Direct API vs Dashboard vs Proxy](#23-comparison--kubectl-vs-direct-api-vs-dashboard-vs-proxy)
24. [ASCII Architecture Diagram](#24-ascii-architecture-diagram)
25. [Real-World Production Use Cases](#25-real-world-production-use-cases)
26. [Best Practices for Production Remote Access](#26-best-practices-for-production-remote-access)
27. [Common Mistakes and Pitfalls](#27-common-mistakes-and-pitfalls)
28. [Hands-On Labs — Guided Exercises](#28-hands-on-labs--guided-exercises)
29. [Interview Questions — Beginner to Advanced](#29-interview-questions--beginner-to-advanced)
30. [Cheat Sheet — Commands, Flags & One-Liners](#30-cheat-sheet--commands-flags--one-liners)
31. [Key Takeaways & Summary](#31-key-takeaways--summary)

---

## 1. Introduction — What Remote Cluster Access Means and Why It Matters

**Remote cluster access** is the practice of managing and interacting with a Kubernetes cluster from a machine that is not a node in that cluster — typically an engineer's laptop, a CI/CD runner, a monitoring server, or an administrative jump host. This is the reality for virtually every production Kubernetes deployment: the cluster runs in a datacenter or cloud VPC while the humans and automation pipelines that manage it operate from entirely separate machines.

Understanding how to set up, secure, and troubleshoot remote access is therefore not an optional skill — it is a **foundational operational requirement** for anyone working with Kubernetes in production.

### Why This Is More Complex Than It Appears

Unlike connecting to a server via SSH (one protocol, one credential, one endpoint), remote Kubernetes cluster access involves:

- **Network path establishment:** The remote machine must be able to reach the API server's IP and port (6443) — which may be behind VPNs, firewalls, private subnets, or cloud load balancers
- **Multi-layer authentication:** TLS mutual authentication, X.509 certificates, bearer tokens, or OIDC flows — all configured in a single `kubeconfig` file
- **Authorization mapping:** The remote user's identity must map to RBAC rules that control exactly what they can do
- **Context management:** A production engineer may manage 10+ clusters simultaneously; kubeconfig provides a unified interface
- **Security governance:** Remote access credentials must be provisioned, scoped, rotated, audited, and revoked throughout their lifecycle

### Production Scenarios That Require Remote Access

| Scenario | Who Needs Remote Access | What They Need to Do |
|---|---|---|
| Daily operations | Platform engineers | Deploy, scale, debug, inspect workloads |
| On-call incident response | SRE on-call | Read logs, exec into pods, check node status |
| CI/CD pipeline | GitHub Actions / Jenkins / ArgoCD | Deploy new images, update ConfigMaps |
| Security audit | Security engineer | Read RBAC, inspect Secrets, check NetworkPolicies |
| Capacity planning | Architect | Read resource utilization, node inventory |
| Customer support | L3 support engineer | Read application logs and events |
| Monitoring | Prometheus, Grafana | Scrape metrics from cluster components |

Every one of these requires carefully configured remote access — and every one represents a unique security surface that must be governed.

---

## 2. Core Identity Table — Remote Access Components

| Component | Binary / Resource | Port | Protocol | Role in Remote Access |
|---|---|---|---|---|
| **kube-apiserver** | `kube-apiserver` | 6443 | HTTPS | Receives and validates all remote kubectl/API requests |
| **etcd** | `etcd` | 2379, 2380 | HTTPS/gRPC | Stores kubeconfigs (cluster-side), RBAC rules, SA tokens |
| **kubectl** | `kubectl` (client CLI) | N/A | HTTPS client | Primary tool for remote human/automated access |
| **kubeconfig** | `~/.kube/config` (YAML file) | N/A | N/A | Credential and endpoint configuration for remote clients |
| **ServiceAccount** | `serviceaccount` resource | N/A | K8s resource | Identity for CI/CD and in-cluster automation |
| **RBAC Role** | `role` / `clusterrole` | N/A | K8s resource | Defines what the remote user can do |
| **RBAC Binding** | `rolebinding` / `clusterrolebinding` | N/A | K8s resource | Assigns roles to remote users |
| **CertificateSigningRequest** | `csr` resource | N/A | K8s resource | Used to issue client certificates for remote users |
| **X.509 Client Certificate** | `*.crt` / `*.key` files | N/A | TLS | Remote user authentication credential |
| **Bearer Token** | String in kubeconfig | N/A | HTTP header | ServiceAccount / OIDC token for remote auth |
| **kube-controller-manager** | `kube-controller-manager` | 10257 | HTTPS | Manages ServiceAccount token lifecycle |
| **kube-scheduler** | `kube-scheduler` | 10259 | HTTPS | Not directly involved in remote access; but schedulable |
| **CoreDNS** | `coredns` | 53, 9153 | UDP/TCP | Resolves `kubernetes.default.svc` for in-cluster access |
| **HAProxy / Cloud LB** | External LB | 6443 | TCP | Load balancer fronting HA API server endpoints |
| **SSH Tunnel / VPN** | `ssh`, VPN daemon | 22, 1194, etc. | TCP/UDP | Network transport for remote access in private clusters |
| **kubelet** | `kubelet` | 10250 | HTTPS | Node-level endpoint accessed for logs, exec, port-forward |

---

## 3. How Remote Access Works — The Complete Request Pipeline

Every `kubectl get pods`, `kubectl apply -f`, and `kubectl logs` command from a remote machine travels through a well-defined pipeline:

```
Remote Machine
    │
    │ 1. kubectl reads ~/.kube/config
    │    - API server URL: https://k8s-api.example.com:6443
    │    - Credentials: client cert + key (or token)
    │    - CA cert: to verify the API server's TLS certificate
    │
    │ 2. Network path: Remote → Internet/VPN → Firewall → API server
    │
    │ 3. TLS handshake
    │    - API server presents its certificate (signed by cluster CA)
    │    - Remote machine verifies API server cert against CA
    │    - Remote machine presents client certificate
    │    - API server verifies client cert against cluster CA
    │
    │ 4. Authentication (AuthN)
    │    - API server extracts username from certificate CN field
    │    - Username: "jane", Groups: ["developers"] (from O field)
    │
    │ 5. Authorization (AuthZ — RBAC)
    │    - Does jane have permission for: GET /api/v1/namespaces/prod/pods?
    │    - RBAC check: RoleBindings for jane in "prod" namespace
    │    - Result: Allow (200 OK) or Deny (403 Forbidden)
    │
    │ 6. Admission Control
    │    - For mutating requests: validate/mutate via admission webhooks
    │
    │ 7. etcd read/write (API server only)
    │    - API server reads from or writes to etcd
    │    - Returns serialized JSON/YAML response
    │
    ▼
Response back to kubectl on remote machine
```

### 3.1 What the Remote Machine Needs

| Requirement | Details |
|---|---|
| `kubectl` installed | Version within ±1 minor version of cluster |
| Valid kubeconfig | Contains: server URL, CA cert, client credentials |
| Network access | Can reach API server IP:6443 (via VPN or direct) |
| Valid credentials | Client cert not expired, token not expired |
| RBAC permissions | User has bindings granting needed operations |

---

## 4. The Controller Pattern — Watch → Compare → Act → Loop

When a remote user submits a request (e.g., `kubectl apply -f deployment.yaml`), the API server accepts and stores the desired state. Built-in controllers then take over to **reconcile** actual state toward the desired state. Understanding this pattern helps remote operators predict and diagnose what happens after their commands.

### 4.1 The Four-Phase Reconciliation Loop

```
┌──────────────────────────────────────────────────────────────────┐
│               KUBERNETES CONTROLLER RECONCILIATION LOOP          │
│                                                                  │
│   You run: kubectl apply -f deployment.yaml  (from remote)       │
│   API server stores desired state in etcd                        │
│                                                                  │
│   ┌─────────────┐                                                │
│   │  1. WATCH   │  Informer detects new/changed Deployment       │
│   │             │  object via API server watch stream            │
│   └──────┬──────┘                                                │
│          │ Event: ADDED                                          │
│          ▼                                                        │
│   ┌─────────────┐                                                │
│   │  2. COMPARE │  Desired: spec.replicas=3                      │
│   │             │  Current: 0 running Pods                       │
│   │             │  Delta: need 3 more Pods                       │
│   └──────┬──────┘                                                │
│          │ Delta found                                           │
│          ▼                                                        │
│   ┌─────────────┐                                                │
│   │  3. ACT     │  Create ReplicaSet                             │
│   │             │  ReplicaSet controller creates 3 Pods          │
│   │             │  Scheduler assigns Pods to Nodes               │
│   └──────┬──────┘                                                │
│          │ State written back via API server                      │
│          ▼                                                        │
│   ┌─────────────┐                                                │
│   │  4. LOOP    │  Returns to WATCH                              │
│   │             │  Triggered on next event or resync             │
│   └─────────────┘                                                │
└──────────────────────────────────────────────────────────────────┘
```

### 4.2 Why This Matters for Remote Operations

From a remote machine:
- `kubectl apply` → stores desired state → triggers controller loop
- `kubectl delete` → triggers garbage collection cascade
- `kubectl scale` → triggers ReplicaSet controller reconciliation
- `kubectl rollout restart` → triggers Deployment controller rolling update
- Every operation is **asynchronous** — the command returns before the actual state change completes

This is why `kubectl rollout status` and `kubectl wait` exist — to allow remote operators to synchronously monitor asynchronous reconciliation.

---

## 5. Pre-Lab Requirements and Environment Setup

### 5.1 Lab Environment Overview

```
┌──────────────────────────────────────────────────────────────────┐
│  LAB TOPOLOGY                                                    │
│                                                                  │
│  Remote Machine (your laptop/workstation):                       │
│  - OS: Linux / macOS / Windows                                   │
│  - IP: 192.168.1.100 (example)                                   │
│  - Needs: kubectl + kubeconfig                                   │
│                                                                  │
│  Kubernetes Cluster (separate machines):                         │
│  - Control Plane: 10.0.0.10 (k8s-cp-1)                          │
│  - Worker Node 1: 10.0.0.20 (k8s-worker-1)                      │
│  - Worker Node 2: 10.0.0.21 (k8s-worker-2)                      │
│  - API server: https://10.0.0.10:6443                            │
│                                                                  │
│  Network:                                                        │
│  - Remote machine can reach 10.0.0.10:6443 via VPN              │
│  - Or via public IP if cluster is cloud-hosted                   │
└──────────────────────────────────────────────────────────────────┘
```

### 5.2 What You Need Before Starting

```bash
# On the remote machine, verify basic connectivity:

# 1. Check if you can reach the API server (TCP connection)
nc -zv 10.0.0.10 6443
# Expected: Connection to 10.0.0.10 6443 port [tcp/*] succeeded!

# 2. Check if you have curl (for testing)
curl --version

# 3. Check your OS
uname -a   # Linux / macOS
# or
systeminfo  # Windows

# 4. Check available disk space (kubeconfig, kubectl are small)
df -h ~/.kube/

# 5. If behind a corporate proxy, note proxy settings
echo $http_proxy
echo $https_proxy
echo $no_proxy
```

### 5.3 Lab Variables (Replace with Your Values)

```bash
# Set these once and use throughout the lab:
export K8S_API_SERVER="https://10.0.0.10:6443"   # Your API server IP/hostname
export CLUSTER_NAME="production-cluster"
export REMOTE_USER="devops-user"                  # The user you're creating
export NAMESPACE="default"                         # Working namespace

echo "API Server: $K8S_API_SERVER"
echo "Cluster: $CLUSTER_NAME"
echo "User: $REMOTE_USER"
```

---

## 6. Step 1 — Installing kubectl on the Remote Machine

### 6.1 Linux Installation

```bash
# Method 1: Direct binary download (recommended for pinned versions)
# Always match your cluster version ± 1 minor version
K8S_VERSION="v1.29.0"

curl -LO "https://dl.k8s.io/release/${K8S_VERSION}/bin/linux/amd64/kubectl"

# Verify the download integrity
curl -LO "https://dl.k8s.io/release/${K8S_VERSION}/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256) kubectl" | sha256sum --check
# Expected: kubectl: OK

# Install kubectl
chmod +x kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Verify installation
kubectl version --client
# Client Version: v1.29.0

# Method 2: Package manager (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gnupg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubectl

# Pin to prevent accidental upgrade
sudo apt-mark hold kubectl

# Method 3: Snap (Ubuntu)
sudo snap install kubectl --classic --channel=1.29
```

### 6.2 macOS Installation

```bash
# Method 1: Homebrew (easiest)
brew install kubectl

# Pin to specific version
brew install kubectl@1.29
echo 'export PATH="/usr/local/opt/kubectl@1.29/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# Method 2: Direct binary
K8S_VERSION="v1.29.0"
curl -LO "https://dl.k8s.io/release/${K8S_VERSION}/bin/darwin/amd64/kubectl"
# For Apple Silicon (M1/M2/M3):
# curl -LO "https://dl.k8s.io/release/${K8S_VERSION}/bin/darwin/arm64/kubectl"

chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
sudo chown root: /usr/local/bin/kubectl

# Verify
kubectl version --client
```

### 6.3 Windows Installation

```powershell
# Method 1: chocolatey
choco install kubernetes-cli --version=1.29.0

# Method 2: winget
winget install -e --id Kubernetes.kubectl

# Method 3: Direct download
$K8S_VERSION = "v1.29.0"
$url = "https://dl.k8s.io/release/$K8S_VERSION/bin/windows/amd64/kubectl.exe"
Invoke-WebRequest -Uri $url -OutFile kubectl.exe

# Add to PATH
# Move kubectl.exe to C:\Windows\System32\ or add its directory to PATH

# Verify
kubectl version --client
```

### 6.4 kubectl Version Compatibility Matrix

| kubectl Version | Compatible Server Versions | Notes |
|---|---|---|
| v1.29 | v1.28, v1.29, v1.30 | ±1 minor version rule |
| v1.28 | v1.27, v1.28, v1.29 | Always use current or one behind |
| v1.27 | v1.26, v1.27, v1.28 | Earlier versions may lack newer features |

```bash
# Check version skew
kubectl version
# Look at: Client Version vs Server Version
# They should be within 1 minor version of each other

# Check if the connection works (will show server version if accessible)
kubectl cluster-info
```

### 6.5 Shell Autocompletion Setup

```bash
# Bash (Linux)
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc

# Zsh (macOS/Linux)
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
echo 'alias k=kubectl' >> ~/.zshrc
source ~/.zshrc

# Fish
kubectl completion fish | source
# Or permanently:
kubectl completion fish > ~/.config/fish/completions/kubectl.fish

# Verify completion works
kubectl get <TAB><TAB>
# Should show: pods, deployments, services, etc.
```

---

## 7. Step 2 — Obtaining and Configuring the kubeconfig

### 7.1 Understanding Where kubeconfig Comes From

| Source | How to Get It | Use Case |
|---|---|---|
| **kubeadm cluster admin.conf** | SSH to control plane, copy `/etc/kubernetes/admin.conf` | Initial cluster admin access |
| **kubeadm generated user cert** | Generate cert for specific user (step-by-step below) | User-specific limited access |
| **Cloud provider CLI** | `aws eks update-kubeconfig`, `gcloud container clusters get-credentials` | Managed Kubernetes (EKS/GKE/AKS) |
| **Shared by cluster admin** | Admin sends you a kubeconfig file | Team access (use with caution) |
| **Generated by CI/CD tool** | ArgoCD, Flux, Jenkins generate scoped kubeconfigs | Automated pipeline access |

### 7.2 Method 1 — Copying Admin kubeconfig (Emergency / Initial Setup)

```bash
# ON THE CONTROL PLANE NODE (not remote machine):
# The admin.conf gives full cluster-admin access — handle with extreme care!

cat /etc/kubernetes/admin.conf
# This file contains:
# - The API server URL
# - The cluster CA certificate
# - The admin client certificate and key (embedded as base64)

# Copy to remote machine securely
# Option A: SCP
scp ubuntu@10.0.0.10:/etc/kubernetes/admin.conf ~/admin.conf

# Option B: SSH + cat (pipe directly)
ssh ubuntu@10.0.0.10 'sudo cat /etc/kubernetes/admin.conf' > ~/admin.conf

# CRITICAL: Fix the server address if it's set to localhost
# The admin.conf often has server: https://127.0.0.1:6443
# This works on the control plane but NOT from remote machines
cat ~/admin.conf | grep server
# If it shows https://127.0.0.1:6443, you must change it

# Replace localhost with the actual control plane IP/hostname
sed -i 's|https://127.0.0.1:6443|https://10.0.0.10:6443|g' ~/admin.conf
# Or for macOS (BSD sed):
sed -i '' 's|https://127.0.0.1:6443|https://10.0.0.10:6443|g' ~/admin.conf

# Test with the modified config
kubectl --kubeconfig=~/admin.conf get nodes
```

### 7.3 Method 2 — Setting Up the kubeconfig File Manually

```bash
# Create kubeconfig directory
mkdir -p ~/.kube

# Option A: Copy the obtained kubeconfig to default location
cp ~/admin.conf ~/.kube/config

# Set secure permissions (CRITICAL — kubeconfig contains credentials)
chmod 600 ~/.kube/config

# Verify the file location and permissions
ls -la ~/.kube/config
# Expected: -rw------- 1 youruser yourgroup XXXX ~/.kube/config

# Test the connection
kubectl cluster-info
# Expected output:
# Kubernetes control plane is running at https://10.0.0.10:6443
# CoreDNS is running at https://10.0.0.10:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

### 7.4 Method 3 — Merging Multiple kubeconfigs

```bash
# If you already have a kubeconfig and want to add a new cluster:

# Step 1: Set KUBECONFIG env to merge files on-the-fly
export KUBECONFIG=~/.kube/config:~/admin.conf

# Step 2: View merged result
kubectl config view

# Step 3: Save the merged config permanently
kubectl config view --flatten > ~/.kube/config.merged
mv ~/.kube/config ~/.kube/config.bak
mv ~/.kube/config.merged ~/.kube/config
chmod 600 ~/.kube/config

# Step 4: List all contexts in the merged config
kubectl config get-contexts
```

### 7.5 Examining the kubeconfig Structure

```bash
# View kubeconfig without raw credentials
kubectl config view

# View with raw credentials (contains actual certs/tokens — be careful)
kubectl config view --raw

# View only the current context
kubectl config view --minify

# Check what server the current context points to
kubectl config view -o jsonpath='{.clusters[?(@.name=="production-cluster")].cluster.server}'

# Check current context
kubectl config current-context

# Check the user in current context
kubectl config view \
  -o jsonpath='{.contexts[?(@.name=="production")].context.user}'
```

### 7.6 kubeconfig for Cloud Providers

```bash
# AWS EKS
aws eks update-kubeconfig \
  --region us-east-1 \
  --name production-cluster \
  --kubeconfig ~/.kube/config  # Or specify a different file

# View what was added
kubectl config get-contexts | grep production-cluster

# GKE (Google Kubernetes Engine)
gcloud container clusters get-credentials production-cluster \
  --region us-central1 \
  --project my-project

# AKS (Azure Kubernetes Service)
az aks get-credentials \
  --resource-group production-rg \
  --name production-cluster \
  --file ~/.kube/config

# Verify connection to any of the above
kubectl get nodes
```

---

## 8. Step 3 — Network Connectivity and Firewall Configuration

### 8.1 Diagnosing Network Connectivity

```bash
# Test 1: Basic TCP connectivity to API server port
# If this fails, it's a network/firewall issue — not a Kubernetes issue
nc -zv 10.0.0.10 6443
# Success: Connection to 10.0.0.10 6443 port [tcp/*] succeeded!
# Failure: nc: connectx to 10.0.0.10 port 6443 (tcp) failed: Connection refused
#          nc: getaddrinfo for host "10.0.0.10" port 6443: Name or service not known

# Test 2: Ping the control plane (tests network path, not port)
ping -c 3 10.0.0.10

# Test 3: Traceroute (see where the connection fails)
traceroute 10.0.0.10
# or on macOS:
traceroute -P tcp -p 6443 10.0.0.10

# Test 4: HTTPS connection test (raw TLS without certificate validation)
curl -k https://10.0.0.10:6443/version
# If you get JSON back → network works, TLS works, basic connectivity confirmed
# If you get "connection refused" → port not reachable
# If you get "no route to host" → network path issue

# Test 5: DNS resolution (if using hostname)
nslookup k8s-api.example.com
dig k8s-api.example.com

# Test 6: Check if the problem is TLS certificate validation
curl -v https://10.0.0.10:6443/version 2>&1 | grep -E "SSL|TLS|certificate"
```

### 8.2 Firewall Rules Required for Remote Access

```bash
# The remote machine needs OUTBOUND access to:
# API server: TCP 6443 (kubectl, all API calls)
# kubelet:    TCP 10250 (kubectl logs, exec, port-forward)
# NodePort:   TCP 30000-32767 (optional: accessing NodePort services)

# On the cluster side (inbound rules needed):
# From remote machine IP → control plane port 6443
# From remote machine IP → all nodes port 10250 (for kubectl exec/logs)

# Test kubelet connectivity (needed for kubectl exec and kubectl logs)
nc -zv 10.0.0.20 10250  # Worker node 1
nc -zv 10.0.0.21 10250  # Worker node 2

# If firewall is Linux iptables (on cluster nodes):
# Allow port 6443 from remote IP
sudo iptables -A INPUT -p tcp --dport 6443 \
  -s 192.168.1.100 -j ACCEPT  # Replace with your remote IP

# If firewall is AWS Security Group:
# Add inbound rule: TCP 6443 from your IP CIDR

# If firewall is UFW (Ubuntu):
sudo ufw allow from 192.168.1.100 to any port 6443 proto tcp
sudo ufw allow from 192.168.1.100 to any port 10250 proto tcp
```

### 8.3 Corporate Proxy Configuration

```bash
# If accessing through a corporate HTTP proxy:

# Set proxy for kubectl (HTTP_PROXY / HTTPS_PROXY)
export HTTPS_PROXY=http://proxy.corporate.com:8080
export HTTP_PROXY=http://proxy.corporate.com:8080

# CRITICAL: Exclude the Kubernetes API server from proxy
# Otherwise kubectl traffic goes through the proxy, which may not support raw TCP
export NO_PROXY=10.0.0.10,10.0.0.20,10.0.0.21,localhost,127.0.0.1

# Make these permanent in ~/.bashrc or ~/.zshrc
echo 'export HTTPS_PROXY=http://proxy.corporate.com:8080' >> ~/.bashrc
echo 'export NO_PROXY=10.0.0.10,kubernetes.default' >> ~/.bashrc

# Test through proxy
curl -v \
  --proxy http://proxy.corporate.com:8080 \
  https://10.0.0.10:6443/version
```

### 8.4 VPN Setup for Private Clusters

```bash
# Most production clusters are in private networks
# Remote access requires VPN first

# OpenVPN (common corporate VPN)
# Connect to VPN:
sudo openvpn --config /etc/openvpn/corporate.ovpn

# After VPN connection, verify routing to cluster
ip route show | grep 10.0.0.0
# Should show: 10.0.0.0/8 via <VPN gateway>

# WireGuard (modern, fast VPN)
sudo wg-quick up wg0   # Connect WireGuard tunnel

# Tailscale (easiest zero-config VPN)
tailscale up
tailscale ip  # Get your Tailscale IP
# Other cluster nodes running Tailscale are accessible directly

# Verify VPN is working
ping 10.0.0.10
nc -zv 10.0.0.10 6443
```

---

## 9. Step 4 — Certificate and TLS Verification

### 9.1 Understanding the TLS Trust Chain

```
Kubernetes PKI Trust Chain:
─────────────────────────────────────────────────────────
Root CA (ca.crt / ca.key)
    ├── API Server Certificate (apiserver.crt)
    │   Signs: CN=kube-apiserver, SANs: [10.0.0.10, k8s-api.example.com, kubernetes, ...]
    │
    ├── Client Certificate for admin (admin.crt)
    │   Signs: CN=kubernetes-admin, O=system:masters
    │
    ├── Client Certificate for jane (jane.crt)
    │   Signs: CN=jane, O=developers
    │
    └── kubelet Certificate for each node
        Signs: CN=system:node:worker-1, O=system:nodes
```

### 9.2 Verifying the API Server Certificate

```bash
# Check what certificate the API server presents
openssl s_client \
  -connect 10.0.0.10:6443 \
  -servername 10.0.0.10 \
  </dev/null 2>/dev/null | \
  openssl x509 -noout -text | \
  grep -A 20 "Subject Alternative Name\|Validity\|Subject:"

# Verify the Subject Alternative Names (SANs)
# The API server cert MUST have SANs for every hostname/IP you use to connect
# Missing SAN = "x509: certificate is valid for X, not Y" error

# Verify the certificate is signed by the cluster CA
# First, get the cluster CA cert (from kubeconfig or control plane)
kubectl config view --raw \
  -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | \
  base64 -d > /tmp/cluster-ca.crt

# Now verify the API server cert
openssl verify \
  -CAfile /tmp/cluster-ca.crt \
  <(openssl s_client -connect 10.0.0.10:6443 </dev/null 2>/dev/null | \
    openssl x509)
# Expected: stdin: OK

# Check certificate expiry
openssl s_client -connect 10.0.0.10:6443 </dev/null 2>/dev/null | \
  openssl x509 -noout -dates
# notBefore=Jan  1 00:00:00 2025 GMT
# notAfter=Jan  1 00:00:00 2026 GMT
```

### 9.3 Extracting and Trusting the Cluster CA

```bash
# Method 1: Extract CA from existing kubeconfig
kubectl config view --raw \
  -o jsonpath='{.clusters[?(@.name=="production-cluster")].cluster.certificate-authority-data}' | \
  base64 -d > cluster-ca.crt

# Method 2: Copy from control plane
scp ubuntu@10.0.0.10:/etc/kubernetes/pki/ca.crt ./cluster-ca.crt

# Method 3: Extract from admin.conf
grep 'certificate-authority-data:' ~/admin.conf | \
  awk '{print $2}' | base64 -d > cluster-ca.crt

# Verify the CA certificate
openssl x509 -in cluster-ca.crt -noout -text | \
  grep -E "Subject:|Validity|Not After"

# Test connection using the CA cert (no certificate errors)
curl \
  --cacert cluster-ca.crt \
  https://10.0.0.10:6443/version
# Should return JSON without SSL errors

# Add to kubeconfig
kubectl config set-cluster production-cluster \
  --server=https://10.0.0.10:6443 \
  --certificate-authority=cluster-ca.crt \
  --embed-certs=true  # Embeds the CA cert in kubeconfig (portable)
```

### 9.4 Handling Certificate Errors

```bash
# Error 1: "x509: certificate signed by unknown authority"
# Cause: Your kubeconfig doesn't have the correct CA cert
# Fix: Update certificate-authority-data in kubeconfig

# Error 2: "x509: certificate is valid for 127.0.0.1, not 10.0.0.10"
# Cause: API server cert doesn't have 10.0.0.10 as a SAN
# Fix A: Add the IP as a SAN to the API server cert (on control plane)
#   kubeadm certs renew apiserver --apiserver-cert-extra-sans=10.0.0.10

# Fix B (temporary/testing ONLY — NEVER IN PRODUCTION):
kubectl get nodes \
  --insecure-skip-tls-verify=true  # Disables ALL TLS verification!

# Error 3: "x509: certificate has expired or is not yet valid"
# Cause: API server certificate expired or system clock skew
# Fix: Renew certificate on control plane
#   kubeadm certs renew apiserver

# Check current API server certificate SANs (run on control plane)
openssl x509 \
  -in /etc/kubernetes/pki/apiserver.crt \
  -noout -text | grep -A 5 "Subject Alternative Name"

# Re-issue API server cert with additional SANs
kubeadm certs renew apiserver \
  --apiserver-cert-extra-sans=10.0.0.10,k8s-api.example.com,mycluster.example.com
```

---

## 10. Step 5 — Creating Dedicated User Credentials for Remote Access

### 10.1 The Right Way: X.509 Client Certificates via CSR API

```bash
# BEST PRACTICE: Never give remote users the admin.conf
# Instead, create individual certificates with limited permissions

# ── ON THE REMOTE MACHINE ──────────────────────────────────────────────

# Step 1: Generate the user's private key (stays on remote machine — never shared)
openssl genrsa -out ${REMOTE_USER}.key 2048

# Step 2: Create a Certificate Signing Request (CSR)
# CN = username in Kubernetes
# O = group membership in Kubernetes (used by RBAC)
openssl req -new \
  -key ${REMOTE_USER}.key \
  -out ${REMOTE_USER}.csr \
  -subj "/CN=${REMOTE_USER}/O=developers/O=team-a"
  #       ^username           ^group1        ^group2

# View the CSR to verify it's correct
openssl req -in ${REMOTE_USER}.csr -noout -text | \
  grep -A 5 "Subject:"

# ── ON THE CONTROL PLANE OR BY CLUSTER ADMIN ────────────────────────────

# Step 3: Submit CSR to Kubernetes (admin does this, or user if they have permission)
# The remote user sends their .csr file to the admin
# Admin submits it to the cluster:

CSR_BASE64=$(cat ${REMOTE_USER}.csr | base64 | tr -d '\n')

cat << EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: ${REMOTE_USER}-csr
spec:
  request: ${CSR_BASE64}
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 31536000   # 1 year; reduce for higher security
  usages:
    - client auth
EOF

# Step 4: Approve the CSR
kubectl certificate approve ${REMOTE_USER}-csr

# Verify approval
kubectl get csr ${REMOTE_USER}-csr
# STATUS should show: Approved,Issued

# Step 5: Extract the signed certificate
kubectl get csr ${REMOTE_USER}-csr \
  -o jsonpath='{.status.certificate}' | \
  base64 -d > ${REMOTE_USER}.crt

# Verify the certificate
openssl x509 -in ${REMOTE_USER}.crt -noout -text | \
  grep -E "Subject:|Validity|Not After"

# ── BACK ON REMOTE MACHINE ──────────────────────────────────────────────

# Step 6: Build kubeconfig for the remote user
# (Admin gives the user their .crt file and the cluster CA cert)

# Set up the cluster reference (with CA cert)
kubectl config set-cluster ${CLUSTER_NAME} \
  --server=${K8S_API_SERVER} \
  --certificate-authority=cluster-ca.crt \
  --embed-certs=true

# Add the user credentials
kubectl config set-credentials ${REMOTE_USER} \
  --client-certificate=${REMOTE_USER}.crt \
  --client-key=${REMOTE_USER}.key \
  --embed-certs=true

# Create a context binding them
kubectl config set-context ${REMOTE_USER}-context \
  --cluster=${CLUSTER_NAME} \
  --user=${REMOTE_USER} \
  --namespace=default

# Switch to the new context
kubectl config use-context ${REMOTE_USER}-context

# Verify identity (should show the username)
kubectl auth whoami
# NAME              GROUPS
# devops-user       [developers team-a system:authenticated]

# Test access (will get 403 until RBAC is configured)
kubectl get pods
# Error from server (Forbidden): pods is forbidden: User "devops-user" cannot...
```

### 10.2 ServiceAccount Tokens for CI/CD Remote Access

```bash
# For CI/CD pipelines that need remote access:

# Step 1: Create a dedicated ServiceAccount
kubectl create serviceaccount ci-pipeline -n kube-system
# Or in a specific namespace:
kubectl create serviceaccount github-actions -n production

# Step 2: Create a time-limited token (recommended)
kubectl create token github-actions \
  --duration=8760h \  # 1 year
  -n production

# Step 3: Or create a long-lived token via Secret (legacy but common)
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: github-actions-token
  namespace: production
  annotations:
    kubernetes.io/service-account.name: github-actions
type: kubernetes.io/service-account-token
EOF

# Extract the token
TOKEN=$(kubectl get secret github-actions-token \
  -n production \
  -o jsonpath='{.data.token}' | base64 -d)

# Step 4: Build kubeconfig using the token
cat > ci-kubeconfig.yaml << EOF
apiVersion: v1
kind: Config
clusters:
  - name: ${CLUSTER_NAME}
    cluster:
      server: ${K8S_API_SERVER}
      certificate-authority-data: $(kubectl config view --raw \
        -o jsonpath='{.clusters[0].cluster.certificate-authority-data}')
users:
  - name: github-actions
    user:
      token: ${TOKEN}
contexts:
  - name: github-actions-context
    context:
      cluster: ${CLUSTER_NAME}
      user: github-actions
      namespace: production
current-context: github-actions-context
EOF

# Step 5: Store in CI/CD secret (GitHub Secrets, Vault, etc.)
# In GitHub Actions, store as secret KUBECONFIG_PRODUCTION
# Usage in workflow:
# echo "${{ secrets.KUBECONFIG_PRODUCTION }}" > /tmp/kubeconfig.yaml
# kubectl --kubeconfig=/tmp/kubeconfig.yaml get pods
```

---

## 11. Step 6 — RBAC Configuration for Remote Users

### 11.1 The RBAC Model for Remote Access

```bash
# Remote access is only as powerful as the RBAC rules allow
# Always create the minimum necessary permissions

# ── ROLE 1: Developer in their namespace ────────────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-access
  namespace: production
rules:
  # Read workloads
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/status", "services", "endpoints", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
    verbs: ["get", "list", "watch"]
  # Deploy workloads
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets"]
    verbs: ["update", "patch"]
  # Debug access
  - apiGroups: [""]
    resources: ["pods/exec", "pods/portforward"]
    verbs: ["create"]
  # View events
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-access-binding
  namespace: production
subjects:
  - kind: User
    name: devops-user
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: team-a
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-access
  apiGroup: rbac.authorization.k8s.io
EOF

# ── ROLE 2: Read-only for entire cluster (SRE observer) ─────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sre-readonly-binding
subjects:
  - kind: User
    name: devops-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view           # Built-in read-only cluster role
  apiGroup: rbac.authorization.k8s.io
EOF

# ── ROLE 3: CI/CD pipeline with deploy permissions ──────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: ci-deployer
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "update", "patch"]
  - apiGroups: ["apps"]
    resources: ["deployments/scale"]
    verbs: ["update"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list"]
  # Allow reading secrets for deployment (but not creating/deleting)
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]
    # Restrict by resource names if possible:
    resourceNames: ["registry-credentials", "app-config-secret"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: github-actions
    namespace: production
roleRef:
  kind: Role
  name: ci-deployer
  apiGroup: rbac.authorization.k8s.io
EOF
```

### 11.2 Verifying RBAC Works for Remote Users

```bash
# Test from the remote machine (after applying RBAC):

# Does the user have access to pods?
kubectl auth can-i get pods -n production
# yes

# Does the user have access to secrets?
kubectl auth can-i get secrets -n production
# no

# Full permissions list for current user
kubectl auth can-i --list -n production

# Test as a specific user (requires impersonate RBAC permission)
kubectl auth can-i get pods -n production \
  --as=devops-user
kubectl auth can-i delete namespaces \
  --as=devops-user
# no

# Test as ServiceAccount
kubectl auth can-i update deployments -n production \
  --as=system:serviceaccount:production:github-actions
# yes
```

---

## 12. Step 7 — Multi-Cluster and Multi-Context Access Management

### 12.1 Managing Multiple kubeconfigs

```bash
# Install kubectx and kubens (context and namespace switchers)
# macOS
brew install kubectx

# Linux
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -s /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -s /opt/kubectx/kubens /usr/local/bin/kubens

# Or via krew
kubectl krew install ctx
kubectl krew install ns

# List all contexts
kubectx
# or
kubectl config get-contexts

# Switch to production context
kubectx production
# or
kubectl config use-context production

# Switch to previous context
kubectx -

# Switch namespace
kubens kube-system
# or
kubectl config set-context --current --namespace=kube-system

# List current context and namespace
kubectl config current-context
kubectl config view --minify | grep namespace
```

### 12.2 Organizing Multiple Cluster kubeconfigs

```bash
# Best practice: Separate kubeconfig files per cluster
# Then merge them for daily use

# File structure:
mkdir -p ~/.kube/clusters
~/.kube/clusters/production.yaml
~/.kube/clusters/staging.yaml
~/.kube/clusters/development.yaml
~/.kube/config  # Merged file

# Merge all cluster configs:
KUBECONFIG=$(find ~/.kube/clusters -name "*.yaml" | tr '\n' ':')
export KUBECONFIG
kubectl config view --flatten > ~/.kube/config
unset KUBECONFIG
chmod 600 ~/.kube/config

# Rename contexts to be meaningful
kubectl config rename-context \
  kubernetes-admin@kubernetes \
  production

kubectl config rename-context \
  arn:aws:eks:us-east-1:123456789:cluster/staging \
  staging

# List renamed contexts
kubectl config get-contexts
# CURRENT   NAME         CLUSTER              AUTHINFO      NAMESPACE
# *         production   production-cluster   admin         default
#           staging      staging-cluster      eks-user      default
#           development  dev-cluster          dev-user      dev
```

### 12.3 Context Switching Safety for Production

```bash
# DANGER: Accidentally running production commands from wrong context
# Install kubie for isolated context shells

# Install kubie (prevents cross-context accidents)
brew install kubie  # macOS
# or download from: https://github.com/sbstp/kubie

# Open isolated shell with production context
kubie ctx production
# All kubectl commands in this shell ONLY affect production
kubectl get nodes  # Safe: only production
exit  # Returns to parent shell with previous context

# Alternative: prompt indicator showing current context
# Add to ~/.bashrc or ~/.zshrc:
KUBE_PS1_SYMBOL_ENABLE=false
source ~/.kube/kube-ps1/kube-ps1.sh
PS1='$(kube_ps1) \$ '

# Color-code production red:
kube_ps1_ctx_color() {
  local ctx=$(kubectl config current-context 2>/dev/null)
  [[ "$ctx" == *"production"* ]] && echo "red" || echo "green"
}
```

---

## 13. Built-in Controllers Deep Dive — Understanding What You're Accessing

When you access a Kubernetes cluster remotely, you're interacting with the API server, which manages state that built-in controllers continuously reconcile. Understanding each controller makes you a more effective operator.

### 13.1 ReplicaSet Controller

**What it does:** Ensures a specified number of Pod replicas are running at all times. The self-healing backbone of stateless applications.

**Reconciliation logic:**
```
desired = replicaset.spec.replicas
current = count(pods matching replicaset.spec.selector that are NOT terminating)
delta = desired - current

if delta > 0: create |delta| new Pods
if delta < 0: delete |delta| Pods (preferring those without a Node, then failing, then oldest)
```

**Remote access perspective:**
```bash
# Observe ReplicaSet controller in action
kubectl apply -f - << 'EOF'
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
EOF

# Watch Pods being created by the RS controller (from remote machine)
kubectl get pods -l app=nginx -w

# Delete a Pod and watch self-healing
kubectl delete pod $(kubectl get pods -l app=nginx -o name | head -1)
kubectl get pods -l app=nginx -w
# A new Pod is created within seconds
```

### 13.2 Deployment Controller

**What it does:** Manages ReplicaSets to implement versioned, rollable updates. The primary way to deploy stateless applications.

**Reconciliation logic:**
```
When Pod template changes:
  1. Create new ReplicaSet with new template
  2. Scale up new RS (by maxSurge per iteration)
  3. Scale down old RS (by maxUnavailable per iteration)
  4. Repeat until new RS has all replicas, old RS has 0
```

**Remote access perspective:**
```bash
# Trigger rolling update from remote machine
kubectl set image deployment/my-app \
  container-name=my-app:v2.0 \
  -n production

# Monitor the rollout from remote
kubectl rollout status deployment/my-app -n production
# Waiting for deployment "my-app" rollout to finish: 1 out of 3 new replicas updated

# Watch in real-time
kubectl get replicasets -l app=my-app -w

# Rollback if something goes wrong
kubectl rollout undo deployment/my-app -n production

# View rollout history
kubectl rollout history deployment/my-app -n production
```

### 13.3 Node Controller

**What it does:** Monitors node health by tracking kubelet heartbeats. Evicts Pods from unhealthy nodes and assigns Pod CIDRs to new nodes.

**Remote access perspective:**
```bash
# Check node health status from remote machine
kubectl get nodes -o wide

# Detailed node conditions (Ready, MemoryPressure, DiskPressure, PIDPressure)
kubectl describe nodes | grep -A 5 "Conditions:"

# Watch for node condition changes
kubectl get nodes -w

# The Node controller behavior when a node goes NotReady:
# T+0:    Node kubelet stops sending heartbeats
# T+40s:  node-monitor-grace-period expires → condition Unknown
# T+5m:   pod-eviction-timeout → Pods evicted (with taint NoExecute)

# Check when node-controller last updated a node
kubectl get node k8s-worker-1 \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")].lastHeartbeatTime}'
```

### 13.4 Service Controller

**What it does:** Provisions cloud load balancers for Services of type LoadBalancer. For bare-metal clusters without cloud providers, it remains idle unless you use MetalLB or similar.

**Remote access perspective:**
```bash
# Create a LoadBalancer service and watch the controller provision the LB
kubectl expose deployment my-app \
  --type=LoadBalancer \
  --port=80 \
  --target-port=8080 \
  -n production

# Watch external IP assignment
kubectl get svc my-app -n production -w
# Initially: EXTERNAL-IP = <pending>
# After LB provisioned: EXTERNAL-IP = 52.1.2.3

# Check Service controller events
kubectl describe svc my-app -n production | grep -A 10 "Events:"
```

### 13.5 Namespace Controller

**What it does:** Manages Namespace lifecycle. When a Namespace is deleted, it cascades deletion of ALL resources in that namespace.

```bash
# Demonstrate namespace controller cascade
kubectl create namespace test-delete
kubectl create deployment test-app \
  --image=nginx \
  --replicas=3 \
  -n test-delete

# Watch cascade deletion
kubectl delete namespace test-delete &
kubectl get pods -n test-delete -w
# All Pods, Deployments, Services, ConfigMaps etc. deleted automatically
```

### 13.6 Job Controller

**What it does:** Creates Pods to run tasks to completion. Manages parallelism, retries, and completion tracking.

```bash
# Create a batch Job from remote machine
cat << 'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
spec:
  completions: 3        # Run 3 successful completions
  parallelism: 2        # Run 2 at a time
  backoffLimit: 4       # Retry up to 4 times on failure
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: my-migrator:v1.0
          command: ["./migrate.sh"]
EOF

# Monitor Job progress from remote
kubectl get job data-migration -w
kubectl get pods -l job-name=data-migration
```

### 13.7 StatefulSet Controller

**What it does:** Manages stateful applications that need stable network identities and persistent storage. Creates Pods in order (0, 1, 2) and updates in reverse (2, 1, 0).

```bash
# StatefulSet requires a headless Service (for stable DNS)
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: "postgres-headless"  # Reference headless service
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
          image: postgres:14
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:  # Each Pod gets its OWN PVC
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
EOF

# From remote: connect to specific StatefulSet pod by stable DNS name
kubectl exec -it postgres-0 -- \
  psql -U postgres -h postgres-0.postgres-headless.default.svc.cluster.local
```

### 13.8 DaemonSet Controller

**What it does:** Ensures one Pod runs on every node (or selected nodes). Used for node-level agents (monitoring, logging, networking).

```bash
# DaemonSet example: Deploy Prometheus node-exporter on every worker
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      tolerations:         # Run even on control-plane nodes (optional)
        - key: node-role.kubernetes.io/control-plane
          effect: NoSchedule
      hostNetwork: true
      containers:
        - name: node-exporter
          image: prom/node-exporter:latest
          ports:
            - containerPort: 9100
              hostPort: 9100
EOF

# Verify one Pod per node from remote
kubectl get pods -n monitoring -l app=node-exporter -o wide
# NAME                   READY   NODE           IP
# node-exporter-abc12    1/1     k8s-worker-1   10.0.0.20
# node-exporter-def34    1/1     k8s-worker-2   10.0.0.21
# node-exporter-xyz56    1/1     k8s-cp-1       10.0.0.10
```

### 13.9 Garbage Collector

**What it does:** Deletes "orphaned" resources using `ownerReferences`. When a parent (Deployment) is deleted, the Garbage Collector deletes the children (ReplicaSet → Pods).

```bash
# See ownerReferences in action
kubectl get pod <pod-name> \
  -o jsonpath='{.metadata.ownerReferences}' | jq
# [{"apiVersion":"apps/v1","blockOwnerDeletion":true,
#   "controller":true,"kind":"ReplicaSet","name":"nginx-rs-abc123",...}]

# Cascade deletion (default):
kubectl delete deployment nginx
# Deletes Deployment → ReplicaSet → Pods (in background)

# Orphan deletion (keep children):
kubectl delete deployment nginx \
  --cascade=orphan
# Deletes Deployment only; ReplicaSet and Pods remain (orphaned)

# Find orphaned ReplicaSets (common after manual deletion)
kubectl get replicasets --all-namespaces \
  -o json | jq \
  '.items[] | select(.spec.replicas == 0) | .metadata.name'
```

### 13.10 PersistentVolume Controller

**What it does:** Binds PersistentVolumeClaims to PersistentVolumes. Manages the lifecycle of storage.

```bash
# Create a PVC and watch the controller bind it
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
EOF

# Watch binding from remote
kubectl get pvc app-data -n production -w
# Initially: STATUS = Pending
# After binding: STATUS = Bound

# Check why a PVC is stuck in Pending
kubectl describe pvc app-data -n production
# Look for: "no persistent volumes available" or
#           "waiting for first consumer"
```

---

## 14. Internal Working Concepts — Informers, Work Queues & Reconciliation

### 14.1 The Informer Pattern — Why It Matters for Remote Operators

When you run `kubectl get pods -w` from a remote machine, you're receiving events through the same informer mechanism that controllers use internally.

```
┌─────────────────────────────────────────────────────────────────┐
│                    INFORMER INTERNALS                           │
│                                                                 │
│  Remote kubectl -w command                                      │
│       │                                                         │
│       │ HTTP GET /api/v1/pods?watch=true&resourceVersion=X      │
│       ▼                                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  kube-apiserver                         │   │
│  │  ┌─────────────────────────────────────────────────┐   │   │
│  │  │  Watch Cache (in-memory)                        │   │   │
│  │  │  Backed by etcd watch (not one per client!)     │   │   │
│  │  │  Multiplexes to all watchers                    │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
│       │                                                         │
│       │ Streaming events: ADDED, MODIFIED, DELETED             │
│       ▼                                                         │
│  kubectl receives and formats events for display                │
│                                                                 │
│  Controllers (in kube-controller-manager):                      │
│  Same mechanism — but they process events in work queues        │
└─────────────────────────────────────────────────────────────────┘
```

### 14.2 Work Queue — Why Commands Don't Return Immediately

```bash
# kubectl apply returns immediately — but the work isn't done yet
kubectl apply -f large-deployment.yaml
# Command returns: deployment.apps/large-app configured

# But the Pods aren't created yet! The work is in the controller queue.
kubectl get pods -l app=large-app
# May show 0/10 pods ready initially

# Wait for the reconciliation to complete
kubectl rollout status deployment/large-app --timeout=5m
# Waiting for deployment "large-app" rollout to finish: 3 of 10 updated...
# deployment "large-app" successfully rolled out

# Under the hood:
# 1. kubectl apply PUT request to API server
# 2. API server writes to etcd
# 3. Deployment controller informer receives MODIFIED event
# 4. Work queue item added: "default/large-app"
# 5. Worker goroutine picks up item
# 6. Reconcile: calculate desired vs current (10 pods vs 0)
# 7. Create ReplicaSet → Pods, 10 at a time with rolling update limits
```

### 14.3 Reconciliation Loop — Observing It From Remote

```bash
# Artificially create a reconciliation trigger and observe
# Create a deployment with 3 replicas
kubectl create deployment demo --image=nginx --replicas=3

# In Terminal 1 (remote): Watch pods
kubectl get pods -l app=demo -w &

# In Terminal 2 (remote): Delete a pod directly
kubectl delete pod $(kubectl get pods -l app=demo -o name | head -1)

# Terminal 1 shows:
# demo-xxx   0/1  Terminating   0    2m
# demo-yyy   0/1  ContainerCreating 0  0s   ← New pod, created by RS controller
# demo-yyy   1/1  Running       0    3s

# The reconciliation happened in < 100ms:
# 1. Pod deletion → etcd event
# 2. RS controller informer notified
# 3. Work queue item: "default/demo-rs-abc"
# 4. Compare: desired=3, current=2 → create 1 pod
# 5. New pod created

kill %1  # Stop background watch
```

---

## 15. API Server and etcd Interaction — The Access Enforcement Layer

### 15.1 The Core Principle

```
╔═══════════════════════════════════════════════════════════════════╗
║                                                                   ║
║  REMOTE CLIENTS NEVER ACCESS ETCD DIRECTLY.                       ║
║                                                                   ║
║  kubectl → API server → etcd                                      ║
║                                                                   ║
║  The API server enforces:                                         ║
║  1. TLS encryption in transit                                     ║
║  2. Authentication (who are you?)                                 ║
║  3. Authorization / RBAC (what can you do?)                       ║
║  4. Admission control (is this request valid?)                    ║
║  5. Audit logging (recording every action)                        ║
║                                                                   ║
║  etcd is only reachable BY the API server.                        ║
║  It's firewalled at port 2379 to only accept connections          ║
║  from the control plane nodes where kube-apiserver runs.          ║
║                                                                   ║
╚═══════════════════════════════════════════════════════════════════╝
```

### 15.2 Tracing a Remote kubectl Command to etcd

```bash
# What happens when you run (from remote machine):
kubectl get deployments -n production

# Step 1: kubectl builds HTTP request
# GET https://10.0.0.10:6443/apis/apps/v1/namespaces/production/deployments
# Headers: Authorization: Bearer <token>
#          Accept: application/json

# Step 2: API server receives request
# - Verifies TLS client certificate
# - Extracts identity: CN=devops-user, O=developers
# - RBAC check: can devops-user GET deployments in production?
# - Reads from watch cache (served from memory, not etcd for reads!)
# - Returns JSON list of Deployment objects

# See exactly what HTTP calls kubectl makes:
kubectl get deployments -n production -v=6 2>&1 | grep "GET\|Response"
# GET https://10.0.0.10:6443/apis/apps/v1/namespaces/production/deployments 200 OK

# For writes (apply, delete), etcd IS written to:
kubectl apply -f deployment.yaml -v=6 2>&1 | grep "PATCH\|PUT\|POST"
# POST https://10.0.0.10:6443/apis/apps/v1/namespaces/production/deployments 201 Created
```

### 15.3 API Server Response Codes from Remote Perspective

| HTTP Code | Meaning | Common Remote Access Cause |
|---|---|---|
| 200 OK | Success | Read operation succeeded |
| 201 Created | Resource created | POST (create) succeeded |
| 404 Not Found | Resource doesn't exist | Wrong namespace or name |
| 401 Unauthorized | Authentication failed | Invalid/expired credentials |
| 403 Forbidden | Authorization denied | No RBAC permission |
| 409 Conflict | Resource version conflict | Concurrent modification |
| 422 Unprocessable | Validation failed | Invalid manifest YAML |
| 429 Too Many Requests | Rate limited | Too many API calls |
| 500 Internal Server Error | Server error | etcd or controller issue |
| 503 Service Unavailable | API server not ready | Control plane issue |

---

## 16. Leader Election — HA Considerations for Remote Access Infrastructure

### 16.1 How Leader Election Affects Remote Access

In HA setups, multiple API servers, controller managers, and schedulers run simultaneously. Remote clients connect to any available API server (through a load balancer). Understanding leader election helps explain behavior during failovers.

```bash
# The kube-controller-manager uses leader election
# Only ONE instance actively reconciles at a time
# Others are warm standbys

# Check who is the current leader
kubectl get lease kube-controller-manager -n kube-system -o yaml
# spec.holderIdentity: "k8s-cp-1_abc-123"  ← Current leader
# spec.renewTime: "2025-01-01T14:30:00Z"  ← Must renew to stay leader
# spec.leaseDurationSeconds: 30           ← Lease expires after 30s without renewal

# Monitor leader changes (in an HA cluster)
kubectl get lease kube-controller-manager -n kube-system -w

# What happens to remote access if the current leader dies?
# 1. Lease expires (after leaseDurationSeconds: 30s default)
# 2. A standby instance acquires the lease and becomes leader
# 3. New leader re-Lists all resources from API server
# 4. Reconciliation resumes

# During the ~30s leadership gap:
# - Existing Pods/Services continue running (kube-proxy rules intact)
# - No new Pods created for scaling events
# - Rolling updates pause
# - Remote kubectl get/list/watch commands still work (API server is HA)
```

### 16.2 Leader Election Flags

```bash
# View current kube-controller-manager flags (from remote)
kubectl get pod kube-controller-manager-k8s-cp-1 -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | \
  tr ',' '\n' | grep 'leader'

# Typical settings:
# --leader-elect=true
# --leader-elect-lease-duration=30s    # How long before leader considered dead
# --leader-elect-renew-deadline=25s    # Leader must renew this often
# --leader-elect-retry-period=5s       # How often standbys try to acquire
```

### 16.3 HA API Server for Reliable Remote Access

```
For reliable remote access in production:
┌─────────────────────────────────────────────────────────────┐
│  Remote Machine (your laptop)                               │
│         │                                                   │
│         │ HTTPS :6443                                       │
│         ▼                                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  HAProxy / Cloud Load Balancer (VIP: 10.0.0.100)   │   │
│  │  health-check: GET /healthz                         │   │
│  └──────────┬────────────────┬──────────────────────────┘  │
│             │                │                              │
│  ┌──────────▼──────┐  ┌──────▼──────────┐                  │
│  │  kube-apiserver │  │  kube-apiserver │  (Active-Active)  │
│  │  on k8s-cp-1    │  │  on k8s-cp-2   │                   │
│  └─────────────────┘  └─────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

```bash
# For HA remote access, use the load balancer IP in kubeconfig
kubectl config set-cluster production-cluster \
  --server=https://10.0.0.100:6443 \  # VIP, not individual node IP
  --certificate-authority=cluster-ca.crt \
  --embed-certs=true

# Verify HA connectivity
kubectl get nodes  # Works even if one API server is down
```

---

## 17. Advanced Remote Access Patterns

### 17.1 kubectl Port-Forward — Accessing Services Remotely

```bash
# Access a Service or Pod directly without exposing it externally
# Traffic flows: remote machine → API server → kubelet → Pod

# Forward local port to a Service
kubectl port-forward svc/postgres \
  5432:5432 \
  -n production &

# Connect from remote machine
psql -h localhost -p 5432 -U postgres

# Forward local port to a specific Pod
kubectl port-forward pod/my-app-7d4f9-abc \
  8080:8080 \
  -n production &

# Access the pod's HTTP endpoint
curl http://localhost:8080/health

# Bind to specific address (default binds to localhost only)
kubectl port-forward \
  --address 0.0.0.0 \  # Allow connections from other machines too
  svc/my-service \
  8080:80 \
  -n production

# Run in background with logging
kubectl port-forward svc/grafana 3000:3000 -n monitoring \
  > /tmp/port-forward.log 2>&1 &
PF_PID=$!
echo "Port-forward PID: $PF_PID"

# Cleanup
kill $PF_PID
```

### 17.2 kubectl Proxy — API Server Proxy

```bash
# Start a local HTTP proxy to the Kubernetes API server
# This is useful for exploring the API without TLS/auth setup
kubectl proxy --port=8001 &

# Now access the API via localhost (no authentication needed through proxy)
curl http://localhost:8001/api/v1/namespaces
curl http://localhost:8001/api/v1/namespaces/production/pods

# Access specific pod through proxy (via API server subresource)
curl http://localhost:8001/api/v1/namespaces/production/pods/nginx-pod/proxy/

# WARNING: kubectl proxy has the SAME permissions as your kubeconfig user
# Anyone who can reach localhost:8001 has your cluster access level
# ALWAYS bind to 127.0.0.1 (default) — never to 0.0.0.0 in production

# Stop the proxy
kill %1

# Alternative: Use for accessing Kubernetes Dashboard without Ingress
kubectl port-forward svc/kubernetes-dashboard \
  8443:443 \
  -n kubernetes-dashboard &
# Then access: https://localhost:8443
```

### 17.3 SSH Tunneling for Private Clusters

```bash
# When API server is on private network (10.0.0.10)
# and you have SSH access to a bastion/jump host in the same network

# Option 1: SSH -L tunnel (local port forwarding)
# Creates local port 6443 → bastion → API server 10.0.0.10:6443
ssh -L 6443:10.0.0.10:6443 \
    -i ~/.ssh/bastion-key.pem \
    ubuntu@bastion.example.com \
    -N &   # -N: no remote command, just forward
TUNNEL_PID=$!

# Update kubeconfig to use localhost
kubectl config set-cluster production-cluster \
  --server=https://localhost:6443

# Now kubectl works through the tunnel
kubectl get nodes

# Cleanup
kill $TUNNEL_PID
# Restore original server
kubectl config set-cluster production-cluster \
  --server=https://10.0.0.10:6443

# Option 2: ProxyJump (SSH jump host — cleaner)
# In ~/.ssh/config:
cat >> ~/.ssh/config << 'EOF'
Host k8s-cp-1
    HostName 10.0.0.10
    User ubuntu
    ProxyJump bastion.example.com
    IdentityFile ~/.ssh/cluster-key.pem
EOF

# Now SSH directly to control plane via bastion
ssh k8s-cp-1
# And port-forward via jump
ssh -L 6443:localhost:6443 k8s-cp-1 -N &
```

### 17.4 Direct API Access (Without kubectl)

```bash
# Sometimes you need to access the API directly (custom scripts, non-kubectl tools)

# Method 1: Using curl with TLS
TOKEN=$(kubectl config view --raw \
  -o jsonpath='{.users[?(@.name=="devops-user")].user.token}')

CACERT_DATA=$(kubectl config view --raw \
  -o jsonpath='{.clusters[0].cluster.certificate-authority-data}' | base64 -d)

echo "$CACERT_DATA" > /tmp/ca.crt

curl \
  --cacert /tmp/ca.crt \
  -H "Authorization: Bearer ${TOKEN}" \
  "${K8S_API_SERVER}/api/v1/namespaces/production/pods" | \
  jq '.items[].metadata.name'

# Method 2: Using client certificate
CERT_DATA=$(kubectl config view --raw \
  -o jsonpath='{.users[?(@.name=="devops-user")].user.client-certificate-data}' | base64 -d)
KEY_DATA=$(kubectl config view --raw \
  -o jsonpath='{.users[?(@.name=="devops-user")].user.client-key-data}' | base64 -d)

echo "$CERT_DATA" > /tmp/client.crt
echo "$KEY_DATA" > /tmp/client.key

curl \
  --cacert /tmp/ca.crt \
  --cert /tmp/client.crt \
  --key /tmp/client.key \
  "${K8S_API_SERVER}/apis/apps/v1/namespaces/production/deployments" | \
  jq '.items[].metadata.name'

# Method 3: Using Python kubernetes client
pip install kubernetes

python3 << 'EOF'
from kubernetes import client, config

# Load from kubeconfig
config.load_kube_config()

v1 = client.CoreV1Api()
pods = v1.list_namespaced_pod("production")
for pod in pods.items:
    print(f"Pod: {pod.metadata.name}, Status: {pod.status.phase}")
EOF
```

---

## 18. Performance Tuning for Remote API Access

### 18.1 API Server Request Limits

```bash
# Check current API server concurrency limits
kubectl get pod kube-apiserver-k8s-cp-1 -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | \
  tr ' ' '\n' | grep 'inflight'

# Key flags to tune for heavy remote access:
# --max-requests-inflight=800         (default: 400)
# --max-mutating-requests-inflight=400 (default: 200)

# Monitor inflight requests
kubectl get --raw /metrics 2>/dev/null | \
  grep 'apiserver_current_inflight_requests'
```

### 18.2 Client-Side Performance Optimization

```bash
# Problem: Using --all-namespaces with large clusters is slow
kubectl get pods --all-namespaces  # Queries every namespace sequentially

# Better: Use namespace-specific queries
kubectl get pods -n production    # Much faster

# Problem: Watching with -w creates a persistent connection
# Use shorter watches with timeouts
kubectl get pods -w \
  --request-timeout=60s   # Reconnect every 60s

# Problem: -o yaml on large collections is very slow
kubectl get pods --all-namespaces -o yaml  # Returns huge payload

# Better: Use jsonpath to get only what you need
kubectl get pods --all-namespaces \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'

# Or use jq for filtering
kubectl get pods --all-namespaces -o json | \
  jq '.items[] | select(.status.phase == "Pending") | .metadata.name'

# Set appropriate request timeout
kubectl get pods --request-timeout=30s

# Use --chunk-size for very large collections
kubectl get pods --all-namespaces --chunk-size=100
```

### 18.3 Concurrent Access Performance

```bash
# For parallel operations (avoid hitting API rate limits)

# BAD: Sequential loops
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  kubectl get pods -n $ns  # Each call = separate API request
done

# BETTER: Single API call with all namespaces
kubectl get pods --all-namespaces

# BEST: Use labels and field selectors
kubectl get pods --all-namespaces \
  -l environment=production \
  --field-selector status.phase=Running

# For batch operations, use background jobs with rate limiting
for deploy in deployment1 deployment2 deployment3; do
  kubectl rollout restart deployment/$deploy -n production &
  sleep 0.5  # Small delay between requests
done
wait  # Wait for all background jobs
```

### 18.4 Important Performance Flags Reference

| Flag | Component | Default | Production Recommendation |
|---|---|---|---|
| `--max-requests-inflight` | kube-apiserver | 400 | 800-1200 for large clusters |
| `--max-mutating-requests-inflight` | kube-apiserver | 200 | 400-600 |
| `--request-timeout` | kubectl (client) | 0 (no timeout) | 60s for interactive, 300s for CI/CD |
| `--watch-cache-sizes` | kube-apiserver | auto | Tune per resource type |
| `QPS` | Go client | 5 | 50-100 for automation |
| `Burst` | Go client | 10 | 100-200 for automation |

---

## 19. Security Hardening for Remote Access

### 19.1 RBAC Best Practices

```bash
# Principle 1: Never use wildcards in production RBAC
# BAD:
cat << 'BADEOF'
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
BADEOF

# GOOD: Specific resources and verbs only
cat << 'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-updater
  namespace: production
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "update", "patch"]
    resourceNames: ["my-app", "my-api"]  # Restrict to specific deployments
EOF

# Principle 2: Audit all ClusterRoleBindings quarterly
kubectl get clusterrolebindings -o json | \
  jq '.items[] | 
      select(.roleRef.name == "cluster-admin") |
      {name: .metadata.name, subjects: .subjects}'

# Principle 3: Use groups for RBAC, not individual users
# Groups from certificate O= field or OIDC groups claim
# Adding a user to a group gives them all associated permissions
# Removing from group removes all permissions immediately
```

### 19.2 TLS Configuration for Remote Access

```bash
# Verify minimum TLS version for API server
kubectl get pod kube-apiserver-k8s-cp-1 -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | \
  tr ' ' '\n' | grep 'tls-min-version'

# Should be at minimum: --tls-min-version=VersionTLS12
# For new clusters: --tls-min-version=VersionTLS13

# Test TLS version from remote machine
openssl s_client \
  -connect 10.0.0.10:6443 \
  -tls1_2 </dev/null 2>&1 | grep "Protocol\|Cipher"

# Verify cipher suites
openssl s_client \
  -connect 10.0.0.10:6443 \
  -cipher 'ECDHE-RSA-AES256-GCM-SHA384' </dev/null 2>&1 | \
  grep "Cipher\|Protocol"

# CRITICAL: Never use insecure-skip-tls-verify in production
# This completely disables certificate validation!
# Anyone can MITM your kubectl commands
kubectl get nodes --insecure-skip-tls-verify  # NEVER IN PRODUCTION!
```

### 19.3 ServiceAccount Security for Automation

```bash
# Use dedicated ServiceAccounts per application/pipeline (never share)
# Disable automount for ServiceAccounts that don't need API access

# Disable automounting globally for default SA
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  kubectl patch serviceaccount default \
    -n $ns \
    -p '{"automountServiceAccountToken": false}'
done

# For CI/CD: Use short-lived tokens instead of long-lived secrets
# GitHub Actions OIDC federation (no long-lived secrets needed)
cat << 'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: github-actions-oidc
subjects:
  - kind: User
    # GitHub OIDC format: repo:owner/repo:ref:refs/heads/main
    name: "repo:myorg/myrepo:ref:refs/heads/main"
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: ci-deployer-role
  apiGroup: rbac.authorization.k8s.io
EOF

# Audit ServiceAccounts with non-default permissions
kubectl get rolebindings,clusterrolebindings \
  --all-namespaces -o json | \
  jq '.items[] | 
      select(.subjects != null) |
      select(.subjects[] | .kind == "ServiceAccount") |
      {binding: .metadata.name, sa: [.subjects[] | .name]}'
```

### 19.4 kubeconfig Security Practices

```bash
# File permissions (must be 600 — no group or world access)
chmod 600 ~/.kube/config
chmod 600 ~/.kube/clusters/*.yaml

# Verify permissions
ls -la ~/.kube/config
# -rw------- (600) is correct
# -rw-r--r-- (644) is WRONG — others can read your credentials

# CRITICAL: Check kubeconfig is not in git history
git log --all --full-history -- '~/.kube/config' 2>/dev/null
# Should be empty

# Add to .gitignore
echo '*.kubeconfig' >> .gitignore
echo '.kube/' >> ~/.gitignore_global
git config --global core.excludesfile ~/.gitignore_global

# Use vault for kubeconfig storage in CI/CD
# Instead of KUBECONFIG in CI env vars, use HashiCorp Vault:
vault kv put secret/k8s/production kubeconfig=@~/.kube/config

# Retrieve in CI:
vault kv get -field=kubeconfig secret/k8s/production > /tmp/kubeconfig.yaml
kubectl --kubeconfig=/tmp/kubeconfig.yaml get pods
shred -u /tmp/kubeconfig.yaml  # Securely delete after use
```

---

## 20. Monitoring and Observability for Remote Access Sessions

### 20.1 Audit Logs — Every Remote Action is Recorded

```bash
# Check if audit logging is enabled (on control plane)
kubectl get pod kube-apiserver-k8s-cp-1 -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | \
  tr ' ' '\n' | grep 'audit'

# Expected output:
# --audit-log-path=/var/log/kubernetes/audit.log
# --audit-log-maxage=30
# --audit-policy-file=/etc/kubernetes/audit-policy.yaml

# From remote machine: View recent audit events
# (requires kubectl exec into the API server pod or SSH to control plane)
ssh ubuntu@10.0.0.10 \
  "tail -100 /var/log/kubernetes/audit.log" | \
  jq 'select(.user.username != "system:apiserver") | 
      {user: .user.username, 
       verb: .verb, 
       resource: .objectRef.resource,
       namespace: .objectRef.namespace,
       time: .requestReceivedTimestamp}'

# Find all 403 Forbidden events (failed access attempts)
ssh ubuntu@10.0.0.10 \
  "tail -1000 /var/log/kubernetes/audit.log" | \
  jq 'select(.responseStatus.code == 403) | 
      {user: .user.username, 
       verb: .verb, 
       resource: .objectRef.resource}'

# Find who deleted what
ssh ubuntu@10.0.0.10 \
  "grep '\"verb\":\"delete\"' /var/log/kubernetes/audit.log" | \
  tail -20 | \
  jq '{user: .user.username, 
       resource: .objectRef.resource, 
       name: .objectRef.name,
       time: .requestReceivedTimestamp}'
```

### 20.2 Key Prometheus Metrics for Remote Access Monitoring

| Metric | Type | Description | Alert Threshold |
|---|---|---|---|
| `apiserver_request_total{code="401"}` | Counter | Auth failures | > 5/minute |
| `apiserver_request_total{code="403"}` | Counter | RBAC denials | > 20/minute |
| `apiserver_request_duration_seconds` | Histogram | Request latency | p99 > 1s |
| `apiserver_current_inflight_requests` | Gauge | Concurrent requests | > 80% of max |
| `rest_client_request_duration_seconds` | Histogram | kubectl call latency | p99 > 500ms |
| `rest_client_requests_total{code="5xx"}` | Counter | Server errors | > 0 sustained |
| `apiserver_audit_requests_rejected_total` | Counter | Audit write failures | > 0 |
| `kubernetes_client_certificate_expiration_seconds` | Gauge | Cert expiry | < 7 days |

```bash
# Check API server metrics from remote machine
# (requires appropriate ClusterRole permission)
kubectl get --raw /metrics | \
  grep 'apiserver_request_total' | \
  grep -v "^#" | \
  sort -t= -k4 -rn | head -20

# Check request latency histogram
kubectl get --raw /metrics | \
  grep 'apiserver_request_duration_seconds_bucket{.*verb="LIST"'

# Alert setup (Prometheus AlertManager rule)
cat << 'EOF' > /tmp/k8s-access-alerts.yaml
groups:
  - name: remote-access
    rules:
      - alert: HighKubernetesAuthFailureRate
        expr: sum(rate(apiserver_request_total{code="401"}[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High Kubernetes authentication failure rate"

      - alert: KubernetesCertExpiringSoon
        expr: |
          min(
            apiserver_client_certificate_expiration_seconds_count{job="apiserver"}
          ) < 604800
        labels:
          severity: critical
        annotations:
          summary: "Kubernetes client certificate expiring within 7 days"
EOF
kubectl apply -f /tmp/k8s-access-alerts.yaml
```

### 20.3 Monitoring Your Remote Access Sessions

```bash
# See currently connected clients to API server
kubectl get --raw /metrics | \
  grep 'rest_client_requests_total' | \
  grep -v "^#" | head -10

# Monitor active watches (persistent connections from remote tools)
kubectl get --raw /metrics | \
  grep 'apiserver_longrunning_requests' | \
  grep -v "^#"

# Check who is currently watching what
kubectl get --raw /metrics | \
  grep 'apiserver_watch_events_total' | \
  grep -v "^#"

# Real-time access logging (from remote machine, stream audit log)
kubectl exec -n kube-system kube-apiserver-k8s-cp-1 -- \
  tail -f /var/log/kubernetes/audit.log 2>/dev/null | \
  jq 'select(.verb != "watch" and .verb != "get") |
      {user: .user.username, verb: .verb, resource: .objectRef.resource}'
```

---

## 21. Troubleshooting Remote Access Issues

### 21.1 Diagnostic Decision Tree

```
kubectl command fails
       │
       ├── Error: "The connection to the server X:6443 was refused"
       │   → Network issue: firewall, VPN, wrong IP
       │   → Check: nc -zv <api-server-ip> 6443
       │
       ├── Error: "x509: certificate signed by unknown authority"
       │   → Wrong CA cert in kubeconfig
       │   → Fix: Update certificate-authority-data
       │
       ├── Error: "x509: certificate is valid for X, not Y"
       │   → API server cert doesn't have your IP/hostname as SAN
       │   → Fix: kubeadm certs renew apiserver --apiserver-cert-extra-sans=Y
       │
       ├── Error: "Unauthorized" (401)
       │   → Invalid/expired credentials
       │   → Check: cert expiry, token expiry, correct user
       │
       ├── Error: "Forbidden" (403)
       │   → Authentication passed, RBAC denies
       │   → Check: kubectl auth can-i <verb> <resource>
       │
       ├── Error: "context deadline exceeded" / "timeout"
       │   → Network latency, API server overloaded
       │   → Check: ping, traceroute, API server metrics
       │
       └── Error: "error: no configuration has been provided"
           → No kubeconfig found
           → Check: ~/.kube/config exists, KUBECONFIG env var
```

### 21.2 Pods Not Created After Remote kubectl apply

```bash
# Step 1: Verify the resource was created
kubectl get deployment <name> -n <namespace>
# If "NotFound" → kubectl apply failed or went to wrong namespace

# Step 2: Check if ReplicaSet was created
kubectl get replicasets -n <namespace> -l app=<app-name>
# If missing → Deployment controller may be down

# Step 3: Describe the deployment for events
kubectl describe deployment <name> -n <namespace>
# Look for: Events section, conditions

# Step 4: Check events
kubectl get events -n <namespace> \
  --field-selector involvedObject.name=<deployment-name> \
  --sort-by='.lastTimestamp'

# Common causes and fixes:
# "Insufficient CPU/memory" → scale cluster or reduce resource requests
kubectl describe nodes | grep -A 5 "Allocated resources:"

# "ImagePullBackOff" → invalid image or registry credentials
kubectl describe pod <failing-pod> | grep -A 10 "Events:"
kubectl create secret docker-registry regcred \
  --docker-server=<registry> \
  --docker-username=<user> \
  --docker-password=<pass> \
  -n <namespace>

# "0/3 nodes are available: 3 node(s) had taint" → node taints preventing scheduling
kubectl get nodes -o custom-columns="NAME:.metadata.name,TAINTS:.spec.taints"
kubectl taint nodes k8s-worker-1 key=value:NoSchedule-  # Remove taint

# Check kube-controller-manager is running (controls Pod creation)
kubectl get pods -n kube-system -l component=kube-controller-manager
kubectl logs -n kube-system kube-controller-manager-k8s-cp-1 --tail=50
```

### 21.3 Deployment Stuck — Rolling Update Not Progressing

```bash
# Check rollout status
kubectl rollout status deployment/<name> -n <namespace> --timeout=10s

# View current state of ReplicaSets
kubectl get replicasets -n <namespace> -l app=<name> -o wide

# Describe deployment for conditions
kubectl describe deployment <name> -n <namespace> | \
  grep -A 5 "Conditions:"
# Look for: ProgressDeadlineExceeded, ReplicaFailure

# Check why new Pods aren't becoming Ready
kubectl get pods -n <namespace> -l app=<name>
# STATUS column: CrashLoopBackOff, ImagePullBackOff, Pending, Error

# Debug failing new Pod
NEW_POD=$(kubectl get pods -n <namespace> -l app=<name> \
  --sort-by=.metadata.creationTimestamp -o name | tail -1)
kubectl describe $NEW_POD -n <namespace>
kubectl logs $NEW_POD -n <namespace>
kubectl logs $NEW_POD -n <namespace> --previous  # If crashed

# Common fixes:
# readinessProbe failing → fix health check endpoint or increase initialDelaySeconds
# OOMKilled → increase memory limits
# Wrong image tag → kubectl set image deployment/<name> container=image:correct-tag
# Config missing → check ConfigMaps and Secrets exist

# Rollback if needed
kubectl rollout undo deployment/<name> -n <namespace>
kubectl rollout status deployment/<name> -n <namespace>
```

### 21.4 Node NotReady — Remote Diagnosis

```bash
# Identify problem nodes
kubectl get nodes | grep NotReady
kubectl describe node <node-name> | grep -A 10 "Conditions:"

# Check what's running on the problem node
kubectl get pods --all-namespaces \
  --field-selector spec.nodeName=<node-name>

# If you have SSH access to the node (for deeper diagnosis):
ssh ubuntu@<node-ip>
systemctl status kubelet
journalctl -u kubelet -n 100 --no-pager | grep -i "error\|fail"

# Common causes:
# "network plugin is not ready" → CNI pod crashed on this node
kubectl get pods -n kube-system -o wide | grep <node-name>  # Check CNI pod

# "failed to connect to apiserver" → network issue
ping 10.0.0.10  # From the node

# "disk pressure" → node running out of disk space
df -h  # On the node

# "memory pressure" → node running out of memory
free -m  # On the node

# Remote remediation options:
# Cordon (stop scheduling new Pods)
kubectl cordon <node-name>

# Drain (evict all Pods)
kubectl drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data \
  --force

# After fixing the node:
kubectl uncordon <node-name>
```

### 21.5 Remote Access Specific Errors

```bash
# Error: "Unable to connect to the server: dial tcp X.X.X.X:6443: i/o timeout"
# → Network timeout (firewall blocking or VPN not connected)
ping 10.0.0.10              # Test ICMP
nc -zv 10.0.0.10 6443      # Test TCP port
traceroute 10.0.0.10       # Find where it stops

# Error: "context deadline exceeded"
# → Connection succeeded but response took too long
# Check API server load:
kubectl get --raw /healthz
kubectl get --raw /readyz

# Error: "Error loading config file ~/.kube/config"
# → Corrupted or invalid kubeconfig YAML
kubectl config view    # Will show YAML parse error
# Fix: Restore from backup or regenerate

# Error: "server doesn't have a resource type pods"
# → kubectl version much newer/older than cluster
kubectl version         # Check version skew
# Solution: Install matching kubectl version

# Error: "no such host" (DNS error)
# → API server hostname not resolvable
nslookup k8s-api.example.com
dig k8s-api.example.com
# Fix: Update /etc/hosts or fix DNS entry

# Test all of these systematically:
kubectl cluster-info dump --output-directory=/tmp/cluster-dump
# Dumps complete cluster state for offline analysis
```

---

## 22. Disaster Recovery — Maintaining Access During Failures

### 22.1 Stateless Controllers — Why Access Recovery Is Fast

```bash
# The kube-controller-manager is completely stateless
# All access state (RBAC rules, ServiceAccount tokens) lives in etcd
# Restarting the controller manager doesn't affect remote access

# What happens when controller-manager crashes:
# 1. Existing kubectl connections to API server continue working
# 2. RBAC enforcement continues (RBAC cache in API server)
# 3. ServiceAccount tokens remain valid (verified by API server)
# 4. New Pod scheduling pauses (but running Pods continue)
# 5. New tokens are not issued (until new leader takes over)

# Recovery is automatic:
# 1. kubelet detects static pod died → recreates it
# 2. New controller-manager instance starts
# 3. Leader election: wins Lease object
# 4. Re-Lists all resources from API server
# 5. Full reconciliation resumes

# From remote machine: Monitor recovery
kubectl get pods -n kube-system -l component=kube-controller-manager -w
```

### 22.2 etcd Backup — Protecting Access Configuration

```bash
# All RBAC rules, ServiceAccounts, and access config live in etcd
# If etcd is lost without backup, all access config must be rebuilt from scratch

# From remote machine (via API): List all RBAC resources to backup
kubectl get clusterroles,clusterrolebindings \
  -o yaml > /tmp/clusterrbac-backup.yaml

kubectl get roles,rolebindings \
  --all-namespaces \
  -o yaml > /tmp/namespacedrbac-backup.yaml

kubectl get serviceaccounts \
  --all-namespaces \
  -o yaml > /tmp/serviceaccounts-backup.yaml

# On control plane (physical etcd backup):
ETCDCTL_CMD="etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key"

kubectl exec -n kube-system etcd-k8s-cp-1 -- \
  $ETCDCTL_CMD snapshot save /tmp/etcd-backup.db

# Verify backup
kubectl exec -n kube-system etcd-k8s-cp-1 -- \
  $ETCDCTL_CMD snapshot status /tmp/etcd-backup.db \
  --write-out=table

# Extract backup from pod
kubectl cp kube-system/etcd-k8s-cp-1:/tmp/etcd-backup.db \
  ~/etcd-backup-$(date +%Y%m%d).db

# GitOps backup (strongly recommended)
# Store all Kubernetes manifests in Git
# Including RBAC, ServiceAccounts, kubeconfig templates
# If cluster is lost, git clone + kubectl apply restores everything
```

### 22.3 Emergency Access When API Server Is Down

```bash
# Last resort: Use admin.conf directly on the control plane node

# Step 1: SSH to control plane
ssh ubuntu@10.0.0.10

# Step 2: Use local admin credentials (doesn't need API server to be "ready")
kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes

# Step 3: Check why API server is down
sudo systemctl status kubelet
sudo crictl ps | grep apiserver
sudo crictl logs $(sudo crictl ps | grep apiserver | awk '{print $1}')

# Step 4: Check API server static pod manifest
sudo cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Step 5: Common fixes
# Certificate expired:
sudo kubeadm certs renew all
sudo systemctl restart kubelet

# etcd not responding:
sudo crictl ps | grep etcd
# If etcd is down, check its logs:
sudo crictl logs $(sudo crictl ps | grep etcd | awk '{print $1}')

# API server manifest corrupted:
# Restore from backup or kubeadm-generated manifest
```

---

## 23. Comparison — kubectl vs Direct API vs Dashboard vs Proxy

### 23.1 Remote Access Methods Comparison Table

| Method | Authentication | Use Case | Remote-Friendly | Audit Trail | Complexity |
|---|---|---|---|---|---|
| **kubectl (human)** | kubeconfig cert/token/OIDC | Daily ops, debugging | ✅ Yes | Full audit log | Low |
| **kubectl (CI/CD)** | ServiceAccount token | Automated deployment | ✅ Yes | Full audit log | Low |
| **Direct REST API** | Same as kubectl | Custom tooling | ✅ Yes | Full audit log | Medium |
| **Python/Go SDK** | Same as kubectl | Operators, controllers | ✅ Yes | Full audit log | High |
| **Kubernetes Dashboard** | Same as kubectl | Visual management | ✅ Yes (with port-forward) | Partial | Low |
| **kubectl proxy** | Same as kubectl | API exploration | ✅ Local only | Full audit log | Low |
| **kubectl port-forward** | Same as kubectl | Service access | ✅ Yes | Partial | Low |
| **SSH + kubectl** | SSH key + kubeconfig | Private cluster access | ✅ Yes | SSH + audit log | Medium |
| **Lens / k9s** | kubeconfig | Visual terminal UI | ✅ Yes | Via kubeconfig | Low |
| **Teleport** | SSO (no kubeconfig) | Zero-trust access | ✅ Yes | Centralized | Medium |

### 23.2 kubectl vs kube-apiserver vs kube-scheduler

| Aspect | kubectl (client) | kube-apiserver | kube-scheduler |
|---|---|---|---|
| **What it is** | CLI tool on your machine | Control plane server process | Control plane server process |
| **Location** | Remote machine | Control plane node | Control plane node |
| **Role** | Sends HTTP requests to API server | Receives and validates all requests | Assigns Pods to Nodes |
| **Authentication** | Presents credentials (kubeconfig) | Verifies credentials | Uses its own kubeconfig |
| **Authorization** | Subject to RBAC | Enforces RBAC | Has ClusterAdmin-equivalent via SA |
| **Port** | None (client) | 6443 (HTTPS) | 10259 (metrics) |
| **HA Model** | N/A (stateless CLI) | Active-Active (all instances serve) | Active-Passive (leader election) |
| **Persistence** | Reads kubeconfig from disk | Reads/writes etcd | Reads from API server only |
| **Failure impact** | User loses access (local issue) | Cluster management stops | New Pods not scheduled |

---

## 24. ASCII Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║         LAB: ACCESSING KUBERNETES CLUSTER FROM REMOTE MACHINE                   ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║                                                                                  ║
║  ┌─────────────────────────────────────────────────────────────────────────┐    ║
║  │                    REMOTE MACHINE (Your Laptop)                        │    ║
║  │                    IP: 192.168.1.100                                   │    ║
║  │                                                                         │    ║
║  │  ┌────────────────────────────────────────────────────────────────┐    │    ║
║  │  │  ~/.kube/config (kubeconfig)                                   │    │    ║
║  │  │  ┌─────────────────────┐  ┌──────────────────┐  ┌──────────┐  │    │    ║
║  │  │  │  clusters:          │  │  users:          │  │ contexts │  │    │    ║
║  │  │  │  - server: :6443    │  │  - cert+key      │  │ (binding)│  │    │    ║
║  │  │  │  - ca-cert-data     │  │  - or: token     │  │          │  │    │    ║
║  │  │  └─────────────────────┘  └──────────────────┘  └──────────┘  │    │    ║
║  │  └────────────────────────────────────────────────────────────────┘    │    ║
║  │                                                                         │    ║
║  │  kubectl  │  Python SDK  │  Helm  │  ArgoCD agent  │  Custom tools      │    ║
║  └─────────────────────────────────────────────────────────────────────────┘    ║
║                              │                                                   ║
║                              │ HTTPS :6443                                       ║
║                              │ (VPN / Direct / SSH tunnel)                       ║
║                              │                                                   ║
║  ┌─────────────────────────────────────────────────────────────────────────┐    ║
║  │                   NETWORK LAYER                                        │    ║
║  │                                                                         │    ║
║  │  ┌──────────┐    ┌──────────────┐    ┌──────────────────────────────┐  │    ║
║  │  │  VPN /   │    │  Firewall    │    │  HAProxy / Cloud LB          │  │    ║
║  │  │  Bastion │───►│  Rules       │───►│  VIP: 10.0.0.100:6443        │  │    ║
║  │  │  (Jump   │    │  TCP 6443    │    │  (for HA multi-CP clusters)  │  │    ║
║  │  │  Host)   │    │  allowed     │    └──────────────────────────────┘  │    ║
║  │  └──────────┘    └──────────────┘                                      │    ║
║  └─────────────────────────────────────────────────────────────────────────┘    ║
║                              │                                                   ║
║                              ▼                                                   ║
║  ┌─────────────────────────────────────────────────────────────────────────┐    ║
║  │                   KUBERNETES CONTROL PLANE                             │    ║
║  │                   IP: 10.0.0.10                                        │    ║
║  │                                                                         │    ║
║  │  ┌──────────────────────────────────────────────────────────────────┐  │    ║
║  │  │                  kube-apiserver :6443                            │  │    ║
║  │  │                                                                  │  │    ║
║  │  │  1. TLS Termination (verify client cert against CA)             │  │    ║
║  │  │  2. Authentication → username: "devops-user", groups: [dev]     │  │    ║
║  │  │  3. RBAC Authorization → allowed? yes/no                        │  │    ║
║  │  │  4. Admission Control → valid? mutate?                          │  │    ║
║  │  │  5. Audit Logging → who did what when                           │  │    ║
║  │  └───────────────────────────────┬──────────────────────────────────┘  │    ║
║  │                                  │                                      │    ║
║  │                    ┌─────────────▼────────────┐                         │    ║
║  │                    │        etcd :2379         │                         │    ║
║  │                    │  (Only API server talks   │                         │    ║
║  │                    │   directly to etcd)       │                         │    ║
║  │                    │  Stores: RBAC, SAs,       │                         │    ║
║  │                    │         all resources     │                         │    ║
║  │                    └──────────────────────────┘                         │    ║
║  │                                                                         │    ║
║  │  ┌─────────────────────────────────────────────────────────────────┐   │    ║
║  │  │  kube-controller-manager :10257  │  kube-scheduler :10259       │   │    ║
║  │  │  (Leader elected)                │  (Leader elected)             │   │    ║
║  │  │  - ReplicaSet controller         │  - Binds Pods to Nodes       │   │    ║
║  │  │  - Deployment controller         │  - Uses Node labels/taints   │   │    ║
║  │  │  - ServiceAccount controller     │                               │   │    ║
║  │  │  - Node controller               │                               │   │    ║
║  │  │  - All talk to API server only   │                               │   │    ║
║  │  └─────────────────────────────────────────────────────────────────┘   │    ║
║  └─────────────────────────────────────────────────────────────────────────┘    ║
║                              │                                                   ║
║              ┌───────────────┴───────────────┐                                  ║
║              │                               │                                   ║
║  ┌───────────▼───────────┐     ┌─────────────▼──────────┐                       ║
║  │  WORKER NODE 1        │     │  WORKER NODE 2         │                       ║
║  │  IP: 10.0.0.20        │     │  IP: 10.0.0.21         │                       ║
║  │                       │     │                        │                       ║
║  │  kubelet :10250       │     │  kubelet :10250        │                       ║
║  │  kube-proxy           │     │  kube-proxy            │                       ║
║  │  containerd (CRI)     │     │  containerd (CRI)      │                       ║
║  │  CNI plugin           │     │  CNI plugin            │                       ║
║  │                       │     │                        │                       ║
║  │  ┌─────────────────┐  │     │  ┌─────────────────┐   │                       ║
║  │  │  App Pods       │  │     │  │  App Pods       │   │                       ║
║  │  │  ┌───┐ ┌───┐    │  │     │  │  ┌───┐ ┌───┐   │   │                       ║
║  │  │  │P1 │ │P2 │    │  │     │  │  │P3 │ │P4 │   │   │                       ║
║  │  │  └───┘ └───┘    │  │     │  │  └───┘ └───┘   │   │                       ║
║  │  └─────────────────┘  │     │  └─────────────────┘   │                       ║
║  └───────────────────────┘     └────────────────────────┘                       ║
║                                                                                  ║
║  REMOTE ACCESS PATH:                                                             ║
║  kubectl → VPN/Bastion → Firewall → LB → API server → etcd/controllers/nodes    ║
╚══════════════════════════════════════════════════════════════════════════════════╝
```

---

## 25. Real-World Production Use Cases

### 25.1 Multi-Region Enterprise Platform

**Scenario:** A global enterprise with Kubernetes clusters in US-East, EU-West, and AP-South. Platform team of 15 engineers, 200 developers across 20 teams.

```bash
# kubeconfig structure for multi-region access
~/.kube/clusters/us-east-production.yaml   # US production
~/.kube/clusters/eu-west-production.yaml   # EU production
~/.kube/clusters/ap-south-production.yaml  # AP production
~/.kube/clusters/us-east-staging.yaml      # Staging

# Merged into single kubeconfig with clear context names
kubectx us-east-prod    # Switch to US production
kubectx eu-west-prod    # Switch to EU production

# RBAC model:
# - Developers: edit in their team namespace only
# - Platform engineers: admin across all clusters
# - Security team: view with no exec access
# - CI/CD (per team): deploy only in team's namespace
```

### 25.2 Financial Services with Strict Audit Requirements

```bash
# All remote access via Teleport (zero-trust access proxy)
# No kubeconfig files on laptops → no credential theft risk
# All commands recorded in centralized audit database

tsh login --proxy=teleport.bank.com --auth=ActiveDirectory
tsh kube login production-cluster

# All kubectl commands are:
# 1. Verified via AD credentials
# 2. Subject to Teleport RBAC policies
# 3. Recorded in Teleport audit log (searchable)
# 4. Forwarded to Kubernetes audit log
# 5. Available for compliance review in SIEM

kubectl get pods -n banking-app  # Fully audited
kubectl exec -it pod/banking-processor -- bash  # Session recorded
```

### 25.3 GitOps-First Organization

```bash
# Developers have READ-ONLY remote access via kubectl
# All changes go through Git → ArgoCD → cluster (not direct kubectl apply)

# Developer kubeconfig: view ClusterRole (read-only)
kubectl get pods -n my-app      # ✅ Works
kubectl apply -f change.yaml     # ❌ Forbidden (RBAC denies write)

# To deploy: submit PR to Git
# ArgoCD detects change → applies to cluster
# ArgoCD ServiceAccount: limited to specific namespaces

# Humans can still exec and port-forward for debugging:
kubectl exec -it pod/my-app-xxx -- bash  # ✅ Debug access allowed
kubectl port-forward svc/my-db 5432:5432  # ✅ Read data for debugging
```

---

## 26. Best Practices for Production Remote Access

### 26.1 Credential Management

- **Never embed credentials in scripts or Docker images** — use environment variables or mounted secrets
- **Rotate kubeconfig credentials regularly** — certificate-based: annually; token-based: monthly
- **Use separate kubeconfigs per environment** — never have production and staging in the same context file on shared machines
- **Use exec credential plugins for cloud clusters** — AWS, GCP, Azure handle token refresh automatically
- **Implement kubeconfig version control** — track changes to cluster configurations (not credentials!)

### 26.2 Network Security

- **Keep API server on private network** — never expose :6443 to the public internet without additional controls
- **Require VPN for production cluster access** — before any network path exists to the API server
- **Use allowlists for API server access** — if cloud-hosted, restrict Security Group/Firewall to known IPs
- **Audit access from external IPs** — monitor API server audit logs for unexpected source IPs

### 26.3 RBAC Design

- **Namespace-scope permissions** when possible — `Role` + `RoleBinding` over `ClusterRole` + `ClusterRoleBinding`
- **No wildcards in production** — always specify exact resources and verbs
- **Break-glass procedure for cluster-admin** — temporary binding with TTL, requires approval
- **Quarterly RBAC audits** — review and clean up stale bindings
- **Document every RBAC decision** — use annotations on RoleBindings to explain purpose and approver

### 26.4 Operational Hygiene

```bash
# Always verify your context before running dangerous commands
kubectl config current-context
kubectl config view --minify | grep namespace

# Use --dry-run for validation before applying
kubectl apply -f deployment.yaml --dry-run=server
kubectl delete deployment my-app -n production --dry-run=client

# Use diff to preview changes
kubectl diff -f deployment.yaml

# For critical operations: get second pair of eyes
# Use asciinema to record sessions for review
asciinema rec /tmp/kubectl-session-$(date +%Y%m%d-%H%M%S).cast
kubectl apply -f important-change.yaml
# Press Ctrl+D to stop recording
```

---

## 27. Common Mistakes and Pitfalls

### 27.1 Using localhost in Server URL

**Mistake:** Copying admin.conf as-is when it contains `server: https://127.0.0.1:6443`.  
**Impact:** Works on the control plane node but fails from remote machine.  
**Fix:** `sed -i 's|127.0.0.1:6443|10.0.0.10:6443|g' kubeconfig.yaml`

---

### 27.2 insecure-skip-tls-verify in kubeconfig

**Mistake:**
```yaml
cluster:
  insecure-skip-tls-verify: true  # NEVER IN PRODUCTION!
```
**Impact:** Allows man-in-the-middle attacks; anyone between you and the API server can intercept/modify traffic.  
**Fix:** Properly configure `certificate-authority-data` with the actual cluster CA cert.

---

### 27.3 Sharing kubeconfig with cluster-admin

**Mistake:** Admin shares admin.conf with all team members.  
**Impact:** If any credential is leaked, full cluster is compromised. No per-user audit trail.  
**Fix:** Create individual credentials per user with appropriate RBAC scoping.

---

### 27.4 kubectl context mistake (running production commands in wrong context)

**Mistake:** Engineer thinks they're in staging but accidentally runs `kubectl delete deployment` in production.  
**Impact:** Production outage.  
**Fix:** Use `kubie ctx production` for isolated shells, add context indicator to shell prompt, use `kubectl confirm` tools.

---

### 27.5 Storing kubeconfig in Git

**Mistake:** Accidentally committing `~/.kube/config` to a repository.  
**Impact:** Credentials are now in git history, accessible to anyone with repo access.  
**Fix:** Immediately rotate all credentials. Add `.kube/` to `.gitignore`. Use `git filter-branch` to remove from history.

---

### 27.6 Not Handling kubeconfig for CI/CD Securely

**Mistake:** Storing entire kubeconfig in CI/CD environment variable as plain text.  
**Impact:** Visible in CI/CD logs; accessible to all engineers with CI/CD access.  
**Fix:** Use proper secret management (HashiCorp Vault, AWS Secrets Manager). Scope SA tokens to minimal permissions. Use OIDC federation where available.

---

### 27.7 Certificate Expiry Not Monitored

**Mistake:** No alert for expiring client certificates.  
**Impact:** Sudden access failure when cert expires, usually at the worst time.  
**Fix:** Add Prometheus alert for `kubernetes_client_certificate_expiration_seconds`. Set calendar reminders 30 days before expiry.

---

## 28. Hands-On Labs — Guided Exercises

### Lab 1: Complete Remote Access Setup (Full Lab)

```bash
# GOAL: Set up remote kubectl access to a kubeadm cluster
# TIME: ~30 minutes

# ── PREREQUISITES ──────────────────────────────────────────────────────
# 1. A running kubeadm cluster on 10.0.0.10
# 2. SSH access to the control plane
# 3. A remote machine (your laptop)

# ── STEP 1: Install kubectl ─────────────────────────────────────────────
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
kubectl version --client

# ── STEP 2: Get kubeconfig and fix server address ───────────────────────
scp ubuntu@10.0.0.10:/etc/kubernetes/admin.conf ~/remote-lab-kubeconfig.yaml
sed -i 's|https://127.0.0.1:6443|https://10.0.0.10:6443|g' ~/remote-lab-kubeconfig.yaml
chmod 600 ~/remote-lab-kubeconfig.yaml

# ── STEP 3: Test initial connection ─────────────────────────────────────
kubectl --kubeconfig=~/remote-lab-kubeconfig.yaml cluster-info
kubectl --kubeconfig=~/remote-lab-kubeconfig.yaml get nodes

# ── STEP 4: Set as default kubeconfig ───────────────────────────────────
mkdir -p ~/.kube
cp ~/remote-lab-kubeconfig.yaml ~/.kube/config
chmod 600 ~/.kube/config

# ── STEP 5: Create a dedicated user (instead of admin) ──────────────────
# Generate user credentials
openssl genrsa -out lab-user.key 2048
openssl req -new -key lab-user.key -out lab-user.csr \
  -subj "/CN=lab-user/O=lab-developers"

# Submit CSR to cluster
cat << EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: lab-user-csr
spec:
  request: $(cat lab-user.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
    - client auth
EOF

kubectl certificate approve lab-user-csr
kubectl get csr lab-user-csr -o jsonpath='{.status.certificate}' | \
  base64 -d > lab-user.crt

# ── STEP 6: Create RBAC for lab-user ────────────────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: lab-user-role
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "namespaces", "nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: lab-user-binding
subjects:
  - kind: User
    name: lab-user
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: lab-user-role
  apiGroup: rbac.authorization.k8s.io
EOF

# ── STEP 7: Build kubeconfig for lab-user ───────────────────────────────
kubectl config set-credentials lab-user \
  --client-certificate=lab-user.crt \
  --client-key=lab-user.key \
  --embed-certs=true

kubectl config set-context lab-user-context \
  --cluster=$(kubectl config view -o jsonpath='{.clusters[0].name}') \
  --user=lab-user \
  --namespace=default

# ── STEP 8: Switch to lab-user context and test ─────────────────────────
kubectl config use-context lab-user-context

kubectl get nodes       # Should work (ClusterRole grants this)
kubectl get pods -A     # Should work
kubectl create deployment test --image=nginx  # Should FAIL (no create permission)
# Error from server (Forbidden): deployments.apps is forbidden

# ── CLEANUP ─────────────────────────────────────────────────────────────
kubectl config use-context $(kubectl config get-contexts | grep admin | awk '{print $2}')
kubectl delete csr lab-user-csr
kubectl delete clusterrolebinding lab-user-binding
kubectl delete clusterrole lab-user-role
kubectl config delete-context lab-user-context
kubectl config delete-user lab-user
rm lab-user.key lab-user.crt lab-user.csr
```

### Lab 2: kubectl Port-Forward and Proxy

```bash
# GOAL: Access cluster services from remote machine without Ingress/NodePort

# Step 1: Create a test application
kubectl create deployment web --image=nginx --replicas=2
kubectl expose deployment web --port=80 --target-port=80

# Step 2: Port-forward to the service
kubectl port-forward svc/web 8080:80 &
PF_PID=$!

# Step 3: Access via localhost
curl http://localhost:8080
# Should return nginx welcome page

# Step 4: Port-forward to specific pod
POD=$(kubectl get pods -l app=web -o name | head -1)
kubectl port-forward $POD 8081:80 &

curl http://localhost:8081
# Same result but targeting specific pod

# Step 5: Use kubectl proxy to explore the API
kubectl proxy --port=8001 &
curl http://localhost:8001/api/v1/namespaces/default/pods | jq '.items[].metadata.name'
curl http://localhost:8001/healthz

# Cleanup
kill $PF_PID
jobs -l | grep port-forward | awk '{print $2}' | xargs kill 2>/dev/null
kubectl delete deployment web
kubectl delete service web
```

### Lab 3: Multi-Context Management

```bash
# GOAL: Manage multiple cluster contexts safely

# Step 1: Create two simulated contexts (using same cluster with different namespaces)
kubectl create namespace staging-sim
kubectl create namespace production-sim

kubectl config set-context staging \
  --cluster=$(kubectl config view -o jsonpath='{.clusters[0].name}') \
  --user=$(kubectl config view -o jsonpath='{.users[0].name}') \
  --namespace=staging-sim

kubectl config set-context production \
  --cluster=$(kubectl config view -o jsonpath='{.clusters[0].name}') \
  --user=$(kubectl config view -o jsonpath='{.users[0].name}') \
  --namespace=production-sim

# Step 2: Switch between contexts
kubectl config use-context staging
kubectl config current-context     # staging
kubectl get pods                   # Shows pods in staging-sim

kubectl config use-context production
kubectl config current-context     # production
kubectl get pods                   # Shows pods in production-sim

# Step 3: Demonstrate danger of wrong context
kubectl config use-context staging
kubectl create deployment staging-app --image=nginx --replicas=2

kubectl config use-context production
# DANGER: Without checking context, you might delete staging's deployment
# kubectl delete deployment staging-app  # DON'T RUN!

# Step 4: Always check before running
kubectl config current-context && \
  kubectl delete deployment staging-app --dry-run=client

# Cleanup
kubectl config delete-context staging
kubectl config delete-context production
kubectl delete namespace staging-sim
kubectl delete namespace production-sim
```

---

## 29. Interview Questions — Beginner to Advanced

### Beginner Level

**Q1: A colleague says "just copy the admin.conf file to your laptop." What's wrong with this approach?**

**A:** Several problems: First, admin.conf usually contains `server: https://127.0.0.1:6443` which only works from the control plane node, not remotely — you must update it with the actual IP/hostname. Second and more critically, admin.conf grants **full cluster-admin access** — if your laptop is compromised or the file is accidentally shared, the attacker has complete control over the cluster. 

The correct approach for most users is to create individual credentials (X.509 cert or token) with appropriate RBAC-scoped permissions. Only emergency/platform-admin access should use cluster-admin credentials, and even then it should be protected by MFA, VPN, and audited.

---

**Q2: You run `kubectl get pods` from your laptop and get "connection refused". What are the first three things you check?**

**A:** In order:
1. **Network path**: `nc -zv <api-server-ip> 6443` — tests if the API server port is reachable at all. If this fails, it's a network issue (VPN not connected, firewall blocking, wrong IP).
2. **Server URL in kubeconfig**: `kubectl config view | grep server` — verify the URL points to the correct IP/hostname and not localhost or an old IP.
3. **API server is running**: From the control plane node, `kubectl get pods -n kube-system | grep apiserver` — the API server pod might be down.

---

**Q3: What is a kubeconfig context and why are they important?**

**A:** A context is a named combination of three elements in kubeconfig: a **cluster** (which API server to connect to), a **user** (which credentials to use), and an optional **namespace** (default namespace for commands). 

Contexts are important because they allow a single `~/.kube/config` file to store access configuration for multiple clusters simultaneously. Instead of maintaining separate files and remembering to switch between them, you simply run `kubectl config use-context production` or `kubectx staging`. 

For production safety, separate contexts mean you can clearly see which cluster you're operating on — reducing the risk of accidentally running a production command in staging or vice versa.

---

### Intermediate Level

**Q4: Explain the complete flow of a `kubectl get pods` command from your laptop to the API server response.**

**A:** Full flow:
1. `kubectl` reads `~/.kube/config` — extracts API server URL, CA cert, and client credentials
2. kubectl builds HTTP GET request: `https://<api-server>:6443/api/v1/namespaces/<ns>/pods`
3. **TLS handshake**: kubectl connects; API server presents its TLS cert; kubectl verifies it against the CA cert in kubeconfig; if client cert auth, kubectl presents its cert and the API server verifies it
4. **Authentication**: API server extracts identity from the client certificate's CN field (username) and O field (groups). If token auth, it validates the token via TokenReview
5. **RBAC Authorization**: API server checks: does this username/group have `GET` permission on `pods` in this namespace? If no matching RoleBinding → 403 Forbidden
6. **Read from watch cache**: API server serves the response from its in-memory watch cache (backed by etcd watches) — not a direct etcd read for most GET requests
7. Response JSON is serialized and returned to kubectl
8. kubectl formats the response as a table and displays it

---

**Q5: How would you set up remote access for a GitHub Actions CI/CD pipeline to deploy to Kubernetes?**

**A:** Best practice for GitHub Actions:

**Option A (OIDC Federation — Recommended):** Configure GitHub Actions OIDC with the cluster's OIDC provider. GitHub provides an ID token for each workflow run. The Kubernetes API server validates it without any static secrets. This is the most secure approach.

**Option B (ServiceAccount + Token):** Create a dedicated ServiceAccount with minimal RBAC (only update deployments in specific namespaces). Generate a token: `kubectl create token github-actions-sa -n production --duration=8760h`. Store this token in GitHub Secrets as `KUBECONFIG_PRODUCTION`. In the workflow, write it to a temp file and use `kubectl --kubeconfig=/tmp/kube.yaml` for the deployment commands. Delete the temp file after use.

In either case: never give CI/CD cluster-admin permissions; scope to specific namespaces and resource types; rotate tokens on a schedule; monitor for unusual access patterns in audit logs.

---

**Q6: What is the difference between running `kubectl` from inside a Pod vs from a remote machine?**

**A:** Key differences:

**Remote machine:** Uses kubeconfig file from `~/.kube/config` or `KUBECONFIG` env var. Credentials are explicitly configured (cert/key/token in the file). The API server is at an external IP/hostname. TLS is full mutual authentication.

**Inside a Pod:** Uses the ServiceAccount token automatically mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`. The CA cert is at `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`. The API server is at `https://kubernetes.default.svc` (resolved via CoreDNS). The token is a **bound token** — scoped to the specific Pod and ServiceAccount, with configurable expiry. RBAC permissions are those of the ServiceAccount, not a human user.

In code: `config.InClusterConfig()` vs `config.BuildConfigFromFlags("", kubeconfig)` in Go; `kubernetes.config.load_incluster_config()` vs `load_kube_config()` in Python.

---

### Advanced Level

**Q7: How would you design remote access for a 1000-engineer organization with 50 Kubernetes clusters across 3 cloud providers?**

**A:** At this scale, individual kubeconfig files are unmanageable. The solution uses layered tooling:

**Authentication layer:** Centralized SSO (Okta/Azure AD) integrated with Kubernetes via OIDC. All clusters configured with the same OIDC provider. Users log in once (to the IdP) and get short-lived JWTs. No individual certificates to manage.

**Access management layer:** Use a tool like **Teleport** (zero-trust access proxy) or **Rancher** (multi-cluster management) as a centralized broker. Engineers never have direct access to cluster API servers. All access is proxied through the broker, which enforces company policies and provides centralized audit logs.

**RBAC:** Defined as code in Git (GitOps). Common RBAC templates per persona (developer, SRE, security-auditor) applied across all clusters. Team-specific RBAC in team namespaces. Automated quarterly audits comparing actual RBAC to approved state.

**Monitoring:** Centralized audit log aggregation (Elasticsearch/Splunk). Alerts for: new cluster-admin bindings, cert expiry < 30 days, auth failure spikes, access from unexpected geolocations.

**Disaster recovery:** Every cluster's RBAC configuration is stored in Git. If a cluster is rebuilt, `kubectl apply` of the Git repo restores all access configuration.

---

**Q8: You notice that `kubectl get pods` works but `kubectl logs <pod-name>` fails with "connection refused". How do you diagnose this?**

**A:** This is a **kubelet connectivity issue**, not an API server issue. The kubectl logs command works by: kubectl → API server → kubelet on the node running the pod → container runtime → log stream.

When `get pods` works but `logs` fails:
1. API server is reachable ✅ (get pods works)
2. API server can't reach the kubelet on port 10250 ❌

Diagnosis steps:
```bash
# Find which node the pod is on
kubectl get pod <pod-name> -o wide
# NODE: k8s-worker-1

# Check if kubelet port is accessible from control plane
kubectl exec -n kube-system kube-apiserver-k8s-cp-1 -- \
  nc -zv 10.0.0.20 10250

# Check firewall rules (from remote machine to nodes)
nc -zv 10.0.0.20 10250

# Check kubelet status on the worker node
ssh ubuntu@10.0.0.20 "systemctl status kubelet"

# Check if kubelet is serving on correct port
ssh ubuntu@10.0.0.20 "ss -tlnp | grep 10250"
```

Common causes: Firewall rule blocking 10250; kubelet is down or crashed; kubelet serving certificate expired (`journalctl -u kubelet | grep "cert"`). Fix depends on the specific cause.

---

## 30. Cheat Sheet — Commands, Flags & One-Liners

### 30.1 kubectl Installation

```bash
# Install specific version
curl -LO "https://dl.k8s.io/release/v1.29.0/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Check version
kubectl version --client --short

# Enable shell completion
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
source ~/.bashrc
```

### 30.2 kubeconfig Management

```bash
kubectl config current-context           # Show active context
kubectl config get-contexts              # List all contexts
kubectl config use-context <name>        # Switch context
kubectl config set-context --current \
  --namespace=<ns>                       # Set default namespace

kubectl config view                      # View kubeconfig
kubectl config view --raw                # With credentials
kubectl config view --minify             # Current context only

# Fix localhost server URL
sed -i 's|127.0.0.1:6443|10.0.0.10:6443|g' kubeconfig.yaml
chmod 600 kubeconfig.yaml

# Merge kubeconfigs
KUBECONFIG=~/.kube/config:~/new-cluster.yaml \
  kubectl config view --flatten > ~/.kube/config.new
mv ~/.kube/config.new ~/.kube/config
chmod 600 ~/.kube/config

# Use custom kubeconfig
kubectl --kubeconfig=/path/to/config get pods

# Use via environment variable
export KUBECONFIG=/path/to/kubeconfig.yaml
```

### 30.3 Creating User Credentials

```bash
# Generate user key and CSR
openssl genrsa -out user.key 2048
openssl req -new -key user.key -out user.csr \
  -subj "/CN=username/O=group1"

# Submit and approve CSR
kubectl apply -f csr-manifest.yaml
kubectl certificate approve user-csr
kubectl get csr user-csr \
  -o jsonpath='{.status.certificate}' | base64 -d > user.crt

# Build kubeconfig entry
kubectl config set-credentials username \
  --client-certificate=user.crt \
  --client-key=user.key \
  --embed-certs=true
kubectl config set-context username-ctx \
  --cluster=my-cluster \
  --user=username \
  --namespace=default
```

### 30.4 RBAC Commands

```bash
kubectl auth can-i <verb> <resource> -n <ns>  # Test permission
kubectl auth can-i --list -n <ns>             # All permissions
kubectl auth can-i get pods --as=<user>       # Test as user
kubectl auth can-i --list --as=system:serviceaccount:<ns>:<sa>

kubectl get clusterroles,clusterrolebindings
kubectl get roles,rolebindings -n <ns>
kubectl describe clusterrole <name>
kubectl describe rolebinding <name> -n <ns>

kubectl create role <n> --verb=get,list --resource=pods -n <ns>
kubectl create rolebinding <n> --role=<n> --user=<user> -n <ns>
kubectl create clusterrolebinding <n> --clusterrole=<n> --user=<user>
```

### 30.5 Network Testing

```bash
# Test API server connectivity
nc -zv <api-ip> 6443
curl -k https://<api-ip>:6443/version
curl -k https://<api-ip>:6443/healthz

# Test with CA cert
curl --cacert cluster-ca.crt https://<api-ip>:6443/version

# Check TLS certificate
openssl s_client -connect <api-ip>:6443 </dev/null 2>/dev/null | \
  openssl x509 -noout -text | grep -A 5 "Subject Alternative Name"

# SSH tunnel
ssh -L 6443:<api-ip>:6443 user@bastion -N &
```

### 30.6 Port-Forward and Proxy

```bash
kubectl port-forward svc/<name> 8080:80 -n <ns> &
kubectl port-forward pod/<name> 8080:80 -n <ns> &
kubectl port-forward --address 0.0.0.0 svc/<name> 8080:80 -n <ns>

kubectl proxy --port=8001 &
curl http://localhost:8001/api/v1/namespaces/<ns>/pods

# Kill port-forward
kill $(lsof -t -i:8080)
```

### 30.7 Troubleshooting

```bash
# Debug connection
kubectl get pods -v=6          # Show HTTP requests
kubectl get pods -v=9          # Full HTTP request/response

# Check cluster health
kubectl cluster-info
kubectl get nodes
kubectl get pods --all-namespaces | grep -v "Running\|Completed"
kubectl get events --all-namespaces --sort-by='.lastTimestamp' | tail -20

# Check API server
kubectl get --raw /healthz
kubectl get --raw /readyz
kubectl get --raw /version

# Certificate check
kubeadm certs check-expiration  # From control plane
openssl x509 -in cert.crt -noout -dates

# Forced context check before dangerous commands
kubectl config current-context && kubectl <dangerous-command> --dry-run=client
```

### 30.8 Important API Server Flags

| Flag | Default | Purpose |
|---|---|---|
| `--bind-address` | `0.0.0.0` | Address to listen on |
| `--secure-port` | `6443` | HTTPS port |
| `--anonymous-auth` | `true` | Disable for security: `false` |
| `--authorization-mode` | `Node,RBAC` | Always include RBAC |
| `--tls-min-version` | `VersionTLS12` | Set to TLS13 for new clusters |
| `--audit-log-path` | none | Set for compliance |
| `--max-requests-inflight` | `400` | Increase for large clusters |

---

## 31. Key Takeaways & Summary

### The Remote Access Setup in 7 Steps

```
1. Install kubectl      → The tool
2. Get kubeconfig       → The credentials + endpoint
3. Fix server URL       → Point to real IP, not localhost
4. Verify TLS           → Certificate trust chain
5. Create user creds    → Individual X.509 certs or tokens
6. Configure RBAC       → Least-privilege permissions
7. Manage contexts      → Safe multi-cluster access
```

### The 10 Non-Negotiable Rules for Production Remote Access

1. **Fix the server URL.** `127.0.0.1:6443` only works from the control plane. Remote access requires the actual IP or hostname.

2. **Never disable TLS verification.** `insecure-skip-tls-verify: true` is the Kubernetes equivalent of "disable all security." Fix the CA cert, not the verification.

3. **Controllers never touch etcd directly.** All remote access goes through the API server, which enforces all security controls. This is the foundation of Kubernetes security.

4. **Create individual credentials.** Never share admin.conf with team members. Create scoped, individual certs or tokens.

5. **Apply RBAC before production access.** Least privilege isn't optional. Namespace-scope permissions whenever possible.

6. **Protect kubeconfig like passwords.** `chmod 600`, not in Git, not in CI logs, not in chat. It IS a password.

7. **Check your context.** Before any mutating command in production, verify `kubectl config current-context`. One wrong context, one wrong command, one major outage.

8. **Audit everything.** Enable API server audit logging. Every remote action is recorded: user, verb, resource, timestamp. This is your security camera.

9. **Monitor certificate expiry.** Certificates expire silently and suddenly. Monitor expiry 30+ days in advance. Automate renewal where possible.

10. **Design for recovery.** Store all RBAC configuration in Git. If the cluster is lost, you should be able to restore access from version control, not memory.

---

> **This guide covers Kubernetes v1.29+ remote access patterns. Always consult the official documentation at https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/ for the most current information.**

---

*End of: Lab: Accessing a Kubernetes Cluster From a Remote Machine — Complete Production Guide*
