# Complete ETCD in Kubernetes: Beginner to Production Expert

## 1. What is etcd and Why Kubernetes Uses It

### What is etcd?

**etcd** is a distributed, reliable key-value store that is designed to store critical data for distributed systems. The name comes from "/etc" (Unix configuration directory) + "d" (distributed).

**Key Characteristics:**
- **Consistent**: Uses the Raft consensus algorithm to ensure all nodes agree on data
- **Distributed**: Can run as a cluster of multiple nodes
- **Reliable**: Fault-tolerant and highly available
- **Fast**: Optimized for reads and small writes
- **Secure**: Supports TLS, RBAC, and encryption



<img width="11889" height="1392" alt="deepseek_mermaid_20260202_0bb1ea" src="https://github.com/user-attachments/assets/455f5a46-788c-435e-aadf-91b421908086" />




### Why Kubernetes Uses etcd

Kubernetes needs a single source of truth for the entire cluster state. etcd serves as the "brain" of Kubernetes:

```
┌─────────────────────────────────────────┐
│         Kubernetes Cluster              │
│                                         │
│  ┌──────────┐    ┌──────────┐          │
│  │   API    │───▶│   etcd   │          │
│  │  Server  │◀───│ (Database)│         │
│  └──────────┘    └──────────┘          │
│       │                                 │
│       ▼                                 │
│  ┌──────────────────────┐               │
│  │ Scheduler, Controller│               │
│  │ Manager, etc.        │               │
│  └──────────────────────┘               │
└─────────────────────────────────────────┘
```

**What Kubernetes Stores in etcd:**
- Cluster configuration and state
- Pod definitions and current status
- Secrets and ConfigMaps
- Service endpoints
- Resource quotas and limits
- Network policies
- RBAC policies
- Custom Resource Definitions (CRDs)

**Why etcd specifically:**
1. **Strong consistency**: Every read returns the most recent write
2. **Watch capability**: Components can watch for changes in real-time
3. **Lease mechanism**: Supports TTL for temporary data
4. **Multi-version concurrency control (MVCC)**: Maintains history of changes
5. **Proven reliability**: Battle-tested in production environments

---

## 2. How etcd Works Internally

### The Raft Consensus Algorithm

etcd uses **Raft** to maintain consistency across cluster members. Raft is easier to understand than alternatives like Paxos.

**Core Concepts:**

```
┌─────────────────────────────────────────────┐
│           Raft Cluster (3 nodes)            │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │  Leader  │  │ Follower │  │ Follower │  │
│  │ (Node 1) │  │ (Node 2) │  │ (Node 3) │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│       │              │              │       │
│       └──────────────┴──────────────┘       │
│            Heartbeat & Log Replication      │
└─────────────────────────────────────────────┘
```

### Node States in Raft

1. **Leader**: Handles all client requests, sends heartbeats
2. **Follower**: Receives logs from leader, responds to requests
3. **Candidate**: Seeking votes to become leader

### Leader Election Process

```
Step 1: Follower timeout (no heartbeat from leader)
   Follower → Candidate

Step 2: Candidate requests votes
   Candidate → All nodes: "Vote for me (term X)"

Step 3: Nodes vote (max 1 vote per term)
   Nodes → Candidate: "Here's my vote"

Step 4: Majority wins
   If votes > cluster_size/2:
      Candidate → Leader
   Else:
      Candidate → Follower (retry later)
```

**Election Example (5-node cluster):**

```bash
# Initial state - Leader exists
Node1: Leader
Node2: Follower
Node3: Follower
Node4: Follower
Node5: Follower

# Leader fails
Node1: DOWN ❌
Node2: Follower (waiting for heartbeat...)
Node3: Follower (waiting for heartbeat...)
Node4: Follower (waiting for heartbeat...)
Node5: Follower (waiting for heartbeat...)

# Timeout occurs (typically 150-300ms)
Node2: Candidate (Term 5) "Vote for me!"
Node3: Follower → Votes for Node2
Node4: Follower → Votes for Node2
Node5: Follower → Votes for Node2

# Node2 receives 3 votes (majority of 5)
Node2: Candidate → LEADER (Term 5) ✓
```

### Quorum

**Quorum** = Minimum number of nodes needed to make decisions

```
Formula: (N/2) + 1
where N = total number of nodes

Examples:
- 3 nodes: quorum = 2 (can tolerate 1 failure)
- 5 nodes: quorum = 3 (can tolerate 2 failures)
- 7 nodes: quorum = 4 (can tolerate 3 failures)
```

**Important**: Always use odd numbers (3, 5, 7) to avoid split-brain scenarios.

### Write Process with Raft

```
1. Client writes to Leader
   Client → Leader: SET /key "value"

2. Leader appends to its log (uncommitted)
   Leader Log: [... , SET /key "value" (uncommitted)]

3. Leader replicates to Followers
   Leader → Followers: "Here's the new entry"

4. Followers append to their logs
   Follower Logs: [... , SET /key "value" (uncommitted)]

5. Followers acknowledge
   Followers → Leader: "ACK"

6. Leader receives majority ACKs
   Leader commits the entry

7. Leader responds to client
   Leader → Client: "Success"

8. Leader notifies Followers to commit
   Leader → Followers: "Commit entry at index X"
```

**Consistency Guarantee**: A write is only considered successful when a majority (quorum) has stored it.

### Read Process

etcd offers different consistency modes:

**1. Linearizable Read (Default - Strongest)**
```
Client → Leader: GET /key
Leader → Followers: "Am I still the leader?"
Followers → Leader: "Yes" (majority confirms)
Leader → Client: value
```

**2. Serializable Read (Faster, may be stale)**
```
Client → Any node: GET /key
Node → Client: value (from local state, no consensus check)
```

---

## 3. Kubernetes Control Plane Interaction with etcd

### Architecture Overview

```
┌───────────────────────────────────────────────────────┐
│                    Control Plane                       │
│                                                        │
│  ┌──────────────────────────────────────────────┐     │
│  │           kube-apiserver                     │     │
│  │  - Only component that talks to etcd         │     │
│  │  - REST API for all operations               │     │
│  │  - Watches and notifies on changes           │     │
│  └──────────────────┬───────────────────────────┘     │
│                     │                                  │
│                     │ gRPC                             │
│                     ▼                                  │
│  ┌──────────────────────────────────────────────┐     │
│  │                 etcd                         │     │
│  │  - Stores all cluster data                   │     │
│  │  - Provides consistency guarantees           │     │
│  └──────────────────────────────────────────────┘     │
│                                                        │
│  ┌──────────────┐  ┌─────────────┐  ┌──────────────┐ │
│  │  Scheduler   │  │ Controller  │  │  Cloud       │ │
│  │              │  │  Manager    │  │  Controller  │ │
│  └──────┬───────┘  └──────┬──────┘  └──────┬───────┘ │
│         │                 │                 │         │
│         └─────────────────┴─────────────────┘         │
│                All use kube-apiserver                 │
│                (never direct etcd access)             │
└───────────────────────────────────────────────────────┘
```

### Key Interactions

**1. Write Operations**

```bash
# Example: Creating a Pod
kubectl create -f pod.yaml

Flow:
1. kubectl → kube-apiserver: POST /api/v1/namespaces/default/pods
2. kube-apiserver validates request (RBAC, admission controllers)
3. kube-apiserver → etcd: Write to /registry/pods/default/my-pod
4. etcd confirms write (Raft consensus)
5. kube-apiserver → kubectl: "Pod created"
6. kube-apiserver notifies watchers (scheduler, kubelet)
```

**2. Watch Operations**

Components use watches to react to changes:

```go
// Pseudo-code of how scheduler watches
watcher := apiserver.Watch("/api/v1/pods")
for event := range watcher {
    if event.Type == "ADDED" && event.Object.Spec.NodeName == "" {
        // This is an unscheduled pod
        schedulePod(event.Object)
    }
}
```

**3. List Operations**

```bash
# kubectl get pods
kubectl → apiserver: GET /api/v1/namespaces/default/pods
apiserver → etcd: LIST /registry/pods/default/
etcd → apiserver: [pod1, pod2, pod3, ...]
apiserver → kubectl: Display table
```

### Data Storage Patterns in etcd

**Kubernetes uses hierarchical keys:**

```
/registry/
├── pods/
│   ├── default/
│   │   ├── nginx-pod
│   │   └── redis-pod
│   └── kube-system/
│       ├── coredns-...
│       └── kube-proxy-...
├── services/
│   └── default/
│       └── kubernetes
├── secrets/
│   └── default/
│       └── my-secret
├── configmaps/
├── deployments/
├── replicasets/
└── ...
```

**Example actual etcd key:**
```
/registry/pods/default/nginx-deployment-7fb96c846b-xjz2m
```

---

## 4. etcd Architecture in Single-Node and Multi-Node Clusters

### Single-Node etcd (Development/Testing Only)

```
┌─────────────────────────────┐
│    Kubernetes Cluster       │
│                             │
│  ┌──────────────┐           │
│  │ API Server   │           │
│  └──────┬───────┘           │
│         │                   │
│         ▼                   │
│  ┌──────────────┐           │
│  │  etcd (1)    │           │
│  │  No HA!      │           │
│  └──────────────┘           │
│                             │
└─────────────────────────────┘
```

**Characteristics:**
- **Simple** to set up
- **NO high availability** - single point of failure
- **NO fault tolerance**
- Used in: minikube, kind, development environments

**Setup (kubeadm example):**
```bash
# Single control plane node
sudo kubeadm init

# etcd runs as a static pod
ls /etc/kubernetes/manifests/etcd.yaml
```

### Multi-Node etcd Cluster (Production)

#### Stacked etcd Topology (Most Common)

etcd runs on the same nodes as other control plane components:

```
┌────────────────────────────────────────────────────────┐
│              Production Cluster                         │
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  Master 1   │  │  Master 2   │  │  Master 3   │    │
│  │             │  │             │  │             │    │
│  │ API Server  │  │ API Server  │  │ API Server  │    │
│  │ Scheduler   │  │ Scheduler   │  │ Scheduler   │    │
│  │ Controller  │  │ Controller  │  │ Controller  │    │
│  │   Manager   │  │   Manager   │  │   Manager   │    │
│  │             │  │             │  │             │    │
│  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │    │
│  │ │  etcd   │ │  │ │  etcd   │ │  │ │  etcd   │ │    │
│  │ │ (Leader)│ │  │ │(Follower│ │  │ │(Follower│ │    │
│  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │    │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘    │
│         └─────────────────┴─────────────────┘          │
│                etcd cluster communication              │
└────────────────────────────────────────────────────────┘
```

**Pros:**
- Fewer servers needed
- Simpler to manage
- Lower cost

**Cons:**
- Control plane failure = etcd failure
- Resource contention possible

**Setup:**
```bash
# Initialize first control plane node
sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:6443" \
  --upload-certs

# Join additional control plane nodes
sudo kubeadm join LOAD_BALANCER_DNS:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <key>
```

#### External etcd Topology (Maximum HA)

etcd runs on dedicated nodes:

```
┌────────────────────────────────────────────────────────┐
│                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  Master 1   │  │  Master 2   │  │  Master 3   │    │
│  │ API Server  │  │ API Server  │  │ API Server  │    │
│  │ Scheduler   │  │ Scheduler   │  │ Scheduler   │    │
│  │ Controller  │  │ Controller  │  │ Controller  │    │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘    │
│         └─────────────────┴─────────────────┘          │
│                          │                             │
│                          ▼                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │   etcd 1    │  │   etcd 2    │  │   etcd 3    │    │
│  │  (Leader)   │  │ (Follower)  │  │ (Follower)  │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
│         Dedicated etcd cluster                         │
└────────────────────────────────────────────────────────┘
```

**Pros:**
- Maximum resilience
- etcd can survive control plane failures
- Better resource isolation
- Can scale etcd independently

**Cons:**
- More servers required (6+ for full HA)
- More complex setup and management

### Cluster Size Recommendations

```
┌──────────┬─────────┬──────────────┬─────────────┐
│  Nodes   │ Quorum  │   Failures   │  Use Case   │
│          │         │   Tolerated  │             │
├──────────┼─────────┼──────────────┼─────────────┤
│    1     │    1    │      0       │  Dev only   │
│    3     │    2    │      1       │  Small prod │
│    5     │    3    │      2       │  Production │
│    7     │    4    │      3       │  Large prod │
└──────────┴─────────┴──────────────┴─────────────┘
```

**Why not more than 7?**
- Write latency increases with cluster size
- More network traffic for consensus
- Diminishing returns on availability
- 7 nodes can tolerate 3 failures (99.99%+ availability)

---

## 5. Data Model (Keys, Values, Prefixes, Leases, TTL)

### Keys and Values

etcd stores data as **key-value pairs** with byte array keys and values.

**Key Characteristics:**
- Keys are byte strings (typically UTF-8)
- Values are byte strings (any binary data)
- Keys are sorted lexicographically
- Maximum key size: typically 1.5 MB
- Maximum value size: default 1.5 MB (configurable)

**Example:**
```bash
# Set a key-value pair
etcdctl put /app/config/database "postgresql://host:5432/db"

# Get the value
etcdctl get /app/config/database
# Output:
# /app/config/database
# postgresql://host:5432/db
```

### Hierarchical Keys (Prefixes)

While etcd doesn't have "directories," you can simulate hierarchy with `/`:

```
/production/
  /production/web/
    /production/web/replicas = "3"
    /production/web/image = "nginx:1.21"
  /production/database/
    /production/database/host = "postgres.local"
    /production/database/port = "5432"

/staging/
  /staging/web/
    /staging/web/replicas = "1"
```

**List all keys with a prefix:**
```bash
etcdctl get /production/ --prefix

# Output:
# /production/database/host
# postgres.local
# /production/database/port
# 5432
# /production/web/image
# nginx:1.21
# /production/web/replicas
# 3
```

### Revisions and Versioning (MVCC)

etcd uses **Multi-Version Concurrency Control**:

```
Timeline of changes to /config/replicas:

Revision 1: /config/replicas = "1"  (Create)
Revision 2: /config/replicas = "2"  (Update)
Revision 3: /config/replicas = "3"  (Update)
Revision 5: /config/replicas deleted (Delete)
           ↑ (Revision 4 was another key)

Current state: key doesn't exist
History: preserved until compaction
```

**Working with revisions:**

```bash
# Get current value
etcdctl get /config/replicas

# Get value at specific revision
etcdctl get /config/replicas --rev=2
# Output: /config/replicas = "2"

# Get all changes since revision 100
etcdctl get /config/ --prefix --rev=100

# Watch from a specific revision
etcdctl watch /config/replicas --rev=1
# Shows all historical changes from revision 1 to now
```

### Leases and TTL

**Leases** provide automatic key expiration:

```
┌──────────────────────────────────────┐
│  Create Lease (TTL: 60s)             │
│  Lease ID: 123456789                 │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  Attach keys to lease:               │
│  /session/node1 → Lease 123456789    │
│  /lock/resource → Lease 123456789    │
└──────────────────┬───────────────────┘
                   │
                   ▼
┌──────────────────────────────────────┐
│  Keep-alive (refresh TTL)            │
│  Every 20s: "I'm still alive!"       │
└──────────────────┬───────────────────┘
                   │
                   ▼
        ┌──────────┴──────────┐
        │                     │
        ▼                     ▼
  Keep-alive OK       Keep-alive FAILS
  (lease renewed)     (after 60s, keys deleted)
```

**Practical example - Session management:**

```bash
# Grant a lease with 30 second TTL
etcdctl lease grant 30
# Output: lease 694d7f6d6f6d7f6e granted with TTL(30s)

LEASE_ID=694d7f6d6f6d7f6e

# Attach a key to the lease
etcdctl put /sessions/user123 "active" --lease=$LEASE_ID

# Keep the lease alive
etcdctl lease keep-alive $LEASE_ID
# This will continuously refresh the lease

# In another terminal, watch the key
etcdctl watch /sessions/user123
# If keep-alive stops, after 30s you'll see: DELETE

# Revoke lease manually
etcdctl lease revoke $LEASE_ID
# All keys attached to this lease are immediately deleted
```

**Use cases:**
1. **Service Discovery**: Register service, auto-deregister on crash
2. **Leader Election**: Leader holds lease, others watch for expiration
3. **Distributed Locks**: Lock holder must keep-alive
4. **Ephemeral Data**: Temporary configuration or state

### Transactions

etcd supports atomic multi-key operations:

```bash
# Syntax:
# etcdctl txn [options]
# Then enter:
# - Comparison conditions
# - Success operations (if conditions true)
# - Failure operations (if conditions false)

# Example: Atomic increment with compare-and-swap
etcdctl txn <<EOF
compare:
value(/counter) = "5"

success:
put /counter "6"

failure:
get /counter
EOF
```

**Real-world example - Atomic configuration update:**

```bash
# Update database connection only if current version matches
etcdctl txn <<EOF
compare:
value(/config/version) = "v1"

success:
put /config/database "new-postgres-host:5432"
put /config/version "v2"

failure:
get /config/version
EOF
```

---

## 6. Consistency, Durability, and Failure Handling

### Consistency Guarantees

etcd provides **linearizability** - the strongest consistency model:

```
What linearizability means:
┌──────────────────────────────────────┐
│ Time ─────────────────────────────▶  │
│                                      │
│ Write(x=1) ──┐                       │
│              │                       │
│              └──[Committed]          │
│                     │                │
│                     ▼                │
│ Read(x) ────────────────────▶ x=1   │
│                                      │
│ Every read after commit sees x=1    │
│ Reads never return stale data       │
└──────────────────────────────────────┘
```

**CAP Theorem and etcd:**

etcd chooses **CP** (Consistency + Partition tolerance):

```
C (Consistency) ✓ - Always consistent
A (Availability) ✗ - May reject requests during partition
P (Partition tolerance) ✓ - Continues with majority
```

During a network partition:

```
┌──────────────────────────────────────────────┐
│  Cluster split: 2 nodes | 3 nodes            │
│                                              │
│  ┌────────┐  ┌────────┐ │ ┌────────┐        │
│  │ Node 1 │  │ Node 2 │ │ │ Node 3 │        │
│  │(Leader)│  │        │ │ │        │        │
│  └────────┘  └────────┘ │ └────────┘        │
│                         │ ┌────────┐        │
│                         │ │ Node 4 │        │
│  Minority side:         │ │        │        │
│  - Cannot elect leader  │ └────────┘        │
│  - Rejects writes ✗     │ ┌────────┐        │
│  - Read-only mode       │ │ Node 5 │        │
│                         │ │        │        │
│                         │ └────────┘        │
│                         │                   │
│                         │ Majority side:    │
│                         │ - Elects new      │
│                         │   leader          │
│                         │ - Accepts writes✓ │
└──────────────────────────────────────────────┘
```

### Durability

**Write-Ahead Log (WAL):**

Every change is first written to disk before being applied:

```
1. Client request arrives
   ↓
2. Leader writes to WAL on disk
   (fsync ensures disk commit)
   ↓
3. Leader replicates to followers
   ↓
4. Followers write to their WALs
   ↓
5. Majority confirms → Committed
   ↓
6. Applied to state machine
   ↓
7. Response to client
```

**WAL Location:**
```bash
# Typically stored at:
/var/lib/etcd/member/wal/
  ├── 0000000000000000-0000000000000000.wal
  └── ...

# Each WAL file is a segment of operations
```

**Snapshots:**

Periodically, etcd creates snapshots to compact WAL:

```
WAL: [op1, op2, op3, ... op10000] (growing)
         ↓
   Create snapshot at op10000
         ↓
Snapshot: Complete state at op10000
WAL: [op10001, op10002, ...] (can discard old WAL)
```

### Failure Handling Scenarios

#### Scenario 1: Single Node Failure (5-node cluster)

```
Before:
  Node1(L), Node2(F), Node3(F), Node4(F), Node5(F)
  Quorum: 3

After Node3 fails:
  Node1(L), Node2(F), Node3(✗), Node4(F), Node5(F)
  Remaining: 4 nodes
  Quorum: Still 3 (>2.5)
  Status: ✓ HEALTHY - cluster continues normally
```

#### Scenario 2: Leader Failure

```
Time 0: Node1(Leader) crashes
  Node1(✗), Node2(F), Node3(F), Node4(F), Node5(F)

Time 0-300ms: Election timeout
  Nodes wait for heartbeat from leader

Time 300ms: Node2 becomes candidate
  Node2(C) starts election (Term 5)
  Requests votes from Node3, Node4, Node5

Time 350ms: Election completes
  Node2 receives majority votes
  Node2(Leader), Node3(F), Node4(F), Node5(F)

Time 351ms: Normal operation resumes
  New leader starts sending heartbeats
  Total downtime: ~350ms
```

#### Scenario 3: Network Partition (Split-Brain Prevention)

```
Initial: 5 nodes, all connected
  Node1(L), Node2(F), Node3(F), Node4(F), Node5(F)

Partition occurs:
  Group A: Node1, Node2 (2 nodes - minority)
  Group B: Node3, Node4, Node5 (3 nodes - majority)

Group A:
  - Node1 detects it can't reach majority
  - Node1 steps down from leader role
  - Group A becomes read-only
  - Writes REJECTED ✗

Group B:
  - Has quorum (3 > 2.5)
  - Elects new leader (e.g., Node3)
  - Accepts writes ✓
  - Continues normal operation

Result: No split-brain! Only one group can write.
```

#### Scenario 4: Majority Failure (Disaster)

```
Initial: 5 nodes
  Node1(L), Node2(F), Node3(F), Node4(F), Node5(F)

Disaster: 3 nodes fail
  Node1(✗), Node2(✗), Node3(✗), Node4(F), Node5(F)

Result:
  Remaining: 2 nodes
  Quorum needed: 3
  Status: ✗ CLUSTER DOWN
  
Behavior:
  - No leader can be elected
  - All writes REJECTED
  - Reads may be stale
  - Cluster is unavailable

Recovery:
  - Restore at least one failed node, OR
  - Force new cluster from snapshot (data loss risk)
```

### Data Loss Prevention

**Scenario: What if leader crashes after acknowledging write but before followers commit?**

```
1. Client writes to Leader
2. Leader writes to WAL and local state
3. Leader responds to client: "Success" ✓
4. Leader crashes before replicating ✗

Result: IMPOSSIBLE in etcd!

Why? Leader only responds after quorum confirms:

Actual flow:
1. Client → Leader: Write
2. Leader → Followers: Replicate
3. Followers → Leader: ACK (quorum reached)
4. Leader commits
5. Leader → Client: "Success" ✓

If leader crashes at step 4, the new leader will have the data from the followers who confirmed.
```

---

## 7. Performance and Tuning

### Performance Characteristics

**Typical Benchmarks (SSD, good network):**

```
┌─────────────────┬──────────────┬─────────────┐
│   Operation     │   Latency    │  Throughput │
├─────────────────┼──────────────┼─────────────┤
│  Write (small)  │   < 10ms     │  10K ops/s  │
│  Read (local)   │   < 1ms      │  100K ops/s │
│  Read (quorum)  │   < 10ms     │  10K ops/s  │
│  Watch          │   Real-time  │  Millions   │
└─────────────────┴──────────────┴─────────────┘
```

**Factors affecting performance:**
- Network latency between nodes
- Disk I/O (WAL writes)
- CPU (for encryption, compression)
- Memory (for caching)
- Cluster size (more nodes = more replication)

### Key Configuration Parameters

**1. Heartbeat and Election Timeouts**

```yaml
# /etc/kubernetes/manifests/etcd.yaml
spec:
  containers:
  - command:
    - etcd
    - --heartbeat-interval=100        # Leader sends heartbeat every 100ms
    - --election-timeout=1000         # Follower waits 1000ms before election
```

**Rule of thumb:**
```
election-timeout >= 10 × heartbeat-interval

Default: heartbeat=100ms, election=1000ms
Low latency network: heartbeat=50ms, election=500ms
High latency/WAN: heartbeat=500ms, election=5000ms
```

**Impact:**
- **Lower timeouts**: Faster failure detection, but more CPU and false positives
- **Higher timeouts**: More stable, but slower failure recovery

**2. Snapshot Settings**

```yaml
- --snapshot-count=10000              # Snapshot after 10K operations

# Lower value (5000):
  - More frequent snapshots
  - Faster recovery
  - More disk I/O

# Higher value (100000):
  - Less disk I/O
  - Slower recovery (more WAL to replay)
  - Larger WAL files
```

**3. Quota and Limits**

```yaml
- --quota-backend-bytes=8589934592    # 8GB database size limit

# When quota exceeded:
# - etcd enters maintenance mode
# - Rejects writes
# - Must compact and defragment
```

**Check current size:**
```bash
etcdctl endpoint status --write-out=table
# Look at DB SIZE column

# If approaching quota:
# 1. Compact old revisions
etcdctl compact $(etcdctl endpoint status --write-out="json" | jq -r '.[0].Status.header.revision')

# 2. Defragment
etcdctl defrag

# 3. Check size again
etcdctl endpoint status --write-out=table
```

**4. Performance Tuning Flags**

```yaml
# Improve write performance
- --max-request-bytes=1572864         # 1.5MB max request size
- --max-txn-ops=128                   # Max operations per transaction

# Network tuning
- --peer-transport-security=true      # Use TLS (slight overhead)
- --auto-compaction-mode=periodic
- --auto-compaction-retention=1h      # Auto-compact every hour
```

### Kubernetes-Specific Tuning

**For large clusters (>1000 nodes):**

```yaml
# In kube-apiserver configuration
- --etcd-compaction-interval=5m       # Compact more frequently
- --etcd-count-metric-poll-period=1m  # Monitor etcd size
```

**Watch optimization:**
```yaml
# Reduce watch cache impact
- --watch-cache-sizes=pods=1000,nodes=500
```

### Storage Performance

**Disk Requirements:**

```
┌─────────────────┬─────────────┬───────────────┐
│   Disk Type     │  IOPS       │  Recommended  │
├─────────────────┼─────────────┼───────────────┤
│  HDD (7200rpm)  │  ~100       │  ✗ Never      │
│  SATA SSD       │  ~500       │  ⚠ Small only │
│  NVMe SSD       │  10,000+    │  ✓ Ideal      │
│  EBS gp3 (AWS)  │  3,000-16K  │  ✓ Good       │
└─────────────────┴─────────────┴───────────────┘
```

**Test disk performance:**

```bash
# Write test
sudo fio --name=write_iops --directory=/var/lib/etcd \
  --size=4G --time_based --runtime=60s --ramp_time=2s \
  --ioengine=libaio --direct=1 --verify=0 --bs=4K \
  --iodepth=64 --rw=randwrite --group_reporting=1

# Look for IOPS > 3000 for production
```

**Mount options for etcd disk:**

```bash
# /etc/fstab
/dev/nvme1n1  /var/lib/etcd  xfs  noatime,nodiratime  0 2

# noatime: Don't update access time (performance)
# nodiratime: Don't update directory access time
```

### Memory Tuning

**Memory usage pattern:**

```
etcd memory = WAL + Snapshot + Watch cache + MVCC

Typical breakdown:
- WAL buffer: 100-500 MB
- Snapshot cache: Based on snapshot-count
- Watch cache: Based on active watchers (K8s has many!)
- MVCC store: Based on database size
```

**For Kubernetes:**

```bash
# Minimum: 2-4 GB
# Recommended for 100 nodes: 8 GB
# Recommended for 1000+ nodes: 16-32 GB
```

### Network Optimization

**Network requirements:**

```
Latency: < 10ms RTT between etcd nodes (critical!)
Bandwidth: 1 Gbps minimum, 10 Gbps for large clusters
```

**Check network latency:**

```bash
# Between etcd nodes
ping -c 100 etcd-node-2

# Acceptable: < 10ms average
# Warning: 10-50ms
# Problem: > 50ms (consider different deployment)
```

**Monitoring network:**

```bash
# Watch etcd metrics
curl http://localhost:2379/metrics | grep network_peer

# Look for:
# - etcd_network_peer_round_trip_time_seconds
#   (should be < 0.01, i.e., 10ms)
```

### Benchmarking etcd

```bash
# Write performance test
benchmark --endpoints=https://127.0.0.1:2379 \
  --conns=1 --clients=1 \
  put --key-size=8 --sequential-keys \
  --total=10000 --val-size=256

# Read performance test
benchmark --endpoints=https://127.0.0.1:2379 \
  --conns=1 --clients=1 \
  range key --total=10000

# Watch performance test  
benchmark --endpoints=https://127.0.0.1:2379 \
  --conns=1 --clients=1 \
  watch --total=10000
```

**Interpreting results:**

```
Good performance indicators:
- Write latency: < 10ms (p99)
- Read latency: < 1ms (p99)
- Throughput: > 5000 writes/sec

Warning signs:
- Write latency: > 50ms
- Many leader changes
- Frequent timeout errors
```

---

## 8. Security (TLS, Auth, Encryption at Rest)

### TLS (Transport Layer Security)

etcd supports TLS for **two** types of communication:

```
┌────────────────────────────────────────────────────┐
│                                                    │
│  1. Client-to-Server (API access)                 │
│     kube-apiserver ──TLS──▶ etcd                   │
│                                                    │
│  2. Peer-to-Peer (cluster replication)            │
│     etcd-1 ◀──TLS──▶ etcd-2 ◀──TLS──▶ etcd-3     │
│                                                    │
└────────────────────────────────────────────────────┘
```

#### Setting up TLS

**Step 1: Generate Certificates**

```bash
# Using cfssl (recommended) or openssl

# CA certificate
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "etcd": {
        "usages": ["signing", "key encipherment", "server auth", "client auth"],
        "expiry": "8760h"
      }
    }
  }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "etcd-ca",
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

# Server certificate (for each etcd node)
cat > server-csr.json <<EOF
{
  "CN": "etcd-server",
  "hosts": [
    "127.0.0.1",
    "etcd1.example.com",
    "etcd2.example.com",
    "etcd3.example.com",
    "10.0.0.1",
    "10.0.0.2",
    "10.0.0.3"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=etcd \
  server-csr.json | cfssljson -bare server

# Peer certificate (for etcd-to-etcd communication)
cat > peer-csr.json <<EOF
{
  "CN": "etcd-peer",
  "hosts": [
    "etcd1.example.com",
    "etcd2.example.com",
    "etcd3.example.com",
    "10.0.0.1",
    "10.0.0.2",
    "10.0.0.3"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=etcd \
  peer-csr.json | cfssljson -bare peer

# Client certificate (for kube-apiserver)
cat > client-csr.json <<EOF
{
  "CN": "etcd-client",
  "key": {
    "algo": "rsa",
    "size": 2048
  }
}
EOF

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=etcd \
  client-csr.json | cfssljson -bare client
```

**Step 2: Configure etcd with TLS**

```yaml
# /etc/kubernetes/manifests/etcd.yaml
spec:
  containers:
  - command:
    - etcd
    # Client TLS
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --client-cert-auth=true
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    
    # Peer TLS
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-client-cert-auth=true
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    
    # Listen addresses (HTTPS)
    - --listen-client-urls=https://0.0.0.0:2379
    - --listen-peer-urls=https://0.0.0.0:2380
    - --advertise-client-urls=https://10.0.0.1:2379
    - --initial-advertise-peer-urls=https://10.0.0.1:2380
    
    volumeMounts:
    - mountPath: /etc/kubernetes/pki/etcd
      name: etcd-certs
      readOnly: true
```

**Step 3: Configure kube-apiserver to use TLS**

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --etcd-servers=https://10.0.0.1:2379,https://10.0.0.2:2379,https://10.0.0.3:2379
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/etcd/client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/etcd/client.key
```

**Step 4: Use etcdctl with TLS**

```bash
# Set environment variables
export ETCDCTL_API=3
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/client.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/client.key
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379

# Now etcdctl commands work
etcdctl member list
etcdctl endpoint health
```

### Authentication (RBAC)

etcd supports role-based access control:

```
┌──────────────────────────────────────┐
│     etcd Authentication Model        │
│                                      │
│  Users ──belongs to──▶ Roles         │
│                         │            │
│                         ▼            │
│                    Permissions       │
│                   (Read/Write/...)   │
│                         │            │
│                         ▼            │
│                    Resources         │
│                    (Key ranges)      │
└──────────────────────────────────────┘
```

**Enabling authentication:**

```bash
# 1. Create root user
etcdctl user add root
# Enter password when prompted

# 2. Create a role with specific permissions
etcdctl role add app-readonly

# Grant read permission on /app/ prefix
etcdctl role grant-permission app-readonly read /app/ --prefix=true

# 3. Create user and assign role
etcdctl user add app-user
etcdctl user grant-role app-user app-readonly

# 4. Enable authentication
etcdctl auth enable

# Now authentication is required!
```

**Using authenticated access:**

```bash
# Method 1: With flags
etcdctl --user=app-user:password get /app/config

# Method 2: With environment variables
export ETCDCTL_USER=app-user:password
etcdctl get /app/config

# Root user can do everything
etcdctl --user=root:rootpassword user list
```

**Common roles setup:**

```bash
# Read-only role for monitoring
etcdctl role add readonly
etcdctl role grant-permission readonly read / --prefix=true

# Admin role (everything except user management)
etcdctl role add admin
etcdctl role grant-permission admin readwrite / --prefix=true

# Application-specific role
etcdctl role add app-service
etcdctl role grant-permission app-service readwrite /app/service1/ --prefix=true
etcdctl role grant-permission app-service read /app/shared/ --prefix=true
```

### Encryption at Rest

Kubernetes can encrypt data before storing it in etcd:

```
┌──────────────────────────────────────────────────┐
│   Without Encryption at Rest                     │
│                                                  │
│   Secret "my-password" = "plain123"              │
│         │                                        │
│         ▼                                        │
│   kube-apiserver                                 │
│         │                                        │
│         ▼                                        │
│   etcd stores: "plain123" ◀── VISIBLE IN DISK!  │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│   With Encryption at Rest                        │
│                                                  │
│   Secret "my-password" = "plain123"              │
│         │                                        │
│         ▼                                        │
│   kube-apiserver encrypts with key               │
│         │                                        │
│         ▼                                        │
│   etcd stores: "k8s:enc:aescbc:v1:key1:Xf7g..." │
│                     └── ENCRYPTED! ✓             │
└──────────────────────────────────────────────────┘
```

**Step 1: Create encryption configuration**

```bash
# Generate a random encryption key
head -c 32 /dev/urandom | base64

# Create encryption config file
cat > /etc/kubernetes/pki/encryption-config.yaml <<EOF
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
          secret: <BASE64_KEY_FROM_ABOVE>
    - identity: {}  # Fallback for reading old unencrypted data
EOF
```

**Step 2: Configure kube-apiserver**

```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --encryption-provider-config=/etc/kubernetes/pki/encryption-config.yaml
    
    volumeMounts:
    - mountPath: /etc/kubernetes/pki/encryption-config.yaml
      name: encryption-config
      readOnly: true
  
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/encryption-config.yaml
      type: FileOrCreate
    name: encryption-config
```

**Step 3: Encrypt existing data**

```bash
# Encryption only applies to NEW writes
# To encrypt existing secrets:
kubectl get secrets --all-namespaces -o json | kubectl replace -f -

# Verify encryption
ETCDCTL_API=3 etcdctl get /registry/secrets/default/my-secret | hexdump -C
# Should see "k8s:enc:aescbc:v1:..." instead of plain text
```

**Multiple encryption keys (key rotation):**

```yaml
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key2
          secret: <NEW_KEY>  # Used for new writes
        - name: key1
          secret: <OLD_KEY>  # Used for reading old data
    - identity: {}
```

**Rotating encryption keys:**

```bash
# 1. Add new key as first entry in config
# 2. Restart kube-apiserver
# 3. Re-encrypt all secrets
kubectl get secrets --all-namespaces -o json | kubectl replace -f -
# 4. Remove old key from config
# 5. Restart kube-apiserver again
```

### Security Best Practices Summary

```
✓ Always use TLS for client and peer communication
✓ Use client certificate authentication
✓ Enable etcd RBAC for fine-grained access
✓ Use encryption at rest for sensitive data
✓ Restrict network access (firewall rules)
✓ Keep certificates rotated (before expiry)
✓ Limit physical access to etcd nodes
✓ Monitor access logs
✓ Use separate etcd cluster for multi-tenancy
✓ Regular security audits
```

---

## 9. Backup and Restore (etcdctl snapshots)

### Why Backups Are Critical

etcd is the **single source of truth** for your entire Kubernetes cluster:

```
If etcd is lost:
❌ All pod definitions gone
❌ All services gone
❌ All secrets gone
❌ All configuration gone
❌ Cluster is completely unrecoverable without backup
```

### Creating Snapshots

**Basic snapshot:**

```bash
# Set up environment
export ETCDCTL_API=3
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379

# Create snapshot
etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db

# Output:
# Snapshot saved at /backup/etcd-snapshot-20240215-143022.db
```

**Verify snapshot integrity:**

```bash
etcdctl snapshot status /backup/etcd-snapshot-20240215-143022.db \
  --write-out=table

# Output:
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | a1b2c3d4 |   123456 |       1543 | 4.2 MB     |
# +----------+----------+------------+------------+
```

### Automated Backup Script

**Production-ready backup script:**

```bash
#!/bin/bash
# /usr/local/bin/etcd-backup.sh

set -euo pipefail

# Configuration
BACKUP_DIR="/var/backups/etcd"
RETENTION_DAYS=7
ENDPOINTS="https://127.0.0.1:2379"
ETCDCTL_API=3

# TLS certificates
CACERT="/etc/kubernetes/pki/etcd/ca.crt"
CERT="/etc/kubernetes/pki/etcd/server.crt"
KEY="/etc/kubernetes/pki/etcd/server.key"

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Timestamp
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="$BACKUP_DIR/etcd-snapshot-$TIMESTAMP.db"

# Create snapshot
echo "Creating etcd snapshot..."
etcdctl \
  --endpoints="$ENDPOINTS" \
  --cacert="$CACERT" \
  --cert="$CERT" \
  --key="$KEY" \
  snapshot save "$BACKUP_FILE"

# Verify snapshot
echo "Verifying snapshot..."
etcdctl \
  --endpoints="$ENDPOINTS" \
  snapshot status "$BACKUP_FILE" \
  --write-out=table

# Compress snapshot
echo "Compressing snapshot..."
gzip "$BACKUP_FILE"

# Delete old backups
echo "Cleaning up old backups (older than $RETENTION_DAYS days)..."
find "$BACKUP_DIR" -name "etcd-snapshot-*.db.gz" -mtime +$RETENTION_DAYS -delete

# Upload to remote storage (optional)
# aws s3 cp "$BACKUP_FILE.gz" s3://my-etcd-backups/
# OR
# gsutil cp "$BACKUP_FILE.gz" gs://my-etcd-backups/

echo "Backup completed: $BACKUP_FILE.gz"

# Send notification (optional)
# curl -X POST https://hooks.slack.com/... -d "{\"text\":\"etcd backup completed\"}"
```

**Set up cron job:**

```bash
# Make script executable
chmod +x /usr/local/bin/etcd-backup.sh

# Add to crontab (backup every 6 hours)
sudo crontab -e

# Add this line:
0 */6 * * * /usr/local/bin/etcd-backup.sh >> /var/log/etcd-backup.log 2>&1
```

### Restoring from Snapshot

#### Scenario 1: Single-Node Cluster Restore

```bash
# 1. Stop kube-apiserver and etcd
sudo systemctl stop kube-apiserver
sudo systemctl stop etcd
# OR if using static pods:
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# 2. Remove existing etcd data
sudo rm -rf /var/lib/etcd

# 3. Restore from snapshot
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd \
  --name=master-1 \
  --initial-cluster=master-1=https://10.0.0.1:2380 \
  --initial-advertise-peer-urls=https://10.0.0.1:2380

# 4. Set correct permissions
sudo chown -R etcd:etcd /var/lib/etcd

# 5. Start etcd and kube-apiserver
sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
# OR
# sudo systemctl start etcd
# sudo systemctl start kube-apiserver

# 6. Verify cluster
kubectl get nodes
kubectl get pods --all-namespaces
```

#### Scenario 2: Multi-Node Cluster Restore

**More complex - must restore ALL nodes:**

```bash
# On EACH etcd node (master-1, master-2, master-3):

# 1. Stop etcd
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# 2. Remove old data
sudo rm -rf /var/lib/etcd

# 3. Restore with correct member info
# On master-1:
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd \
  --name=master-1 \
  --initial-cluster=master-1=https://10.0.0.1:2380,master-2=https://10.0.0.2:2380,master-3=https://10.0.0.3:2380 \
  --initial-advertise-peer-urls=https://10.0.0.1:2380 \
  --initial-cluster-token=etcd-cluster-restore

# On master-2:
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd \
  --name=master-2 \
  --initial-cluster=master-1=https://10.0.0.1:2380,master-2=https://10.0.0.2:2380,master-3=https://10.0.0.3:2380 \
  --initial-advertise-peer-urls=https://10.0.0.2:2380 \
  --initial-cluster-token=etcd-cluster-restore

# On master-3:
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd \
  --name=master-3 \
  --initial-cluster=master-1=https://10.0.0.1:2380,master-2=https://10.0.0.2:2380,master-3=https://10.0.0.3:2380 \
  --initial-advertise-peer-urls=https://10.0.0.3:2380 \
  --initial-cluster-token=etcd-cluster-restore

# 4. Fix permissions on all nodes
sudo chown -R etcd:etcd /var/lib/etcd

# 5. Start etcd on all nodes (one by one)
sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/

# 6. Verify cluster health
etcdctl endpoint health --cluster
etcdctl member list
```

### Backup Best Practices

**1. Frequency:**
```
Development: Daily
Production: Every 6 hours or more frequently
Critical Production: Every 1-2 hours + before changes
```

**2. Retention:**
```
Keep at least:
- Last 7 days of daily backups
- Last 4 weeks of weekly backups
- Last 12 months of monthly backups
```

**3. Storage locations:**
```
✓ Local disk (fast restore)
✓ Remote storage (S3, GCS, Azure Blob)
✓ Different region (disaster recovery)
✓ Encrypted backups
```

**4. Testing:**
```bash
# Test restore regularly (monthly)
# Use a separate test cluster

# Automated test restore script
#!/bin/bash
LATEST_BACKUP=$(ls -t /backup/etcd-*.db.gz | head -1)
gunzip -c "$LATEST_BACKUP" > /tmp/test-restore.db

etcdctl snapshot restore /tmp/test-restore.db \
  --data-dir=/tmp/etcd-test

# Verify it contains data
etcdctl --data-dir=/tmp/etcd-test member list

# Cleanup
rm -rf /tmp/etcd-test /tmp/test-restore.db
```

### What Gets Backed Up

```
Snapshot includes:
✓ All key-value data
✓ Current cluster state
✓ Revision history (within retention)
✓ Cluster membership info

Snapshot does NOT include:
✗ etcd configuration files
✗ TLS certificates
✗ WAL files (not needed if using snapshot)
```

---

## 10. Disaster Recovery Scenarios

### Scenario 1: Single etcd Member Failure (Easy)

**Problem:**
```
Cluster: 3 nodes
Node1: ✓ Healthy
Node2: ✗ Failed (disk crash)
Node3: ✓ Healthy
```

**Solution:**

```bash
# 1. Remove failed member
etcdctl member list
# Output shows Node2 ID: abc123

etcdctl member remove abc123

# 2. Deploy new node with same name
# On new Node2:

# Join as new member
etcdctl member add node2 \
  --peer-urls=https://10.0.0.2:2380

# This returns join command like:
# ETCD_NAME="node2"
# ETCD_INITIAL_CLUSTER="node1=https://10.0.0.1:2380,node2=https://10.0.0.2:2380,node3=https://10.0.0.3:2380"
# ETCD_INITIAL_CLUSTER_STATE="existing"

# 3. Start new etcd with these parameters
# Edit /etc/kubernetes/manifests/etcd.yaml with join info

# 4. Verify
etcdctl member list
etcdctl endpoint health --cluster
```

**Time to recovery:** 5-10 minutes  
**Data loss:** None (replicated data)

---

### Scenario 2: Majority Failure (Critical)

**Problem:**
```
Cluster: 5 nodes
Node1: ✗ Failed
Node2: ✗ Failed  
Node3: ✗ Failed  
Node4: ✓ Healthy (has data but can't form quorum)
Node5: ✓ Healthy (has data but can't form quorum)

Quorum needed: 3
Available: 2
Status: CLUSTER DOWN ❌
```

**Solution A: Restore from Snapshot (Recommended)**

```bash
# 1. Choose one healthy node as base (Node4)
# On Node4:

# Stop etcd on all nodes
# On each node:
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# 2. Get latest backup
BACKUP_FILE=/backup/etcd-snapshot-latest.db

# 3. Restore on each node with new cluster config

# On Node4 (will be new member):
etcdctl snapshot restore $BACKUP_FILE \
  --data-dir=/var/lib/etcd-new \
  --name=node4-new \
  --initial-cluster=node4-new=https://10.0.0.4:2380,node5-new=https://10.0.0.5:2380 \
  --initial-advertise-peer-urls=https://10.0.0.4:2380 \
  --initial-cluster-token=etcd-cluster-disaster-recovery

# On Node5:
etcdctl snapshot restore $BACKUP_FILE \
  --data-dir=/var/lib/etcd-new \
  --name=node5-new \
  --initial-cluster=node4-new=https://10.0.0.4:2380,node5-new=https://10.0.0.5:2380 \
  --initial-advertise-peer-urls=https://10.0.0.5:2380 \
  --initial-cluster-token=etcd-cluster-disaster-recovery

# 4. Update etcd manifests to use new data directory
# Edit /tmp/etcd.yaml:
#   --data-dir=/var/lib/etcd-new
#   Update --initial-cluster
#   Update --name

# 5. Start new cluster
sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/

# 6. Verify
etcdctl endpoint health --cluster

# 7. Add new members to reach desired size
# Deploy 3 new nodes and join them
```

**Solution B: Force New Cluster from Survivor (Data Loss Risk!)**

```bash
# ⚠️ WARNING: Only if no backup available
# This creates new cluster from one member's data
# May lose uncommitted writes!

# On Node4:

# 1. Stop etcd
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# 2. Force new cluster
etcdctl snapshot save /tmp/snapshot.db
etcdctl snapshot restore /tmp/snapshot.db \
  --data-dir=/var/lib/etcd-forced \
  --name=node4-forced \
  --initial-cluster=node4-forced=https://10.0.0.4:2380 \
  --initial-advertise-peer-urls=https://10.0.0.4:2380

# 3. Update etcd config
# Edit etcd.yaml to use /var/lib/etcd-forced

# 4. Start single-node cluster
sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/

# 5. Add members back
# Deploy new nodes and join
```

**Time to recovery:** 30-60 minutes  
**Data loss:** Depends on backup freshness (minutes to hours)

---

### Scenario 3: Complete Cluster Destruction

**Problem:**
```
All etcd nodes destroyed
All Kubernetes nodes destroyed
Only backup survives
```

**Solution: Full Cluster Rebuild**

```bash
# Phase 1: Rebuild infrastructure
# 1. Provision new control plane nodes
# 2. Install Kubernetes components
# 3. DO NOT run kubeadm init yet

# Phase 2: Restore etcd
# On first control plane node:

# 1. Extract backup
gunzip -c /restore/etcd-snapshot.db.gz > /tmp/etcd-snapshot.db

# 2. Restore
etcdctl snapshot restore /tmp/etcd-snapshot.db \
  --data-dir=/var/lib/etcd \
  --name=master-1 \
  --initial-cluster=master-1=https://10.0.1.1:2380,master-2=https://10.0.1.2:2380,master-3=https://10.0.1.3:2380 \
  --initial-advertise-peer-urls=https://10.0.1.1:2380

# 3. Copy restored data to other nodes
scp -r /var/lib/etcd root@master-2:/var/lib/
scp -r /var/lib/etcd root@master-3:/var/lib/

# On master-2 and master-3, run restore with their respective names

# Phase 3: Restore certificates
# 1. Restore from backup:
#    - /etc/kubernetes/pki/
#    - /etc/kubernetes/*.conf

# Phase 4: Start control plane
# On each master:
sudo mv /etc/kubernetes/manifests/*.yaml.backup /etc/kubernetes/manifests/

# Phase 5: Verify
kubectl get nodes
kubectl get pods -A

# Phase 6: Rejoin worker nodes
# On each worker:
kubeadm join ...
```

**Time to recovery:** 2-4 hours  
**Data loss:** Based on backup age

---

### Scenario 4: Data Corruption

**Problem:**
```
etcd database corrupted
Cluster running but returning errors
```

**Symptoms:**
```bash
etcdctl get /registry/pods/default/nginx
# Error: mvcc: required revision has been compacted

# Or
etcdctl endpoint health
# Error: database space exceeded
```

**Solution:**

```bash
# 1. Check database size
etcdctl endpoint status --write-out=table

# 2. If space exceeded:
# Compact old revisions
CURRENT_REV=$(etcdctl endpoint status --write-out="json" | \
  jq -r '.[0].Status.header.revision')

etcdctl compact $CURRENT_REV

# Defragment to reclaim space
etcdctl defrag

# 3. If still corrupted:
# Restore from backup
# (See Scenario 2)
```

---

### Scenario 5: Accidental Data Deletion

**Problem:**
```bash
# Oops! Deleted critical namespace
kubectl delete namespace production
```

**Solution:**

```bash
# Option 1: If deleted recently (within minutes)
# Restore from most recent snapshot

# Option 2: Point-in-time restore
# Find backup before deletion
ls -lh /backup/

# Restore to temporary cluster
etcdctl snapshot restore /backup/etcd-snapshot-before-delete.db \
  --data-dir=/var/lib/etcd-recovery

# Extract specific resources
# (Complex - requires custom scripts)

# Option 3: If multi-cluster setup
# Copy from staging cluster
kubectl get namespace production -o yaml --context=staging | \
  kubectl apply --context=production -f -
```

---

### Disaster Recovery Checklist

**Before disaster:**
```
□ Automated backups running
□ Backups tested monthly
□ Offsite backup storage configured
□ DR runbook documented
□ Team trained on restore procedures
□ Certificates backed up
□ Configuration files backed up
□ Recovery time objective (RTO) defined
□ Recovery point objective (RPO) defined
□ Emergency contact list updated
```

**During disaster:**
```
□ Assess extent of failure
□ Notify stakeholders
□ Identify most recent good backup
□ Follow appropriate scenario steps
□ Document actions taken
□ Monitor recovery progress
```

**After recovery:**
```
□ Verify all services working
□ Check for data inconsistencies
□ Review what caused disaster
□ Update DR procedures
□ Conduct post-mortem
□ Improve monitoring/alerting
```

---

## 11. etcd Monitoring and Alerting

### Key Metrics to Monitor

**1. Cluster Health**

```bash
# Check overall health
etcdctl endpoint health --cluster

# Metrics to track:
etcd_server_has_leader          # 1 = has leader, 0 = no leader
etcd_server_leader_changes_seen_total  # Frequent changes = problem
etcd_server_proposals_failed_total     # Failed Raft proposals
```

**2. Performance Metrics**

```bash
# Disk sync duration (critical!)
etcd_disk_wal_fsync_duration_seconds
# Alert if p99 > 100ms

# Backend commit duration
etcd_disk_backend_commit_duration_seconds
# Alert if p99 > 100ms

# Leader election latency
etcd_server_leader_changes_seen_total
# Track frequency
```

**3. Database Size**

```bash
# Current database size
etcd_mvcc_db_total_size_in_bytes
# Alert if approaching quota (default 2GB)

# In-use size (after defrag)
etcd_mvcc_db_total_size_in_use_in_bytes
```

**4. Network Health**

```bash
# Round-trip time between peers
etcd_network_peer_round_trip_time_seconds
# Alert if > 50ms

# Sent/received bytes
etcd_network_client_grpc_sent_bytes_total
etcd_network_client_grpc_received_bytes_total
```

**5. Client Requests**

```bash
# Request rate
etcd_server_requests_total{type="write"}
etcd_server_requests_total{type="read"}

# Request duration
etcd_http_request_duration_seconds
```

### Setting Up Prometheus Monitoring

**1. etcd exposes metrics on port 2379:**

```bash
curl http://localhost:2379/metrics

# Or with TLS:
curl --cacert /etc/kubernetes/pki/etcd/ca.crt \
     --cert /etc/kubernetes/pki/etcd/server.crt \
     --key /etc/kubernetes/pki/etcd/server.key \
     https://localhost:2379/metrics
```

**2. Prometheus ServiceMonitor (if using Prometheus Operator):**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: etcd
  namespace: kube-system
  labels:
    component: etcd
spec:
  type: ClusterIP
  clusterIP: None
  ports:
  - name: metrics
    port: 2379
    protocol: TCP
---
apiVersion: v1
kind: Endpoints
metadata:
  name: etcd
  namespace: kube-system
  labels:
    component: etcd
subsets:
- addresses:
  - ip: 10.0.0.1  # etcd node 1
  - ip: 10.0.0.2  # etcd node 2
  - ip: 10.0.0.3  # etcd node 3
  ports:
  - name: metrics
    port: 2379
    protocol: TCP
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: etcd
  namespace: kube-system
  labels:
    component: etcd
spec:
  jobLabel: component
  endpoints:
  - port: metrics
    interval: 30s
    scheme: https
    tlsConfig:
      caFile: /etc/prometheus/secrets/etcd-certs/ca.crt
      certFile: /etc/prometheus/secrets/etcd-certs/client.crt
      keyFile: /etc/prometheus/secrets/etcd-certs/client.key
      insecureSkipVerify: false
  selector:
    matchLabels:
      component: etcd
  namespaceSelector:
    matchNames:
    - kube-system
```

**3. Create secret with etcd certs:**

```bash
kubectl create secret generic etcd-certs \
  --from-file=ca.crt=/etc/kubernetes/pki/etcd/ca.crt \
  --from-file=client.crt=/etc/kubernetes/pki/etcd/client.crt \
  --from-file=client.key=/etc/kubernetes/pki/etcd/client.key \
  -n monitoring
```

### Critical Prometheus Alerts

**alerts.yaml:**

```yaml
groups:
- name: etcd
  interval: 30s
  rules:
  
  # No leader alert
  - alert: etcdNoLeader
    expr: etcd_server_has_leader == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "etcd has no leader"
      description: "etcd cluster has no leader for more than 1 minute"
  
  # High number of leader changes
  - alert: etcdHighNumberOfLeaderChanges
    expr: increase(etcd_server_leader_changes_seen_total[1h]) > 3
    labels:
      severity: warning
    annotations:
      summary: "High number of leader changes"
      description: "etcd has seen {{ $value }} leader changes in the last hour"
  
  # Slow fsync
  - alert: etcdSlowFsync
    expr: histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])) > 0.1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "etcd slow fsync"
      description: "etcd WAL fsync p99 is {{ $value }}s (threshold: 0.1s)"
  
  # Database size approaching quota
  - alert: etcdDatabaseQuotaLowSpace
    expr: etcd_mvcc_db_total_size_in_bytes / etcd_server_quota_backend_bytes > 0.8
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "etcd database size is approaching quota"
      description: "etcd database is {{ $value | humanizePercentage }} full"
  
  # High commit duration
  - alert: etcdHighCommitDuration
    expr: histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket[5m])) > 0.1
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High backend commit duration"
      description: "etcd backend commit p99 is {{ $value }}s"
  
  # Failed proposals
  - alert: etcdHighNumberOfFailedProposals
    expr: increase(etcd_server_proposals_failed_total[1h]) > 5
    labels:
      severity: warning
    annotations:
      summary: "High number of failed proposals"
      description: "{{ $value }} proposals failed in the last hour"
  
  # Member communication issues
  - alert: etcdMemberCommunicationSlow
    expr: histogram_quantile(0.99, rate(etcd_network_peer_round_trip_time_seconds_bucket[5m])) > 0.05
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "etcd member communication is slow"
      description: "Peer round-trip time p99 is {{ $value }}s"
  
  # Insufficient members
  - alert: etcdInsufficientMembers
    expr: count(up{job="etcd"} == 1) < 3
    for: 3m
    labels:
      severity: critical
    annotations:
      summary: "etcd has insufficient members"
      description: "Only {{ $value }} etcd members are up (minimum: 3)"
```

### Grafana Dashboard

**Key panels to include:**

```
Dashboard: etcd Cluster Health

Row 1: Cluster Overview
  - Has Leader (gauge)
  - Leader Changes (graph)
  - Cluster Members Up (gauge)
  - Proposals Failed (counter)

Row 2: Performance
  - WAL Fsync Duration p99 (graph)
  - Backend Commit Duration p99 (graph)
  - Network RTT (graph)
  - Request Rate (graph)

Row 3: Database
  - DB Size (gauge)
  - DB Size in Use (gauge)
  - Keys Count (gauge)
  - Revision (counter)

Row 4: Operations
  - Read Requests/sec (graph)
  - Write Requests/sec (graph)
  - Request Duration p99 (graph)
  - Watch Streams (gauge)
```

**Example panel query (Fsync duration):**

```promql
histogram_quantile(0.99, 
  rate(etcd_disk_wal_fsync_duration_seconds_bucket[5m])
)
```

### Log Monitoring

**Important log patterns to watch:**

```bash
# Leader elections
journalctl -u etcd | grep "elected leader"

# Slow requests
journalctl -u etcd | grep "apply request took too long"

# Peer communication issues
journalctl -u etcd | grep "failed to send"

# Database size warnings
journalctl -u etcd | grep "database space exceeded"
```

**Set up log alerts (Loki/ELK):**

```yaml
# Example Loki alert
- alert: etcdSlowRequests
  expr: |
    count_over_time({job="etcd"} |~ "apply request took too long"[5m]) > 10
  labels:
    severity: warning
  annotations:
    summary: "etcd is experiencing slow requests"
```

### Health Check Script

**Automated health checker:**

```bash
#!/bin/bash
# /usr/local/bin/etcd-health-check.sh

ENDPOINTS="https://10.0.0.1:2379,https://10.0.0.2:2379,https://10.0.0.3:2379"
CACERT="/etc/kubernetes/pki/etcd/ca.crt"
CERT="/etc/kubernetes/pki/etcd/client.crt"
KEY="/etc/kubernetes/pki/etcd/client.key"

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'

echo "=== etcd Health Check ==="
echo

# 1. Check endpoint health
echo "1. Endpoint Health:"
etcdctl --endpoints="$ENDPOINTS" \
  --cacert="$CACERT" --cert="$CERT" --key="$KEY" \
  endpoint health

# 2. Check member list
echo
echo "2. Member List:"
etcdctl --endpoints="$ENDPOINTS" \
  --cacert="$CACERT" --cert="$CERT" --key="$KEY" \
  member list --write-out=table

# 3. Check database size
echo
echo "3. Database Size:"
etcdctl --endpoints="$ENDPOINTS" \
  --cacert="$CACERT" --cert="$CERT" --key="$KEY" \
  endpoint status --write-out=table

# 4. Check for leader
HAS_LEADER=$(etcdctl --endpoints="$ENDPOINTS" \
  --cacert="$CACERT" --cert="$CERT" --key="$KEY" \
  endpoint status --write-out=json | jq -r '.[0].Status.leader' | wc -c)

if [ "$HAS_LEADER" -gt 1 ]; then
  echo -e "${GREEN}✓ Leader elected${NC}"
else
  echo -e "${RED}✗ No leader!${NC}"
  exit 1
fi

# 5. Performance test
echo
echo "4. Performance Test (10 writes):"
START=$(date +%s%N)
for i in {1..10}; do
  etcdctl --endpoints="$ENDPOINTS" \
    --cacert="$CACERT" --cert="$CERT" --key="$KEY" \
    put /health-check/$i "test" > /dev/null
done
END=$(date +%s%N)
DURATION=$(( ($END - $START) / 1000000 ))
AVG=$(( $DURATION / 10 ))

if [ "$AVG" -lt 50 ]; then
  echo -e "${GREEN}✓ Average write latency: ${AVG}ms${NC}"
elif [ "$AVG" -lt 100 ]; then
  echo -e "${YELLOW}⚠ Average write latency: ${AVG}ms${NC}"
else
  echo -e "${RED}✗ Average write latency: ${AVG}ms (slow!)${NC}"
fi

# Cleanup
etcdctl --endpoints="$ENDPOINTS" \
  --cacert="$CACERT" --cert="$CERT" --key="$KEY" \
  del /health-check/ --prefix > /dev/null

echo
echo "=== Health Check Complete ==="
```

**Run periodically:**

```bash
# Add to cron (every 5 minutes)
*/5 * * * * /usr/local/bin/etcd-health-check.sh >> /var/log/etcd-health.log 2>&1
```

---

## 12. Common etcd Issues and Troubleshooting

### Issue 1: "mvcc: database space exceeded"

**Symptoms:**
```bash
etcdctl put /test value
# Error: etcdserver: mvcc: database space exceeded
```

**Cause:** Database reached quota (default 2GB)

**Diagnosis:**
```bash
# Check current size
etcdctl endpoint status --write-out=table

# Example output showing problem:
+------------------+------------------+---------+---------+-----------+
|    ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER |
+------------------+------------------+---------+---------+-----------+
| 127.0.0.1:2379   | abc123           | 3.5.0   | 2.1 GB  | true      |
+------------------+------------------+---------+---------+-----------+
```

**Solution:**
```bash
# 1. Get current revision
REV=$(etcdctl endpoint status --write-out="json" | \
  jq -r '.[0].Status.header.revision')

# 2. Compact old revisions
etcdctl compact $REV
# This marks old revisions for deletion

# 3. Defragment to reclaim space
etcdctl defrag
# This actually frees the space

# 4. Verify
etcdctl endpoint status --write-out=table
# DB SIZE should be reduced

# 5. If still too large, increase quota (carefully!)
# Edit /etc/kubernetes/manifests/etcd.yaml
# Add: --quota-backend-bytes=4294967296  # 4GB
```

**Prevention:**
```yaml
# Enable auto-compaction
- --auto-compaction-mode=periodic
- --auto-compaction-retention=1h

# OR revision-based
- --auto-compaction-mode=revision
- --auto-compaction-retention=1000
```

---

### Issue 2: "etcdserver: request timed out"

**Symptoms:**
```bash
kubectl get pods
# Error: etcdserver: request timed out

etcdctl get /test
# Error: context deadline exceeded
```

**Causes:**
1. Slow disk I/O
2. Network latency
3. CPU starvation
4. Large number of watchers

**Diagnosis:**
```bash
# 1. Check fsync duration
curl -k https://localhost:2379/metrics | grep fsync_duration

# If p99 > 100ms, disk is slow:
# etcd_disk_wal_fsync_duration_seconds{quantile="0.99"} 0.523

# 2. Check CPU usage
top -p $(pgrep etcd)

# 3. Check network latency
# Between etcd nodes:
ping -c 100 etcd-node-2

# 4. Check for excessive watchers
curl -k https://localhost:2379/metrics | grep etcd_debugging_mvcc_watcher_total
```

**Solutions:**

**A. Slow Disk:**
```bash
# Test disk performance
sudo fio --name=write_iops --directory=/var/lib/etcd \
  --size=4G --time_based --runtime=60s \
  --ramp_time=2s --ioengine=libaio --direct=1 \
  --verify=0 --bs=4K --iodepth=64 --rw=randwrite

# If IOPS < 3000:
# - Use SSD/NVMe
# - Enable write caching on RAID controller
# - Use dedicated disk for etcd

# Optimize mount options
# /etc/fstab:
/dev/nvme1n1  /var/lib/etcd  xfs  noatime,nodiratime  0 2
```

**B. Network Issues:**
```bash
# Increase timeouts (last resort!)
# In /etc/kubernetes/manifests/etcd.yaml:
- --heartbeat-interval=200
- --election-timeout=2000
```

**C. Too Many Watchers:**
```bash
# Check watcher count
etcdctl watch --rev=0 /registry --prefix &
# Kill excessive watchers

# In kube-apiserver, limit watch caches:
- --watch-cache-sizes=pods=1000,nodes=500
```

---

### Issue 3: "no available endpoints" / "all endpoints are unhealthy"

**Symptoms:**
```bash
etcdctl endpoint health
# Error: {"level":"warn","ts":"...","msg":"endpoints are unhealthy","endpoints":["https://127.0.0.1:2379"]}
```

**Diagnosis:**
```bash
# 1. Check if etcd is running
systemctl status etcd
# OR
kubectl get pods -n kube-system | grep etcd

# 2. Check logs
journalctl -u etcd -f
# OR
kubectl logs -n kube-system etcd-master-1

# 3. Check if port is listening
netstat -tlnp | grep 2379
ss -tlnp | grep 2379

# 4. Check TLS certificates
openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -text -noout
# Look at:
# - Validity dates
# - Subject Alternative Names (must include IPs)

# 5. Test connection
curl -k https://localhost:2379/health
```

**Common causes and fixes:**

**A. Certificate Issues:**
```bash
# Check cert expiry
for cert in /etc/kubernetes/pki/etcd/*.crt; do
  echo "=== $cert ==="
  openssl x509 -in $cert -noout -dates
done

# If expired, regenerate:
kubeadm certs renew etcd-server
kubeadm certs renew etcd-peer
systemctl restart kubelet
```

**B. Wrong Listen Address:**
```yaml
# In etcd.yaml, ensure:
- --listen-client-urls=https://0.0.0.0:2379,https://127.0.0.1:2379
# NOT just https://127.0.0.1:2379
```

**C. Firewall:**
```bash
# Check firewall rules
sudo iptables -L -n | grep 2379
sudo iptables -L -n | grep 2380

# Open ports if needed
sudo firewall-cmd --permanent --add-port=2379-2380/tcp
sudo firewall-cmd --reload
```

---

### Issue 4: Frequent Leader Elections

**Symptoms:**
```bash
# Logs show constant leader changes
journalctl -u etcd | grep "elected leader"

# Metrics show high leader changes
curl -k https://localhost:2379/metrics | grep leader_changes_seen_total
# etcd_server_leader_changes_seen_total 47  # In last hour = BAD
```

**Causes:**
1. Network instability
2. Disk I/O latency spikes
3. CPU throttling
4. Heartbeat timeout too low

**Diagnosis:**
```bash
# 1. Check network stability
ping -c 1000 etcd-node-2 | tail -5
# Packet loss should be 0%

# 2. Check for I/O waits
iostat -x 1 10
# Look at %util and await columns

# 3. Monitor heartbeat failures
curl -k https://localhost:2379/metrics | grep heartbeat_send_failures_total
```

**Solutions:**

```bash
# 1. Increase timeouts (conservative approach)
# In etcd.yaml:
- --heartbeat-interval=200    # was 100
- --election-timeout=2000     # was 1000

# 2. Fix underlying issues:
# - Upgrade network hardware
# - Use faster disks
# - Reduce CPU contention (dedicated nodes)

# 3. Check for clock skew
# On all nodes:
timedatectl status
# Ensure clocks are synchronized (use NTP)
```

---

### Issue 5: Split-Brain After Network Partition

**Symptoms:**
```
Two groups of nodes, each thinks it's the valid cluster
Data diverges between groups
```

**Reality Check:**
```
This CANNOT happen in etcd if configured correctly!
etcd's Raft prevents split-brain by requiring quorum.

If you see "split-brain symptoms":
1. Configuration error (wrong initial-cluster)
2. Manual override was used incorrectly
3. Network partition with asymmetric connectivity
```

**Diagnosis:**
```bash
# Check each node's view of the cluster
# On each node:
etcdctl member list

# All nodes should show SAME member list
# If different lists = problem!

# Check which nodes can reach others
# From each node:
for ip in 10.0.0.1 10.0.0.2 10.0.0.3; do
  echo "Testing $ip:"
  nc -zv $ip 2380
done
```

**Solution:**
```bash
# If truly split (rare):
# 1. Identify majority side (more nodes)
# 2. Shut down minority side
# 3. Fix network
# 4. Restore minority nodes from majority's data
# 5. Or restore entire cluster from backup
```

---

### Issue 6: "required revision has been compacted"

**Symptoms:**
```bash
etcdctl get --rev=1000 /mykey
# Error: mvcc: required revision has been compacted
```

**Cause:** Requested historical revision has been compacted away

**Context:**
```
etcd maintains history, but not forever:
Rev 1000-1999: ✗ Compacted
Rev 2000-3499: ✓ Available
Rev 3500: Current
```

**Solution:**
```bash
# This is expected behavior after compaction
# You cannot retrieve compacted revisions

# To get current value:
etcdctl get /mykey

# To prevent excessive compaction:
- --auto-compaction-retention=24h  # Keep 24h of history
```

---

### Issue 7: etcd Won't Start After Restore

**Symptoms:**
```bash
# After snapshot restore:
systemctl start etcd
# Failed!

journalctl -u etcd -n 50
# Error: "member ID mismatch"
# or "cluster ID mismatch"
```

**Cause:** Restore created new cluster ID/member IDs that don't match manifest

**Solution:**
```bash
# 1. During restore, use --name and --initial-cluster correctly
etcdctl snapshot restore snapshot.db \
  --name=master-1 \
  --initial-cluster=master-1=https://10.0.0.1:2380 \
  --data-dir=/var/lib/etcd-new

# 2. Update etcd.yaml to match:
- --name=master-1
- --initial-cluster=master-1=https://10.0.0.1:2380
- --initial-cluster-state=new  # Important!
- --data-dir=/var/lib/etcd-new

# 3. Remove old data
rm -rf /var/lib/etcd  # Make sure you have backup!

# 4. Start etcd
```

---

### Issue 8: High Memory Usage

**Symptoms:**
```bash
top
# etcd using 8GB+ RAM

kubectl top node
# Master node OOM
```

**Causes:**
1. Large number of keys
2. Large values
3. Many concurrent watchers
4. Memory leak (rare, upgrade etcd)

**Diagnosis:**
```bash
# 1. Check key count and size
etcdctl endpoint status --write-out=table

# 2. Find large keys
etcdctl get / --prefix --keys-only | while read key; do
  size=$(etcdctl get "$key" | wc -c)
  echo "$size $key"
done | sort -rn | head -20

# 3. Count watchers
curl -k https://localhost:2379/metrics | grep watcher_total
```

**Solutions:**
```bash
# 1. Split large values
# Instead of:
# /config/all-settings = 5MB JSON
# Use:
# /config/db/host = "postgres"
# /config/db/port = "5432"
# ...

# 2. Limit
retention
- --auto-compaction-retention=1h

# 3. Increase memory if needed
# Edit systemd service or pod resources

# 4. Use ConfigMaps for large configs instead of Secrets
```

---

### Troubleshooting Toolkit

**Essential commands:**

```bash
# Health check
etcdctl endpoint health --cluster

# Member list
etcdctl member list --write-out=table

# Status
etcdctl endpoint status --write-out=table

# Metrics
curl -k https://localhost:2379/metrics

# Logs
journalctl -u etcd -f --since "10 minutes ago"

# Check certificate
openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -text -noout

# Test write/read
etcdctl put /test "$(date)"
etcdctl get /test

# Performance test
etcdctl check perf
```

**Quick diagnostic script:**

```bash
#!/bin/bash
echo "=== etcd Quick Diagnostics ==="
echo "1. Service Status:"
systemctl is-active etcd || echo "etcd not running!"
echo
echo "2. Endpoint Health:"
etcdctl endpoint health --cluster
echo
echo "3. Leader:"
etcdctl endpoint status --write-out=table | grep true
echo
echo "4. Database Size:"
etcdctl endpoint status --write-out=table
echo
echo "5. Recent Errors:"
journalctl -u etcd --since "5 minutes ago" | grep -i error | tail -10
```

---

## 13. Real-World Production Best Practices

### Infrastructure Best Practices

**1. Dedicated Nodes**

```
✓ DO: Run etcd on dedicated control plane nodes
✗ DON'T: Co-locate etcd with heavy workloads

Rationale:
- etcd is I/O and CPU intensive
- Workload interference = latency = cluster instability
```

**2. Odd Number of Nodes**

```
Cluster Size Decision Tree:
├─ Development/Testing: 1 node (no HA)
├─ Small Production: 3 nodes (1 failure tolerance)
├─ Standard Production: 5 nodes (2 failures tolerance)
└─ Large Production: 7 nodes (3 failures tolerance)

Never use even numbers (2, 4, 6)!
- 2 nodes: worse than 1 (split-brain possible)
- 4 nodes: same failure tolerance as 3
- 6 nodes: same failure tolerance as 5
```

**3. Use Fast SSDs**

```
Minimum Requirements:
- IOPS: > 3,000
- Latency: < 10ms (99th percentile)
- Type: NVMe SSD (best), SATA SSD (acceptable)

Test before deployment:
sudo fio --name=etcd_test \
  --filename=/var/lib/etcd/test \
  --size=22m --time_based --runtime=60s \
  --ioengine=libaio --fdatasync=1 \
  --direct=1 --numjobs=1 --rw=write \
  --blocksize=2300
```

**4. Network Topology**

```
Requirements:
- RTT < 10ms (between etcd nodes)
- Bandwidth: 1Gbps minimum
- Same datacenter/availability zone

For multi-region:
✗ DON'T spread etcd across regions (high latency)
✓ DO use separate clusters per region
✓ DO use external replication for cross-region DR
```

**5. Resource Allocation**

```yaml
# Minimum for production (per etcd pod)
resources:
  requests:
    cpu: "1000m"      # 1 CPU
    memory: "2Gi"
  limits:
    cpu: "2000m"      # 2 CPU
    memory: "8Gi"

# For large clusters (1000+ nodes)
resources:
  requests:
    cpu: "2000m"
    memory: "8Gi"
  limits:
    cpu: "4000m"
    memory: "16Gi"
```

---

### Configuration Best Practices

**1. Optimal etcd Configuration**

```yaml
# /etc/kubernetes/manifests/etcd.yaml
spec:
  containers:
  - command:
    - etcd
    
    # Data directory (use dedicated disk/volume)
    - --data-dir=/var/lib/etcd
    
    # Networking
    - --listen-client-urls=https://0.0.0.0:2379
    - --listen-peer-urls=https://0.0.0.0:2380
    - --advertise-client-urls=https://10.0.0.1:2379
    - --initial-advertise-peer-urls=https://10.0.0.1:2380
    
    # Cluster
    - --initial-cluster=master-1=https://10.0.0.1:2380,master-2=https://10.0.0.2:2380,master-3=https://10.0.0.3:2380
    - --initial-cluster-state=existing  # Or 'new' for initial setup
    - --initial-cluster-token=my-k8s-cluster  # Unique per cluster
    
    # Performance
    - --heartbeat-interval=100
    - --election-timeout=1000
    - --snapshot-count=10000
    - --max-snapshots=5
    - --max-wals=5
    
    # Quotas
    - --quota-backend-bytes=8589934592  # 8GB
    
    # Auto-compaction
    - --auto-compaction-mode=periodic
    - --auto-compaction-retention=8h
    
    # Security
    - --client-cert-auth=true
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --peer-client-cert-auth=true
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    
    # Logging
    - --logger=zap
    - --log-level=info  # Use 'warn' in production for less verbosity
    
    # Metrics
    - --listen-metrics-urls=http://0.0.0.0:2381
```

**2. kube-apiserver Configuration**

```yaml
# Optimize for etcd
- --etcd-servers=https://10.0.0.1:2379,https://10.0.0.2:2379,https://10.0.0.3:2379
- --etcd-servers-overrides=/events#https://10.0.0.1:2379;https://10.0.0.2:2379;https://10.0.0.3:2379

# Separate etcd for events (optional, for large clusters)
# Reduces load on main etcd

# Compaction
- --etcd-compaction-interval=5m

# Watch cache (reduce etcd load)
- --watch-cache-sizes=pods=1000,nodes=500
```

---

### Operational Best Practices

**1. Backup Strategy**

```
Schedule:
├─ Automatic: Every 6 hours
├─ Pre-change: Before any major cluster change
└─ Off-site: Daily sync to remote storage

Retention:
├─ Hourly: Last 24 hours
├─ Daily: Last 7 days
├─ Weekly: Last 4 weeks
└─ Monthly: Last 12 months

Storage:
├─ Local: Fast restore
├─ S3/GCS: Durability
└─ Cross-region: Disaster recovery
```

**2. Monitoring and Alerting**

```yaml
Critical Alerts (Page immediately):
- etcd has no leader (1 min)
- Majority of members down (immediate)
- Database quota exceeded (immediate)
- Disk fsync > 500ms (5 min)

Warning Alerts (Slack/email):
- Leader changes > 3/hour
- Disk fsync > 100ms (10 min)
- Database > 80% quota (30 min)
- High network latency > 50ms (15 min)
- Certificate expiring < 30 days

Metrics to Track:
- etcd_server_has_leader
- etcd_disk_wal_fsync_duration_seconds
- etcd_mvcc_db_total_size_in_bytes
- etcd_network_peer_round_trip_time_seconds
- etcd_server_proposals_failed_total
```

**3. Change Management**

```
Pre-change Checklist:
□ Backup taken and verified
□ Maintenance window scheduled
□ Rollback plan documented
□ Team notified
□ Monitoring alerts adjusted

During Change:
□ One node at a time
□ Verify quorum after each change
□ Monitor metrics continuously
□ Document any issues

Post-change:
□ Verify cluster health
□ Check performance metrics
□ Restore normal monitoring
□ Update documentation
```

**4. Capacity Planning**

```bash
# Monitor growth trends
# Database size growth
curl -k https://localhost:2379/metrics | \
  grep etcd_mvcc_db_total_size_in_bytes

# Track over time to predict:
# - When to increase quota
# - When to add nodes
# - When to optimize data

# Rule of thumb:
# Plan for 3x current size before quota increase needed
```

**5. Disaster Recovery Drills**

```
Quarterly DR Test:
1. Restore from backup to test cluster
2. Verify data integrity
3. Measure recovery time
4. Document any issues
5. Update procedures

Annual DR Test:
1. Simulate total loss
2. Full cluster rebuild
3. Restore from backup
4. Verify all applications work
5. Measure total recovery time
```

---

### Security Best Practices

**1. Network Security**

```bash
# Firewall rules (iptables example)
# Allow only necessary traffic

# etcd client port (from API server only)
iptables -A INPUT -p tcp --dport 2379 \
  -s 10.0.0.0/24 -j ACCEPT
iptables -A INPUT -p tcp --dport 2379 -j DROP

# etcd peer port (from other etcd nodes only)
iptables -A INPUT -p tcp --dport 2380 \
  -s 10.0.0.1 -j ACCEPT
iptables -A INPUT -p tcp --dport 2380 \
  -s 10.0.0.2 -j ACCEPT
iptables -A INPUT -p tcp --dport 2380 \
  -s 10.0.0.3 -j ACCEPT
iptables -A INPUT -p tcp --dport 2380 -j DROP
```

**2. Certificate Management**

```bash
# Monitor certificate expiry
for cert in /etc/kubernetes/pki/etcd/*.crt; do
  echo "Checking $cert"
  openssl x509 -in $cert -noout -dates
done

# Set up automated renewal
# Add to cron:
0 0 1 * * kubeadm certs renew etcd-server && \
              kubeadm certs renew etcd-peer && \
              systemctl restart kubelet
```

**3. Access Control**

```
✓ Enable etcd authentication
✓ Use least-privilege RBAC
✓ Separate roles for different services
✓ Audit access logs
✓ Rotate credentials regularly
```

**4. Encryption**

```
✓ TLS for all communication
✓ Encryption at rest for secrets
✓ Encrypt backups
✓ Secure key management (KMS)
```

---

### Scaling Best Practices

**1. Horizontal Scaling (More Nodes)**

```
When to add nodes:
✗ Don't add nodes just because
✓ Add when failure tolerance insufficient
✓ Current: 3 nodes → Add to 5 for more resilience

Warning:
- More nodes = More replication overhead
- Diminishing returns after 7 nodes
- Consider splitting cluster instead
```

**2. Vertical Scaling (Bigger Nodes)**

```
Scale up when:
- Database size growing
- High CPU usage
- Memory pressure
- Slow response times

Typical progression:
Small:  2 CPU, 4GB RAM  (< 100 nodes)
Medium: 4 CPU, 8GB RAM  (100-500 nodes)
Large:  8 CPU, 16GB RAM (500-1000 nodes)
XLarge: 16 CPU, 32GB RAM (1000+ nodes)
```

**3. Data Optimization**

```bash
# Regular maintenance
# 1. Compact frequently
- --auto-compaction-retention=1h

# 2. Defragment monthly
etcdctl defrag --cluster

# 3. Monitor key count
etcdctl get / --prefix --keys-only | wc -l

# 4. Clean up old data
# Example: Delete old events
kubectl delete events --all --all-namespaces
```

---

### High Availability Best Practices

**1. Multi-Zone Deployment**

```
Ideal 3-node setup:
- Node 1: Zone A
- Node 2: Zone B  
- Node 3: Zone C

Benefits:
- Zone failure tolerance
- Lower blast radius

Constraints:
- Network latency < 10ms between zones
- Usually means same region
```

**2. Load Balancer Configuration**

```yaml
# For kube-apiserver to etcd
# Use all endpoints, not LB
- --etcd-servers=https://etcd-1:2379,https://etcd-2:2379,https://etcd-3:2379

# Client will try endpoints in order
# Automatic failover to healthy members
```

**3. Graceful Degradation**

```
Design for partial failures:
- kube-apiserver caches frequently accessed data
- Watch connections survive brief etcd restarts
- Workloads continue during etcd maintenance
```

---

### Development vs Production

```
┌────────────────────┬─────────────────┬──────────────────┐
│    Aspect          │   Development   │   Production     │
├────────────────────┼─────────────────┼──────────────────┤
│ Nodes              │ 1               │ 3 or 5           │
│ Storage            │ Any disk        │ NVMe SSD         │
│ Backups            │ Optional        │ Every 6 hours    │
│ Monitoring         │ Basic           │ Full observ.     │
│ Encryption         │ Optional        │ Always           │
│ High Availability  │ No              │ Yes              │
│ Resources          │ 1 CPU, 2GB      │ 2+ CPU, 8+ GB    │
└────────────────────┴─────────────────┴──────────────────┘
```

---

## 14. Hands-On Examples with etcdctl

### Setup etcdctl Environment

```bash
# Install etcdctl
ETCD_VER=v3.5.15
wget https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar -xzvf etcd-${ETCD_VER}-linux-amd64.tar.gz
sudo cp etcd-${ETCD_VER}-linux-amd64/etcdctl /usr/local/bin/
sudo chmod +x /usr/local/bin/etcdctl

# Set environment variables
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key

# Add to ~/.bashrc for persistence
cat >> ~/.bashrc <<'EOF'
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key
alias e='etcdctl'
EOF
```

### Example 1: Basic CRUD Operations

```bash
# CREATE/UPDATE (PUT)
etcdctl put /myapp/config/database "postgresql://localhost:5432/mydb"
etcdctl put /myapp/config/port "8080"
etcdctl put /myapp/config/debug "true"

# READ (GET)
etcdctl get /myapp/config/database
# Output:
# /myapp/config/database
# postgresql://localhost:5432/mydb

# Get only the value
etcdctl get /myapp/config/database --print-value-only
# Output:
# postgresql://localhost:5432/mydb

# Get all keys with a prefix
etcdctl get /myapp/config/ --prefix
# Output:
# /myapp/config/database
# postgresql://localhost:5432/mydb
# /myapp/config/debug
# true
# /myapp/config/port
# 8080

# Get only keys (no values)
etcdctl get /myapp/ --prefix --keys-only
# Output:
# /myapp/config/database
# /myapp/config/debug
# /myapp/config/port

# DELETE
etcdctl del /myapp/config/debug

# Delete all keys with prefix
etcdctl del /myapp/config/ --prefix

# DELETE with previous value
etcdctl del /myapp/config/port --prev-kv
# Output shows deleted key-value
```

### Example 2: Working with Revisions

```bash
# Put multiple versions
etcdctl put /counter "1"
etcdctl put /counter "2"
etcdctl put /counter "3"

# Get current value
etcdctl get /counter
# /counter
# 3

# Get value at specific revision
# First, find revision numbers
etcdctl get /counter --write-out=json | jq .header.revision
# Output: 12345

# Get previous revision
etcdctl get /counter --rev=12344
# /counter
# 2

# Watch from specific revision (replay history)
etcdctl watch /counter --rev=1
# Shows all changes from revision 1 to now
```

### Example 3: Transactions (Compare-and-Swap)

```bash
# Atomic compare-and-swap
# Only update if current value matches

etcdctl put /lock/myservice "available"

# Transaction: Update only if value is "available"
etcdctl txn <<EOF
compare:
value("/lock/myservice") = "available"

success:
put /lock/myservice "acquired"

failure:
get /lock/myservice
EOF

# If successful, lock is acquired
# If failed, shows current value

# More complex transaction
etcdctl txn <<EOF
compare:
value("/counter") = "5"
version("/config/enabled") > 0

success:
put /counter "6"
put /log "Counter incremented"

failure:
get /counter
get /config/enabled
EOF
```

### Example 4: Leases and TTL

```bash
# Grant a lease with 30-second TTL
etcdctl lease grant 30
# Output: lease 694d7f6d6f6d7f6e granted with TTL(30s)

LEASE_ID=694d7f6d6f6d7f6e

# Attach key to lease
etcdctl put /session/user123 "active" --lease=$LEASE_ID

# Keep lease alive (in background)
etcdctl lease keep-alive $LEASE_ID &
KEEPALIVE_PID=$!

# Check lease info
etcdctl lease timetolive $LEASE_ID
# Output: lease 694d7f6d6f6d7f6e granted with TTL(30s), remaining(25s)

# Stop keep-alive (key will expire after 30s)
kill $KEEPALIVE_PID

# Watch key disappear
etcdctl watch /session/user123
# Wait 30 seconds...
# DELETE event appears

# Revoke lease immediately
LEASE_ID=$(etcdctl lease grant 60 | awk '{print $2}')
etcdctl put /temp/data "test" --lease=$LEASE_ID
etcdctl lease revoke $LEASE_ID
# Key immediately deleted
```

### Example 5: Watching for Changes

```bash
# Watch a single key
etcdctl watch /myapp/config/database
# Terminal blocks, waiting for changes

# In another terminal, make a change:
etcdctl put /myapp/config/database "new-value"

# First terminal shows:
# PUT
# /myapp/config/database
# new-value

# Watch with prefix
etcdctl watch /myapp/config/ --prefix

# Watch and execute command on change
etcdctl watch /myapp/config/ --prefix -- sh -c 'echo "Config changed at $(date)"'

# Watch from specific revision
etcdctl watch /myapp/ --rev=1000 --prefix
# Shows all changes since revision 1000

# Interactive watch
etcdctl watch /myapp/ --interactive --prefix
# Allows multiple watch patterns
```

### Example 6: Kubernetes Data Exploration

```bash
# List all namespaces
etcdctl get /registry/namespaces --prefix --keys-only

# Get specific namespace
etcdctl get /registry/namespaces/default

# List all pods in a namespace
etcdctl get /registry/pods/default/ --prefix --keys-only

# Get specific pod data
etcdctl get /registry/pods/default/nginx-pod | jq .

# List all secrets
etcdctl get /registry/secrets --prefix --keys-only

# Count resources
echo "Pods: $(etcdctl get /registry/pods --prefix --keys-only | wc -l)"
echo "Services: $(etcdctl get /registry/services --prefix --keys-only | wc -l)"
echo "Secrets: $(etcdctl get /registry/secrets --prefix --keys-only | wc -l)"

# Watch for pod creations
etcdctl watch /registry/pods/ --prefix
# Then: kubectl run test --image=nginx
```

### Example 7: Backup and Restore

```bash
# Create backup
BACKUP_FILE=/tmp/etcd-backup-$(date +%Y%m%d-%H%M%S).db
etcdctl snapshot save $BACKUP_FILE

# Verify backup
etcdctl snapshot status $BACKUP_FILE --write-out=table
# Output:
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | a1b2c3d4 |   123456 |       1543 | 4.2 MB     |
# +----------+----------+------------+------------+

# Simulate disaster (DON'T DO THIS IN PRODUCTION!)
# sudo rm -rf /var/lib/etcd/*

# Restore from backup
sudo systemctl stop etcd  # or move manifest
etcdctl snapshot restore $BACKUP_FILE \
  --data-dir=/var/lib/etcd-restored \
  --name=master-1 \
  --initial-cluster=master-1=https://10.0.0.1:2380 \
  --initial-advertise-peer-urls=https://10.0.0.1:2380

# Update etcd config to use restored data
# sudo systemctl start etcd
```

### Example 8: Cluster Management

```bash
# List members
etcdctl member list --write-out=table
# +------------------+---------+---------+------------------------+
# |        ID        | STATUS  |  NAME   |      PEER ADDRS        |
# +------------------+---------+---------+------------------------+
# | 8e9e05c52164694d | started | master-1| https://10.0.0.1:2380 |
# | 91bc3c398fb3c146 | started | master-2| https://10.0.0.2:2380 |
# | fd422379fda50e48 | started | master-3| https://10.0.0.3:2380 |
# +------------------+---------+---------+------------------------+

# Add a new member
etcdctl member add master-4 \
  --peer-urls=https://10.0.0.4:2380

# This outputs join command for new node

# Remove a member
etcdctl member remove 8e9e05c52164694d

# Update member peer URLs
etcdctl member update 91bc3c398fb3c146 \
  --peer-urls=https://10.0.0.2:2380,https://10.0.0.2:2390
```

### Example 9: Performance Testing

```bash
# Write performance
etcdctl check perf

# Custom benchmark
# 1000 sequential writes
for i in {1..1000}; do
  etcdctl put /bench/key-$i "value-$i"
done

# Measure time
time for i in {1..100}; do
  etcdctl put /test-$i "value"
done

# Read performance
time for i in {1..100}; do
  etcdctl get /test-$i
done

# Cleanup
etcdctl del /test- --prefix
etcdctl del /bench/ --prefix
```

### Example 10: Debugging

```bash
# Check endpoint health
etcdctl endpoint health --cluster

# Get status of all members
etcdctl endpoint status --cluster --write-out=table

# Get metrics
curl -k https://localhost:2379/metrics | grep -E '(fsync|commit_duration|leader)'

# Check database size
etcdctl endpoint status --write-out=json | \
  jq -r '.[].Status | "DB Size: \(.dbSize) bytes, In Use: \(.dbSizeInUse) bytes"'

# Alarm list
etcdctl alarm list

# Compact
REV=$(etcdctl endpoint status --write-out="json" | jq -r '.[0].Status.header.revision')
etcdctl compact $REV

# Defragment
etcdctl defrag --cluster

# Clear alarms
etcdctl alarm disarm
```

---

## 15. How etcd Topics Appear in CKA Exam

The Certified Kubernetes Administrator (CKA) exam tests practical etcd skills. Here's what you need to know:

### Common CKA Tasks

**Task 1: Backup etcd**

```bash
# Typical question format:
# "Create a backup of the etcd cluster and save it to /opt/etcd-backup.db"

# Solution:
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify
ETCDCTL_API=3 etcdctl snapshot status /opt/etcd-backup.db --write-out=table
```

**Task 2: Restore etcd from Backup**

```bash
# Typical question:
# "Restore etcd from backup located at /opt/etcd-backup.db"

# Solution steps:
# 1. Move manifests (stop etcd and kube-apiserver)
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# 2. Restore
ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-from-backup

# 3. Update etcd manifest
sudo vi /tmp/etcd.yaml
# Change: --data-dir=/var/lib/etcd-from-backup

# 4. Move manifests back
sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# 5. Wait for etcd to be ready (check with crictl ps)
```

**Task 3: Check etcd Cluster Health**

```bash
# Question: "Verify the etcd cluster is healthy"

# Solution:
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health
```

**Task 4: Find etcd Configuration**

```bash
# Question: "What is the data directory used by etcd?"

# Solution:
# Check the static pod manifest
cat /etc/kubernetes/manifests/etcd.yaml | grep data-dir

# Or check the running process
ps aux | grep etcd | grep data-dir
```

**Task 5: Count Resources in etcd**

```bash
# Question: "How many pods are stored in etcd in the default namespace?"

# Solution:
ETCDCTL_API=3 etcdctl get /registry/pods/default/ --prefix --keys-only \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key | wc -l
```

### CKA Exam Tips

**1. Know These Paths by Heart:**
```
Certificates:
/etc/kubernetes/pki/etcd/ca.crt
/etc/kubernetes/pki/etcd/server.crt
/etc/kubernetes/pki/etcd/server.key

Manifests:
/etc/kubernetes/manifests/etcd.yaml
/etc/kubernetes/manifests/kube-apiserver.yaml

Data:
/var/lib/etcd (default data directory)
```

**2. Memorize This Template:**
```bash
export ETCDCTL_API=3
etcdctl <command> \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**3. Quick Reference Card for Exam:**
```bash
# Backup
etcdctl snapshot save <file>

# Restore
etcdctl snapshot restore <file> --data-dir=<path>

# Health
etcdctl endpoint health

# Member list
etcdctl member list
```

**4. Time-Saving Alias:**
```bash
# Set this up at start of exam
alias e='ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key'

# Then use: e snapshot save /backup.db
```

**5. Common Mistakes to Avoid:**
```
❌ Forgetting ETCDCTL_API=3
❌ Wrong certificate paths
❌ Not verifying backup after creation
❌ Forgetting to update etcd.yaml after restore
❌ Not waiting for etcd to restart fully
```

**6. Exam Strategy:**
```
1. Read question carefully (backup vs restore vs both)
2. Check if certificates are in default location
3. Use tab completion for paths
4. Verify each step before proceeding
5. Test the result (e.g., kubectl get pods still works)
```

---

## 16. Real-World Production Use Cases

### Use Case 1: E-commerce Platform - High-Volume Cluster

**Scenario:**
```
Company: Online retailer
Scale: 500 microservices, 2000 pods
Traffic: 100K requests/second
Requirements: 99.99% uptime
```

**etcd Configuration:**
```yaml
# 5-node external etcd cluster
Cluster Size: 5 nodes
Resources per node:
  - CPU: 8 cores
  - RAM: 32 GB
  - Disk: 1TB NVMe SSD (dedicated)
  - Network: 10 Gbps

# Optimized settings
- --quota-backend-bytes=17179869184  # 16GB
- --auto-compaction-retention=30m
- --snapshot-count=5000

# Separated event storage
- Dedicated 3-node etcd cluster for events
- Keeps main cluster performant

Monitoring:
- Prometheus + Grafana
- PagerDuty integration
- SLO: 99.99% availability
- Alert if p99 latency > 50ms

Backup:
- Every 2 hours to S3
- Cross-region replication
- Tested restore: monthly
- RTO: 30 minutes, RPO: 2 hours
```

**Lessons Learned:**
- Separated events reduced main etcd load by 40%
- Auto-compaction essential at this scale
- Network bandwidth was bottleneck initially
- Regular defragmentation required (weekly)

---

### Use Case 2: Multi-Tenant SaaS Platform

**Scenario:**
```
Company: SaaS provider
Tenants: 50 customers
Isolation: Namespace per tenant
Challenge: Fair resource distribution
```

**Architecture:**
```
# Cluster per environment
Production:
  - 5-node etcd (external)
  - Quotas per tenant enforced
  - Separate etcd for each large tenant

Staging:
  - 3-node stacked etcd
  - Shared with multiple stages

Development:
  - 1-node etcd per dev
```

**Security Implementation:**
```yaml
# etcd RBAC per tenant
# Tenant A user
etcdctl user add tenant-a
etcdctl role add tenant-a-role
etcdctl role grant-permission tenant-a-role \
  readwrite /registry/secrets/tenant-a/ --prefix=true
etcdctl role grant-permission tenant-a-role \
  read /registry/pods/tenant-a/ --prefix=true
etcdctl user grant-role tenant-a tenant-a-role

# Encryption at rest
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key-tenant-a
          secret: <UNIQUE_KEY>

# Audit logging
- --audit-policy-file=/etc/kubernetes/audit-policy.yaml
- --audit-log-path=/var/log/kube-apiserver-audit.log
```

**Challenges Solved:**
- noisy neighbor (resource quotas)
- Data isolation (encryption per tenant)
- Cost allocation (metrics per namespace)

---

### Use Case 3: Financial Services - Compliance Requirements

**Scenario:**
```
Company: Bank
Compliance: PCI-DSS, SOC 2, GDPR
Requirements:
  - Zero data loss
  - Full audit trail
  - Encryption everywhere
  - Geographic restrictions
```

**Setup:**
```yaml
# Air-gapped environment
Deployment:
  - 3 separate regions (active-active-passive)
  - 7-node etcd per region
  - No internet access (all mirrors internal)

Security:
  - mTLS for all communication
  - Encryption at rest (FIPS-compliant)
  - Hardware Security Module (HSM) for keys
  - Certificate rotation: every 30 days
  - etcd authentication required

Backup:
  - Continuous replication to tape
  - Hourly snapshots
  - Quarterly off-site storage
  - Retention: 7 years

Monitoring:
  - All access logged
  - Anomaly detection
  - Change approval required
  - Break-glass procedures documented
```

**Compliance Features:**
```bash
# Audit logging
etcdctl auth enable
etcdctl user add auditor --no-password
etcdctl role add audit
etcdctl role grant-permission audit read / --prefix=true
etcdctl user grant-role auditor audit

# Log all access
journalctl -u etcd -o json | \
  jq -r '{time: .MESSAGE, user: .USER, action: .ACTION}'

# Immutable backup
aws s3 cp backup.db s3://backups/ \
  --storage-class GLACIER \
  --sse aws:kms \
  --sse-kms-key-id alias/etcd-backup
```

---

### Use Case 4: IoT Platform - Massive Scale

**Scenario:**
```
Company: IoT device manager
Scale: 1 million devices
Data: Device state, configs, telemetry
Pattern: High read, moderate write
```

**Solution:**
```
# Distributed architecture
Main etcd cluster:
  - 5 nodes
  - Stores critical state only
  - Strict quotas

Regional etcd clusters:
  - 3 nodes per region (10 regions)
  - Caches device data locally
  - Syncs to main async

Optimization:
  - Aggressive compaction (5min)
  - Watch-based sync (not polling)
  - Batch updates (not individual)
  - TTL on ephemeral data (5min)

Performance:
  - 50K reads/sec per cluster
  - 5K writes/sec per cluster
  - p99 latency: < 10ms
```

**Device State Management:**
```bash
# Device heartbeat with lease
# Each device has a lease
DEVICE_ID="sensor-12345"
LEASE=$(etcdctl lease grant 300)  # 5 minutes
etcdctl put /devices/$DEVICE_ID/heartbeat "$(date)" --lease=$LEASE

# Background process keeps lease alive
while true; do
  etcdctl lease keep-alive $LEASE
  sleep 60
done &

# Watch for device disconnections
etcdctl watch /devices/ --prefix | \
  while read event; do
    if [[ $event == DELETE* ]]; then
      # Device lost connection
      alert_ops $event
    fi
  done
```

---

### Use Case 5: Disaster Recovery Test - Major Outage

**Scenario:**
```
Incident: Data center fire
Impact: All 3 etcd nodes destroyed
Time: Friday 2 PM
Recovery: From off-site backup
```

**Timeline:**
```
14:00 - Fire alarm, DC evacuated
14:15 - Confirmed total loss
14:20 - DR team activated
14:25 - Retrieved latest backup (13:00 snapshot)
14:30 - Provisioned new infrastructure (cloud)
15:00 - Restored etcd from backup
15:15 - Rebuilt control plane
15:30 - Validated data integrity
16:00 - Production workloads migrated
16:30 - All services operational

Total downtime: 2.5 hours
Data loss: 1 hour (14:00-15:00 changes)
```

**Post-Mortem Actions:**
```
Improvements:
1. Added real-time replication to cloud
2. Increased backup frequency (hourly → every 15 min)
3. Automated DR drills (monthly)
4. Multi-region deployment
5. Updated RTO/RPO targets

New Architecture:
- Primary: On-prem (5 nodes)
- DR: Cloud (3 nodes, hot standby)
- Sync: Real-time via VPN
- Failover: Automated (< 5 min)
```

---

## 17. Common Mistakes and Pitfalls

### Mistake 1: Even Number of Nodes

```
❌ WRONG: 2, 4, or 6 node clusters

Why it's bad:
- 2 nodes: No fault tolerance (worse than 1)
- 4 nodes: Same tolerance as 3 (wasted resources)
- 6 nodes: Same tolerance as 5 (wasted resources)

Failure tolerance:
1 node: 0 failures
2 nodes: 0 failures (WORSE: split-brain risk!)
3 nodes: 1 failure
4 nodes: 1 failure (same as 3!)
5 nodes: 2 failures
6 nodes: 2 failures (same as 5!)

✓ CORRECT: Always use odd numbers (1, 3, 5, 7)
```

---

### Mistake 2: Ignoring Disk Performance

```
❌ WRONG: Using spinning disks or slow SSDs

Symptoms:
- Frequent "apply request took too long" warnings
- Leader elections
- Timeouts
- Cluster instability

Real example:
# Before (HDD):
etcd_disk_wal_fsync_duration_seconds{quantile="0.99"} 0.523
# 523ms fsync = TERRIBLE

# After (NVMe SSD):
etcd_disk_wal_fsync_duration_seconds{quantile="0.99"} 0.003
# 3ms fsync = GOOD

✓ CORRECT:
- Use NVMe SSD (best)
- Or high-performance SATA SSD (acceptable)
- Dedicated disk for etcd
- Test with fio before deployment
```

---

### Mistake 3: Not Backing Up Regularly

```
❌ WRONG: "We'll back up when we remember"

Real disaster story:
- Company ran etcd for 2 years
- Never tested backups
- Accidental `kubectl delete` of critical namespace
- No recent backup
- Lost 6 months of configuration
- 3 days to manually recreate

✓ CORRECT:
- Automated backups every 6 hours
- Test restores monthly
- Multiple backup locations
- Off-site storage
```

**Proper backup script:**
```bash
#!/bin/bash
# Production backup script
set -euo pipefail

BACKUP_DIR="/var/backups/etcd"
S3_BUCKET="s3://company-etcd-backups"
RETENTION_DAYS=30

# Create backup
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="$BACKUP_DIR/etcd-$TIMESTAMP.db"

etcdctl snapshot save "$BACKUP_FILE"
gzip "$BACKUP_FILE"

# Upload to S3
aws s3 cp "$BACKUP_FILE.gz" "$S3_BUCKET/"

# Verify remotely
aws s3 ls "$S3_BUCKET/etcd-$TIMESTAMP.db.gz"

# Clean old backups
find "$BACKUP_DIR" -name "*.db.gz" -mtime +$RETENTION_DAYS -delete
aws s3 ls "$S3_BUCKET/" | \
  awk '{print $4}' | \
  while read file; do
    # Delete files older than retention
  done

# Test restore (monthly, separate cluster)
if [ "$(date +%d)" = "01" ]; then
  echo "Monthly restore test..."
  # Test restore to dev cluster
fi
```

---

### Mistake 4: Forgetting Encryption

```
❌ WRONG: Secrets stored in plain text

Risk:
- Anyone with etcd access can read secrets
- Backups contain plain-text secrets
- Disk theft = data breach

Example:
etcdctl get /registry/secrets/default/db-password
# Outputs: {"kind":"Secret","data":{"password":"cGFzc3dvcmQxMjM="}}
echo "cGFzc3dvcmQxMjM=" | base64 -d
# password123 <- VISIBLE!

✓ CORRECT: Enable encryption at rest
```

---

### Mistake 5: Single Point of Failure

```
❌ WRONG: Single control plane node

Architecture:
┌─────────────────┐
│   Master Node   │
│  - API Server   │
│  - etcd         │  ← If this fails, ENTIRE cluster down
│  - Scheduler    │
│  - Controller   │
└─────────────────┘

✓ CORRECT: Multi-master setup

┌──────────┐  ┌──────────┐  ┌──────────┐
│ Master 1 │  │ Master 2 │  │ Master 3 │
│  + etcd  │  │  + etcd  │  │  + etcd  │
└──────────┘  └──────────┘  └──────────┘
     ↓             ↓             ↓
     └─────────────┴─────────────┘
          Load Balancer
```

---

### Mistake 6: Ignoring Compaction

```
❌ WRONG: Never compacting etcd

What happens:
1. Database grows infinitely
2. Hits quota (2GB default)
3. etcd enters read-only mode
4. Cluster stops accepting writes
5. PRODUCTION DOWN

Real metrics:
# Before compaction
etcd_mvcc_db_total_size_in_bytes 2147483648  # 2GB (at quota!)

# After compaction + defrag
etcd_mvcc_db_total_size_in_bytes 524288000   # 500MB

✓ CORRECT: Enable auto-compaction
- --auto-compaction-mode=periodic
- --auto-compaction-retention=1h

# Or schedule manual compaction
0 */6 * * * etcdctl compact $(etcdctl endpoint status --write-out="json" | jq -r '.[0].Status.header.revision') && etcdctl defrag
```

---

### Mistake 7: Co-locating with Heavy Workloads

```
❌ WRONG: Running etcd on same node as database/ML workload

Result:
- CPU contention → slow etcd
- Disk I/O contention → fsync latency
- Memory pressure → OOM kills
- etcd instability → entire cluster unstable

Example scenario:
Node running: etcd + PostgreSQL + Elasticsearch
- Postgres does heavy writes
- etcd fsync delayed
- Leader thinks followers are slow
- Unnecessary leader election
- Cluster instability

✓ CORRECT:
- Dedicated nodes for etcd
- Or at least: dedicated CPU cores (CPU pinning)
- Dedicated disk for etcd data
- Memory guarantees (no swap)
```

---

### Mistake 8: Not Monitoring

```
❌ WRONG: "etcd just works, no need to monitor"

Reality:
- Silent degradation over months
- One day: catastrophic failure
- No warning signs captured
- No historical data to debug

Must-have alerts:
1. etcd_server_has_leader == 0
2. etcd_disk_wal_fsync_duration_seconds{quantile="0.99"} > 0.1
3. etcd_mvcc_db_total_size_in_bytes / quota > 0.8
4. rate(etcd_server_leader_changes_seen_total[1h]) > 3
5. up{job="etcd"} < 3

✓ CORRECT: Full observability stack
```

---

### Mistake 9: Manual Cluster Changes in Production

```
❌ WRONG: Ad-hoc etcdctl commands in prod

Disaster waiting to happen:
# Oops, wrong member ID!
etcdctl member remove abc123  # Removed wrong node
# Cluster now has 2 nodes, lost quorum!

# Or worse:
etcdctl del / --prefix  # Deleted EVERYTHING

✓ CORRECT:
- All changes via automation (Ansible, Terraform)
- Change approval process
- Tested in staging first
- Rollback plan ready
- Backup taken immediately before
```

---

### Mistake 10: Undersized Resources

```
❌ WRONG: "etcd is small, 1 CPU is enough"

Reality for 100+ node cluster:
Requests:
  cpu: 100m      ← WAY TOO LOW
  memory: 512Mi  ← WAY TOO LOW

Symptoms:
- CPU throttling
- OOM kills
- Slow responses
- Cluster instability

✓ CORRECT: Adequate resources

# Small cluster (< 100 nodes)
resources:
  requests:
    cpu: 1000m
    memory: 2Gi
  limits:
    cpu: 2000m
    memory: 4Gi

# Large cluster (500+ nodes)
resources:
  requests:
    cpu: 2000m
    memory: 8Gi
  limits:
    cpu: 4000m
    memory: 16Gi
```

---

### Mistake 11: Forgetting Network Segmentation

```
❌ WRONG: etcd accessible from anywhere

Risk:
- Direct access from workload pods
- Accidental or malicious deletion
- Data exfiltration
- DoS attacks

✓ CORRECT: Network policies

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: etcd-access
  namespace: kube-system
spec:
  podSelector:
    matchLabels:
      component: etcd
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          component: kube-apiserver
    ports:
    - protocol: TCP
      port: 2379
  - from:  # Peer communication
    - podSelector:
        matchLabels:
          component: etcd
    ports:
    - protocol: TCP
      port: 2380
```

---

### Mistake 12: Not Testing Disaster Recovery

```
❌ WRONG: Backups exist but never tested

Common failures during real DR:
- Backup corrupted
- Restore procedure outdated
- Certificates missing
- Team doesn't know steps
- Takes 10x longer than expected

Story:
- Company had automated backups for 3 years
- Major outage occurred
- Tried to restore
- Discovered backups were using wrong directory
- All backups were empty
- Result: 2 days of downtime

✓ CORRECT: Regular DR drills
- Monthly: Restore to test cluster
- Quarterly: Full DR simulation
- Document every step
- Measure and improve RTO/RPO
- Train entire team
```

---

## 18. Interview Questions with Answers

### Basic Level

**Q1: What is etcd and why does Kubernetes use it?**

**Answer:**
etcd is a distributed, reliable key-value store that stores all Kubernetes cluster data. Kubernetes uses it as the single source of truth for:
- Cluster state (pods, services, deployments)
- Configuration data (ConfigMaps, Secrets)
- Cluster membership
- Service discovery

Key features that make it suitable:
- Strong consistency (Raft consensus)
- High availability (distributed)
- Watch capability (real-time updates)
- Versioning (MVCC for history)

---

**Q2: Explain the difference between etcd v2 and v3.**

**Answer:**
| Feature | etcd v2 | etcd v3 |
|---------|---------|---------|
| API | HTTP/JSON | gRPC |
| Data model | Hierarchical (directories) | Flat key-value |
| Transactions | Limited | Full ACID |
| Watch | Per key | Range-based |
| Leases | TTL per key | Leases attach to multiple keys |
| Performance | Slower | Much faster |
| Current | Deprecated | Active |

Kubernetes uses etcd v3. Always set `ETCDCTL_API=3`.

---

**Q3: What is the Raft consensus algorithm?**

**Answer:**
Raft is the algorithm etcd uses to maintain consistency across cluster members. It ensures all nodes agree on the same data even during failures.

Key concepts:
1. **Leader election**: One node is elected leader
2. **Log replication**: Leader replicates writes to followers
3. **Quorum**: Majority must confirm before commit
4. **Term**: Each election increases term number

Process:
```
Client → Leader: Write request
Leader → Followers: Replicate entry
Followers → Leader: Acknowledge
Leader commits when quorum reached (N/2 + 1)
Leader → Client: Success
```

---

### Intermediate Level

**Q4: How do you backup and restore etcd in a Kubernetes cluster?**

**Answer:**

**Backup:**
```bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**Restore:**
```bash
# 1. Stop kube-apiserver and etcd
sudo mv /etc/kubernetes/manifests/{etcd,kube-apiserver}.yaml /tmp/

# 2. Restore snapshot
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd.db \
  --data-dir=/var/lib/etcd-restored

# 3. Update etcd.yaml with new data directory
# 4. Start services
sudo mv /tmp/{etcd,kube-apiserver}.yaml /etc/kubernetes/manifests/
```

---

**Q5: What happens if etcd loses quorum?**

**Answer:**
When quorum is lost (less than N/2 + 1 nodes available):

**Behavior:**
- No new leader can be elected
- All writes are rejected
- Cluster enters read-only mode
- Kubernetes API server can't commit changes

**Example (5-node cluster):**
```
Quorum needed: 3
If 3 nodes fail:
  Remaining: 2 nodes
  Cannot form quorum
  Cluster unavailable
```

**Recovery:**
1. Restore failed nodes to reach quorum, OR
2. Force new cluster from snapshot (data loss risk)

**Prevention:**
- Use odd numbers (3, 5, 7)
- Monitor member health
- Automated alerting
- Regular backups

---

**Q6: Explain etcd's MVCC and how Kubernetes uses it.**

**Answer:**
**MVCC (Multi-Version Concurrency Control)** maintains history of all changes:

```
Timeline:
Revision 1: PUT /pods/nginx {"image": "nginx:1.14"}
Revision 2: PUT /pods/nginx {"image": "nginx:1.15"}
Revision 3: PUT /pods/nginx {"image": "nginx:1.16"}
Revision 4: DELETE /pods/nginx

Current state: key doesn't exist
History: All revisions preserved (until compaction)
```

**Kubernetes usage:**
1. **Optimistic Concurrency:** ResourceVersion in every object
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     resourceVersion: "12345"  # From etcd revision
   ```
2. **Watch from revision:** Resume watches after disconnect
3. **List with consistency:** `kubectl get pods --resource-version=0`

---

### Advanced Level

**Q7: How would you troubleshoot slow etcd performance?**

**Answer:**

**Step-by-step diagnosis:**

1. **Check disk I/O:**
```bash
# Fsync latency (should be < 10ms)
curl -k https://localhost:2379/metrics | grep fsync_duration
```

2. **Network latency:**
```bash
# Between nodes (should be < 10ms)
for node in etcd-{1,2,3}; do
  ping -c 100 $node | tail -1
done
```

3. **Database size:**
```bash
etcdctl endpoint status --write-out=table
# If approaching quota → compact + defrag
```

4. **CPU/Memory:**
```bash
top -p $(pgrep etcd)
# Check for CPU throttling or memory pressure
```

5. **Leader stability:**
```bash
# Check for frequent elections
curl -k https://localhost:2379/metrics | grep leader_changes
```

**Common fixes:**
- Upgrade to NVMe SSD (if fsync > 100ms)
- Enable auto-compaction
- Increase resources
- Reduce watch caches in kube-apiserver
- Move to dedicated nodes

---

**Q8: Explain etcd security best practices.**

**Answer:**

**1. TLS Everywhere:**
```yaml
# Client-server TLS
--client-cert-auth=true
--trusted-ca-file=/path/to/ca.crt
--cert-file=/path/to/server.crt
--key-file=/path/to/server.key

# Peer TLS
--peer-client-cert-auth=true
--peer-trusted-ca-file=/path/to/ca.crt
--peer-cert-file=/path/to/peer.crt
--peer-key-file=/path/to/peer.key
```

**2. Authentication:**
```bash
etcdctl auth enable
etcdctl user add admin
etcdctl role add readonly
etcdctl role grant-permission readonly read / --prefix=true
```

**3. Encryption at Rest:**
```yaml
# Kubernetes EncryptionConfiguration
resources:
  - resources: [secrets]
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: <base64-key>
```

**4. Network Segmentation:**
- Firewall rules (only API server → etcd)
- Network policies
- Private network for etcd cluster

**5. Access Control:**
- Least privilege principle
- Audit logging
- Regular credential rotation

---

**Q9: How do you migrate from a 3-node to a 5-node etcd cluster?**

**Answer:**

**Step 1: Add first new member**
```bash
# On existing cluster
etcdctl member add etcd-4 --peer-urls=https://10.0.0.4:2380

# Output shows environment variables for new node
ETCD_NAME="etcd-4"
ETCD_INITIAL_CLUSTER="etcd-1=https://10.0.0.1:2380,etcd-2=https://10.0.0.2:2380,etcd-3=https://10.0.0.3:2380,etcd-4=https://10.0.0.4:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"
```

**Step 2: Start new member**
```bash
# On new node (etcd-4)
# Configure with provided variables
etcd \
  --name=etcd-4 \
  --initial-cluster="etcd-1=...,etcd-2=...,etcd-3=...,etcd-4=..." \
  --initial-cluster-state=existing \
  --initial-advertise-peer-urls=https://10.0.0.4:2380 \
  ...
```

**Step 3: Wait for sync**
```bash
# Verify member joined
etcdctl member list
# Wait for it to catch up
etcdctl endpoint status --cluster
```

**Step 4: Repeat for 5th node**
```bash
etcdctl member add etcd-5 --peer-urls=https://10.0.0.5:2380
# Start etcd-5 with "existing" state
```

**Important:**
- Add one at a time
- Wait for full sync before adding next
- Update kube-apiserver endpoints
- New quorum: 3 → can tolerate 2 failures

---

**Q10: Design an etcd setup for a 10,000-node Kubernetes cluster.**

**Answer:**

**Architecture:**

1. **External etcd cluster (separate from masters):**
```
7 etcd nodes (dedicated hardware)
- Tolerates 3 failures
- Better isolation
- Independent scaling
```

2. **Per-node specs:**
```
CPU: 8 cores (dedicated)
Memory: 32 GB
Disk: 2TB NVMe SSD (dedicated, RAID 10)
Network: 25 Gbps, < 5ms RTT
```

3. **Configuration:**
```yaml
--quota-backend-bytes=17179869184  # 16GB
--auto-compaction-mode=periodic
--auto-compaction-retention=15m
--snapshot-count=5000
--heartbeat-interval=100
--election-timeout=1000
```

4. **Separate event storage:**
```
Dedicated 3-node etcd for events
Reduces main cluster load
Events are high-volume, low-value
```

5. **Monitoring:**
```
- Prometheus with 1min scrape interval
- Grafana dashboards
- PagerDuty integration
- SLO: 99.99% availability, p99 < 20ms
```

6. **Backup:**
```
- Every 30 minutes
- Retention: 30 days local, 1 year S3
- Cross-region replication
- Automated restore testing
```

7. **Scaling strategy:**
```
- Horizontal: Can't go past 7 nodes (diminishing returns)
- Vertical: Upgrade hardware as needed
- Optimization: Aggressive compaction, watch tuning
- Alternative: Shard cluster by function (not recommended)
```

**Expected performance:**
- Write throughput: 10,000 ops/sec
- Read throughput: 100,000 ops/sec
- p99 latency: < 20ms
- Failure recovery: < 10 seconds

---

## 19. Comparison with Similar Technologies

### etcd vs Consul

```
┌─────────────────┬──────────────────┬──────────────────┐
│    Feature      │      etcd        │     Consul       │
├─────────────────┼──────────────────┼──────────────────┤
│ Primary Use     │ Config storage   │ Service mesh     │
│ Consensus       │ Raft             │ Raft             │
│ Data Model      │ Key-value        │ Key-value + DNS  │
│ Service Disc.   │ Via watches      │ Built-in         │
│ Health Checks   │ Manual           │ Built-in         │
│ Multi-DC        │ Not native       │ Native           │
│ UI              │ No               │ Yes              │
│ K8s Integration │ Native (control) │ Add-on           │
│ Use Case        │ Cluster state    │ Microservices    │
└─────────────────┴──────────────────┴──────────────────┘
```

**When to use etcd:**
- Kubernetes cluster (required)
- Need strong consistency guarantees
- Simple key-value storage
- High write throughput

**When to use Consul:**
- Service mesh with observability
- Cross-datacenter replication
- DNS-based service discovery
- Built-in health checking

---

### etcd vs ZooKeeper

```
┌─────────────────┬──────────────────┬──────────────────┐
│    Feature      │      etcd        │    ZooKeeper     │
├─────────────────┼──────────────────┼──────────────────┤
│ Consensus       │ Raft             │ ZAB (Paxos-like) │
│ API             │ gRPC, HTTP       │ Custom protocol  │
│ Language        │ Go               │ Java             │
│ Setup           │ Simple           │ Complex          │
│ Learning Curve  │ Easy             │ Steep            │
│ Performance     │ Fast             │ Fast             │
│ Watches         │ Efficient        │ Less efficient   │
│ Transactions    │ Full ACID        │ Limited          │
│ Memory          │ Lower            │ Higher (JVM)     │
└─────────────────┴──────────────────┴──────────────────┘
```

**ZooKeeper advantages:**
- Battle-tested (20+ years)
- Hadoop ecosystem integration
- More mature tooling

**etcd advantages:**
- Simpler to understand and operate
- Better performance for small values
- Native to cloud-native ecosystem
- Smaller resource footprint

---

### etcd vs Redis (with persistence)

```
┌─────────────────┬──────────────────┬──────────────────┐
│    Feature      │      etcd        │      Redis       │
├─────────────────┼──────────────────┼──────────────────┤
│ Consistency     │ Strong (CP)      │ Eventual (AP)    │
│ Data Structures │ Key-value        │ Rich (lists,     │
│                 │                  │ sets, sorted,etc)│
│ Persistence     │ Always           │ Optional         │
│ Distribution    │ Built-in         │ Redis Cluster    │
│ Performance     │ Good             │ Excellent        │
│ Use Case        │ Config, state    │ Cache, queue     │
│ Watch           │ Native           │ Pub/sub          │
│ Transactions    │ Yes              │ Limited          │
└─────────────────┴──────────────────┴──────────────────┘
```

**When to use Redis:**
- Caching layer
- Session storage
- Real-time analytics
- Message queues
- High-performance reads (millions/sec)

**When to use etcd:**
- Distributed configuration
- Leader election
- Distributed locks
- Service discovery
- Cluster coordination

**Can they coexist?**
Yes! Common pattern:
- etcd: Cluster state, configuration
- Redis: Application caching, sessions

---

### etcd vs Database (PostgreSQL)

```
┌─────────────────┬──────────────────┬──────────────────┐
│    Feature      │      etcd        │   PostgreSQL     │
├─────────────────┼──────────────────┼──────────────────┤
│ Model           │ Key-value        │ Relational       │
│ Query Language  │ Key/prefix       │ SQL              │
│ Consistency     │ Strong           │ Strong           │
│ Transactions    │ Simple           │ Complex          │
│ Distribution    │ Native           │ Extensions       │
│ Watch           │ Built-in         │ LISTEN/NOTIFY    │
│ Size Limit      │ ~10 GB (typical) │ Terabytes        │
│ Use Case        │ Coordination     │ Application data │
└─────────────────┴──────────────────┴──────────────────┘
```

**Don't use etcd as a database!**

❌ Bad use of etcd:
- Storing large datasets
- Complex queries
- Relationships between entities
- Analytics workloads

✓ Correct use:
- Small, critical configuration
- Cluster coordination
- Distributed locks
- Leader election

---

## 20. Summary Cheat Sheet

### Essential Commands

```bash
# Setup
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key

# Health & Status
etcdctl endpoint health --cluster
etcdctl endpoint status --cluster --write-out=table
etcdctl member list --write-out=table

# CRUD Operations
etcdctl put /key "value"
etcdctl get /key
etcdctl get /prefix/ --prefix
etcdctl del /key

# Backup & Restore
etcdctl snapshot save backup.db
etcdctl snapshot restore backup.db --data-dir=/new/path

# Maintenance
etcdctl compact <revision>
etcdctl defrag --cluster
etcdctl alarm list

# Cluster Management
etcdctl member add <name> --peer-urls=<url>
etcdctl member remove <id>
etcdctl member update <id> --peer-urls=<url>
```

---

### Key Concepts

```
Raft Consensus:
- Leader election
- Log replication  
- Quorum (N/2 + 1)

Cluster Sizes:
- 3 nodes: 1 failure tolerance
- 5 nodes: 2 failures tolerance
- 7 nodes: 3 failures tolerance (max recommended)

Performance Requirements:
- Disk: > 3000 IOPS, < 10ms latency
- Network: < 10ms RTT between nodes
- CPU: 2-4 cores per node
- Memory: 4-16 GB per node

Data Model:
- Keys: Byte strings (UTF-8)
- Values: Byte strings (max 1.5 MB)
- Revisions: MVCC history
- Leases: TTL mechanism
```

---

### Configuration Flags

```yaml
# Essential flags
--data-dir=/var/lib/etcd
--name=<node-name>
--initial-cluster=node1=https://...,node2=https://...
--listen-client-urls=https://0.0.0.0:2379
--advertise-client-urls=https://<IP>:2379
--listen-peer-urls=https://0.0.0.0:2380
--initial-advertise-peer-urls=https://<IP>:2380

# Security
--client-cert-auth=true
--trusted-ca-file=<ca.crt>
--cert-file=<server.crt>
--key-file=<server.key>

# Performance
--snapshot-count=10000
--quota-backend-bytes=8589934592  # 8GB
--auto-compaction-retention=1h
--heartbeat-interval=100
--election-timeout=1000
```

---

### Monitoring Metrics

```promql
# Critical metrics
etcd_server_has_leader              # 0 = no leader
etcd_disk_wal_fsync_duration_seconds{quantile="0.99"}  # < 0.1
etcd_mvcc_db_total_size_in_bytes    # Watch for quota
etcd_server_leader_changes_seen_total  # Low = stable
etcd_network_peer_round_trip_time_seconds  # < 0.05
```

---

### Troubleshooting Quick Reference

```
Problem: "database space exceeded"
Fix: etcdctl compact <rev> && etcdctl defrag

Problem: "request timed out"
Check: Disk fsync latency, network RTT
Fix: Upgrade disk, reduce timeouts

Problem: "no leader"
Check: Network connectivity, quorum
Fix: Restore connectivity, check logs

Problem: Frequent leader changes
Check: Heartbeat interval, network stability
Fix: Increase timeouts, fix network

Problem: "all endpoints unhealthy"
Check: TLS certificates, firewall
Fix: Renew certs, open ports
```

---

### Security Checklist

```
□ TLS enabled for client communication
□ TLS enabled for peer communication
□ Client certificate authentication enabled
□ Encryption at rest configured
□ etcd authentication enabled (RBAC)
□ Network policies restrict access
□ Firewall rules configured
□ Certificates rotated regularly
□ Audit logging enabled
□ Access monitored and alerted
```

---

### Backup & DR Checklist

```
□ Automated backups every 6 hours
□ Backups stored locally + remotely
□ Backup retention policy defined
□ Restore tested monthly
□ DR runbook documented
□ Team trained on procedures
□ RTO/RPO defined and measured
□ Certificates backed up
□ Configuration files backed up
```

---

### Production Checklist

```
□ 3 or 5 node cluster (odd number)
□ NVMe SSD storage (tested with fio)
□ Dedicated nodes or resources
□ Network latency < 10ms between nodes
□ Auto-compaction enabled
□ Quotas set appropriately
□ Monitoring and alerting configured
□ Backup automation running
□ Security hardened (TLS, auth, firewall)
□ Documentation complete
□ Team trained
□ DR plan tested
```

---

### Quick Architecture Decision Tree

```
How many nodes?
├─ Dev/Test → 1 node
├─ Small prod → 3 nodes
├─ Standard prod → 5 nodes
└─ Large prod → 7 nodes (max)

Topology?
├─ < 100 nodes → Stacked (etcd on masters)
└─ > 100 nodes → External (dedicated etcd cluster)

Storage?
├─ Dev → Any SSD
└─ Prod → NVMe SSD, dedicated, > 3K IOPS

Resources per node?
├─ < 100 K8s nodes → 2 CPU, 4-8 GB RAM
├─ 100-500 K8s nodes → 4 CPU, 8-16 GB RAM
└─ > 500 K8s nodes → 8 CPU, 16-32 GB RAM

Backup frequency?
├─ Dev → Daily
├─ Prod → Every 6 hours
└─ Critical → Every 1-2 hours
```

---

**This concludes the comprehensive etcd guide from beginner to production expert level!**

Key takeaways:
1. etcd is the brain of Kubernetes - treat it with care
2. Always use odd-numbered clusters (3, 5, or 7 nodes)
3. Fast disks are non-negotiable (NVMe SSD)
4. Backup religiously and test restores regularly
5. Monitor everything - don't wait for failure
6. Security must be layered (TLS + Auth + Encryption + Network)
7. Plan for disaster - it's not if, but when

Master these concepts and you'll be well-equipped to run production Kubernetes clusters with confidence!
