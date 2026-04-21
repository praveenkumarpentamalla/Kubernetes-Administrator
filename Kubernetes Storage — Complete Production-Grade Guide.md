## Table of Contents

1. [Introduction — Why Persistent Storage Is Critical in Kubernetes](#1-introduction)
2. [Core Identity Table — Storage Components](#2-core-identity-table)
3. [Understanding the Need for Persistent Storage](#3-need-for-persistent-storage)
4. [The Controller Pattern — Watch → Compare → Act → Loop](#4-controller-pattern)
5. [Understanding Storage Classes](#5-storage-classes)
6. [Lab: Adding NFS as a Storage Provider](#6-lab-nfs-storage-provider)
7. [Understanding Persistent Volumes](#7-persistent-volumes)
8. [Understanding Persistent Volume Claims](#8-persistent-volume-claims)
9. [Lab: Setting Up Persistent Volumes](#9-lab-persistent-volumes)
10. [Lab: Setting Up Persistent Volume Claims](#10-lab-persistent-volume-claims)
11. [Lab: Creating Pods with Volumes Attached](#11-lab-pods-with-volumes)
12. [Built-in Controllers Deep Dive](#12-built-in-controllers)
13. [Internal Working Concepts — Informers, Work Queues & Reconciliation](#13-internal-concepts)
14. [API Server and etcd Interaction](#14-api-server-etcd)
15. [Leader Election — HA for Storage Controllers](#15-leader-election)
16. [Volume Topology and Scheduling Integration](#16-volume-topology)
17. [Performance Tuning for Storage Operations](#17-performance-tuning)
18. [Security Hardening Practices](#18-security-hardening)
19. [Monitoring and Observability for Storage](#19-monitoring)
20. [Troubleshooting Storage Issues](#20-troubleshooting)
21. [Disaster Recovery — Storage and etcd Backups](#21-disaster-recovery)
22. [Comparison — kube-apiserver vs kube-scheduler vs kube-controller-manager](#22-comparison)
23. [ASCII Architecture Diagram](#23-ascii-diagram)
24. [Real-World Production Use Cases](#24-production-use-cases)
25. [Best Practices for Production Storage](#25-best-practices)
26. [Common Mistakes and Pitfalls](#26-mistakes-pitfalls)
27. [Interview Questions — Beginner to Advanced](#27-interview-questions)
28. [Cheat Sheet — Commands, Flags & Manifests](#28-cheat-sheet)
29. [Key Takeaways & Summary](#29-key-takeaways)

---

## 1. Introduction — Why Persistent Storage Is Critical in Kubernetes {#1-introduction}

Kubernetes was designed for stateless applications first. Containers are ephemeral by nature — when a container crashes, restarts, or is rescheduled to a different node, everything written to the container's local filesystem is lost. This is by design for stateless services (web servers, API proxies, microservices), but completely unacceptable for stateful workloads:

- A PostgreSQL database that loses its data on every pod restart
- A Kafka broker that forgets all messages after a rescheduling
- An Elasticsearch index that starts empty after a node failure
- A file-upload service that can't persist uploaded files

**Kubernetes Persistent Storage** solves this with a layered abstraction model:

```
Application Layer:  Pod → mounts a volume at /data
Claim Layer:        PersistentVolumeClaim → requests storage (size, access mode, class)
Provision Layer:    PersistentVolume → actual storage resource (NFS, EBS, NVMe, etc.)
Policy Layer:       StorageClass → defines HOW storage is provisioned
Infrastructure:     NFS server, AWS EBS, GCP PD, Azure Disk, Ceph, Longhorn, etc.
```

### Why This Abstraction Matters

| Without Storage Abstraction | With Kubernetes Persistent Storage |
|---|---|
| App code knows about specific storage backend (AWS vs on-prem) | App declares "I need 10Gi" — backend irrelevant |
| Pod tied to specific node (local disk) | Pod rescheduled anywhere; storage follows |
| Manual disk provisioning and attachment | Dynamic provisioning via StorageClass |
| Data lost on pod restart | Data persists across pod crashes, restarts, rescheduling |
| No access control on storage | RBAC controls who can create PVCs |
| Manual cleanup of unused disks | ReclaimPolicy automates volume lifecycle |

### The Production Reality

In modern cloud-native architectures, even "stateless" services often need some persistent storage — configuration caches, temporary file processing, shared asset storage. Understanding Kubernetes storage is not optional for production Kubernetes engineers.

---

## 2. Core Identity Table — Storage Components {#2-core-identity-table}

| Component | Kind / Binary | API Group | Namespace Scoped | Role |
|---|---|---|---|---|
| **PersistentVolume** | `persistentvolume` (PV) | `core/v1` | **No** (cluster-scoped) | Actual storage resource; provisioned by admin or dynamically |
| **PersistentVolumeClaim** | `persistentvolumeclaim` (PVC) | `core/v1` | **Yes** | Storage request from user; bound to a PV |
| **StorageClass** | `storageclass` | `storage.k8s.io/v1` | **No** | Defines provisioner, parameters, reclaim policy |
| **VolumeAttachment** | `volumeattachment` | `storage.k8s.io/v1` | No | Records attachment of PV to Node |
| **CSIDriver** | `csidriver` | `storage.k8s.io/v1` | No | Registers CSI driver capabilities |
| **CSINode** | `csinode` | `storage.k8s.io/v1` | No | Node-level CSI driver topology info |
| **kube-controller-manager** | Binary | N/A | N/A | PV controller (binding), AttachDetach controller |
| **kube-apiserver** | Binary | 6443 | N/A | Stores PV/PVC/StorageClass; validates requests |
| **kube-scheduler** | Binary | 10259 | N/A | VolumeBinding scheduler plugin (respects volume topology) |
| **external-provisioner** | Sidecar container | N/A | N/A | CSI: watches PVCs and calls CSI CreateVolume |
| **external-attacher** | Sidecar container | N/A | N/A | CSI: watches VolumeAttachments and calls CSI ControllerPublishVolume |
| **kubelet** | Binary | 10250 | N/A | Mounts volumes into pods; calls NodeStageVolume, NodePublishVolume |
| **CSI Driver Plugin** | DaemonSet | N/A | N/A | Node-level storage operations (mount, format, unmount) |
| **NFS Subdir Provisioner** | Deployment | N/A | N/A | Dynamic NFS provisioner using subdirectories |
| **etcd** | Binary | 2379 | N/A | Stores all PV/PVC/StorageClass objects; exclusively accessed by API server |

---

## 3. Understanding the Need for Persistent Storage {#3-need-for-persistent-storage}

### 3.1 The Ephemeral Container Problem

```
Container Filesystem Layers:
─────────────────────────────────────────────────────────────────────
┌─────────────────────────────────────────────────┐
│  Read-Write Layer (container layer)             │
│  ← All runtime writes go here                  │
│  ← DESTROYED when container is removed         │
├─────────────────────────────────────────────────┤
│  Image Layer N (read-only)                      │
├─────────────────────────────────────────────────┤
│  Image Layer N-1 (read-only)                    │
├─────────────────────────────────────────────────┤
│  Base Image Layer (read-only)                   │
└─────────────────────────────────────────────────┘

When container restarts:
  Read-Write Layer is recreated EMPTY
  All data written during runtime is LOST
```

### 3.2 Volume Types in Kubernetes

Kubernetes supports multiple volume types, each with different persistence characteristics:

| Volume Type | Persistent Across Pod Restart? | Persistent Across Node Failure? | Shared Between Pods? | Use Case |
|---|---|---|---|---|
| `emptyDir` | No | No | Same pod only | Temp scratch space, shared between containers in pod |
| `hostPath` | Yes (same node) | No | Same node only | Dev only; breaks portability |
| `configMap` / `secret` | Yes | Yes | Yes | Config injection |
| `nfs` | Yes | Yes | Yes (RWX) | Shared file storage |
| `persistentVolumeClaim` | Yes | Yes (if RWO on same node, or RWX) | Depends on access mode | Production stateful apps |
| `awsElasticBlockStore` | Yes | No (AZ-scoped) | No | AWS single-instance DBs |
| `gcePersistentDisk` | Yes | No (zone-scoped) | No | GCP single-instance DBs |
| `csi` (generic) | Yes | Depends on driver | Depends on driver | Production (preferred) |

### 3.3 The Three Storage Scenarios in Production

**Scenario 1: Single-Instance Database** (most common)
```
PostgreSQL pod → PVC (ReadWriteOnce, 100Gi) → PV → EBS/GCP PD/Ceph RBD
When pod fails: rescheduled to same/different node; PV detached and re-attached
```

**Scenario 2: Shared File Storage** (content management, CI/CD artifacts)
```
Multiple Pods → PVC (ReadWriteMany, 500Gi) → PV → NFS/CephFS/Azure Files
Multiple pods in different namespaces read/write same volume simultaneously
```

**Scenario 3: StatefulSet Cluster** (Kafka, Cassandra, Elasticsearch)
```
3 StatefulSet pods, each with individual PVC:
  kafka-0 → pvc-kafka-0 → PV (SSD, 100Gi) → node-1
  kafka-1 → pvc-kafka-1 → PV (SSD, 100Gi) → node-2
  kafka-2 → pvc-kafka-2 → PV (SSD, 100Gi) → node-3
Each pod keeps its identity and data across restarts
```

---

## 4. The Controller Pattern — Watch → Compare → Act → Loop {#4-controller-pattern}

### 4.1 PersistentVolume Controller Reconciliation

The PV controller (inside `kube-controller-manager`) manages the binding lifecycle between PVCs and PVs.

```
┌──────────────────────────────────────────────────────────────────────┐
│           PV CONTROLLER — RECONCILIATION LOOP                        │
│                                                                      │
│  WATCH:                                                              │
│  Informer watches PV and PVC objects for ADDED/MODIFIED/DELETED      │
│  Events queued: "pv/<n>" or "default/my-pvc"                        │
│                                                                      │
│  COMPARE:                                                            │
│  PVC.status.phase = Pending + PV.status.phase = Available            │
│  Do they match? (access modes, storage size, storageClass, labels)   │
│                                                                      │
│  ACT:                                                                │
│  Binding: PVC.spec.volumeName = PV.name                             │
│           PV.spec.claimRef = PVC reference                          │
│           PVC.status.phase → Bound                                  │
│           PV.status.phase → Bound                                   │
│                                                                      │
│  LOOP:                                                               │
│  Return to WATCH                                                     │
│  Also handles: Released → Reclaim (delete/retain/recycle)           │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.2 The Complete PVC Lifecycle

```
PVC Created (Pending)
       │
       ▼ Static: PV Controller finds matching Available PV
       │ Dynamic: StorageClass provisioner creates PV
       │
PVC Bound ────────────────────── PV Bound
       │
       │ (Pod references PVC)
       ▼
kubelet: VolumeBinding (scheduler ensures pod + volume on compatible node)
       │
       ▼
AttachDetach Controller: Attach volume to node (cloud volumes)
       │
       ▼
kubelet: Mount volume into pod's container filesystem
       │
       ▼
RUNNING (Pod uses volume)
       │
       ▼ (Pod deleted)
kubelet: Unmount volume
       │
       ▼
AttachDetach Controller: Detach volume from node
       │
PVC Released → PV Released
       │
       ▼ (based on ReclaimPolicy)
  Retain: PV stays (manual cleanup needed)
  Delete: PV and underlying storage deleted
  Recycle: Basic scrub (deprecated)
```

---

## 5. Understanding Storage Classes {#5-storage-classes}

### 5.1 What Is a StorageClass?

A StorageClass defines a **"class" of storage** with specific characteristics. It tells Kubernetes:
- **Which provisioner** to use (AWS EBS CSI driver, NFS subdir provisioner, etc.)
- **What parameters** to pass (disk type, encryption, IOPS, etc.)
- **What happens to data** when the PVC is deleted (reclaimPolicy)
- **When volumes should be provisioned** (volumeBindingMode)

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"  # Optional: set as default
provisioner: ebs.csi.aws.com    # Which CSI driver handles this class
parameters:
  type: gp3                      # Driver-specific parameters
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Retain            # What to do with PV when PVC is deleted
allowVolumeExpansion: true       # Allow PVCs to request more storage
volumeBindingMode: WaitForFirstConsumer  # Delay provisioning until pod is scheduled
mountOptions:
  - discard                      # Pass mount options to the filesystem
```

### 5.2 ReclaimPolicy Comparison

| ReclaimPolicy | Behavior When PVC Deleted | Data Preserved? | Use Case |
|---|---|---|---|
| **Delete** | PV and underlying storage deleted automatically | ❌ No | Ephemeral dev data, recreatable data |
| **Retain** | PV stays in "Released" state; admin manually cleans up | ✅ Yes | Production databases, valuable data |
| **Recycle** | Basic `rm -rf` scrub (deprecated) | ❌ No | Deprecated in 1.15; use Delete instead |

### 5.3 volumeBindingMode Options

| Mode | Behavior | Use Case |
|---|---|---|
| `Immediate` | PV provisioned as soon as PVC is created | Simple setups; not topology-aware |
| `WaitForFirstConsumer` | PV provisioned only when pod referencing PVC is scheduled | Multi-zone clusters; ensures PV created in same zone as pod |

### 5.4 Common StorageClass Provisioners

| Provisioner | Storage Backend | Type |
|---|---|---|
| `ebs.csi.aws.com` | AWS Elastic Block Store | Block |
| `pd.csi.storage.gke.io` | Google Persistent Disk | Block |
| `disk.csi.azure.com` | Azure Managed Disks | Block |
| `file.csi.azure.com` | Azure Files | File |
| `rook-ceph.rbd.csi.ceph.com` | Ceph RBD | Block |
| `rook-ceph.cephfs.csi.ceph.com` | CephFS | File |
| `nfs.csi.k8s.io` | NFS | File |
| `rancher.io/local-path` | Local path on node | Block (local) |
| `k8s.io/minikube-hostpath` | minikube local | Block (local) |
| `longhorn.io` | Longhorn distributed storage | Block |

### 5.5 Creating and Managing StorageClasses

```bash
# View existing StorageClasses
kubectl get storageclasses
kubectl get sc   # Short form

# Describe a StorageClass
kubectl describe storageclass standard

# Set a default StorageClass
kubectl patch storageclass standard \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Remove default annotation
kubectl patch storageclass standard \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# Check which StorageClass is default
kubectl get sc | grep "(default)"
```

---

## 6. Lab: Adding NFS as a Storage Provider {#6-lab-nfs-storage-provider}

### 6.1 Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                    NFS STORAGE ARCHITECTURE                          │
│                                                                      │
│  NFS Server (separate VM or pod)                                    │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  /srv/nfs/exports/   (exported directory)                    │  │
│  │  IP: 192.168.1.100   Port: 2049                              │  │
│  └────────────────────────────────┬──────────────────────────────┘  │
│                                   │ NFS protocol                    │
│  Kubernetes Cluster               │                                  │
│  ┌────────────────────────────────▼──────────────────────────────┐  │
│  │  NFS Subdir External Provisioner (Deployment in cluster)      │  │
│  │  Watches PVCs with storageClassName: "nfs-client"             │  │
│  │  Creates subdirectories: /exports/ns-pvc-name/               │  │
│  │  Creates PV objects pointing to those subdirectories          │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  StorageClass: nfs-client → NFS Provisioner                         │
│  PVC: requests 10Gi, class: nfs-client → bound to NFS PV            │
│  Pod: mounts PVC at /data → accesses /srv/nfs/exports/ns-pvc/       │
└──────────────────────────────────────────────────────────────────────┘
```

### 6.2 Step 1: Set Up the NFS Server

```bash
# On the NFS server (Ubuntu/Debian)
# ── INSTALL NFS SERVER ────────────────────────────────────────────────
sudo apt-get update
sudo apt-get install -y nfs-kernel-server

# ── CREATE EXPORT DIRECTORY ───────────────────────────────────────────
sudo mkdir -p /srv/nfs/exports
sudo chown nobody:nogroup /srv/nfs/exports
sudo chmod 777 /srv/nfs/exports

# ── CONFIGURE NFS EXPORTS ─────────────────────────────────────────────
# Edit /etc/exports
sudo cat >> /etc/exports << 'EOF'
/srv/nfs/exports  10.0.0.0/8(rw,sync,no_subtree_check,no_root_squash)
# Allow all pods in 10.x.x.x range (adjust CIDR for your cluster)
# rw: read-write access
# sync: synchronous writes (safer)
# no_subtree_check: prevents issues with renamed files
# no_root_squash: allow root access from clients (required for some workloads)
EOF

# Apply exports
sudo exportfs -arv
# exporting 10.0.0.0/8:/srv/nfs/exports

# ── START NFS SERVER ──────────────────────────────────────────────────
sudo systemctl enable nfs-kernel-server
sudo systemctl start nfs-kernel-server
sudo systemctl status nfs-kernel-server

# Get NFS server IP (use internal IP accessible from cluster nodes)
NFS_SERVER_IP=$(ip route get 8.8.8.8 | awk 'NR==1{print $7}')
echo "NFS Server IP: $NFS_SERVER_IP"

# ── VERIFY FROM KUBERNETES NODES ─────────────────────────────────────
# SSH to a Kubernetes node and test:
# apt-get install -y nfs-common
# showmount -e 192.168.1.100
# Expected: /srv/nfs/exports  10.0.0.0/8
```

### 6.3 Step 2: Install NFS Client on All Kubernetes Nodes

```bash
# Run on EVERY Kubernetes node (worker AND control plane)
# This is required for kubelet to mount NFS volumes

# Ubuntu/Debian
sudo apt-get install -y nfs-common

# CentOS/RHEL
sudo yum install -y nfs-utils
sudo systemctl enable rpcbind nfs-mountd
sudo systemctl start rpcbind nfs-mountd

# Verify NFS mount works manually on a node:
sudo mount -t nfs 192.168.1.100:/srv/nfs/exports /mnt/test
ls /mnt/test
sudo umount /mnt/test
```

### 6.4 Step 3: Install NFS Subdir External Provisioner via Helm

```bash
# Add the NFS subdir provisioner Helm chart repository
helm repo add nfs-subdir-external-provisioner \
  https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

# Install the provisioner
# Replace with YOUR NFS server IP and path
NFS_SERVER="192.168.1.100"
NFS_PATH="/srv/nfs/exports"

helm install nfs-provisioner \
  nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace nfs-provisioner \
  --create-namespace \
  --set nfs.server=$NFS_SERVER \
  --set nfs.path=$NFS_PATH \
  --set storageClass.name=nfs-client \
  --set storageClass.defaultClass=false \
  --set storageClass.reclaimPolicy=Retain \
  --set storageClass.archiveOnDelete=true \
  --set storageClass.accessModes=ReadWriteMany

# Verify installation
kubectl get pods -n nfs-provisioner
# nfs-provisioner-... Running

kubectl get storageclass
# NAME         PROVISIONER                                    RECLAIMPOLICY
# nfs-client   cluster.local/nfs-provisioner-...             Retain
```

### 6.5 Step 4: Manual NFS StorageClass (Without Helm)

```bash
# Create ServiceAccount and RBAC for the provisioner
cat << 'EOF' | kubectl apply -f -
---
apiVersion: v1
kind: Namespace
metadata:
  name: nfs-provisioner
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: nfs-provisioner
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: nfs-provisioner
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
EOF

# Deploy the NFS provisioner
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  namespace: nfs-provisioner
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: registry.k8s.io/sig-storage/nfs-subdir-external-provisioner:v4.0.2
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: "192.168.1.100"    # Your NFS server IP
            - name: NFS_PATH
              value: "/srv/nfs/exports"
      volumes:
        - name: nfs-client-root
          nfs:
            server: "192.168.1.100"
            path: "/srv/nfs/exports"
EOF

# Create the StorageClass
cat << 'EOF' | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
parameters:
  archiveOnDelete: "true"    # Rename instead of delete (data preservation)
  pathPattern: "${.PVC.namespace}-${.PVC.name}"  # Subdirectory naming pattern
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: Immediate
EOF

# Verify
kubectl get storageclass
kubectl describe storageclass nfs-client
```

### 6.6 Test NFS StorageClass with a Test PVC

```bash
# Create a test PVC to verify dynamic provisioning works
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-test-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteMany    # NFS supports ReadWriteMany (multiple pods)
  storageClassName: nfs-client
  resources:
    requests:
      storage: 1Gi
EOF

# Watch PVC binding (should go from Pending to Bound within seconds)
kubectl get pvc nfs-test-pvc -w
# NAME           STATUS   VOLUME                                     CAPACITY
# nfs-test-pvc   Pending                                                       ← Provisioner creating PV
# nfs-test-pvc   Bound    pvc-abc123-def456-...                     1Gi       ← Success!

# Verify PV was created
kubectl get pv
kubectl describe pv $(kubectl get pvc nfs-test-pvc -o jsonpath='{.spec.volumeName}')

# Verify on NFS server: subdirectory should exist
ls /srv/nfs/exports/
# default-nfs-test-pvc/   ← Created by provisioner

kubectl delete pvc nfs-test-pvc
```

---

## 7. Understanding Persistent Volumes {#7-persistent-volumes}

### 7.1 PV Architecture and Phases

A PersistentVolume is a **cluster-level resource** that represents a piece of storage. It has an independent lifecycle from any pod.

```
PV Lifecycle Phases:
─────────────────────────────────────────────────────────────────────
Available  → PV created; no PVC bound yet
Bound      → PV bound to a PVC
Released   → PVC deleted; PV not yet reclaimed
Failed     → Automatic reclamation failed
```

### 7.2 PV Access Modes

| Access Mode | Short | Description | Typical Backend |
|---|---|---|---|
| `ReadWriteOnce` | RWO | Single node read-write | AWS EBS, GCP PD, iSCSI |
| `ReadWriteOncePod` | RWOP | Single **pod** read-write (v1.22+) | AWS EBS, GCP PD |
| `ReadOnlyMany` | ROX | Multiple nodes read-only | NFS, CephFS |
| `ReadWriteMany` | RWX | Multiple nodes read-write | NFS, CephFS, Azure Files |

**Critical production note**: Most block storage (EBS, GCP PD, Azure Disk) only supports `ReadWriteOnce`. If you need `ReadWriteMany`, you need shared storage (NFS, CephFS, Azure Files).

### 7.3 PV Capacity and Volume Modes

```yaml
spec:
  capacity:
    storage: 10Gi    # Actual storage size

  volumeMode: Filesystem    # Filesystem (default) or Block
  # Filesystem: formatted and mounted as directory
  # Block: raw block device (for databases like SAP HANA)
```

### 7.4 Static vs Dynamic Provisioning

| Type | Who Creates PV | When PV Created | Use Case |
|---|---|---|---|
| **Static** | Admin manually | Before PVC exists | On-premises, legacy storage |
| **Dynamic** | StorageClass provisioner | When PVC is created | Cloud, self-service environments |

### 7.5 Complete PV YAML (Static Provisioning)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-10gi
  labels:
    type: nfs
    environment: production
    tier: data
spec:
  # ── CAPACITY ────────────────────────────────────────────────────────
  capacity:
    storage: 10Gi

  # ── ACCESS MODES ────────────────────────────────────────────────────
  accessModes:
    - ReadWriteMany    # NFS supports concurrent multi-node access

  # ── VOLUME MODE ─────────────────────────────────────────────────────
  volumeMode: Filesystem

  # ── RECLAIM POLICY ──────────────────────────────────────────────────
  persistentVolumeReclaimPolicy: Retain
  # Options: Retain, Delete, Recycle (deprecated)

  # ── STORAGE CLASS ───────────────────────────────────────────────────
  storageClassName: nfs-client
  # If empty string "", this PV will only bind to PVCs with no storageClassName

  # ── MOUNT OPTIONS ────────────────────────────────────────────────────
  mountOptions:
    - hard           # Hard mount (retry indefinitely on server failure)
    - nfsvers=4.1    # Use NFS v4.1
    - noatime        # Don't update access time (performance)
    - nodiratime     # Don't update directory access time

  # ── NFS-SPECIFIC CONFIG ──────────────────────────────────────────────
  nfs:
    server: 192.168.1.100
    path: /srv/nfs/exports/pv-nfs-10gi
    readOnly: false

  # ── NODE AFFINITY (for local volumes) ────────────────────────────────
  # nodeAffinity:
  #   required:
  #     nodeSelectorTerms:
  #       - matchExpressions:
  #           - key: kubernetes.io/hostname
  #             operator: In
  #             values:
  #               - worker-1
```

---

## 8. Understanding Persistent Volume Claims {#8-persistent-volume-claims}

### 8.1 What Is a PVC?

A PersistentVolumeClaim is a **user request for storage**. It specifies what the user needs (size, access mode, storage class) without caring about the underlying implementation. The PV controller matches PVCs to PVs.

### 8.2 PVC Binding Rules

For a PVC to bind to a PV, ALL of these must match:
1. **Access mode**: PV must support all requested access modes
2. **Capacity**: PV storage ≥ PVC requested storage (PVC may get more than requested)
3. **StorageClass**: Both must have the same `storageClassName` (or both empty)
4. **Selector**: If PVC has a selector, PV must match the labels
5. **VolumeMode**: Must match (both Filesystem or both Block)

### 8.3 PVC Binding Priority

```
When multiple PVs could match a PVC, binding priority:
  1. Smallest PV that satisfies the request (best fit)
  2. PVs without storageClass preferred over those with
  3. First available PV alphabetically (tie-break)

A PVC bound to a LARGER PV wastes storage.
Use specific storageClassName to prevent unexpected bindings.
```

### 8.4 Complete PVC YAML

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data-pvc
  namespace: production
  labels:
    app: postgres
    tier: data
  annotations:
    description: "Persistent storage for PostgreSQL production database"
spec:
  # ── ACCESS MODES ────────────────────────────────────────────────────
  accessModes:
    - ReadWriteOnce    # Only one pod writes at a time (typical for databases)

  # ── VOLUME MODE ─────────────────────────────────────────────────────
  volumeMode: Filesystem

  # ── RESOURCES ───────────────────────────────────────────────────────
  resources:
    requests:
      storage: 50Gi    # Request 50Gi of storage
    limits:
      storage: 100Gi   # Hard cap at 100Gi (not all provisioners support this)

  # ── STORAGE CLASS ───────────────────────────────────────────────────
  storageClassName: fast-ssd    # Which StorageClass to use for dynamic provisioning
  # If omitted: uses default StorageClass
  # If "": must bind to a PV with empty storageClassName (static only)

  # ── SELECTOR (for static PV binding only) ──────────────────────────
  selector:
    matchLabels:
      environment: production
    matchExpressions:
      - key: tier
        operator: In
        values: [data, storage]

  # ── VOLUME NAME (forces binding to specific PV) ─────────────────────
  # volumeName: pv-specific-volume-name
  # (Optional: pre-bind to a specific PV without selector)
```

### 8.5 PVC Expansion (Volume Resizing)

```bash
# Requirements for expansion:
# 1. StorageClass must have allowVolumeExpansion: true
# 2. CSI driver must support volume expansion
# 3. PVC must be in Bound state

# Expand a PVC (just edit the storage request)
kubectl patch pvc postgres-data-pvc -n production \
  -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'

# Or using kubectl edit
kubectl edit pvc postgres-data-pvc -n production
# Change: storage: 50Gi → storage: 100Gi

# Watch expansion status
kubectl get pvc postgres-data-pvc -n production -w
# The CAPACITY column will update when expansion completes

# If filesystem resizing is needed (some drivers require pod restart):
kubectl describe pvc postgres-data-pvc -n production | grep "Resize"
# If "FileSystemResizePending": restart the pod to trigger fs resize
kubectl rollout restart deployment/postgres -n production
```

---

## 9. Lab: Setting Up Persistent Volumes {#9-lab-persistent-volumes}

### 9.1 Creating Static PVs for Different Backends

```bash
# ── STATIC NFS PERSISTENT VOLUME ────────────────────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-prod-db-50gi
  labels:
    type: nfs
    environment: production
    tier: database
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-client
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    server: 192.168.1.100
    path: /srv/nfs/exports/prod-db
EOF

# ── STATIC LOCAL VOLUME (high performance, single node) ─────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-local-ssd-worker1
  labels:
    type: local
    node: k8s-worker-1
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/fast-ssd    # Must exist on the node!
  nodeAffinity:            # REQUIRED for local volumes
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - k8s-worker-1
EOF

# ── STATIC hostPath VOLUME (development only!) ────────────────────────────
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-hostpath-dev
  labels:
    type: hostpath
    environment: development
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: manual
  hostPath:
    path: /tmp/k8s-pv    # Exists on node where pod runs
    type: DirectoryOrCreate
EOF

# Verify PVs created
kubectl get pv
# NAME                      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS
# pv-nfs-prod-db-50gi       50Gi       RWX            Retain           Available
# pv-local-ssd-worker1      100Gi      RWO            Delete           Available
# pv-hostpath-dev           5Gi        RWO            Delete           Available

kubectl describe pv pv-nfs-prod-db-50gi
```

### 9.2 Creating Multiple PVs for StatefulSet

```bash
# Pre-create PVs for a 3-replica Kafka StatefulSet
for i in 0 1 2; do
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-kafka-data-${i}
  labels:
    app: kafka
    node: worker-${i}
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-client
  nfs:
    server: 192.168.1.100
    path: /srv/nfs/exports/kafka-${i}
EOF
done

# Verify all 3 PVs created and Available
kubectl get pv -l app=kafka
```

---

## 10. Lab: Setting Up Persistent Volume Claims {#10-lab-persistent-volume-claims}

### 10.1 Static PVC Binding

```bash
# Create PVC that binds to our static NFS PV
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: default
  labels:
    app: postgres
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-client
  resources:
    requests:
      storage: 50Gi
  selector:
    matchLabels:
      environment: production
      tier: database
EOF

# Watch binding happen (Static PVs bind based on matching criteria)
kubectl get pvc postgres-data -w
# postgres-data   Pending                                          ← Looking for PV
# postgres-data   Bound    pv-nfs-prod-db-50gi    50Gi   RWX      ← Found and bound!

# Verify the binding
kubectl get pvc postgres-data
kubectl get pv pv-nfs-prod-db-50gi
# Both should show BOUND status

kubectl describe pvc postgres-data
# Volume: pv-nfs-prod-db-50gi
# Status: Bound
```

### 10.2 Dynamic PVC Binding via StorageClass

```bash
# Dynamic provisioning: PVC creates PV automatically via StorageClass
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-dynamic
  namespace: default
  labels:
    app: my-application
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-client    # NFS provisioner will create PV automatically
  resources:
    requests:
      storage: 10Gi
EOF

# Watch: provisioner creates PV then binds PVC
kubectl get pvc app-data-dynamic -w
kubectl get pv  # See the dynamically created PV

# Check what was created on NFS server:
# ls /srv/nfs/exports/
# default-app-data-dynamic-pvc-xxxxx/   ← Created by provisioner
```

### 10.3 PVC for StatefulSet (volumeClaimTemplates)

```bash
# StatefulSet creates individual PVCs per pod using volumeClaimTemplates
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres-statefulset
  namespace: default
spec:
  serviceName: "postgres"
  replicas: 1
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
          image: postgres:14-alpine
          env:
            - name: POSTGRES_PASSWORD
              value: "postgrespass"
            - name: PGDATA
              value: "/var/lib/postgresql/data/pgdata"
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: "1"
              memory: 1Gi
  # volumeClaimTemplates creates one PVC PER replica
  volumeClaimTemplates:
    - metadata:
        name: postgres-storage
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: nfs-client
        resources:
          requests:
            storage: 10Gi
EOF

# StatefulSet creates PVC named: <template-name>-<pod-name>
kubectl get pvc
# postgres-storage-postgres-statefulset-0   Bound  ...  10Gi

kubectl get pods
# postgres-statefulset-0  Running

kubectl delete statefulset postgres-statefulset
```

---

## 11. Lab: Creating Pods with Volumes Attached {#11-lab-pods-with-volumes}

### 11.1 Single Pod with NFS Volume

```bash
# First ensure we have a working NFS PVC
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: web-content-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs-client
  resources:
    requests:
      storage: 5Gi
EOF

# Wait for PVC to be bound
kubectl wait pvc/web-content-pvc --for=jsonpath='{.status.phase}'=Bound --timeout=60s

# Create a pod that uses the PVC
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: writer-pod
  labels:
    app: writer
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Writer pod started" >> /data/log.txt
          echo "$(date): Written by pod $POD_NAME" >> /data/log.txt
          while true; do
            echo "$(date): heartbeat" >> /data/log.txt
            sleep 10
          done
      env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
      volumeMounts:
        - name: shared-storage
          mountPath: /data    # Mount point inside container
      resources:
        requests:
          cpu: 50m
          memory: 32Mi
  volumes:
    - name: shared-storage
      persistentVolumeClaim:
        claimName: web-content-pvc    # Reference to our PVC
        readOnly: false
EOF

# Wait for pod
kubectl wait pod/writer-pod --for=condition=Ready --timeout=60s

# Verify volume is mounted
kubectl exec writer-pod -- df -h /data
# /dev/... or NFS mount showing capacity

kubectl exec writer-pod -- ls -la /data/
kubectl exec writer-pod -- cat /data/log.txt
```

### 11.2 Multiple Pods Sharing the Same Volume (NFS RWX)

```bash
# Create a second pod reading from the same NFS volume
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: reader-pod
  labels:
    app: reader
spec:
  containers:
    - name: reader
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          echo "Reader pod watching shared volume..."
          while true; do
            echo "=== $(date) ==="
            tail -5 /data/log.txt 2>/dev/null || echo "No log file yet"
            sleep 5
          done
      volumeMounts:
        - name: shared-storage
          mountPath: /data
          readOnly: true    # Reader only needs read access
      resources:
        requests:
          cpu: 50m
          memory: 32Mi
  volumes:
    - name: shared-storage
      persistentVolumeClaim:
        claimName: web-content-pvc    # SAME PVC as writer-pod!
        readOnly: true
EOF

kubectl wait pod/reader-pod --for=condition=Ready --timeout=60s

# Watch reader-pod's output (should see writer-pod's data)
kubectl logs reader-pod -f &

# Watch writer update the file
kubectl exec writer-pod -- tail -f /data/log.txt &

sleep 30

# Both pods share the same NFS storage successfully!
kill %1 %2
```

### 11.3 Deployment with Persistent Volume

```bash
# Production pattern: Deployment with PVC
cat << 'EOF' | kubectl apply -f -
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: uploads-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteMany    # Multiple replicas can all write to NFS
  storageClassName: nfs-client
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: file-server
spec:
  replicas: 3    # All 3 replicas share the same NFS volume
  selector:
    matchLabels:
      app: file-server
  template:
    metadata:
      labels:
        app: file-server
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          volumeMounts:
            - name: uploads
              mountPath: /usr/share/nginx/html/uploads
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
      volumes:
        - name: uploads
          persistentVolumeClaim:
            claimName: uploads-pvc
EOF

kubectl rollout status deployment/file-server

# Verify all 3 pods have the volume mounted
kubectl get pods -l app=file-server -o wide
for pod in $(kubectl get pods -l app=file-server -o name); do
  echo "=== $pod ==="
  kubectl exec $pod -- df -h /usr/share/nginx/html/uploads
done
```

### 11.4 Pod with Multiple Volume Types

```bash
# Production pattern: multiple volumes in one pod
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: multi-volume-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      volumeMounts:
        # 1. Persistent NFS storage for data
        - name: persistent-data
          mountPath: /app/data
        # 2. ConfigMap as configuration file
        - name: app-config
          mountPath: /etc/config
          readOnly: true
        # 3. Secret as credentials
        - name: db-credentials
          mountPath: /etc/secrets
          readOnly: true
        # 4. EmptyDir for temporary files
        - name: tmp-cache
          mountPath: /tmp/cache
        # 5. DownwardAPI for pod metadata
        - name: pod-info
          mountPath: /etc/pod-info
          readOnly: true
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
  volumes:
    # 1. Persistent NFS volume
    - name: persistent-data
      persistentVolumeClaim:
        claimName: uploads-pvc

    # 2. ConfigMap volume
    - name: app-config
      configMap:
        name: app-config
        optional: true

    # 3. Secret volume (tmpfs, not on disk)
    - name: db-credentials
      secret:
        secretName: db-secret
        optional: true
        defaultMode: 0400    # Read-only for owner only

    # 4. EmptyDir (temporary, destroyed with pod)
    - name: tmp-cache
      emptyDir:
        medium: Memory    # Use RAM for maximum performance
        sizeLimit: 256Mi

    # 5. Downward API
    - name: pod-info
      downwardAPI:
        items:
          - path: "pod-name"
            fieldRef:
              fieldPath: metadata.name
          - path: "namespace"
            fieldRef:
              fieldPath: metadata.namespace
          - path: "cpu-limit"
            resourceFieldRef:
              containerName: app
              resource: limits.cpu
EOF

kubectl wait pod/multi-volume-pod --for=condition=Ready --timeout=60s
kubectl exec multi-volume-pod -- mount | grep -E "nfs|tmpfs"
```

### 11.5 Cleanup Lab Resources

```bash
# Clean up in correct order (pod first, then PVC, then PV for static)
kubectl delete pod writer-pod reader-pod multi-volume-pod
kubectl delete deployment file-server
kubectl delete pvc web-content-pvc uploads-pvc app-data-dynamic

# For dynamically provisioned PVs with reclaimPolicy: Delete,
# PVs are automatically deleted when PVCs are deleted.
# For Retain policy PVs, manually delete and clean up storage backend.

kubectl get pv  # Check if any PVs remain (Released state)
```

---

## 12. Built-in Controllers Deep Dive {#12-built-in-controllers}

### 12.1 PersistentVolume Controller

The most critical controller for storage. Manages the binding between PVCs and PVs.

```
RECONCILIATION LOOP:
  WATCH: PV and PVC objects
  COMPARE:
    For each Pending PVC:
      Find matching Available PV (check: accessModes, storage, storageClass, labels)
      If match found: bind them
    For each Released PV:
      Check reclaimPolicy:
        Delete  → delete the PV object (provisioner deletes storage)
        Retain  → leave PV in Released state
        Recycle → schedule scrub job (deprecated)
  ACT: PATCH PV.spec.claimRef and PVC.spec.volumeName
```

### 12.2 AttachDetach Controller

Manages the attachment and detachment of volumes to/from nodes.

```bash
# AttachDetach controller:
# 1. Monitors which pods need which volumes on which nodes
# 2. Calls CSI ControllerPublishVolume to attach (via VolumeAttachment object)
# 3. Calls CSI ControllerUnpublishVolume to detach when pod is removed

# View current VolumeAttachments
kubectl get volumeattachments
# NAME                                     ATTACHER             PV                   NODE          ATTACHED
# csi-abc123...                            ebs.csi.aws.com     pvc-xxx...            node-1        true

# If a volume is stuck attaching/detaching:
kubectl describe volumeattachment <name>
```

### 12.3 ReplicaSet Controller

RS controller creates Pods. Pods reference PVCs. If a PVC is in Pending state, the Pod starts but the volume mount fails (pod enters Pending or ContainerCreating state until PVC is bound).

```bash
# Pods in ContainerCreating state often have pending PVCs
kubectl describe pod <pod-name> | grep "Events:" -A 10
# Warning  FailedMount  5s  kubelet  Unable to attach or mount volumes:
# waiting for a volume to be created, either by external provisioner or manually
```

### 12.4 Deployment Controller

Manages rolling updates. During a rolling update with PVCs:
- `ReadWriteOnce` PVCs: Old pod must release the volume before new pod can use it (with proper pod deletion)
- `ReadWriteMany` PVCs: Multiple pods can share simultaneously — rolling update works seamlessly

### 12.5 Node Controller

Adds taints to unhealthy nodes. When a node becomes `NotReady` with `ReadWriteOnce` volumes:
- Pods evicted after 5 minutes (default)
- AttachDetach controller detaches volumes from unhealthy node
- Volumes reattached to new node where replacement pods are scheduled
- This is why `ReadWriteOnce` volumes can cause brief downtime on node failure

### 12.6 Service Controller

Not directly related to storage but often combined with StatefulSets that use volumes. Headless Services provide stable DNS names for StatefulSet pods, enabling stable identity for stateful applications with persistent volumes.

### 12.7 Namespace Controller

When a namespace is deleted, cascade deletion includes PVCs. PVs with `Delete` reclaimPolicy are automatically deleted. PVs with `Retain` remain in `Released` state and must be manually reclaimed.

### 12.8 Job Controller

Jobs often use PVCs for storing batch processing results. Important: `ReadWriteOnce` PVCs with multiple job parallelism don't work (only one pod can mount RWO at a time). Use `ReadWriteMany` (NFS) or shared emptyDir for parallel jobs.

### 12.9 StatefulSet Controller

The StatefulSet controller's relationship with storage is unique:
- Uses `volumeClaimTemplates` to create one PVC per pod replica
- PVCs named: `<template-name>-<statefulset-name>-<ordinal>`
- When StatefulSet is scaled down, PVCs are **NOT** deleted (data preserved)
- When scaled back up, the same PVCs are reused (pod gets its exact same data back)

```bash
# Observe StatefulSet PVC creation
kubectl scale statefulset my-db --replicas=3
kubectl get pvc | grep my-db
# data-my-db-0  Bound   (created with pod-0)
# data-my-db-1  Bound   (created with pod-1)
# data-my-db-2  Bound   (created with pod-2)

kubectl scale statefulset my-db --replicas=0
kubectl get pvc | grep my-db
# PVCs still exist! Data preserved.

kubectl scale statefulset my-db --replicas=3
# my-db-0 reattaches to data-my-db-0 (same data!)
```

### 12.10 DaemonSet Controller

DaemonSets often manage node-level storage provisioners (like Longhorn or local-path-provisioner) running as DaemonSets. These use `hostPath` volumes to access local node storage for provisioning purposes.

### 12.11 Garbage Collector

Uses `ownerReferences` for storage cleanup:
- StatefulSet → owns PVC templates (but doesn't auto-delete PVCs when StatefulSet deleted)
- Deployment → owns ReplicaSet → owns Pods (cascade)
- PVCs are NOT owned by the pods that use them (by design — data preservation)

---

## 13. Internal Working Concepts — Informers, Work Queues & Reconciliation {#13-internal-concepts}

### 13.1 PV Controller Informer Setup

```
┌──────────────────────────────────────────────────────────────────────┐
│          PV CONTROLLER — INFORMER ARCHITECTURE                       │
│                                                                      │
│  PV Informer:                                                        │
│  LIST + WATCH /api/v1/persistentvolumes                             │
│  Cache: all PVs indexed by status.phase, capacity, access modes     │
│  OnAdd(pv) → enqueue "pv/<pv-name>" for binding consideration       │
│  OnUpdate(pv) → check if status changed (e.g., Released)            │
│                                                                      │
│  PVC Informer:                                                       │
│  LIST + WATCH /api/v1/persistentvolumeclaims                        │
│  OnAdd(pvc) → enqueue "ns/pvc-name" for binding                     │
│  OnUpdate(pvc) → check if now needs binding or released             │
│                                                                      │
│  Work Queue:                                                         │
│  Keys: "default/postgres-pvc", "pv/pv-nfs-10gi"                    │
│  Deduplication: multiple PV updates → single reconcile              │
│                                                                      │
│  Workers (5 goroutines):                                            │
│  syncClaim(pvc) → find matching PV → bind                          │
│  syncVolume(pv)  → handle reclaim on release                        │
└──────────────────────────────────────────────────────────────────────┘
```

### 13.2 AttachDetach Controller Internal Loop

```
AttachDetach Controller maintains:
  desiredStateOfWorld: which volumes SHOULD be attached to which nodes
  actualStateOfWorld:  which volumes ARE attached to which nodes

Reconciliation:
  For each volume in desiredStateOfWorld but not actualStateOfWorld:
    → Trigger Attach (via VolumeAttachment object)
  For each volume in actualStateOfWorld but not desiredStateOfWorld:
    → Trigger Detach

VolumeAttachment lifecycle:
  1. AttachDetach creates VolumeAttachment object
  2. CSI external-attacher watches VolumeAttachment
  3. external-attacher calls CSI ControllerPublishVolume (API)
  4. Storage backend attaches volume to node
  5. external-attacher updates VolumeAttachment.status.attached = true
  6. kubelet detects volume is attached → mounts into pod
```

### 13.3 Dynamic Provisioning Work Queue

```
CSI External Provisioner (sidecar with CSI driver):
  Watches: PVCs with matching StorageClass and unbound state
  
  On new PVC:
    1. Call CSI CreateVolume API (talks to storage backend)
    2. Create PV object with the provisioned volume details
    3. PV Controller binds PVC → PV

Rate limiting and retry:
  If CreateVolume fails: exponential backoff
  MaxRetries: configurable
  Events recorded on PVC: "ProvisioningFailed"
```

---

## 14. API Server and etcd Interaction {#14-api-server-etcd}

### 14.1 The Golden Rule for Storage

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  ALL STORAGE CONTROLLERS NEVER ACCESS ETCD DIRECTLY.                ║
║                                                                      ║
║  PV Controller → kube-apiserver → etcd                             ║
║  AttachDetach Controller → kube-apiserver → etcd                    ║
║  CSI External Provisioner → kube-apiserver → etcd                   ║
║  kubelet → kube-apiserver → etcd (for volume state updates)         ║
║                                                                      ║
║  kube-apiserver provides for storage:                               ║
║  1. Validates PV/PVC YAML (correct fields, access modes, etc.)      ║
║  2. Admission: ResourceQuota for PVC storage requests               ║
║  3. RBAC: controls who can create/delete PVs and PVCs               ║
║  4. Watch cache: efficient fan-out to PV controller and kubelet      ║
║  5. Optimistic concurrency: prevents two controllers binding        ║
║     same PV to two PVCs simultaneously                              ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 14.2 etcd Key Structure for Storage

```bash
# Storage objects in etcd:
# /registry/persistentvolumes/<pv-name>
# /registry/persistentvolumeclaims/<namespace>/<pvc-name>
# /registry/storageclasses/<sc-name>
# /registry/volumeattachments/<va-name>
# /registry/csidrivers/<driver-name>
# /registry/csinodes/<node-name>

# Verify etcd keys (on control plane node)
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl get /registry/persistentvolumes --prefix --keys-only \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key 2>/dev/null | head -10
```

### 14.3 Optimistic Concurrency in PV Binding

```
BINDING RACE CONDITION PREVENTION:

PVC A and PVC B both want the same PV.
Both PVC controllers detect "PV-1 is Available" simultaneously.

PV Controller for PVC-A:
  1. Read PV-1 (resourceVersion: 12345)
  2. Compute: PVC-A can bind to PV-1
  3. PATCH PV-1: claimRef=PVC-A, version=12345

PV Controller for PVC-B (simultaneously):
  1. Read PV-1 (resourceVersion: 12345)
  2. Compute: PVC-B can bind to PV-1
  3. PATCH PV-1: claimRef=PVC-B, version=12345

API server uses optimistic locking:
  First patch succeeds (resourceVersion matches)
  Second patch FAILS (resourceVersion conflict)
  PVC-B gets error, controller retries with fresh read
  Retry: PV-1 is now Bound to PVC-A → PVC-B finds different PV
```

---

## 15. Leader Election — HA for Storage Controllers {#15-leader-election}

### 15.1 Leader Election for PV Controller

```bash
# PV Controller and AttachDetach Controller run inside kube-controller-manager
# Only one kube-controller-manager instance is active (leader)
# Leader election via Lease object

kubectl get lease kube-controller-manager -n kube-system -o yaml
# spec:
#   holderIdentity: "k8s-cp-1_abc123"  ← Active leader
#   leaseDurationSeconds: 15

# What happens during leader failover (~15-30 seconds)?
#   - PVC binding pauses (new PVCs stay Pending)
#   - AttachDetach operations pause (volumes not attached/detached)
#   - EXISTING mounted volumes keep working (kubelet manages independently)
#   - After new leader elected: full resync
#   - Any missed bindings caught by re-list of PVs and PVCs
```

### 15.2 External Provisioner Leader Election

CSI external provisioners have their own leader election (separate from kube-controller-manager):

```bash
# CSI external provisioners in HA mode run multiple replicas
# but only one actively provisions at a time
# Uses Kubernetes Leases for coordination

kubectl get lease -n kube-system | grep csi
# ebs-csi-aws-com-...   kube-system  ...  ebs-csi-controller-...

# CSI sidecars that support HA:
# - external-provisioner (--leader-election flag)
# - external-attacher (--leader-election flag)
# - external-snapshotter
# - external-resizer
```

### 15.3 Leader Election Flags

| Flag | Default | Description |
|---|---|---|
| `--leader-elect` | `true` | Enable leader election for kube-controller-manager |
| `--leader-elect-lease-duration` | `15s` | How long lease is valid |
| `--leader-elect-renew-deadline` | `10s` | Must renew before this |
| `--leader-elect-retry-period` | `2s` | Standby check interval |
| `--leader-election` | false | CSI sidecar leader election (must enable explicitly) |

---

## 16. Volume Topology and Scheduling Integration {#16-volume-topology}

### 16.1 Why Topology Matters

Block storage like AWS EBS and GCP PD is **zone-specific** — a disk created in `us-east-1a` can only be attached to nodes in `us-east-1a`. If the scheduler places a pod in `us-east-1b`, it can't use an `us-east-1a` EBS volume.

```
PROBLEM WITHOUT TOPOLOGY AWARENESS:
  PVC created → EBS volume provisioned in us-east-1a (random)
  Pod scheduled → lands on node in us-east-1b
  Pod fails: cannot mount EBS volume from different AZ

SOLUTION: WaitForFirstConsumer volumeBindingMode
  PVC created → NO volume provisioned yet
  Pod scheduled → kube-scheduler picks node in us-east-1a (for some reason)
  StorageClass provisioner: "Provision in us-east-1a" (same zone as pod)
  EBS volume created in us-east-1a → successfully attached to pod
```

### 16.2 Configuring Topology-Aware Storage

```yaml
# StorageClass with topology awareness
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-topology-aware
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
# WaitForFirstConsumer: Volume only provisioned after pod is scheduled
# This enables topology-aware provisioning
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
  - matchLabelExpressions:
      - key: topology.kubernetes.io/zone
        values:
          - us-east-1a
          - us-east-1b
          - us-east-1c
```

---

## 17. Performance Tuning for Storage Operations {#17-performance-tuning}

### 17.1 kube-controller-manager Storage Flags

```bash
# PV controller concurrency
# --concurrent-pv-syncs=5    (default: 5)
# --pvclaimbinder-sync-period=15s  (default: 15s, how often to re-sync all PVCs)
# Increase for clusters with many PVCs being created/deleted rapidly

# AttachDetach controller
# --attach-detach-reconcile-sync-period=1m  (default: 1m)
# --disable-attach-detach-reconcile-sync=false
# How often the reconcile loop runs for attach/detach

# Check current settings
kubectl get pod kube-controller-manager-$(hostname) -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | \
  tr ' ' '\n' | grep -E "pv|pvc|attach|detach|volume"
```

### 17.2 NFS Mount Performance Optimization

```yaml
# Performance-optimized NFS mount options
mountOptions:
  - hard          # Retry indefinitely (safer than soft)
  - nfsvers=4.1   # Use NFS v4.1 (most efficient for Kubernetes)
  - rsize=1048576 # Read buffer: 1MB
  - wsize=1048576 # Write buffer: 1MB
  - timeo=600     # Timeout: 60 seconds (10 × 100ms units)
  - retrans=2     # Retransmissions before failure
  - noatime       # Don't update access time (saves IOPS)
  - nodiratime    # Don't update directory access time
  - proto=tcp     # Use TCP (more reliable over WAN)
```

### 17.3 CSI Driver Performance Flags

```bash
# External provisioner performance flags
--worker-threads=100           # Parallel provision/delete operations
--timeout=3m                   # CSI call timeout
--retry-interval-start=1s      # First retry delay
--retry-interval-max=5m        # Max retry interval

# External attacher performance flags
--worker-threads=10            # Parallel attach/detach operations
--timeout=2m                   # CSI attach timeout

# EBS CSI specific
--volume-attach-limit=39       # Max volumes per node (AWS limit varies)
```

### 17.4 Storage Class Performance Profiles

```yaml
# PROFILE 1: High-performance (databases, OLTP)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: database-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: io2
  iops: "64000"        # Maximum IOPS
  throughput: "1000"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true

# PROFILE 2: Cost-optimized (archival, backups)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: archive-hdd
provisioner: ebs.csi.aws.com
parameters:
  type: sc1            # Cold HDD (cheapest)
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

---

## 18. Security Hardening Practices {#18-security-hardening}

### 18.1 RBAC for Storage

```yaml
# Developer role: can create PVCs but not PVs
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: storage-user
  namespace: production
rules:
  # Can manage PVCs in their namespace
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # Can VIEW StorageClasses (but not create)
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
---
# Platform team: can manage PVs (cluster-scoped)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: storage-admin
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["*"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses", "volumeattachments", "csidrivers", "csinodes"]
    verbs: ["*"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update", "patch"]
```

### 18.2 Encryption at Rest

```yaml
# StorageClass with encryption enabled (AWS EBS)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: encrypted-storage
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:us-east-1:123456789:key/your-key-id"
  # Use customer-managed key for FIPS/compliance requirements
```

### 18.3 PVC Access Controls — LimitRange for Storage

```yaml
# Prevent oversized PVC requests
apiVersion: v1
kind: LimitRange
metadata:
  name: storage-limits
  namespace: production
spec:
  limits:
    - type: PersistentVolumeClaim
      min:
        storage: 100Mi    # Must request at least 100Mi
      max:
        storage: 500Gi    # Cannot request more than 500Gi
```

### 18.4 ResourceQuota for Storage

```yaml
# Limit total storage per namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: production
spec:
  hard:
    requests.storage: 1Ti          # Total storage requests ≤ 1Ti
    persistentvolumeclaims: "20"   # Max 20 PVCs
    fast-ssd.storageclass.storage.k8s.io/requests.storage: 500Gi  # Per-class quota
    nfs-client.storageclass.storage.k8s.io/requests.storage: 2Ti
    nfs-client.storageclass.storage.k8s.io/persistentvolumeclaims: "50"
```

### 18.5 Secure NFS Configuration

```bash
# /etc/exports — secure NFS configuration
# Use specific CIDR (not 0.0.0.0/0)
/srv/nfs/exports 10.0.1.0/24(rw,sync,no_subtree_check,root_squash)
# root_squash: maps root from client to anonymous user on server
# NO no_root_squash in production unless required

# Use NFSv4 with Kerberos for authentication (enterprise)
/srv/secure 10.0.0.0/8(rw,sec=krb5p,no_subtree_check)
# sec=krb5p: Kerberos privacy mode (authentication + encryption)

# Firewall: restrict NFS access
sudo ufw allow from 10.0.0.0/8 to any port 2049 proto tcp
sudo ufw allow from 10.0.0.0/8 to any port 2049 proto udp
```

---

## 19. Monitoring and Observability for Storage {#19-monitoring}

### 19.1 Key Prometheus Metrics

| Metric | Description | Alert Threshold |
|---|---|---|
| `kube_persistentvolumeclaim_status_phase{phase="Pending"}` | PVCs stuck in Pending | > 0 for > 5 min |
| `kube_persistentvolumeclaim_status_phase{phase="Bound"}` | Bound PVCs count | Dashboard |
| `kube_persistentvolume_status_phase{phase="Failed"}` | Failed PVs | > 0 |
| `kube_persistentvolume_status_phase{phase="Released"}` | Released PVs needing cleanup | > 0 |
| `kubelet_volume_stats_available_bytes` | Available bytes per volume | < 20% free |
| `kubelet_volume_stats_capacity_bytes` | Capacity per volume | Trending analysis |
| `kubelet_volume_stats_used_bytes` | Used bytes per volume | > 80% = warning |
| `storage_operation_duration_seconds` | CSI operation latency | p99 > 30s |
| `volume_manager_total_volumes` | Total volumes on a node | Track for limits |

### 19.2 Storage Monitoring Commands

```bash
# Check all PVC states
kubectl get pvc --all-namespaces
kubectl get pvc --all-namespaces | grep -v "Bound"  # Find non-bound PVCs

# Check PV states
kubectl get pv
kubectl get pv | grep -v "Bound"  # Find unbound/released PVs

# Disk usage on a pod's mounted volume
kubectl exec <pod-name> -- df -h /data
kubectl exec <pod-name> -- du -sh /data/*

# Volume metrics via kubelet
curl -sk https://<node-ip>:10250/metrics | \
  grep "kubelet_volume_stats" | grep -v "^#"

# Check volume attachment status
kubectl get volumeattachments
kubectl describe volumeattachment <name>

# Find pods using a specific PVC
kubectl get pods --all-namespaces \
  -o json | jq '.items[] | 
    select(.spec.volumes[]?.persistentVolumeClaim.claimName == "my-pvc") |
    {namespace: .metadata.namespace, name: .metadata.name}'
```

### 19.3 Prometheus Alert Rules

```yaml
groups:
  - name: kubernetes-storage
    rules:
      # PVC stuck in Pending
      - alert: KubePVCPendingTooLong
        expr: |
          kube_persistentvolumeclaim_status_phase{phase="Pending"} == 1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "PVC {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }} pending > 10 min"

      # Volume nearly full
      - alert: KubeVolumeAlmostFull
        expr: |
          (kubelet_volume_stats_available_bytes
            / kubelet_volume_stats_capacity_bytes) < 0.2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Volume {{ $labels.persistentvolumeclaim }} is 80%+ full"

      # PV in Failed state
      - alert: KubePersistentVolumeErrors
        expr: |
          kube_persistentvolume_status_phase{phase=~"Failed|Pending"} > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "PV {{ $labels.persistentvolume }} in {{ $labels.phase }} state"

      # Released PV not reclaimed
      - alert: KubePVReleasedNotReclaimed
        expr: |
          kube_persistentvolume_status_phase{phase="Released"} > 0
        for: 30m
        labels:
          severity: info
        annotations:
          summary: "PV {{ $labels.persistentvolume }} Released but not reclaimed (manual action needed)"
```

---

## 20. Troubleshooting Storage Issues {#20-troubleshooting}

### 20.1 PVC Stuck in Pending

```bash
# ── STEP 1: Check PVC events ─────────────────────────────────────────────
kubectl describe pvc <pvc-name> -n <namespace>
# Look for: Events section, status conditions

# ── STEP 2: Common causes and diagnosis ──────────────────────────────────

# CAUSE 1: No matching PV available (static provisioning)
kubectl get pv | grep Available
kubectl get pvc <n> -o yaml | grep -A 5 "spec:"
# Check: accessModes, storageClassName, resources match a PV

# CAUSE 2: Provisioner not running (dynamic provisioning)
kubectl get pods -n nfs-provisioner
kubectl logs -n nfs-provisioner deployment/nfs-client-provisioner --tail=30

# CAUSE 3: StorageClass not found
kubectl get pvc <n> -o jsonpath='{.spec.storageClassName}'
kubectl get storageclass  # Is that class present?

# CAUSE 4: No default StorageClass and PVC doesn't specify one
kubectl get storageclass | grep "(default)"
# If none: either create default SC or specify storageClassName in PVC

# CAUSE 5: Quota exceeded (ResourceQuota limits storage)
kubectl describe resourcequota -n <namespace>
# Check: requests.storage used vs hard

# CAUSE 6: WaitForFirstConsumer mode (no pod using it yet)
kubectl get storageclass <sc-name> -o jsonpath='{.volumeBindingMode}'
# If WaitForFirstConsumer: PVC stays Pending until a pod references it
```

### 20.2 Pods Stuck in ContainerCreating (Volume Mount Failure)

```bash
# ── STEP 1: Describe the pod ─────────────────────────────────────────────
kubectl describe pod <pod-name> -n <namespace>
# Look for: Events, Volumes section, Mount errors

# ── STEP 2: Common volume mount failures ─────────────────────────────────

# CAUSE 1: PVC not bound
kubectl get pvc <pvc-name> -n <namespace>
# If Pending: fix PVC first (see above)

# CAUSE 2: Volume still attached to old node (RWO volumes)
kubectl get volumeattachment | grep <pv-name>
# If old attachment stuck:
kubectl describe volumeattachment <name>
# Force delete old VolumeAttachment if node is truly dead:
kubectl delete volumeattachment <name>

# CAUSE 3: NFS server unreachable
kubectl exec <any-pod> -- ping 192.168.1.100    # Test NFS server
kubectl exec <any-pod> -- showmount -e 192.168.1.100  # Test NFS export

# CAUSE 4: NFS client not installed on node
kubectl get pod <pod-name> -o wide              # Which node?
ssh <node>
sudo apt-get install -y nfs-common              # Fix: install nfs-common

# CAUSE 5: Permission denied on NFS mount
kubectl exec <pod-name> -- ls /data  # Permission denied?
# Fix: Check NFS server /etc/exports permissions
# Check: no_root_squash vs root_squash settings

# CAUSE 6: CSI driver not running on node
kubectl get pods -n kube-system | grep csi     # Check CSI pods
kubectl get csinode <node-name>                 # Check CSI registration
```

### 20.3 Deployment Stuck Due to Volume Issues

```bash
# Check why new pods are not starting
kubectl rollout status deployment/<n> -n <namespace>
kubectl get pods -l app=<label> -o wide

# For RWO volumes: old pod must fully terminate before new pod mounts
kubectl get pods -l app=<label> | grep Terminating
# If old pod stuck in Terminating:
kubectl delete pod <old-pod> --grace-period=0 --force

# Check if volume is attached to wrong node
kubectl get volumeattachment
kubectl describe volumeattachment <name>

# Check node conditions
kubectl describe node <node-name> | grep -A 5 "Conditions:"
kubectl get node <name> -o jsonpath='{.status.conditions[*].type}'
```

### 20.4 Node NotReady — Impact on Storage

```bash
# When node goes NotReady, volumes need to be forcefully detached
# Default timeout: 6 minutes (pod-eviction-timeout + attach-detach period)

# Check if volumes are stuck on NotReady node
kubectl get volumeattachment | grep <notready-node>

# For emergency (node truly dead): force detach
kubectl delete volumeattachment <va-name>

# Wait for pod eviction and rescheduling
kubectl get pods -l app=<n> -w

# After pod rescheduled: verify volume reattached to new node
kubectl describe pod <new-pod> | grep "Successfully attached"

# If volume is in bad state: manual recovery
kubectl edit pv <pv-name>
# Remove stale claimRef if PV stuck in Released
```

### 20.5 NFS-Specific Troubleshooting

```bash
# Test NFS connectivity from Kubernetes nodes
kubectl run nfs-test --image=busybox:1.36 --restart=Never \
  --command -- sh -c "showmount -e 192.168.1.100 || nc -zv 192.168.1.100 2049"
kubectl logs nfs-test
kubectl delete pod nfs-test

# Check NFS mount options on a pod
kubectl exec <pod> -- mount | grep nfs

# NFS performance test
kubectl exec <pod> -- dd if=/dev/zero of=/data/test.img bs=1M count=100
# Check write speed; if < 50MB/s for SSD NFS: tune network/NFS options

# NFS stale file handle
kubectl exec <pod> -- ls /data
# "Stale file handle" error: NFS server restarted or export changed
# Fix: Pod needs to restart to re-mount

# Check NFS server logs
sudo tail -100 /var/log/syslog | grep nfs
sudo exportfs -v   # Show active exports
```

---

## 21. Disaster Recovery — Storage and etcd Backups {#21-disaster-recovery}

### 21.1 Stateless Controllers — Storage Recovery

```
kube-controller-manager (PV Controller, AttachDetach) is STATELESS.
All PV/PVC/StorageClass data lives in etcd.

If controller-manager crashes:
  → Existing mounted volumes: CONTINUE WORKING (kubelet manages)
  → New PVC binding: PAUSED (~30s until new leader elected)
  → After recovery: controller re-syncs all PVs/PVCs and resumes

etcd backup contains:
  → All PV objects (claimRefs, status, storageClass, etc.)
  → All PVC objects (bound volume names, status)
  → All StorageClass objects
  → All VolumeAttachment objects

After etcd restore:
  → All storage bindings restored
  → CSI drivers pick up VolumeAttachments via watch
  → Existing pods re-attach their volumes
  → Data on storage backends UNAFFECTED (etcd only stores metadata)
```

### 21.2 Critical Distinction: Metadata vs Data

```
KUBERNETES ETCD BACKUP:
  Saves: PV/PVC metadata, bindings, StorageClass definitions
  Does NOT save: Actual data on the storage backend
  
  → Restore etcd: Kubernetes "knows" about storage again
  → Data recovery: depends on storage backend's own backup strategy

DATA BACKUP (separate from etcd):
  EBS: AWS snapshots, EBS Lifecycle Manager
  GCP PD: GCP Snapshots
  NFS: NFS server filesystem backup (rsync, Bacula, etc.)
  Ceph: Ceph snapshots, Velero + Ceph plugin
  
VELERO: Kubernetes backup tool that handles BOTH:
  → Backs up Kubernetes objects (PVs, PVCs, etc.) from etcd/API
  → Backs up volume data (using CSI snapshots or restic)
```

### 21.3 Volume Snapshot Support

```bash
# CSI Volume Snapshots (requires snapshot controller + CRDs)
# Install snapshot CRDs
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v6.3.0/client/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v6.3.0/client/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v6.3.0/client/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml

# Create a VolumeSnapshot
cat << 'EOF' | kubectl apply -f -
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snapshot-$(date +%Y%m%d)
  namespace: production
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: postgres-data
EOF

# Restore from snapshot
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data-restored
  namespace: production
spec:
  dataSource:
    name: postgres-snapshot-20250101
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 50Gi
EOF
```

---

## 22. Comparison — kube-apiserver vs kube-scheduler vs kube-controller-manager {#22-comparison}

| Dimension | kube-apiserver | kube-scheduler | kube-controller-manager |
|---|---|---|---|
| **Storage role** | Stores all PV/PVC/SC objects; validates requests; RBAC | VolumeBinding plugin: ensures pod scheduled on node compatible with volume topology | PV Controller: binds PVCs to PVs; AttachDetach: manages volume attachment to nodes |
| **Volume binding** | Stores the binding (PV.spec.claimRef) | Considers volume topology when selecting node for pod | Executes the binding — writes claimRef to API server |
| **Dynamic provisioning** | Stores newly created PV (created by provisioner) | N/A (topology via WaitForFirstConsumer) | N/A (external provisioner does this) |
| **etcd access** | YES (direct, exclusive) | NO (via API server) | NO (via API server) |
| **HA model** | Active-Active | Active-Passive (leader) | Active-Passive (leader) |
| **Failure impact** | No new PV/PVC ops | New pods not scheduled (affects volume topology binding) | PVC binding paused; AttachDetach paused; existing volumes OK |
| **Port** | 6443 | 10259 | 10257 |
| **Recovery time** | Immediate (behind LB) | ~30s leader election | ~30s leader election |

---

## 23. ASCII Architecture Diagram {#23-ascii-diagram}

```
╔════════════════════════════════════════════════════════════════════════════════════╗
║                   KUBERNETES PERSISTENT STORAGE — COMPLETE ARCHITECTURE           ║
╠════════════════════════════════════════════════════════════════════════════════════╣
║                                                                                    ║
║  ADMIN: kubectl apply -f pv.yaml / storageclass.yaml                              ║
║  USER:  kubectl apply -f pvc.yaml (or via StatefulSet volumeClaimTemplates)       ║
║           │                                                                        ║
║           │ HTTPS :6443                                                            ║
║           ▼                                                                        ║
║  ┌──────────────────────────────────────────────────────────────────────────────┐  ║
║  │                     kube-apiserver :6443                                     │  ║
║  │  Stores: PV, PVC, StorageClass, VolumeAttachment, CSIDriver, CSINode         │  ║
║  │  Validates: access modes, volume mode, storage class references              │  ║
║  │  RBAC: controls who can create PVs vs PVCs vs StorageClasses                 │  ║
║  └──────────────┬───────────────────────────────────────────────────────────────┘  ║
║                 │ Watch events                                                      ║
║   ┌─────────────┼──────────────────────────────────────────────────┐               ║
║   │             │                                                   │               ║
║  ┌▼─────────────────────────┐  ┌──────────────────────┐  ┌─────────▼──────────┐   ║
║  │   etcd :2379              │  │ kube-controller-mgr  │  │  kube-scheduler    │   ║
║  │  /registry/              │  │ :10257 (Leader Elected│  │  :10259            │   ║
║  │  ├── persistentvolumes   │  │                       │  │                    │   ║
║  │  ├── pvc                 │  │  PV Controller:       │  │  VolumeBinding     │   ║
║  │  ├── storageclasses      │  │  Watch PVs + PVCs     │  │  Plugin:           │   ║
║  │  ├── volumeattachments   │  │  Bind matching pairs  │  │  - For            │   ║
║  │  └── csidrivers          │  │  Handle reclaim       │  │    WaitForFirst-  │   ║
║  │                          │  │                       │  │    Consumer SCs   │   ║
║  │  Only API server         │  │  AttachDetach Ctrl:   │  │  - Ensures pod   │   ║
║  │  accesses etcd!          │  │  Manages volume       │  │    scheduled in   │   ║
║  └──────────────────────────┘  │  attach/detach via    │  │    same zone as   │   ║
║                                 │  VolumeAttachment     │  │    volume         │   ║
║                                 │  objects              │  └────────────────────┘  ║
║                                 │                       │                          ║
║                                 │  NEVER talks to etcd! │                          ║
║                                 └───────────────────────┘                          ║
║                                                                                    ║
║  DYNAMIC PROVISIONING FLOW:                                                        ║
║  ┌──────────────────────────────────────────────────────────────────────────────┐  ║
║  │  CSI Driver (DaemonSet on each node)                                         │  ║
║  │  ┌───────────────┐  ┌────────────────────┐  ┌──────────────────────────────┐ │  ║
║  │  │ external-     │  │ external-attacher  │  │ CSI Node Plugin              │ │  ║
║  │  │ provisioner   │  │                    │  │ NodeStageVolume              │ │  ║
║  │  │ Watch PVCs →  │  │ Watch VolumeAttach │  │ NodePublishVolume (mount)    │ │  ║
║  │  │ CreateVolume  │  │ ControllerPublish  │  │ NodeUnpublishVolume (umount) │ │  ║
║  │  │ Create PV     │  │ (attach to node)   │  │                              │ │  ║
║  │  └───────────────┘  └────────────────────┘  └──────────────────────────────┘ │  ║
║  └──────────────────────────────────────────────────────────────────────────────┘  ║
║                                                                                    ║
║  STORAGE BACKENDS:                                                                 ║
║  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐    ║
║  │ NFS      │  │ AWS EBS  │  │ GCP PD   │  │ Ceph     │  │ Azure Disk/Files │    ║
║  │ (RWX)    │  │ (RWO)    │  │ (RWO)    │  │ RBD/CephFS│ │ (RWO/RWX)        │    ║
║  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────────────┘    ║
║                                                                                    ║
║  POD → PVC → (binding) → PV → (mount by kubelet) → /data in container             ║
║  (namespaced)   (binds)  (cluster-scoped)  (stored in etcd as metadata)            ║
╚════════════════════════════════════════════════════════════════════════════════════╝
```

---

## 24. Real-World Production Use Cases {#24-production-use-cases}

### 24.1 PostgreSQL Production Deployment

```yaml
# High-availability PostgreSQL with persistent storage
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd    # SSD for database performance
  resources:
    requests:
      storage: 200Gi
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: production
spec:
  serviceName: postgres
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    spec:
      containers:
        - name: postgres
          image: postgres:14-alpine
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: "2"
              memory: 4Gi
            limits:
              cpu: "4"
              memory: 8Gi
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 200Gi
```

### 24.2 Shared Content Management (NFS RWX)

```yaml
# WordPress cluster with shared media files on NFS
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wordpress-media
  namespace: production
spec:
  accessModes:
    - ReadWriteMany    # ALL WordPress replicas share media files
  storageClassName: nfs-client
  resources:
    requests:
      storage: 100Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: production
spec:
  replicas: 5    # 5 replicas all accessing same NFS volume
  template:
    spec:
      containers:
        - name: wordpress
          image: wordpress:6.4
          volumeMounts:
            - name: media-storage
              mountPath: /var/www/html/wp-content/uploads
      volumes:
        - name: media-storage
          persistentVolumeClaim:
            claimName: wordpress-media
```

### 24.3 Kafka Cluster with Per-Broker Storage

```yaml
# Each Kafka broker gets its own dedicated PVC via volumeClaimTemplates
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kafka
  namespace: messaging
spec:
  replicas: 3
  serviceName: kafka-headless
  template:
    spec:
      containers:
        - name: kafka
          image: confluentinc/cp-kafka:7.5.0
          volumeMounts:
            - name: data
              mountPath: /var/lib/kafka/data
          resources:
            requests:
              cpu: "2"
              memory: 4Gi
  # One PVC per pod: data-kafka-0, data-kafka-1, data-kafka-2
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 500Gi
```

---

## 25. Best Practices for Production Storage {#25-best-practices}

### 25.1 PV and PVC Design

- **Always use StorageClasses for dynamic provisioning** — avoid static PVs unless absolutely necessary
- **Use `WaitForFirstConsumer` for topology-sensitive storage** — prevents zone mismatch issues in multi-zone clusters
- **Set `reclaimPolicy: Retain` for production databases** — never auto-delete critical data
- **Label PVs with metadata** (environment, team, backup-schedule) for governance
- **Specify exact `storageClassName` in PVCs** — never rely on defaults for production

### 25.2 Capacity and Performance

- **Right-size PVC requests** — over-provisioning wastes money; under-provisioning causes failures
- **Set `allowVolumeExpansion: true`** in StorageClass — allows PVC resizing without recreation
- **Monitor volume utilization with Prometheus** — alert at 80% full
- **Use node-local storage for latency-sensitive workloads** (Kafka, time-series DBs)
- **Use NFS for shared access, not for high-IOPS workloads**

### 25.3 Reliability and HA

- **Use `ReadWriteOnce` for databases** — prevents data corruption from concurrent writes
- **Use `ReadWriteMany` (NFS/CephFS) only for stateless shared content**
- **Implement volume snapshots for backup** — use Velero or native CSI snapshots
- **Test DR procedure**: snapshot → delete PVC → restore from snapshot → verify data
- **Use PodDisruptionBudgets** with StatefulSets to protect against simultaneous pod eviction

### 25.4 Security

- **Enable encryption at rest** for all production StorageClasses
- **Use RBAC** to prevent developers from creating cluster-wide PVs
- **ResourceQuota per namespace** to limit total storage consumption
- **LimitRange per namespace** to limit per-PVC storage requests
- **Never expose NFS without network restrictions** — use firewall rules and IP allowlists

---

## 26. Common Mistakes and Pitfalls {#26-mistakes-pitfalls}

### 26.1 Using ReadWriteOnce with Multiple Replicas

**Mistake:** Deployment with 3 replicas all trying to use the same `ReadWriteOnce` PVC.  
**Impact:** Only the pod on the node where the PV is mounted can start; others fail with `Multi-Attach error`.  
**Fix:** Use `ReadWriteMany` (NFS/CephFS) or give each pod its own PVC via StatefulSet.

---

### 26.2 Not Setting reclaimPolicy: Retain for Databases

**Mistake:** Using default `reclaimPolicy: Delete` for database storage.  
**Impact:** Developer deletes namespace or PVC → entire database is destroyed.  
**Fix:** Always use `Retain` for production databases. Create separate StorageClasses.

---

### 26.3 Missing `no_root_squash` on NFS When Required

**Mistake:** NFS server has `root_squash` (default), but container runs as root.  
**Impact:** Container writes fail with "Permission denied" even though directory permissions look correct.  
**Fix:** Either use `no_root_squash` on the NFS server, or configure containers to run as non-root.

---

### 26.4 WaitForFirstConsumer Not Used in Multi-Zone Clusters

**Mistake:** Using `Immediate` volumeBindingMode with zone-specific storage (EBS, GCP PD).  
**Impact:** PVC provisioned in zone A; pod scheduled to zone B → `FailedMount` because volume not in same zone.  
**Fix:** Use `WaitForFirstConsumer` so volume is provisioned in the same zone as the pod.

---

### 26.5 Not Installing NFS Client on All Nodes

**Mistake:** Deploying NFS-backed pods without `nfs-common` on Kubernetes nodes.  
**Impact:** Pod stuck in `ContainerCreating`; kubelet can't mount the NFS volume.  
**Fix:** Install `nfs-common` (Ubuntu) or `nfs-utils` (RHEL) on every node before deploying NFS storage.

---

### 26.6 Forgetting That PVCs Are Not Deleted with StatefulSet

**Mistake:** Deleting a StatefulSet expecting all associated PVCs to be deleted.  
**Impact:** PVCs (and their underlying PVs) remain; waste storage; potentially confuse future deployments.  
**Fix:** Manually delete PVCs after deleting StatefulSet, or use `kubectl delete statefulset --cascade=foreground`.

---

### 26.7 Using hostPath in Production

**Mistake:** Using `hostPath` volumes for production persistent data.  
**Impact:** Pod tied to specific node; data lost if node is replaced; violates portability.  
**Fix:** Use proper PVCs with StorageClass. `hostPath` is only for development/testing.

---

## 27. Interview Questions — Beginner to Advanced {#27-interview-questions}

### Beginner Level

**Q1: What is the difference between a PersistentVolume (PV) and a PersistentVolumeClaim (PVC)?**

**A:** 
- **PersistentVolume (PV)**: A cluster-level storage resource provisioned by an administrator (or automatically via dynamic provisioning). It represents actual physical or virtual storage (NFS share, EBS volume, Ceph RBD image). It is cluster-scoped (not namespace-specific).
- **PersistentVolumeClaim (PVC)**: A namespace-scoped request for storage by a user or application. The PVC specifies storage requirements (size, access mode, storage class) without needing to know the underlying storage implementation.

The relationship is like a Pod to a Node: a PVC requests what it needs (10Gi, RWX), and Kubernetes matches it to a PV that satisfies those requirements. This abstraction separates the concern of providing storage (admin) from consuming storage (developer).

---

**Q2: What are the three main reclaimPolicy options in Kubernetes, and when would you use each?**

**A:**
- **Delete** (default for dynamically provisioned volumes): When a PVC is deleted, the PV and the underlying storage resource are automatically deleted. Use for development, staging, or any data that's easily recreatable.
- **Retain**: When a PVC is deleted, the PV moves to `Released` state but is NOT deleted. The data is preserved on the storage backend until an administrator manually reclaims it. Use for any production data, especially databases where accidental data loss is catastrophic.
- **Recycle** (deprecated): Performs a basic scrub (`rm -rf`) on the volume and makes it available again. Deprecated in Kubernetes 1.15; use dynamic provisioning with `Delete` policy instead.

---

**Q3: What is a StorageClass and why is it important?**

**A:** A StorageClass defines a "class" of storage with specific characteristics. It tells Kubernetes which provisioner to use (which CSI driver), what parameters to pass to that driver (disk type, encryption, IOPS), what reclaim policy to apply, and when to provision the volume (immediately or wait for pod scheduling).

StorageClasses enable **dynamic provisioning** — instead of admins manually creating PVs before users can create PVCs, the StorageClass provisioner automatically creates PVs on demand when a PVC is submitted. This enables self-service storage for developers while maintaining governance through StorageClass definitions.

---

### Intermediate Level

**Q4: What is the difference between `Immediate` and `WaitForFirstConsumer` volumeBindingMode, and when should you use each?**

**A:**
- **Immediate**: The PV is provisioned (and the zone is determined) as soon as the PVC is created, before any pod is scheduled. This can cause problems in multi-zone clusters because the volume may be created in a different zone than where the pod ends up being scheduled.
- **WaitForFirstConsumer**: The PV is NOT provisioned until a pod referencing the PVC is actually scheduled. The scheduler picks a node first (considering all constraints: affinity, resources, taints), and THEN the volume is provisioned in the topology (zone) of that node.

Use `WaitForFirstConsumer` for any zone-specific block storage (AWS EBS, GCP PD, Azure Disk) in multi-zone clusters. Use `Immediate` only for NFS, CephFS, or other storage that is accessible cluster-wide regardless of zone.

---

**Q5: How does Kubernetes handle the scenario where a pod with a ReadWriteOnce volume needs to be rescheduled to a different node?**

**A:** The process involves multiple steps and controllers:

1. Pod deleted from old node (or node goes NotReady)
2. AttachDetach controller detects that the pod no longer needs the volume on the old node
3. AttachDetach controller creates/updates a VolumeAttachment for detach
4. CSI external-attacher calls `ControllerUnpublishVolume` to detach the volume from the old node
5. Once detached, the VolumeAttachment shows `attached: false`
6. Meanwhile, Kubernetes schedules a new pod (replacement for old)
7. AttachDetach controller creates a new VolumeAttachment to attach volume to new node
8. CSI external-attacher calls `ControllerPublishVolume` (attach to new node)
9. kubelet on new node calls `NodeStageVolume` then `NodePublishVolume` (mount)
10. Pod starts successfully on the new node with the same volume

For cloud-provider volumes (EBS, GCP PD), this typically takes 1-2 minutes for the detach/attach cycle. For NFS, it's nearly instant because NFS isn't node-specific.

---

### Advanced Level

**Q6: Explain the complete dynamic provisioning lifecycle for a PVC requesting storage from an NFS StorageClass. Include all controllers and actors involved.**

**A:** Complete lifecycle:

1. **User** submits PVC YAML with `storageClassName: nfs-client`
2. **kube-apiserver** validates, stores PVC in etcd with `status.phase = Pending`
3. **NFS subdir provisioner** (an external provisioner): 
   - Its informer detects the new Pending PVC via watch
   - Verifies the StorageClass provisioner matches its name
   - Creates a subdirectory on the NFS server: `/exports/namespace-pvcname/`
   - Creates a PV object in Kubernetes via API server (sets provisioner annotation)
4. **PV Controller** (in kube-controller-manager):
   - Detects new Available PV and existing Pending PVC
   - Evaluates binding criteria: access modes, capacity, storageClass match
   - PATCHES both objects: PV.spec.claimRef = PVC, PVC.spec.volumeName = PV name
   - Sets PV.status.phase = Bound, PVC.status.phase = Bound
5. **User** creates Pod referencing the Bound PVC
6. **kube-scheduler** selects a node (for NFS, no topology restriction)
7. **kubelet** on selected node:
   - Detects pod is assigned to it
   - Reads pod spec, finds PVC reference
   - Looks up PV bound to PVC
   - Calls `NodeStageVolume` then `NodePublishVolume` (NFS: mounts via kernel NFS client)
   - Mount point created at pod's container filesystem path
8. **Pod** starts running, container accesses `/data` which is now the NFS volume

---

**Q7: A StatefulSet pod is rescheduled after its node fails. How does Kubernetes ensure the pod reconnects to its original data?**

**A:** StatefulSets provide two mechanisms that ensure data continuity:

1. **Stable pod identity**: StatefulSet pods are named `<statefulset>-<ordinal>` (e.g., `kafka-0`). When rescheduled, the pod gets the same name regardless of which node it runs on.

2. **PVC persistence**: The `volumeClaimTemplates` creates PVCs named `<template>-<pod-name>` (e.g., `data-kafka-0`). These PVCs are **never deleted** when the pod is evicted or rescheduled — only when the PVC itself is explicitly deleted.

When pod `kafka-0` is rescheduled after node failure:
- New `kafka-0` pod created (same name = same identity)
- Kubernetes looks for PVC named `data-kafka-0` (exactly the same PVC as before)
- PVC is still `Bound` to its PV
- AttachDetach controller detaches PV from failed node, attaches to new node
- kubelet mounts the PV on new node
- `kafka-0` starts with exactly the same data as before the failure

This is the fundamental difference between StatefulSets and Deployments: StatefulSets provide **stable storage identity** across rescheduling.

---

## 28. Cheat Sheet — Commands, Flags & Manifests {#28-cheat-sheet}

### 28.1 PV Commands

```bash
# View
kubectl get pv
kubectl get pv -o wide
kubectl get pv --sort-by=spec.capacity.storage
kubectl describe pv <n>

# Create
kubectl apply -f pv.yaml

# Check status
kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}: {.status.phase}{"\n"}{end}'

# Delete
kubectl delete pv <n>
# WARNING: Only delete after PVC is unbound; data may be lost with Delete policy
```

### 28.2 PVC Commands

```bash
# View
kubectl get pvc -n <namespace>
kubectl get pvc --all-namespaces
kubectl describe pvc <n> -n <namespace>

# Find pending PVCs
kubectl get pvc --all-namespaces | grep -v "Bound"

# Wait for PVC to bind
kubectl wait pvc/<n> --for=jsonpath='{.status.phase}'=Bound --timeout=120s

# Resize PVC (StorageClass must allow expansion)
kubectl patch pvc <n> -n <namespace> \
  -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'

# Find which pods use a PVC
kubectl get pods --all-namespaces -o json | \
  jq '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName=="my-pvc") | 
      {name: .metadata.name, ns: .metadata.namespace}'

# Delete
kubectl delete pvc <n> -n <namespace>
```

### 28.3 StorageClass Commands

```bash
# View
kubectl get storageclass
kubectl get sc
kubectl describe sc <n>

# Set default
kubectl patch sc <n> \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Remove default
kubectl patch sc <n> \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

### 28.4 Volume Attachment Commands

```bash
# View
kubectl get volumeattachments
kubectl describe volumeattachment <n>

# Check CSI drivers
kubectl get csidriver
kubectl get csinode

# Force delete stuck attachment (DANGEROUS: only when node truly dead)
kubectl delete volumeattachment <n>
```

### 28.5 Disk Usage and Volume Stats

```bash
# Disk usage inside a pod
kubectl exec <pod> -- df -h /mount-path
kubectl exec <pod> -- du -sh /mount-path/*

# Volume stats via Prometheus (kubelet)
# kubelet_volume_stats_available_bytes
# kubelet_volume_stats_capacity_bytes
# kubelet_volume_stats_used_bytes

# NFS specific checks
kubectl exec <pod> -- mount | grep nfs
kubectl exec <pod> -- cat /proc/mounts | grep nfs

# Check if NFS server is accessible
kubectl run nfs-debug --image=busybox --restart=Never \
  -- sh -c "nc -zv 192.168.1.100 2049 && echo SUCCESS"
kubectl logs nfs-debug
kubectl delete pod nfs-debug
```

### 28.6 Quick Templates

```yaml
# ── STORAGECLASS TEMPLATE ───────────────────────────────────────────────
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: my-class
provisioner: ebs.csi.aws.com    # or nfs, ceph, etc.
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

# ── PV TEMPLATE ─────────────────────────────────────────────────────────
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: my-class
  nfs:
    server: 192.168.1.100
    path: /srv/nfs/exports/my-pv

# ── PVC TEMPLATE ─────────────────────────────────────────────────────────
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: my-class
  resources:
    requests:
      storage: 10Gi

# ── POD WITH PVC TEMPLATE ─────────────────────────────────────────────────
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: app
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: my-pvc

# ── STATEFULSET WITH VOLUME CLAIM TEMPLATE ───────────────────────────────
apiVersion: apps/v1
kind: StatefulSet
spec:
  template:
    spec:
      containers:
        - name: app
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: my-class
        resources:
          requests:
            storage: 10Gi
```

### 28.7 Troubleshooting Quick Reference

```bash
# PVC pending
kubectl describe pvc <n>                          # Check events
kubectl get pv                                     # Check if PV available
kubectl get storageclass                           # Check SC exists
kubectl logs -n nfs-provisioner deploy/nfs-client-provisioner  # Dynamic provisioner

# Pod stuck ContainerCreating
kubectl describe pod <n> | grep Events -A 15       # Volume mount errors
kubectl get pvc <n>                                 # PVC bound?
kubectl get volumeattachment                        # Attachment status

# Volume full
kubectl exec <pod> -- df -h /data                  # Usage
kubectl get pvc <n> -o yaml | grep capacity        # Current size
kubectl patch pvc <n> -p '{"spec":{"resources":{"requests":{"storage":"NEW_SIZE"}}}}'

# Permission denied on NFS
kubectl exec <pod> -- id                           # Container user ID
# On NFS server: check /etc/exports for root_squash vs no_root_squash
```

---

## 29. Key Takeaways & Summary {#29-key-takeaways}

### The Storage Abstraction Ladder

```
ABSTRACTION MODEL (from highest to lowest):
─────────────────────────────────────────────────────────────────────

APP DEVELOPER sees:
  Pod → volumeMount at /data → PVC claim "give me 10Gi RWO"

PLATFORM ENGINEER configures:
  StorageClass → "use this provisioner with these parameters"
  PV (static) → "this NFS path is available"

INFRASTRUCTURE:
  NFS server, AWS EBS, GCP PD, Ceph, Longhorn...

KUBERNETES MANAGES:
  PV/PVC binding lifecycle → PV Controller
  Volume attach/detach → AttachDetach Controller
  Volume mount/unmount → kubelet
  Dynamic provisioning → External Provisioner (CSI sidecar)
─────────────────────────────────────────────────────────────────────
```

### The 10 Rules of Production Kubernetes Storage

1. **Use StorageClasses for dynamic provisioning** — manual PV creation doesn't scale
2. **`Retain` reclaimPolicy for all production databases** — never auto-delete critical data
3. **`WaitForFirstConsumer` for zone-specific storage** — prevents zone mismatch failures
4. **`ReadWriteOnce` for databases, `ReadWriteMany` for shared content** — use correct access modes
5. **StatefulSets for stateful apps** — provides stable storage identity across rescheduling
6. **Install NFS client on all nodes** — `nfs-common` must be present before NFS volumes work
7. **Controllers never talk to etcd directly** — all operations via kube-apiserver
8. **Monitor volume capacity proactively** — alert at 80%, act at 90%
9. **Test DR regularly** — snapshot, delete PVC, restore, verify data
10. **RBAC: developers create PVCs, admins create PVs** — storage governance separation

---

> **This guide covers Kubernetes v1.29+ storage management. Consult the official documentation at https://kubernetes.io/docs/concepts/storage/ for the most current information.**

---

*End of: Kubernetes Storage — Complete Production-Grade Guide*
