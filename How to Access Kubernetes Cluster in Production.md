## Table of Contents

1. [Introduction — Why Cluster Access Is a Critical Production Concern](#1-introduction--why-cluster-access-is-a-critical-production-concern)
2. [Core Identity Table — Access-Related Components](#2-core-identity-table--access-related-components)
3. [The Kubernetes Access Architecture — Request Lifecycle](#3-the-kubernetes-access-architecture--request-lifecycle)
4. [The Controller Pattern — Watch → Compare → Act → Loop](#4-the-controller-pattern--watch--compare--act--loop)
5. [kubeconfig — The Gateway to Every Cluster](#5-kubeconfig--the-gateway-to-every-cluster)
6. [Authentication Methods — Who Are You?](#6-authentication-methods--who-are-you)
7. [Authorization — What Can You Do? (RBAC Deep Dive)](#7-authorization--what-can-you-do-rbac-deep-dive)
8. [Admission Controllers — The Final Gate](#8-admission-controllers--the-final-gate)
9. [kubectl — Primary Access Tool](#9-kubectl--primary-access-tool)
10. [Secure Remote Access Patterns](#10-secure-remote-access-patterns)
11. [Service Account Access — In-Cluster Applications](#11-service-account-access--in-cluster-applications)
12. [Multi-Cluster Access Management](#12-multi-cluster-access-management)
13. [Built-in Controllers Relevant to Access Management](#13-built-in-controllers-relevant-to-access-management)
14. [Informers, Work Queues & Reconciliation in Access Control](#14-informers-work-queues--reconciliation-in-access-control)
15. [API Server and etcd Interaction — The Access Enforcement Layer](#15-api-server-and-etcd-interaction--the-access-enforcement-layer)
16. [Leader Election and HA for Access Infrastructure](#16-leader-election-and-ha-for-access-infrastructure)
17. [Performance Tuning for High-Traffic API Access](#17-performance-tuning-for-high-traffic-api-access)
18. [Security Hardening — Zero-Trust Access Practices](#18-security-hardening--zero-trust-access-practices)
19. [Monitoring & Observability for Cluster Access](#19-monitoring--observability-for-cluster-access)
20. [Troubleshooting Access Issues — Real kubectl Commands](#20-troubleshooting-access-issues--real-kubectl-commands)
21. [Disaster Recovery — Access During Failure Scenarios](#21-disaster-recovery--access-during-failure-scenarios)
22. [Comparison: Direct API vs kubectl vs Dashboard vs GitOps](#22-comparison-direct-api-vs-kubectl-vs-dashboard-vs-gitops)
23. [ASCII Architecture Diagram — Complete Access Flow](#23-ascii-architecture-diagram--complete-access-flow)
24. [Real-World Production Use Cases](#24-real-world-production-use-cases)
25. [Best Practices for Production Access Management](#25-best-practices-for-production-access-management)
26. [Common Mistakes and Pitfalls](#26-common-mistakes-and-pitfalls)
27. [Hands-On Labs — Practical Access Exercises](#27-hands-on-labs--practical-access-exercises)
28. [Interview Questions — Beginner to Advanced](#28-interview-questions--beginner-to-advanced)
29. [Cheat Sheet — Commands, Flags & Configs](#29-cheat-sheet--commands-flags--configs)
30. [Key Takeaways & Summary](#30-key-takeaways--summary)

---

## 1. Introduction — Why Cluster Access Is a Critical Production Concern

Kubernetes cluster access is not merely a convenience feature — it is one of the most consequential security and operational concerns in your entire platform. A misconfigured access model can allow a single compromised credential to delete every workload, exfiltrate every secret, and render a cluster irrecoverable. Conversely, an overly restrictive access model creates operational bottlenecks, slows incident response, and forces engineers to work around controls in unsafe ways.

**Production cluster access** encompasses:
- **Who** can reach the Kubernetes API server (network access)
- **How** they authenticate (identity verification)
- **What** they are permitted to do (authorization)
- **What** is validated or mutated before their request succeeds (admission)
- **When** and how access is granted, rotated, and revoked (lifecycle)
- **How** all access is recorded for audit (observability)

### Why This Topic Demands Its Own Guide

In development environments, most teams use a single kubeconfig with admin credentials, a locally accessible API server, and no meaningful access controls. This approach is completely unacceptable in production for several reasons:

| Concern | Risk Without Proper Access Management |
|---|---|
| **Credential exposure** | A leaked kubeconfig with cluster-admin grants full cluster takeover |
| **Blast radius** | No RBAC means a buggy script can delete all namespaces |
| **Audit trail** | Without audit logging, you can't answer "who deleted that deployment?" |
| **Compliance** | PCI DSS, SOC 2, HIPAA, and ISO 27001 require access controls and logs |
| **Insider threat** | Without RBAC, any team member can access any other team's secrets |
| **Automation abuse** | CI/CD service accounts with admin privileges are high-value attack targets |

### The Three Pillars of Production Access

```
┌─────────────────────────────────────────────────────────────┐
│                 PRODUCTION CLUSTER ACCESS                   │
│                                                             │
│   ┌─────────────────┐  ┌─────────────────┐  ┌───────────┐  │
│   │  AUTHENTICATION │  │  AUTHORIZATION  │  │ ADMISSION │  │
│   │                 │  │                 │  │           │  │
│   │  Who are you?   │  │  What can you   │  │  Is this  │  │
│   │  Prove your     │  │  do? RBAC       │  │  request  │  │
│   │  identity.      │  │  enforces       │  │  valid?   │  │
│   │                 │  │  permissions.   │  │           │  │
│   └─────────────────┘  └─────────────────┘  └───────────┘  │
│         AuthN                AuthZ              Admission   │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Core Identity Table — Access-Related Components

| Component | Binary/Resource | Port/Location | Protocol | Role in Access |
|---|---|---|---|---|
| **API Server** | `kube-apiserver` | 6443 | HTTPS | Central access enforcement: AuthN + AuthZ + Admission |
| **etcd** | `etcd` | 2379 | HTTPS/gRPC | Stores RBAC rules, ServiceAccount tokens, audit state |
| **kubelet** | `kubelet` | 10250 | HTTPS | Node-level access; exec/logs/portforward endpoints |
| **kubectl** | `kubectl` CLI | N/A | HTTPS client | Primary human access tool |
| **kubeconfig** | `~/.kube/config` | N/A | YAML file | Credential and context configuration |
| **ServiceAccount** | `serviceaccount` resource | N/A | K8s resource | In-cluster workload identity |
| **OIDC Provider** | External (Dex, Okta, etc.) | 443/HTTPS | OIDC/OAuth2 | SSO-based human identity |
| **Webhook Auth** | External service | Configurable | HTTPS | External authentication decisions |
| **RBAC Role** | `role`, `clusterrole` | N/A | K8s resource | Permission definition |
| **RBAC Binding** | `rolebinding`, `clusterrolebinding` | N/A | K8s resource | Permission assignment |
| **Admission Webhook** | External service | HTTPS | Webhook | Mutation/validation of requests |
| **Audit Log** | File / webhook | Configurable | N/A | Access audit trail |
| **Secrets** | `secret` resource (type:token) | N/A | K8s resource | ServiceAccount token storage |
| **Certificate** | PKI: `/etc/kubernetes/pki/` | N/A | TLS | Component-level authentication |
| **TokenRequest API** | API endpoint | 6443 | HTTPS | Bound service account tokens |

---

## 3. The Kubernetes Access Architecture — Request Lifecycle

Every access request — whether from `kubectl`, a CI/CD system, an in-cluster application, or the Kubernetes dashboard — travels through a **deterministic, ordered pipeline** enforced by the kube-apiserver.

### 3.1 The Five-Stage Request Pipeline

```
Client (kubectl / app / CI-CD)
        │
        │ HTTPS request
        ▼
┌──────────────────────────────────────────────────────────────┐
│                     kube-apiserver                           │
│                                                              │
│  Stage 1: TRANSPORT SECURITY                                 │
│  ─────────────────────────────                               │
│  TLS termination; client certificate validation              │
│                    │                                         │
│                    ▼                                         │
│  Stage 2: AUTHENTICATION (AuthN)                             │
│  ───────────────────────────────                             │
│  "Who are you?"                                              │
│  Methods: client cert, bearer token, OIDC, webhook          │
│  Result: username, groups, extra attributes                  │
│                    │                                         │
│                    ▼                                         │
│  Stage 3: AUTHORIZATION (AuthZ)                              │
│  ─────────────────────────────                               │
│  "Are you allowed to do this?"                               │
│  Methods: RBAC, ABAC, Webhook, Node                         │
│  Result: Allow or Deny                                       │
│                    │                                         │
│                    ▼                                         │
│  Stage 4: ADMISSION CONTROL                                  │
│  ──────────────────────────                                  │
│  "Is this request valid/safe?"                               │
│  Mutating webhooks → Validating webhooks                     │
│  Result: Mutated request / Allow / Deny                      │
│                    │                                         │
│                    ▼                                         │
│  Stage 5: PERSISTENCE & RESPONSE                             │
│  ────────────────────────────────                            │
│  Write to etcd (if mutating) → Return response               │
│                                                              │
└──────────────────────────────────────────────────────────────┘
        │
        ▼
      etcd (state storage — only reachable by API server)
```

### 3.2 What Happens If Any Stage Fails

| Stage | Failure | HTTP Response |
|---|---|---|
| TLS | Invalid cert / expired cert | 400 Bad Request or connection refused |
| Authentication | Invalid token / expired cert | 401 Unauthorized |
| Authorization | RBAC denial | 403 Forbidden |
| Admission | Validation webhook rejects | 400/422 with reason |
| Persistence | etcd write fails | 500 Internal Server Error |

---

## 4. The Controller Pattern — Watch → Compare → Act → Loop

Understanding the controller pattern is essential for understanding access management in Kubernetes, because RBAC itself is managed by controllers that follow this exact pattern. When you create a `RoleBinding`, a controller watches that event, compares the desired permissions state to the current state, and acts to enforce it.

### 4.1 The Reconciliation Loop Applied to Access

```
┌─────────────────────────────────────────────────────────────┐
│           ACCESS CONTROL RECONCILIATION LOOP                │
│                                                             │
│   ┌──────────┐                                              │
│   │  WATCH   │◄── ServiceAccount created                    │
│   │          │◄── RoleBinding modified                      │
│   │          │◄── ClusterRole updated                       │
│   └────┬─────┘                                              │
│        │ Event queued                                        │
│        ▼                                                     │
│   ┌──────────────┐                                          │
│   │   COMPARE    │  Desired: spec.rules / spec.subjects     │
│   │              │  Current: enforced permissions in API    │
│   └──────┬───────┘                                          │
│          │ Delta found                                       │
│          ▼                                                   │
│   ┌──────────┐                                              │
│   │   ACT    │  Update RBAC rules in etcd via API server   │
│   │          │  Provision ServiceAccount token             │
│   │          │  Revoke expired tokens                      │
│   └────┬─────┘                                              │
│        │                                                    │
│        └─────────────► LOOP                                 │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 RBAC Controller — Keeping Access In Sync

The RBAC authorization module in the API server is **not a controller** in the traditional sense — it evaluates permissions at request time from its in-memory cache. However, the ServiceAccount controller (part of kube-controller-manager) **does** follow the controller pattern to:

- Create ServiceAccount tokens when a ServiceAccount is created
- Delete tokens when a ServiceAccount is deleted
- Manage token rotation and expiry

```bash
# Observe the ServiceAccount controller's work
kubectl create serviceaccount my-app -n production
kubectl get secrets -n production | grep my-app   # Token created automatically
kubectl describe serviceaccount my-app -n production
```

---

## 5. kubeconfig — The Gateway to Every Cluster

The `kubeconfig` file is the **primary credential artifact** for kubectl and other Kubernetes clients. It defines clusters, users (credentials), and contexts (cluster + user + namespace combinations).

### 5.1 kubeconfig Structure

```yaml
# ~/.kube/config — Full annotated structure
apiVersion: v1
kind: Config

# ──────────────────────────────────────────────────────────────
# CLUSTERS: Define the API server endpoints
# ──────────────────────────────────────────────────────────────
clusters:
  - name: production-cluster
    cluster:
      server: https://k8s-api.production.example.com:6443
      # Option A: CA cert as base64-encoded data (portable)
      certificate-authority-data: <base64-encoded-ca.crt>
      # Option B: CA cert as file path (node-specific)
      # certificate-authority: /etc/kubernetes/pki/ca.crt

  - name: staging-cluster
    cluster:
      server: https://k8s-api.staging.example.com:6443
      certificate-authority-data: <base64-encoded-ca.crt>
      # insecure-skip-tls-verify: true  # NEVER IN PRODUCTION

# ──────────────────────────────────────────────────────────────
# USERS: Define authentication credentials
# ──────────────────────────────────────────────────────────────
users:
  # Method 1: Client certificate (X.509)
  - name: jane-doe
    user:
      client-certificate-data: <base64-encoded-jane.crt>
      client-key-data: <base64-encoded-jane.key>

  # Method 2: Bearer token (ServiceAccount or static)
  - name: ci-pipeline
    user:
      token: eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...

  # Method 3: OIDC (SSO integration)
  - name: oidc-user
    user:
      auth-provider:
        name: oidc
        config:
          idp-issuer-url: https://accounts.google.com
          client-id: kubernetes
          client-secret: OIDC_CLIENT_SECRET
          id-token: <current-id-token>
          refresh-token: <refresh-token>

  # Method 4: exec credential plugin (AWS EKS, GKE, etc.)
  - name: eks-user
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1beta1
        command: aws
        args:
          - eks
          - get-token
          - --cluster-name
          - production-cluster
          - --region
          - us-east-1

# ──────────────────────────────────────────────────────────────
# CONTEXTS: Bind a cluster + user + namespace
# ──────────────────────────────────────────────────────────────
contexts:
  - name: production
    context:
      cluster: production-cluster
      user: jane-doe
      namespace: team-a-production

  - name: production-readonly
    context:
      cluster: production-cluster
      user: readonly-user
      namespace: default

  - name: staging
    context:
      cluster: staging-cluster
      user: jane-doe
      namespace: default

# Active context (used when --context flag not specified)
current-context: staging
```

### 5.2 kubeconfig Management Commands

```bash
# View current context
kubectl config current-context

# List all contexts
kubectl config get-contexts

# Switch context
kubectl config use-context production

# View full kubeconfig (merged)
kubectl config view

# View with secrets revealed
kubectl config view --raw=true

# Set default namespace for current context
kubectl config set-context --current --namespace=production

# Add a new cluster to kubeconfig
kubectl config set-cluster production-cluster \
  --server=https://k8s-api.production.example.com:6443 \
  --certificate-authority=/path/to/ca.crt \
  --embed-certs=true   # Embeds cert in kubeconfig (portable)

# Add credentials
kubectl config set-credentials jane-doe \
  --client-certificate=/path/to/jane.crt \
  --client-key=/path/to/jane.key \
  --embed-certs=true

# Add context
kubectl config set-context production \
  --cluster=production-cluster \
  --user=jane-doe \
  --namespace=production

# Delete a context
kubectl config delete-context old-context

# Merge multiple kubeconfig files
KUBECONFIG=~/.kube/cluster1:~/.kube/cluster2 \
  kubectl config view --flatten > ~/.kube/config
```

### 5.3 kubeconfig Security Rules

| Rule | Rationale |
|---|---|
| Never commit kubeconfig to version control | Contains credentials; immediate rotation needed if exposed |
| Never use `insecure-skip-tls-verify: true` in production | Allows MITM attacks against the API server |
| Use short-lived tokens over long-lived | Reduces blast radius of credential theft |
| Restrict kubeconfig file permissions | `chmod 600 ~/.kube/config` |
| Separate kubeconfigs per environment | Prevent accidental production commands from staging context |
| Use exec credential plugins for cloud clusters | AWS EKS, GKE, AKS handle token refresh automatically |
| Never share kubeconfig files | Each person/system should have their own credentials |

### 5.4 kubeconfig Locations and Precedence

```bash
# Precedence order (highest to lowest):
# 1. --kubeconfig flag on the command line
# 2. KUBECONFIG environment variable (colon-separated list of files)
# 3. Default: ~/.kube/config

# Override location via flag
kubectl get pods --kubeconfig=/etc/kubernetes/admin.conf

# Override via environment variable
export KUBECONFIG=/home/jane/.kube/production.yaml
kubectl get nodes

# Merge multiple files (temporary override)
export KUBECONFIG=/path/to/config1:/path/to/config2
kubectl config get-contexts  # Shows contexts from both files
```

---

## 6. Authentication Methods — Who Are You?

Authentication in Kubernetes is **pluggable** — the API server supports multiple authentication methods simultaneously, and a request succeeds as long as **any one** of the configured methods validates it.

### 6.1 Authentication Methods Comparison

| Method | How It Works | Use Case | Security Level | Token Lifetime |
|---|---|---|---|---|
| **X.509 Client Certificates** | Client presents signed cert; API server validates against CA | Component auth, admin access | High | Until cert expiry |
| **Bearer Token (Static)** | Token in Authorization header; validated against static file | Legacy, emergency | Low | Until file updated |
| **ServiceAccount Tokens (Legacy)** | JWT stored in Secret; no expiry by default | In-cluster apps | Medium | Until secret deleted |
| **ServiceAccount Tokens (Bound)** | JWT with expiry, audience, pod binding | In-cluster apps (recommended) | High | Configurable (default 1h) |
| **OIDC** | External IdP issues JWT; API server validates | Human users (SSO) | High | Short-lived (refresh-based) |
| **Webhook Token Auth** | External service validates tokens | Custom auth systems | Varies | External service controls |
| **Bootstrap Tokens** | Special tokens for cluster bootstrapping | Node join | Medium | 24h default |
| **Proxy Auth** | Trusted proxy sets user/group headers | Frontend auth systems | High (if proxy trusted) | N/A |

### 6.2 X.509 Client Certificates

The most widely used authentication method for components and admins.

```bash
# Generate a client certificate for a new user "jane"
# Step 1: Generate private key
openssl genrsa -out jane.key 2048

# Step 2: Create Certificate Signing Request (CSR)
openssl req -new \
  -key jane.key \
  -out jane.csr \
  -subj "/CN=jane/O=developers"
  # CN = username in Kubernetes
  # O = group membership in Kubernetes

# Step 3: Sign with cluster CA
openssl x509 -req \
  -in jane.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out jane.crt \
  -days 365

# Step 4: Build kubeconfig for jane
kubectl config set-credentials jane \
  --client-certificate=jane.crt \
  --client-key=jane.key \
  --embed-certs=true

# Production Alternative: Use Kubernetes CertificateSigningRequest API
# (doesn't require direct CA key access)
cat jane.csr | base64 | tr -d '\n' > jane.csr.b64

cat << EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane-csr
spec:
  request: $(cat jane.csr.b64)
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 31536000  # 1 year
  usages:
    - client auth
EOF

# Approve the CSR (requires appropriate RBAC)
kubectl certificate approve jane-csr

# Retrieve the signed certificate
kubectl get csr jane-csr \
  -o jsonpath='{.status.certificate}' | \
  base64 -d > jane.crt
```

### 6.3 OIDC Integration (SSO for Human Users)

OIDC is the **recommended approach for human user authentication** in production. It integrates with existing identity providers (Google, Azure AD, Okta, Keycloak, Dex).

```
Flow:
User → OIDC Provider → Gets ID Token (JWT) → Presents to kubectl
kubectl → Sends ID Token to kube-apiserver
kube-apiserver → Validates token signature against OIDC issuer's JWKS
kube-apiserver → Extracts username from token claim (--oidc-username-claim)
kube-apiserver → Extracts groups from token claim (--oidc-groups-claim)
RBAC → Evaluates permissions for the user/groups
```

**API server OIDC flags:**

```yaml
# In kube-apiserver manifest or config
--oidc-issuer-url=https://accounts.google.com
--oidc-client-id=kubernetes
--oidc-username-claim=email          # Which JWT field = username
--oidc-username-prefix=oidc:         # Prefix to avoid name collisions
--oidc-groups-claim=groups           # Which JWT field = groups
--oidc-groups-prefix=oidc:
--oidc-required-claim=hd=example.com # Only allow example.com domain
```

**kubectl OIDC configuration:**

```bash
# Install kubelogin plugin (handles OIDC flow)
kubectl krew install oidc-login

# Configure kubeconfig to use OIDC
kubectl config set-credentials oidc-user \
  --exec-api-version=client.authentication.k8s.io/v1beta1 \
  --exec-command=kubectl \
  --exec-arg=oidc-login \
  --exec-arg=get-token \
  --exec-arg=--oidc-issuer-url=https://keycloak.example.com/auth/realms/k8s \
  --exec-arg=--oidc-client-id=kubectl \
  --exec-arg=--oidc-client-secret=client-secret
```

### 6.4 ServiceAccount Token Authentication

```bash
# Create a ServiceAccount for a CI/CD pipeline
kubectl create serviceaccount github-actions -n ci-cd

# Create a bound token (recommended — has expiry)
kubectl create token github-actions \
  --duration=8760h \  # 1 year
  -n ci-cd

# Legacy: Long-lived token via Secret (NOT recommended for production)
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: github-actions-token
  namespace: ci-cd
  annotations:
    kubernetes.io/service-account.name: github-actions
type: kubernetes.io/service-account-token
EOF

# Get the token value
kubectl get secret github-actions-token -n ci-cd \
  -o jsonpath='{.data.token}' | base64 -d

# Use token in kubeconfig
kubectl config set-credentials github-actions \
  --token=$(kubectl get secret github-actions-token -n ci-cd \
    -o jsonpath='{.data.token}' | base64 -d)
```

---

## 7. Authorization — What Can You Do? (RBAC Deep Dive)

Once authenticated, every request must be **authorized**. Kubernetes supports four authorization modes, but **RBAC (Role-Based Access Control)** is the industry standard for production clusters.

### 7.1 RBAC Building Blocks

```
┌─────────────────────────────────────────────────────────────┐
│                    RBAC COMPONENTS                          │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  ROLE / CLUSTERROLE                                 │   │
│  │  Defines: WHAT can be done                         │   │
│  │  ┌───────────────────────────────────────────────┐ │   │
│  │  │  Rules: [ { apiGroups, resources, verbs } ]   │ │   │
│  │  └───────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────┘   │
│                          +                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  ROLEBINDING / CLUSTERROLEBINDING                   │   │
│  │  Defines: WHO gets the role                        │   │
│  │  ┌───────────────────────────────────────────────┐ │   │
│  │  │  Subjects: [ Users, Groups, ServiceAccounts ] │ │   │
│  │  │  RoleRef:  → Points to a Role/ClusterRole     │ │   │
│  │  └───────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────┘   │
│                          =                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  PERMISSION: Subject X can do Verb on Resource Y   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 Role vs ClusterRole

| Aspect | Role | ClusterRole |
|---|---|---|
| **Scope** | Namespace-specific | Cluster-wide |
| **Can grant access to** | Namespaced resources in one namespace | Namespaced resources across all namespaces OR cluster-scoped resources |
| **Bound via** | RoleBinding (in same namespace) | ClusterRoleBinding OR RoleBinding (limits scope to namespace) |
| **Typical use** | Application-specific permissions | Admin roles, cross-namespace access |

### 7.3 Production RBAC Patterns

```yaml
# ──────────────────────────────────────────────────────────────
# Pattern 1: Developer read access in a specific namespace
# ──────────────────────────────────────────────────────────────
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-read
  namespace: team-a-production
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "services", "configmaps", "endpoints"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create"]   # Allow exec for debugging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-read-binding
  namespace: team-a-production
subjects:
  - kind: User
    name: jane@example.com   # OIDC email
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: oidc:team-a-developers   # OIDC group
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-read
  apiGroup: rbac.authorization.k8s.io
---
# ──────────────────────────────────────────────────────────────
# Pattern 2: CI/CD pipeline with deploy permissions only
# ──────────────────────────────────────────────────────────────
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
    verbs: ["update", "patch"]
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "create", "update", "patch"]
  # Explicitly NO: delete, secrets, clusterrole access
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: github-actions
    namespace: ci-cd
roleRef:
  kind: Role
  name: ci-deployer
  apiGroup: rbac.authorization.k8s.io
---
# ──────────────────────────────────────────────────────────────
# Pattern 3: SRE on-call with broad read access cluster-wide
# ──────────────────────────────────────────────────────────────
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: sre-oncall-reader
rules:
  - apiGroups: [""]
    resources: ["*"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps", "extensions", "networking.k8s.io", "batch"]
    resources: ["*"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/exec", "pods/portforward"]
    verbs: ["create"]   # Allow debugging
  - nonResourceURLs: ["/healthz", "/metrics", "/readyz"]
    verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sre-oncall-binding
subjects:
  - kind: Group
    name: oidc:sre-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: sre-oncall-reader
  apiGroup: rbac.authorization.k8s.io
```

### 7.4 Built-in RBAC Roles Reference

| ClusterRole | What It Grants | Appropriate For |
|---|---|---|
| `cluster-admin` | All verbs on all resources | Break-glass emergency only |
| `admin` | Most resources in a namespace | Namespace owners |
| `edit` | Create/update/delete most resources | Developers |
| `view` | Read-only on most resources | Observers, dashboards |
| `system:node` | Node-specific operations | kubelet |
| `system:kube-proxy` | kube-proxy operations | kube-proxy ServiceAccount |
| `system:coredns` | CoreDNS operations | CoreDNS ServiceAccount |

### 7.5 RBAC Testing and Auditing Commands

```bash
# Test what a specific user can do
kubectl auth can-i create deployments \
  --as=jane@example.com \
  -n production

kubectl auth can-i delete secrets \
  --as=system:serviceaccount:ci-cd:github-actions \
  -n production

# List ALL permissions for a user/service account
kubectl auth can-i --list \
  --as=system:serviceaccount:production:my-app \
  -n production

# Who has access to secrets in production?
kubectl get rolebindings,clusterrolebindings \
  --all-namespaces -o json | \
  jq '.items[] | select(
    .roleRef.name == "cluster-admin" or
    .roleRef.name == "admin" or
    .roleRef.name == "edit"
  ) | {name: .metadata.name, subjects: .subjects}'

# Verify RBAC is enabled
kubectl get pod kube-apiserver-$(hostname) -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | \
  tr ' ' '\n' | grep authorization-mode
# Should include: RBAC
```

### 7.6 Authorization Mode Configuration

```bash
# API server authorization modes (should always include RBAC in production)
# From kube-apiserver static pod:
--authorization-mode=Node,RBAC
# Node: Required for kubelet authorization
# RBAC: Standard role-based access control

# NEVER use these in production:
# --authorization-mode=AlwaysAllow  (no access control!)
# --authorization-mode=ABAC         (deprecated, file-based)
```

---

## 8. Admission Controllers — The Final Gate

After authentication and authorization, requests pass through **Admission Controllers** — plugins that can **mutate** (modify) or **validate** (accept/reject) requests before they are persisted to etcd.

### 8.1 Mutating vs Validating Admission

```
Request → AuthN → AuthZ → Mutating Admission → Validating Admission → etcd

Mutating:  Can change the request (add labels, inject sidecars, set defaults)
Validating: Can only accept or reject (cannot change the request)
```

### 8.2 Critical Production Admission Controllers

| Controller | Type | Purpose |
|---|---|---|
| `NodeRestriction` | Validating | Prevents kubelets from modifying other nodes' resources |
| `PodSecurity` | Validating | Enforces Pod Security Standards (privileged/baseline/restricted) |
| `ResourceQuota` | Validating | Enforces namespace resource quotas |
| `LimitRanger` | Mutating + Validating | Sets default resource requests/limits; validates ranges |
| `ServiceAccount` | Mutating | Auto-injects service account tokens into Pods |
| `MutatingAdmissionWebhook` | Mutating | Calls external webhooks for mutation |
| `ValidatingAdmissionWebhook` | Validating | Calls external webhooks for validation |
| `PodSecurityPolicy` (deprecated) | Validating | Replaced by PodSecurity in v1.25+ |

### 8.3 Pod Security Standards Configuration

```bash
# Apply Pod Security Standards to a namespace
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# Levels:
# privileged: No restrictions (use for trusted system namespaces)
# baseline:   Prevents known privilege escalation
# restricted: Hardened; requires non-root, no host namespaces, etc.

# Verify current PSS settings
kubectl get namespace production \
  -o jsonpath='{.metadata.labels}' | jq
```

### 8.4 External Admission Webhooks (OPA Gatekeeper)

```bash
# Install OPA Gatekeeper for policy enforcement
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system \
  --create-namespace

# Example constraint: require resource limits on all containers
cat << 'EOF' | kubectl apply -f -
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresources
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResources
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresources
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container %v has no memory limit", [container.name])
        }
EOF
```

---

## 9. kubectl — Primary Access Tool

`kubectl` is the official Kubernetes command-line interface. For production access, understanding its security implications, configuration options, and efficient usage is essential.

### 9.1 kubectl Installation and Version Management

```bash
# Install specific version (always match cluster version ±1 minor)
K8S_VERSION=v1.29.0

# Linux
curl -LO "https://dl.k8s.io/release/${K8S_VERSION}/bin/linux/amd64/kubectl"
chmod +x kubectl
mv kubectl /usr/local/bin/

# macOS (Homebrew)
brew install kubernetes-cli

# Verify version compatibility
kubectl version --client
# Client should be within ±1 minor version of server

# The server version (from API server)
kubectl version --short
```

### 9.2 kubectl Verbosity — Understanding API Calls

```bash
# Verbosity levels (critical for debugging access issues)
kubectl get pods -v=1     # Warning messages only
kubectl get pods -v=3     # Core flow information
kubectl get pods -v=6     # HTTP method + URL
kubectl get pods -v=7     # HTTP request headers
kubectl get pods -v=8     # HTTP request body
kubectl get pods -v=9     # Full HTTP request + response

# Example -v=6 output reveals the exact API call:
# GET https://k8s-api:6443/api/v1/namespaces/default/pods 200 OK

# Useful for:
# 1. Understanding what API calls a command makes
# 2. Debugging 403 Forbidden errors (see exact resource/verb)
# 3. Learning the Kubernetes API structure
```

### 9.3 kubectl Plugins for Production Access

```bash
# Install krew plugin manager
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

# Essential plugins for production access
kubectl krew install ctx           # Context switching (kubectl ctx)
kubectl krew install ns            # Namespace switching (kubectl ns)
kubectl krew install access-matrix # View RBAC permissions as matrix
kubectl krew install who-can       # Who has permission to do X?
kubectl krew install rbac-lookup   # Find RBAC permissions for a subject
kubectl krew install oidc-login    # OIDC authentication flow
kubectl krew install neat          # Clean YAML output (removes clutter)
kubectl krew install tree          # Show resource ownership hierarchy
kubectl krew install kubelogin     # Various cluster login methods

# Usage examples
kubectl ctx production             # Switch to production context
kubectl ns kube-system             # Switch to kube-system namespace
kubectl who-can delete pods -n production
kubectl rbac-lookup jane --kind User
kubectl access-matrix -n production
```

### 9.4 kubectl Impersonation

Impersonation allows you to test what another user/ServiceAccount can do **without** sharing their credentials:

```bash
# Test access as another user (requires impersonate RBAC permission)
kubectl get pods -n production \
  --as=system:serviceaccount:production:my-app

# Test access as a group member
kubectl get secrets \
  --as=jane@example.com \
  --as-group=developers

# Required RBAC for impersonation
cat << 'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: impersonator
rules:
  - apiGroups: [""]
    resources: ["users", "groups", "serviceaccounts"]
    verbs: ["impersonate"]
EOF
```

---

## 10. Secure Remote Access Patterns

### 10.1 VPN + Private API Server (Recommended)

The most secure pattern: the API server is not exposed to the public internet. Access requires first connecting to a VPN.

```
Internet
    │
    ▼ (VPN connection)
Corporate/Cloud VPN
    │
    ▼ (private network)
Internal Load Balancer
    │
    ▼ (private subnet)
kube-apiserver :6443
```

```bash
# API server listening only on private network
# In kube-apiserver:
--advertise-address=10.0.0.10   # Private IP only
--bind-address=10.0.0.10         # NOT 0.0.0.0

# No public IP or security group rule for :6443
```

### 10.2 Bastion Host Pattern

```
Internet
    │ SSH
    ▼
Bastion Host (hardened, minimal packages)
    │ kubectl
    ▼
kube-apiserver :6443 (private subnet)
```

```bash
# Access via bastion
ssh -J bastion.example.com user@k8s-cp-1.internal

# Or use SSH tunneling for local kubectl
ssh -L 6443:k8s-api.internal:6443 bastion.example.com -N &
kubectl config set-cluster local-tunnel \
  --server=https://localhost:6443 \
  --insecure-skip-tls-verify=true  # Only safe because it's localhost tunnel
```

### 10.3 kubectl Port-Forward for Service Access

```bash
# Access a service without exposing it externally
kubectl port-forward svc/my-database \
  5432:5432 \
  -n production &
# Now connect to localhost:5432 to reach the database in-cluster

# Access a Pod directly
kubectl port-forward pod/my-app-7d4f9-abc \
  8080:8080 \
  -n production &

# Bind to specific local address
kubectl port-forward svc/grafana \
  --address=127.0.0.1 \
  3000:3000 \
  -n monitoring

# Background with cleanup
PORT_FORWARD_PID=$!
# ... do work ...
kill $PORT_FORWARD_PID
```

### 10.4 kubectl Proxy

```bash
# Start a local proxy to the API server (no TLS required locally)
kubectl proxy --port=8001 &

# Access API via localhost (no auth needed through proxy)
curl http://localhost:8001/api/v1/namespaces
curl http://localhost:8001/api/v1/namespaces/default/pods

# WARNING: kubectl proxy has NO authorization — anyone who can reach
# localhost:8001 has the same access as the kubeconfig user
# Only run on trusted machines, bind to 127.0.0.1

# Kill the proxy
kill $(lsof -t -i:8001)
```

### 10.5 Cloud-Provider Managed Access

**AWS EKS:**

```bash
# Update kubeconfig for EKS cluster
aws eks update-kubeconfig \
  --region us-east-1 \
  --name production-cluster

# This uses aws-iam-authenticator or aws cli as exec credential
# Tokens are short-lived (15 minutes) and auto-refreshed

# Add IAM user/role to cluster access
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapUsers: |
    - userarn: arn:aws:iam::123456789012:user/jane
      username: jane
      groups:
        - system:masters  # Use specific RBAC groups instead!
  mapRoles: |
    - rolearn: arn:aws:iam::123456789012:role/eks-deployer
      username: deployer
      groups:
        - deployers
EOF
```

**GKE:**

```bash
# Get credentials for GKE cluster
gcloud container clusters get-credentials production-cluster \
  --region us-central1 \
  --project my-project

# Workload Identity for in-cluster access to GCP services
# (avoids storing service account keys in cluster)
```

**Azure AKS:**

```bash
# Get credentials for AKS cluster
az aks get-credentials \
  --resource-group production-rg \
  --name production-cluster

# Azure AD integration (AAD RBAC)
az aks update \
  --resource-group production-rg \
  --name production-cluster \
  --enable-azure-rbac \
  --enable-aad
```

### 10.6 Service Mesh for Service-to-Service Access

```bash
# Istio mTLS for service-to-service authentication
# (alternative to RBAC for east-west traffic)

# Check mTLS policy
kubectl get peerauthentication --all-namespaces

# Strict mTLS for a namespace
cat << 'EOF' | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT
EOF

# AuthorizationPolicy for service-to-service access
cat << 'EOF' | kubectl apply -f -
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  selector:
    matchLabels:
      app: backend
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/production/sa/frontend"]
EOF
```

---

## 11. Service Account Access — In-Cluster Applications

Service Accounts are the Kubernetes-native identity for applications running inside the cluster.

### 11.1 How Pods Use Service Accounts

```
Pod starts
  → kubelet mounts ServiceAccount token at /var/run/secrets/kubernetes.io/serviceaccount/
  → Pod reads token from file
  → Pod sends token in Authorization: Bearer <token> header to API server
  → API server validates token (bound token: validates audience, expiry, pod binding)
  → API server identifies the ServiceAccount
  → RBAC evaluates permissions for system:serviceaccount:<namespace>:<sa-name>
```

```bash
# Inside a Pod, the token is available at:
cat /var/run/secrets/kubernetes.io/serviceaccount/token
cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
cat /var/run/secrets/kubernetes.io/serviceaccount/namespace

# Use the token to call the API server from within a Pod
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
NAMESPACE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)

curl -s \
  --cacert $CACERT \
  -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/api/v1/namespaces/$NAMESPACE/pods
```

### 11.2 ServiceAccount Best Practices

```yaml
# Dedicated ServiceAccount per application (never use default)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app
  namespace: production
automountServiceAccountToken: false  # Disable auto-mount by default
---
# Enable only in Pod spec when needed
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: production
spec:
  template:
    spec:
      serviceAccountName: my-app
      automountServiceAccountToken: true  # Enable only where needed
      containers:
        - name: app
          image: my-app:1.0
```

### 11.3 TokenRequest API — Short-Lived Bound Tokens

```bash
# Request a short-lived token for a ServiceAccount
kubectl create token my-app \
  --duration=1h \
  --audience=my-service \
  -n production

# In a Pod, use projected volumes for bound tokens
```

```yaml
# Projected ServiceAccount token in Pod spec
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: my-app
  volumes:
    - name: token
      projected:
        sources:
          - serviceAccountToken:
              audience: my-service         # Limits token to this audience
              expirationSeconds: 3600      # 1 hour
              path: token
          - configMap:
              name: kube-root-ca.crt
              items:
                - key: ca.crt
                  path: ca.crt
  containers:
    - name: app
      volumeMounts:
        - name: token
          mountPath: /var/run/secrets/my-service/
          readOnly: true
```

---

## 12. Multi-Cluster Access Management

### 12.1 kubeconfig Context Management

```bash
# Install kubectx for fast context switching
brew install kubectx

# List all contexts
kubectx

# Switch to production
kubectx production

# Switch back to previous context
kubectx -

# Delete a context
kubectx -d old-cluster

# Rename a context
kubectx staging=old-staging-name
```

### 12.2 Tools for Multi-Cluster Access

| Tool | Purpose | Best For |
|---|---|---|
| **kubectx/kubens** | Fast context and namespace switching | Daily operations |
| **k9s** | Terminal UI across any context | Visual exploration |
| **Lens IDE** | Desktop GUI multi-cluster management | Visual management |
| **Rancher** | Web UI with multi-cluster management | Enterprise centralized access |
| **ArgoCD** | GitOps multi-cluster deployment | CD pipelines |
| **Teleport** | Zero-trust access proxy for K8s | Security-first access |
| **Kubie** | Isolated shell per cluster context | Prevents context mistakes |

### 12.3 Federation and Cross-Cluster Access

```bash
# Using Kubie for isolated contexts (prevents accidental cross-cluster commands)
kubie ctx production  # Opens new shell with production context
# All kubectl commands in this shell only affect production
exit  # Returns to previous context

# Teleport: Zero-trust access proxy
# Users authenticate to Teleport; Teleport proxies to clusters
# No kubeconfig with long-lived credentials needed
tsh login --proxy=teleport.example.com
tsh kube login production-cluster
kubectl get pods  # Routes through Teleport with audit logging
```

---

## 13. Built-in Controllers Relevant to Access Management

### 13.1 ServiceAccount Controller

The ServiceAccount controller watches ServiceAccount and Secret objects, ensuring every ServiceAccount has a corresponding token Secret (in legacy mode) and managing token lifecycle.

```bash
# Observe ServiceAccount controller creating tokens
kubectl create serviceaccount test-sa -n default
# Controller immediately creates a token secret (legacy behavior in <1.24)
kubectl get secrets -n default | grep test-sa

# In 1.24+, tokens are not auto-created as secrets
# They are generated on-demand via TokenRequest API
kubectl describe serviceaccount test-sa
```

**Reconciliation flow:**
```
ServiceAccount ADDED event → work queue
Controller: does a token Secret exist for this SA?
  If NO → create Secret of type kubernetes.io/service-account-token
  If YES → ensure secret is valid and token is signed correctly
Update SA.secrets to reference the token Secret
```

### 13.2 Token Controller

Part of kube-controller-manager. Signs ServiceAccount tokens with the cluster's signing key, ensuring tokens are cryptographically valid.

```bash
# Signing key location (referenced by token controller)
# --service-account-private-key-file=/etc/kubernetes/pki/sa.key (controller-manager)
# --service-account-key-file=/etc/kubernetes/pki/sa.pub (API server for validation)

# Verify token controller is running
kubectl logs -n kube-system kube-controller-manager-$(hostname) | \
  grep -i "token\|serviceaccount"
```

### 13.3 Node Controller (Node Authorization)

The Node controller works with the Node authorization mode to ensure kubelets can only access resources belonging to their own node — not resources on other nodes.

```bash
# Node authorization is enabled via --authorization-mode=Node,RBAC
# NodeRestriction admission controller further limits kubelet permissions

# Check node authorization config
kubectl get pod kube-apiserver-$(hostname) -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | \
  tr ' ' '\n' | grep 'authorization-mode'
```

### 13.4 Namespace Controller

The Namespace controller manages the lifecycle of Namespace objects. When a namespace is deleted, it cascades deletion to all resources including RBAC bindings and ServiceAccounts in that namespace. This has access implications.

```bash
# When namespace is deleted, all its RoleBindings are deleted too
# This means RBAC permissions scoped to that namespace are automatically removed
kubectl delete namespace dev-experiment
# All Roles, RoleBindings, ServiceAccounts in dev-experiment are deleted
```

### 13.5 Garbage Collector

Manages ownerReferences to clean up orphaned resources. Relevant to access management when:
- A ServiceAccount is deleted but its Secrets remain (not cleaned up correctly)
- RoleBindings reference deleted ServiceAccounts (stale bindings remain as security risk)

```bash
# Find stale RoleBindings (referencing non-existent ServiceAccounts)
kubectl get rolebindings,clusterrolebindings \
  --all-namespaces -o json | \
  jq '.items[] | 
      select(.subjects != null) |
      select(.subjects[] | 
        select(.kind == "ServiceAccount")) |
      {name: .metadata.name, 
       namespace: .metadata.namespace,
       subjects: .subjects}' | head -50

# Manually find non-existent SA references
kubectl get rolebindings -n production -o json | \
  jq '.items[] | 
      select(.subjects[] | 
        select(.kind == "ServiceAccount" and 
               (.name | test(".*-deprecated|.*-old")))) |
      .metadata.name'
```

### 13.6 Deployment, ReplicaSet, StatefulSet, DaemonSet Controllers

These controllers create Pods, and each Pod is associated with a ServiceAccount. The controller follows the reconciliation loop:

```
Deployment ADDED/MODIFIED
→ Deployment controller creates/updates ReplicaSet
→ ReplicaSet controller creates Pods
→ Each Pod gets the ServiceAccount specified in spec.serviceAccountName
→ kubelet starts the Pod with the ServiceAccount token mounted
→ Pod can now call the API server with that identity
```

This means: **the Deployment controller indirectly determines the access identity of running Pods**. Setting `spec.template.spec.serviceAccountName` incorrectly can either over-privilege or under-privilege your application.

### 13.7 Job and CronJob Controllers

Jobs and CronJobs create Pods that run tasks. These Pods need access to external systems or the Kubernetes API itself.

```yaml
# Job with specific ServiceAccount for controlled access
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  namespace: production
spec:
  template:
    spec:
      serviceAccountName: db-migrator   # Specific SA with limited permissions
      automountServiceAccountToken: true
      restartPolicy: OnFailure
      containers:
        - name: migrate
          image: my-migrator:v1.0
```

---

## 14. Informers, Work Queues & Reconciliation in Access Control

### 14.1 How RBAC Rules Are Evaluated in Real-Time

The API server does NOT re-read RBAC rules from etcd on every request. It maintains an **in-memory cache** updated by informers.

```
┌────────────────────────────────────────────────────────────────┐
│               RBAC AUTHORIZATION — INTERNAL FLOW              │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  API Server RBAC Authorizer (in-process)                │  │
│  │                                                         │  │
│  │  ┌─────────────────────────────────────────────────┐   │  │
│  │  │  InMemory Cache (Informer-backed)               │   │  │
│  │  │  - ClusterRoles        ← Watch /apis/rbac.../   │   │  │
│  │  │  - ClusterRoleBindings ← Watch events           │   │  │
│  │  │  - Roles               ← Updated in real-time   │   │  │
│  │  │  - RoleBindings        ← via List+Watch         │   │  │
│  │  └─────────────────────────────────────────────────┘   │  │
│  │                                                         │  │
│  │  When request arrives:                                  │  │
│  │  1. Extract username, groups from authentication        │  │
│  │  2. Find RoleBindings/CRBs matching the subject         │  │
│  │  3. Find Rules from referenced Roles/ClusterRoles       │  │
│  │  4. Check if verb + resource + apiGroup matches         │  │
│  │  5. Return Allow or Deny (no match = Deny)              │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Result: O(1) authorization lookups from memory               │
│  RBAC changes propagate within seconds via informer watch      │
└────────────────────────────────────────────────────────────────┘
```

### 14.2 RBAC Change Propagation Timing

```bash
# How long does a new RoleBinding take to take effect?
# Answer: seconds to minutes depending on informer cache

# Create a new RoleBinding
kubectl apply -f new-rolebinding.yaml

# The API server's RBAC informer receives the watch event immediately
# (typically < 1 second in a healthy cluster)
# The in-memory cache is updated

# Test: immediately try the new permission
kubectl auth can-i get pods -n production --as=new-user
# Should work immediately or within seconds

# In practice, if you observe delay, check:
kubectl get lease kube-controller-manager -n kube-system  # Is leader healthy?
kubectl get --raw /metrics | grep 'watch_cache_capacity'   # Cache health
```

### 14.3 Work Queue for ServiceAccount Token Management

The token controller uses a work queue to manage token creation/deletion:

```
ServiceAccount ADDED → queue key: "default/my-sa"
Worker picks up key →
  Does a token secret exist? NO
  → Create Secret: "my-sa-token-xxxxx"
  → Set SA.secrets reference
  → workQueue.Forget(key)

ServiceAccount token Secret EXPIRES →
  TokenController detects via periodic check
  → Delete old Secret
  → queue key for SA
  → Create new token Secret
```

---

## 15. API Server and etcd Interaction — The Access Enforcement Layer

### 15.1 The Immutable Rule

```
╔══════════════════════════════════════════════════════════════╗
║                                                              ║
║  CONTROLLERS AND CLIENTS NEVER ACCESS ETCD DIRECTLY.        ║
║                                                              ║
║  ALL ACCESS — WHETHER FROM KUBECTL, CONTROLLERS,             ║
║  IN-CLUSTER APPS, OR CI/CD SYSTEMS — FLOWS THROUGH          ║
║  THE KUBE-APISERVER, WHICH ENFORCES:                         ║
║                                                              ║
║  1. TLS transport security                                   ║
║  2. Authentication (who are you?)                            ║
║  3. Authorization (what can you do?)                         ║
║  4. Admission control (is this valid?)                       ║
║  5. Audit logging (record everything)                        ║
║                                                              ║
║  Only the kube-apiserver binary has credentials to           ║
║  connect to etcd at port 2379.                               ║
║                                                              ║
╚══════════════════════════════════════════════════════════════╝
```

### 15.2 Why This Matters for Access Management

| Attack Vector | Prevented By |
|---|---|
| Bypass RBAC by writing directly to etcd | etcd firewall rules; no credentials available to non-apiserver processes |
| Replay old deleted tokens | etcd stores current state; deleted secrets = invalid tokens |
| Forge RBAC rules | Can only be done through API server, which requires own RBAC permission |
| Read secrets from etcd | etcd encrypted at rest + no direct access = double protection |

```bash
# Verify etcd is NOT accessible from worker nodes
# (This should fail on any node that isn't a control plane)
curl -k https://<etcd-ip>:2379/health 2>&1
# Expected: Connection refused or timeout

# Verify etcd is only accessible from API server
# On control plane node (should succeed only here):
curl -k https://127.0.0.1:2379/health
# Only succeeds because the control plane has the etcd client certificates
```

### 15.3 etcd-Level Access Controls

```bash
# etcd mTLS — only the API server has the client certificate
# /etc/kubernetes/pki/apiserver-etcd-client.crt (for API server)
# /etc/kubernetes/pki/etcd/ca.crt (for verifying etcd's certificate)

# These files should NOT be readable by non-root users
ls -la /etc/kubernetes/pki/apiserver-etcd-client.key
# Expected: -rw------- root root

# etcd peer authentication
# /etc/kubernetes/pki/etcd/peer.crt — used for etcd peer communication
# Only etcd instances authenticate to each other — no other process should

# Check etcd network listeners
ss -tnp | grep etcd
# Should only show: 0.0.0.0:2380 (peer) and 0.0.0.0:2379 (client)
# Should NOT be reachable from outside the control plane network
```

---

## 16. Leader Election and HA for Access Infrastructure

### 16.1 Why Leader Election Matters for Access

In a 3-node HA control plane, multiple kube-controller-manager instances run. Without leader election, multiple controllers could simultaneously:
- Create duplicate ServiceAccount tokens
- Issue conflicting CertificateSigningRequest approvals
- Delete resources referenced by valid RBAC bindings

**Leader election ensures exactly one controller manager is active** at any time.

### 16.2 Observing Access-Related Leader Election

```bash
# Who is the active controller manager (handling ServiceAccount tokens)?
kubectl get lease kube-controller-manager -n kube-system \
  -o jsonpath='{.spec.holderIdentity}' && echo

# Monitor lease renewals (healthy = updated every renewDeadline)
kubectl get lease kube-controller-manager -n kube-system \
  -o jsonpath='{.spec.renewTime}' && echo

# Watch for leadership changes
kubectl get lease kube-controller-manager \
  -n kube-system -w

# Check if leader is actively working
kubectl logs -n kube-system \
  $(kubectl get pods -n kube-system \
    -l component=kube-controller-manager \
    -o name | head -1) | \
  grep "serviceaccount\|token" | tail -10
```

### 16.3 Lease Object for Access Controller HA

```yaml
# The Lease object that coordinates leadership
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  acquireTime: "2025-01-01T10:00:00.000000Z"
  holderIdentity: "k8s-cp-1_abc123"    # Active leader
  leaseDurationSeconds: 30
  leaseTransitions: 2
  renewTime: "2025-01-01T14:30:00.000000Z"
```

### 16.4 API Server HA for Access

The API server is **Active-Active** — multiple instances run simultaneously, each capable of serving access requests. A load balancer distributes traffic.

```bash
# Verify multiple API server instances (HA setup)
kubectl get pods -n kube-system -l component=kube-apiserver

# All API server instances:
# - Evaluate RBAC from the same etcd state (eventual consistency via informers)
# - Issue tokens signed with the same private key
# - Validate tokens against the same public key
# - Write to the same etcd cluster (through the same Raft-based storage)

# Check current API server endpoint
kubectl config view --raw | grep server
```

### 16.5 Important Leader Election Flags

```bash
# kube-controller-manager flags for access-related HA
--leader-elect=true                              # Enable leader election
--leader-elect-lease-duration=30s               # Lease validity
--leader-elect-renew-deadline=25s               # Must renew before this
--leader-elect-retry-period=5s                  # Standby poll interval
--service-account-private-key-file=/etc/kubernetes/pki/sa.key  # Token signing key
--controllers=*,bootstrapsigner,tokencleaner    # Enable all including auth controllers
```

---

## 17. Performance Tuning for High-Traffic API Access

### 17.1 API Server Throttling Configuration

```yaml
# In kube-apiserver manifest
--max-requests-inflight=800          # Max concurrent non-mutating requests
--max-mutating-requests-inflight=400  # Max concurrent mutating requests
--request-timeout=60s                # Per-request timeout
--min-request-timeout=1800s          # Minimum for WATCH requests

# For large clusters or high CI/CD throughput:
--max-requests-inflight=1600
--max-mutating-requests-inflight=800
```

### 17.2 Client-Side Rate Limiting

```bash
# kubectl has built-in rate limiting
# Default: 5 QPS, 10 burst

# Override for bulk operations
kubectl get pods --all-namespaces \
  --request-timeout=120s

# For programmatic access (Go client example):
# rest.Config{QPS: 100, Burst: 200}
```

### 17.3 Watch Cache Optimization

```yaml
# API server watch cache sizes (reduces etcd load)
--watch-cache-sizes=pod#5000,node#1000,secret#2000,configmap#5000
--default-watch-cache-size=100       # Default per resource type

# For RBAC-heavy clusters (many role/binding changes):
--watch-cache-sizes=clusterrole#500,clusterrolebinding#500,role#2000,rolebinding#2000
```

### 17.4 Connection Limits and Keep-Alive

```yaml
# API server HTTP/2 settings for efficient connections
--http2-max-streams-per-connection=1000

# etcd connection pooling
--etcd-servers=https://10.0.0.10:2379,https://10.0.0.11:2379,https://10.0.0.12:2379
--etcd-prefix=/registry
--etcd-compaction-interval=5m
```

### 17.5 Controller Manager Concurrency for Access Operations

```yaml
# In kube-controller-manager
--concurrent-serviceaccount-token-syncs=5   # Token reconciliation workers
--controllers=*,bootstrapsigner,tokencleaner  # Ensure auth controllers run
--service-account-private-key-file=/etc/kubernetes/pki/sa.key
```

### 17.6 Performance Metrics Table

| Metric | Healthy Value | Warning Threshold | Critical Threshold |
|---|---|---|---|
| API server p99 latency | < 100ms | > 500ms | > 1000ms |
| API server inflight requests | < 60% of max | > 80% of max | > 95% of max |
| Auth plugin latency | < 10ms | > 50ms | > 100ms |
| etcd request latency | < 10ms | > 50ms | > 100ms |
| RBAC evaluation time | < 1ms | > 10ms | > 50ms |

---

## 18. Security Hardening — Zero-Trust Access Practices

### 18.1 RBAC Hardening

```bash
# Principle 1: Default-Deny for all subjects
# Kubernetes RBAC is default-deny (no role = no access)
# Verify no subject has unexpected permissions

# Audit cluster-admin bindings
kubectl get clusterrolebindings -o json | \
  jq '.items[] | 
      select(.roleRef.name == "cluster-admin") | 
      {name: .metadata.name, subjects: .subjects}'

# Remove cluster-admin from non-emergency accounts
kubectl delete clusterrolebinding old-admin-binding

# Principle 2: Never use wildcards in production RBAC
# BAD:
cat << 'EOF'
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]
EOF

# GOOD:
cat << 'EOF'
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "update", "patch"]
EOF

# Principle 3: Separate service account per application
# Principle 4: Quarterly RBAC audits
kubectl get clusterrolebindings,rolebindings \
  --all-namespaces -o wide | \
  grep -v "system:" > /tmp/custom-rbac-audit.txt
```

### 18.2 TLS Hardening

```bash
# Enforce minimum TLS version
# In kube-apiserver:
--tls-min-version=VersionTLS12
# Even better for new clusters:
--tls-min-version=VersionTLS13

# Use strong cipher suites only
--tls-cipher-suites=TLS_AES_128_GCM_SHA256,\
TLS_AES_256_GCM_SHA384,\
TLS_CHACHA20_POLY1305_SHA256,\
TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256,\
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256

# Disable deprecated TLS settings
# Remove: TLS_RSA_WITH_RC4_128_SHA (RC4 is broken)
# Remove: TLS_RSA_WITH_3DES_EDE_CBC_SHA (3DES is weak)

# Verify TLS settings
openssl s_client -connect k8s-api:6443 \
  -tls1_2 2>&1 | grep "Protocol\|Cipher"
```

### 18.3 Service Account Security

```yaml
# Disable default service account auto-mount cluster-wide
# (In API server startup):
# --disable-admission-plugins=ServiceAccount
# NOT recommended — instead configure per SA:

# Disable automount on the default ServiceAccount in all namespaces
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  kubectl patch serviceaccount default \
    -n $ns \
    -p '{"automountServiceAccountToken": false}'
done

# Verify auto-mount is disabled
kubectl get serviceaccounts --all-namespaces \
  -o json | jq \
  '.items[] | 
   select(.metadata.name == "default") | 
   select(.automountServiceAccountToken != false) |
   {namespace: .metadata.namespace}'
```

### 18.4 API Server Security Flags

```yaml
# Complete security hardening configuration for kube-apiserver
spec:
  containers:
    - name: kube-apiserver
      command:
        - kube-apiserver
        
        # Authentication hardening
        - --anonymous-auth=false              # Disable anonymous access
        - --token-auth-file=""                # No static token file
        - --basic-auth-file=""                # No basic auth (removed in 1.19)
        
        # Authorization
        - --authorization-mode=Node,RBAC      # Only Node + RBAC
        
        # Admission
        - --enable-admission-plugins=NodeRestriction,PodSecurity,ResourceQuota,LimitRanger,ServiceAccount,MutatingAdmissionWebhook,ValidatingAdmissionWebhook
        
        # Audit logging
        - --audit-log-path=/var/log/kubernetes/audit.log
        - --audit-log-maxage=30
        - --audit-log-maxbackup=10
        - --audit-log-maxsize=100
        - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
        
        # TLS
        - --tls-min-version=VersionTLS12
        - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
        - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
        
        # Encryption at rest
        - --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
        
        # ServiceAccount tokens
        - --service-account-signing-key-file=/etc/kubernetes/pki/sa.key
        - --service-account-issuer=https://kubernetes.default.svc
        - --service-account-max-token-expiration=86400s  # 24h max
```

### 18.5 Network-Level Access Controls

```bash
# Restrict who can reach the API server at network level
# AWS Security Group rule (only allow from VPN and bastion):
# Inbound TCP 6443 from VPN CIDR (10.8.0.0/16)
# Inbound TCP 6443 from bastion subnet (10.0.1.0/24)
# NO inbound from 0.0.0.0/0

# Kubernetes NetworkPolicy to protect kube-system
cat << 'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-access-to-kube-system
  namespace: kube-system
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
EOF
```

---

## 19. Monitoring & Observability for Cluster Access

### 19.1 Audit Logging — The Access Paper Trail

```yaml
# /etc/kubernetes/audit-policy.yaml
# Production audit policy
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - RequestReceived   # Don't log receive stage (reduces noise)
rules:
  # Log all access to secrets at RequestResponse level (full body)
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
    verbs: ["get", "list", "watch"]
    # CRITICAL: Know when secrets are accessed

  # Log all RBAC changes
  - level: RequestResponse
    resources:
      - group: "rbac.authorization.k8s.io"
        resources: ["*"]
    verbs: ["create", "update", "patch", "delete"]

  # Log all ServiceAccount changes
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["serviceaccounts", "serviceaccounts/token"]

  # Log auth failures (401/403)
  - level: Metadata
    omitStages:
      - RequestReceived
    # Catch-all for any request resulting in 401/403

  # Log exec and portforward (potential for data exfiltration)
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["pods/exec", "pods/portforward", "pods/attach"]

  # Minimal logging for read operations (reduces volume)
  - level: Metadata
    verbs: ["get", "list", "watch"]
    resources:
      - group: ""
        resources: ["*"]
      - group: "apps"
        resources: ["*"]

  # Log namespace deletion (blast radius risk)
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["namespaces"]
    verbs: ["delete"]

  # Don't log health checks (noise)
  - level: None
    nonResourceURLs: ["/healthz", "/readyz", "/livez", "/metrics"]

  # Don't log node/proxy metrics
  - level: None
    users: ["system:node"]
    verbs: ["get"]
    resources:
      - group: ""
        resources: ["nodes/status"]

  # Default: log everything at Metadata level
  - level: Metadata
```

### 19.2 Key Prometheus Metrics for Access Monitoring

| Metric | Description | Alert |
|---|---|---|
| `apiserver_request_total{code="401"}` | Authentication failures | > 10/minute |
| `apiserver_request_total{code="403"}` | Authorization denials | > 50/minute |
| `apiserver_request_total{code="5xx"}` | Server errors | > 0 sustained |
| `apiserver_authentication_attempts_total{result="error"}` | Auth plugin errors | > 0 |
| `apiserver_authorization_decisions_total{decision="deny"}` | RBAC denials | Baseline + 3σ |
| `kubernetes_client_certificate_expiration_seconds` | Certificate expiry | < 7 days |
| `apiserver_audit_requests_rejected_total` | Audit log failures | > 0 |
| `rest_client_request_duration_seconds` | kubectl/client latency | p99 > 1s |

### 19.3 Prometheus Alerting Rules

```yaml
groups:
  - name: kubernetes-access-security
    rules:
      # High authentication failure rate
      - alert: KubeAPIHighAuthFailureRate
        expr: |
          sum(rate(apiserver_request_total{code="401"}[5m])) > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High authentication failure rate on API server"
          description: "More than 5 auth failures/second sustained for 5 minutes"

      # Cluster-admin binding created outside of approved process
      - alert: KubeClusterAdminBindingCreated
        expr: |
          sum(increase(apiserver_audit_event_total{
            objectRef_resource="clusterrolebindings",
            verb=~"create|update"
          }[1m])) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "ClusterRoleBinding created — verify this was intentional"

      # API server certificate expiring
      - alert: KubeAPIServerCertExpiringSoon
        expr: |
          apiserver_client_certificate_expiration_seconds_count{job="apiserver"} > 0
          and histogram_quantile(0.01, 
            sum by (job) (
              rate(apiserver_client_certificate_expiration_seconds_bucket{job="apiserver"}[5m])
            )
          ) < 604800  # 7 days
        labels:
          severity: warning
        annotations:
          summary: "API server client certificate expiring within 7 days"

      # Anonymous requests (should be zero in hardened clusters)
      - alert: KubeAPIAnonymousRequest
        expr: |
          sum(rate(apiserver_request_total{
            user="system:anonymous"
          }[5m])) > 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Anonymous requests detected on API server"
```

### 19.4 Access Audit Dashboard Queries

```promql
# Authentication failure rate by verb
sum by (verb, resource) (
  rate(apiserver_request_total{code=~"40[13]"}[5m])
)

# Most active API callers (user agents)
topk(10, sum by (user_agent) (
  rate(apiserver_request_total[5m])
))

# Request rate by namespace
sum by (namespace) (
  rate(apiserver_request_total{code="200"}[5m])
)

# Secret access frequency (security-relevant)
sum by (verb, user) (
  rate(apiserver_request_total{resource="secrets"}[1h])
)
```

---

## 20. Troubleshooting Access Issues — Real kubectl Commands

### 20.1 Permission Denied (403 Forbidden)

```bash
# Step 1: Understand exactly what permission is needed
# Error message: "User "jane" cannot create resource "deployments" 
#   in API group "apps" in the namespace "production""
# This tells you: verb=create, resource=deployments, apiGroup=apps, ns=production

# Step 2: Check what jane can do
kubectl auth can-i --list \
  --as=jane@example.com \
  -n production

# Step 3: Check existing bindings for jane
kubectl get rolebindings,clusterrolebindings \
  --all-namespaces -o json | \
  jq '.items[] | 
      select(.subjects != null) |
      select(.subjects[] | 
        select(.name == "jane@example.com")) |
      {name: .metadata.name, role: .roleRef.name, ns: .metadata.namespace}'

# Step 4: Find who can do the needed action
kubectl rbac-lookup jane --kind User 2>/dev/null || \
  echo "Install kubectl-rbac-lookup plugin"

# Step 5: Create/fix the RoleBinding
cat << 'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jane-deployer
  namespace: production
subjects:
  - kind: User
    name: jane@example.com
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: edit
  apiGroup: rbac.authorization.k8s.io
EOF

# Step 6: Test after fixing
kubectl auth can-i create deployments \
  --as=jane@example.com \
  -n production
```

### 20.2 Authentication Failures (401 Unauthorized)

```bash
# Diagnosis 1: Token expired
# Check token expiry from kubeconfig
kubectl config view --raw | \
  grep token | head -1 | \
  awk '{print $2}' | \
  base64 -d 2>/dev/null | \
  python3 -c "import sys,json; data=json.load(sys.stdin); print('Expires:', data.get('exp', 'No expiry'))"

# Diagnosis 2: Certificate expired
openssl x509 -in ~/.kube/client.crt -noout -dates 2>/dev/null
# Or from kubeconfig:
kubectl config view --raw | \
  grep client-certificate-data | head -1 | \
  awk '{print $2}' | \
  base64 -d | \
  openssl x509 -noout -dates

# Diagnosis 3: Certificate not signed by cluster CA
openssl verify \
  -CAfile /etc/kubernetes/pki/ca.crt \
  ~/.kube/client.crt
# Expected: OK

# Diagnosis 4: Wrong server/CA in kubeconfig
kubectl config view | grep server
# Verify this matches the actual API server URL

# Diagnosis 5: API server audit log for details
tail -f /var/log/kubernetes/audit.log | \
  jq 'select(.responseStatus.code == 401) | 
      {user: .user.username, 
       reason: .responseStatus.reason,
       time: .requestReceivedTimestamp}'

# Fix: Regenerate certificate
kubeadm certs renew admin.conf
```

### 20.3 Pods Not Created (Access Denied to Controller)

```bash
# Symptoms: Deployment exists but no Pods created
# Root cause: kube-controller-manager doesn't have permission
# (rare in kubeadm clusters, common in custom RBAC setups)

# Check controller manager logs for RBAC errors
kubectl logs -n kube-system \
  $(kubectl get pods -n kube-system \
    -l component=kube-controller-manager \
    -o name | head -1) | \
  grep -i "forbidden\|unauthorized\|denied"

# Check events for RBAC errors
kubectl get events -n production \
  --field-selector reason=FailedCreate \
  --sort-by='.lastTimestamp'

# Verify system:controller role bindings exist
kubectl get clusterrolebinding \
  system:controller:replicaset-controller \
  system:controller:deployment-controller \
  system:controller:serviceaccount-controller

# If missing, restore from default RBAC:
kubeadm init phase addon coredns
# Or re-apply system RBAC:
kubectl apply -f https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/rbac/...
```

### 20.4 Deployment Stuck — Rolling Update Access Issue

```bash
# Check if image pull is failing (needs registry access)
kubectl describe pod <new-pod> -n production | grep -A 5 "Events:"
# Look for: "ImagePullBackOff", "ErrImagePull"

# Check if imagePullSecret is configured
kubectl get deployment <n> -n production \
  -o jsonpath='{.spec.template.spec.imagePullSecrets}'

# Create registry secret if missing
kubectl create secret docker-registry registry-creds \
  --docker-server=registry.example.com \
  --docker-username=robot-account \
  --docker-password=token \
  -n production

# Add to deployment
kubectl patch deployment <n> -n production \
  -p '{"spec":{"template":{"spec":{"imagePullSecrets":[{"name":"registry-creds"}]}}}}'

# Check if ServiceAccount has access to imagePullSecret
kubectl get serviceaccount <sa> -n production \
  -o jsonpath='{.imagePullSecrets}'
```

### 20.5 Node NotReady — kubelet Authentication Issues

```bash
# Symptom: Node shows NotReady; kubelet can't authenticate to API server

# Check kubelet status
ssh <node>
systemctl status kubelet
journalctl -u kubelet -n 50 | grep -i "auth\|forbidden\|cert\|token"

# Check kubelet certificate expiry
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem \
  -noout -dates

# Check if kubelet bootstrap is needed (new node)
ls /var/lib/kubelet/pki/
# If no kubelet-client-current.pem → needs bootstrap

# Regenerate kubelet certificate
kubeadm alpha certs renew kubelet-client-current

# Check kubelet kubeconfig
cat /etc/kubernetes/kubelet.conf | grep server

# Approve pending CertificateSigningRequests for the node
kubectl get csr | grep <node-name>
kubectl certificate approve <csr-name>

# Check Node authorization
kubectl auth can-i get pods \
  --as=system:node:<node-name> \
  --as-group=system:nodes
```

### 20.6 OIDC Token Not Working

```bash
# Step 1: Decode the OIDC token (JWT)
TOKEN=$(kubectl config view --raw | grep id-token | awk '{print $2}')
echo $TOKEN | cut -d. -f2 | base64 -d 2>/dev/null | jq

# Check fields:
# "iss": Must match --oidc-issuer-url
# "aud": Must contain --oidc-client-id value
# "exp": Must be in the future (Unix timestamp)
# The claim specified by --oidc-username-claim must exist

# Step 2: Verify API server OIDC flags
kubectl get pod kube-apiserver-$(hostname) -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | \
  tr ' ' '\n' | grep oidc

# Step 3: Test token validation manually
curl -k \
  -H "Authorization: Bearer $TOKEN" \
  https://k8s-api:6443/api/v1/namespaces

# Step 4: Check for clock skew (token "iat" vs server time)
date  # On both client and server — must be within 5 minutes

# Step 5: Refresh the OIDC token
kubectl oidc-login get-token \
  --oidc-issuer-url=https://keycloak.example.com/auth/realms/k8s \
  --oidc-client-id=kubectl
```

---

## 21. Disaster Recovery — Access During Failure Scenarios

### 21.1 API Server Unreachable — Emergency Access

```bash
# Scenario: API server is down, need emergency access

# Option 1: Direct etcd access (extremely dangerous — bypasses all controls)
# Only use as absolute last resort; requires etcd client certificates
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/deployments/production/my-app | strings

# Option 2: Access via static kubeconfig on control plane node
# The admin.conf has full cluster-admin access and is stored locally
kubectl --kubeconfig=/etc/kubernetes/admin.conf get pods -n kube-system

# Option 3: Restart API server via static pod
ls /etc/kubernetes/manifests/kube-apiserver.yaml
# Move file out and back to trigger kubelet restart
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sleep 5
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# Option 4: SSH to node and use crictl for node-level management
crictl ps | grep kube-apiserver
crictl logs <container-id>
```

### 21.2 Certificate Expiry — Emergency Certificate Renewal

```bash
# Renew all certificates (requires access to control plane node)
kubeadm certs renew all

# Renew individual certificate
kubeadm certs renew apiserver
kubeadm certs renew apiserver-kubelet-client
kubeadm certs renew front-proxy-client
kubeadm certs renew admin.conf

# After renewal, restart control plane components
for component in kube-apiserver kube-controller-manager kube-scheduler; do
  kubectl -n kube-system delete pod -l component=$component
done

# Verify certificates renewed
kubeadm certs check-expiration
```

### 21.3 RBAC Configuration Lost — Restore from Backup

```bash
# If RBAC rules were accidentally deleted, restore from GitOps repo
git clone https://github.com/my-org/k8s-manifests.git
kubectl apply -f k8s-manifests/rbac/ --recursive

# Or restore from Velero backup
velero restore create rbac-restore \
  --from-backup pre-change-backup \
  --include-resources clusterroles,clusterrolebindings,roles,rolebindings

# Or restore from etcd backup (nuclear option)
# This restores all cluster state to backup point-in-time
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored \
  --name=k8s-cp-1 \
  --initial-cluster="k8s-cp-1=https://10.0.0.10:2380" \
  --initial-cluster-token=etcd-restored \
  --initial-advertise-peer-urls=https://10.0.0.10:2380
```

### 21.4 Stateless Controllers — Access Not Affected by Restart

```bash
# The kube-controller-manager is COMPLETELY STATELESS
# All access state (ServiceAccount tokens, RBAC rules) lives in etcd
# Restarting controller-manager does NOT affect access

# Safe restart procedure
kubectl delete pod -n kube-system \
  $(kubectl get pods -n kube-system \
    -l component=kube-controller-manager \
    -o name)

# New leader elected within 30 seconds
# All token management resumes automatically
# In-cluster applications maintain their tokens (tokens survive controller restart)

# What DOES persist across controller manager restarts:
# ✓ ServiceAccount tokens (in etcd)
# ✓ RBAC rules (in etcd)
# ✓ ServiceAccount definitions (in etcd)

# What is ephemeral:
# - In-memory RBAC cache (rebuilt from etcd on restart)
# - Work queue state (rebuilt from informer re-list)
```

---

## 22. Comparison: Direct API vs kubectl vs Dashboard vs GitOps

### 22.1 Access Method Comparison Table

| Method | Use Case | Security | Auditability | Automation | Complexity |
|---|---|---|---|---|---|
| **kubectl (human)** | Day-to-day ops, debugging | High (RBAC + TLS) | Full audit log | Medium | Low |
| **kubectl (CI/CD)** | Deployment pipelines | High (SA tokens) | Full audit log | High | Low-Medium |
| **Direct REST API** | Custom tooling, integrations | High (same as kubectl) | Full audit log | Very High | High |
| **Kubernetes Dashboard** | Visual overview (read-only) | Medium (needs careful RBAC) | Partial | Low | Low |
| **Lens / k9s** | Visual management (local) | High (uses kubeconfig) | Audit log (indirect) | Low | Low |
| **GitOps (ArgoCD/Flux)** | Declarative continuous delivery | Very High (minimal SA perms) | Git history + audit | Very High | Medium |
| **Helm** | Package deployment | High (uses kubeconfig/SA) | Audit log | High | Medium |
| **Operator pattern** | Complex lifecycle management | High (SA with specific perms) | Full audit log | Very High | High |
| **Rancher/Lens Enterprise** | Multi-cluster management UI | High (centralized auth) | Centralized audit | High | Medium |
| **Teleport** | Zero-trust access proxy | Very High (no kubeconfig) | Centralized audit | Medium | Medium |

### 22.2 When to Use Each Method

| Scenario | Recommended Method |
|---|---|
| Developer debugging a Pod | kubectl + RBAC with exec permissions |
| Reading logs in production | kubectl logs (read-only role) or log aggregation tool |
| Deploying to production | GitOps (ArgoCD) — human approval in Git |
| Emergency production fix | Break-glass procedure: temporary elevated kubectl access |
| Automated testing in CI | kubectl with minimal SA (only needed namespaces/resources) |
| Infrastructure as Code | Helm + GitOps |
| Multi-cluster management | Rancher or Lens Enterprise |
| Security-sensitive environments | Teleport (zero-trust; no kubeconfig files on laptops) |
| Running cron jobs | In-cluster Job/CronJob with SA permissions |

---

## 23. ASCII Architecture Diagram — Complete Access Flow

```
╔═══════════════════════════════════════════════════════════════════════════════════╗
║              KUBERNETES PRODUCTION CLUSTER ACCESS — COMPLETE FLOW                ║
╠═══════════════════════════════════════════════════════════════════════════════════╣
║                                                                                   ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐ ║
║  │                         ACCESS ENTRY POINTS                                │ ║
║  │                                                                             │ ║
║  │  Human Users:          CI/CD Systems:         In-Cluster Apps:             │ ║
║  │  ┌──────────────┐      ┌──────────────┐       ┌──────────────┐             │ ║
║  │  │ kubectl      │      │ GitHub       │       │ Pod w/       │             │ ║
║  │  │ ~/.kube/     │      │ Actions      │       │ ServiceAcct  │             │ ║
║  │  │ config       │      │ SA Token     │       │ Token mount  │             │ ║
║  │  └──────┬───────┘      └──────┬───────┘       └──────┬───────┘             │ ║
║  │         │                     │                       │                    │ ║
║  │  ┌──────▼──────────────────────▼───────────────────────▼────────────────┐  │ ║
║  │  │                     VPN / Bastion Host                               │  │ ║
║  │  │               (Network-level access control)                         │  │ ║
║  │  └──────────────────────────────────────────────────────────────────────┘  │ ║
║  └─────────────────────────────────────────────────────────────────────────────┘ ║
║                                │                                                  ║
║                                │ HTTPS :6443                                      ║
║                                ▼                                                  ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐ ║
║  │                         kube-apiserver                                     │ ║
║  │                                                                             │ ║
║  │  ┌─────────────────────────────────────────────────────────────────────┐   │ ║
║  │  │  1. TRANSPORT SECURITY                                               │   │ ║
║  │  │     TLS 1.2+ / Strong ciphers / Certificate validation              │   │ ║
║  │  └─────────────────────────────────────────────────────────────────────┘   │ ║
║  │                               │                                             │ ║
║  │  ┌─────────────────────────────────────────────────────────────────────┐   │ ║
║  │  │  2. AUTHENTICATION (AuthN)                                           │   │ ║
║  │  │                                                                      │   │ ║
║  │  │  ┌─────────────┐  ┌─────────────┐  ┌──────────┐  ┌──────────────┐  │   │ ║
║  │  │  │ X.509 Cert  │  │ OIDC JWT    │  │ SA Token │  │ Webhook      │  │   │ ║
║  │  │  │ (user/comp) │  │ (SSO users) │  │ (in-clus)│  │ (custom)     │  │   │ ║
║  │  │  └─────────────┘  └─────────────┘  └──────────┘  └──────────────┘  │   │ ║
║  │  │                                                                      │   │ ║
║  │  │  Result: username + groups → passes to AuthZ                        │   │ ║
║  │  └─────────────────────────────────────────────────────────────────────┘   │ ║
║  │                               │                                             │ ║
║  │  ┌─────────────────────────────────────────────────────────────────────┐   │ ║
║  │  │  3. AUTHORIZATION (AuthZ) — RBAC                                     │   │ ║
║  │  │                                                                      │   │ ║
║  │  │  ClusterRoles + Roles (WHAT)                                         │   │ ║
║  │  │  ClusterRoleBindings + RoleBindings (WHO → WHAT)                    │   │ ║
║  │  │                                                                      │   │ ║
║  │  │  Evaluation: verb+resource+namespace → Allow or 403 Forbidden        │   │ ║
║  │  └─────────────────────────────────────────────────────────────────────┘   │ ║
║  │                               │                                             │ ║
║  │  ┌─────────────────────────────────────────────────────────────────────┐   │ ║
║  │  │  4. ADMISSION CONTROL                                                │   │ ║
║  │  │                                                                      │   │ ║
║  │  │  Mutating Webhooks → PodSecurity → ResourceQuota → Validating       │   │ ║
║  │  │                                                                      │   │ ║
║  │  │  Result: Request rejected, mutated, or allowed                       │   │ ║
║  │  └─────────────────────────────────────────────────────────────────────┘   │ ║
║  │                               │                                             │ ║
║  │  ┌─────────────────────────────────────────────────────────────────────┐   │ ║
║  │  │  5. AUDIT LOGGING                                                    │   │ ║
║  │  │  Every request logged: user, verb, resource, response code          │   │ ║
║  │  └─────────────────────────────────────────────────────────────────────┘   │ ║
║  │                               │                                             │ ║
║  └───────────────────────────────┼─────────────────────────────────────────────┘ ║
║                                  │ Only kube-apiserver                            ║
║                                  ▼ has etcd credentials                          ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐ ║
║  │                              etcd                                           │ ║
║  │  /registry/rbac/  /registry/secrets/  /registry/serviceaccounts/           │ ║
║  │  mTLS required  │  Encrypted at rest  │  Access audit trail                │ ║
║  └─────────────────────────────────────────────────────────────────────────────┘ ║
║                                                                                   ║
║  ┌─────────────────────────────────────────────────────────────────────────────┐ ║
║  │                         CONTROLLERS                                         │ ║
║  │  kube-controller-manager (ServiceAccount, Token, RBAC-adjacent)            │ ║
║  │  All controller → API server (NEVER → etcd directly)                       │ ║
║  │  Leader Election via Lease object in kube-system                           │ ║
║  └─────────────────────────────────────────────────────────────────────────────┘ ║
╚═══════════════════════════════════════════════════════════════════════════════════╝
```

---

## 24. Real-World Production Use Cases

### 24.1 Financial Services — PCI DSS Compliant Access

**Requirements:** No admin access to production, all access audited, time-limited permissions.

```bash
# Break-glass procedure: temporary elevated access
# Step 1: Emergency access request via ticketing system (out-of-band)
# Step 2: Apply time-limited ClusterRoleBinding
cat << 'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: emergency-admin-jane
  annotations:
    access.company.com/ticket: "INC-12345"
    access.company.com/expires: "2025-01-01T18:00:00Z"
    access.company.com/approver: "manager@company.com"
subjects:
  - kind: User
    name: jane@company.com
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF

# Step 3: Automated cleanup after TTL (via CronJob or external tool)
# Step 4: Audit log review after incident

# RBAC for compliance: read-only for most, write only via CI/CD
```

### 24.2 Multi-Team Platform Engineering

**Structure:** Platform team manages cluster; product teams have namespace-level access.

```yaml
# Pattern: Namespace per team with delegated admin
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-namespace-admin
  namespace: team-a-production
subjects:
  - kind: Group
    name: oidc:team-a-engineers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin     # Full namespace admin (no cluster-scope)
  apiGroup: rbac.authorization.k8s.io
```

### 24.3 GitOps-First Organization

```bash
# In a GitOps-first org:
# Humans have READ-ONLY kubectl access (they can look but not touch)
# All writes go through Git → ArgoCD → cluster

# Human RBAC: read-only cluster-wide
cat << 'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: all-engineers-readonly
subjects:
  - kind: Group
    name: oidc:all-engineers
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
EOF

# ArgoCD service account has deploy permissions
# All changes tracked in Git with PR reviews
```

### 24.4 Security-Sensitive Environment (Teleport)

```bash
# Zero-trust access model with Teleport
# No kubeconfig files on developer laptops
# All access proxied through Teleport

# Developer workflow:
tsh login --proxy=teleport.example.com --auth=okta
tsh kube login production-cluster --as=readonly-user

# All commands go through Teleport which:
# 1. Verifies Okta SSO identity
# 2. Issues short-lived certificates (1 hour)
# 3. Records all kubectl commands in centralized audit log
# 4. Enforces MFA for sensitive namespaces

kubectl get pods -n production  # Fully audited through Teleport
```

---

## 25. Best Practices for Production Access Management

### 25.1 Identity Management

- **Use SSO (OIDC)** for all human user access — never create individual X.509 certs per user in large teams
- **Map OIDC groups to RBAC groups** — adding a user to the IdP group instantly grants/revokes access
- **Short-lived tokens everywhere** — configure `--service-account-max-token-expiration` on the API server
- **Rotate service account tokens** automatically using projected volumes with short expiry
- **Separate service accounts per application** — never share the `default` service account

### 25.2 RBAC Design

- **Namespace-scoped over cluster-scoped** — prefer `Role`/`RoleBinding` over `ClusterRole`/`ClusterRoleBinding` wherever possible
- **Minimal viable permissions** — start with `view` and add only what is needed
- **No wildcards in production** — `resources: ["*"]` or `verbs: ["*"]` are code smells
- **Quarterly RBAC audits** — stale bindings are a persistent security risk
- **Name bindings descriptively** — `jane-deploy-production` not `binding-123`
- **Document intent** — use annotations on RoleBindings to explain why they exist

### 25.3 Operational Security

- **Never share kubeconfig files** — credentials should be individual
- **Never commit kubeconfig or tokens to Git**
- **Use separate kubeconfigs per environment** — prevent accidentally running production commands from staging context
- **Implement the 4-eyes principle** for production access — require approval for sensitive actions
- **Log everything** — comprehensive audit policy from day one

### 25.4 Network Security

- **Private API server** — not exposed to the public internet
- **VPN required for API access** — network-level access control before AuthN
- **NodePort range firewall** — restrict 30000-32767 to necessary sources
- **LoadBalancer source ranges** — always specify `loadBalancerSourceRanges` for public Services

### 25.5 CI/CD Access

- **Minimal CI/CD permissions** — only the resources and verbs needed for deployment
- **Use OIDC federation for CI/CD** — GitHub Actions OIDC → AWS → EKS (no long-lived secrets)
- **Namespace-scoped CI/CD accounts** — CI pipeline for team-a should not touch team-b's namespace
- **Separate CI accounts per environment** — different SA for staging vs production CI

---

## 26. Common Mistakes and Pitfalls

### 26.1 Using cluster-admin for Everything

**Mistake:**
```bash
kubectl create clusterrolebinding all-access \
  --clusterrole=cluster-admin \
  --user=developer@company.com
```
**Impact:** Full cluster compromise if credentials leaked; no blast radius limitation.  
**Fix:** Use namespace-scoped `edit` or custom roles with only needed permissions.

---

### 26.2 Storing Tokens in CI/CD Secrets Without Expiry

**Mistake:** Creating a ServiceAccount secret with no expiry and storing the token in GitHub Secrets.  
**Impact:** Token never expires; cannot be rotated without credential rotation.  
**Fix:** Use OIDC federation (GitHub Actions → AWS OIDC → EKS) or time-bounded tokens.

---

### 26.3 insecure-skip-tls-verify in Production

**Mistake:**
```yaml
clusters:
  - cluster:
      insecure-skip-tls-verify: true   # NEVER in production!
```
**Impact:** Completely defeats TLS; enables man-in-the-middle attacks.  
**Fix:** Obtain the actual CA certificate and set `certificate-authority-data`.

---

### 26.4 Single kubeconfig File for Multiple Environments

**Mistake:** Keeping one `~/.kube/config` with production and staging contexts side-by-side and no safeguards.  
**Impact:** Accidental `kubectl delete deployment` in production when staging was intended.  
**Fix:** Use separate files, Kubie for isolated context shells, or `kubectl confirm` tools.

---

### 26.5 Anonymous Authentication Left Enabled

**Mistake:** Not adding `--anonymous-auth=false` to API server configuration.  
**Impact:** Unauthenticated requests can reach the API server and get 401 (manageable) or worse, accidentally hit discovery endpoints.  
**Fix:** `--anonymous-auth=false` in kube-apiserver flags.

---

### 26.6 Forgetting to Audit stale RoleBindings

**Mistake:** An employee leaves the company. Their OIDC user is deactivated in the IdP. But their RoleBinding still exists in Kubernetes.  
**Impact:** If the account is reactivated or another account gets the same email, access is immediately restored.  
**Fix:** Regular RBAC audits; automate cleanup of bindings for users who are no longer in the IdP.

---

### 26.7 Using Node Port for API Server Access

**Mistake:** Exposing the API server via a NodePort Service for external access.  
**Impact:** API server exposed on every node's IP on the NodePort range.  
**Fix:** Use a dedicated load balancer, VPN, or bastion host.

---

### 26.8 Service Account with No Resource Limits Accessing API Server at High Rate

**Mistake:** An in-cluster application with no rate limiting calls `kubectl get pods --all-namespaces` in a tight loop.  
**Impact:** API server rate limiting kicks in; other users/components experience throttling.  
**Fix:** Implement client-side rate limiting (QPS/Burst in client config); use informers instead of polling.

---

## 27. Hands-On Labs — Practical Access Exercises

### Lab 1: Create a New User with Limited Access

```bash
# Objective: Create a user "bob" with read-only access to the "development" namespace

# Step 1: Generate Bob's certificates
openssl genrsa -out bob.key 2048
openssl req -new -key bob.key -out bob.csr \
  -subj "/CN=bob/O=developers"

# Step 2: Submit CSR to Kubernetes
cat << EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: bob-csr
spec:
  request: $(cat bob.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400
  usages:
    - client auth
EOF

# Step 3: Approve the CSR
kubectl certificate approve bob-csr

# Step 4: Retrieve signed certificate
kubectl get csr bob-csr \
  -o jsonpath='{.status.certificate}' | base64 -d > bob.crt

# Step 5: Create RoleBinding
cat << 'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bob-readonly
  namespace: development
subjects:
  - kind: User
    name: bob
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
EOF

# Step 6: Build Bob's kubeconfig
kubectl config set-credentials bob \
  --client-certificate=bob.crt \
  --client-key=bob.key \
  --embed-certs=true

kubectl config set-context bob-context \
  --cluster=$(kubectl config get-clusters | tail -1) \
  --user=bob \
  --namespace=development

# Step 7: Test access as Bob
kubectl auth can-i get pods -n development --as=bob
kubectl auth can-i create pods -n development --as=bob  # Should fail

# Step 8: Verify with -v flag (see the 403)
kubectl get pods -n production \
  --as=bob -v=6 2>&1 | grep -E "GET|403|200"

# Cleanup
kubectl delete csr bob-csr
kubectl delete rolebinding bob-readonly -n development
rm bob.key bob.crt bob.csr
```

### Lab 2: RBAC Audit and Permission Discovery

```bash
# Objective: Enumerate all permissions in a cluster

# Part 1: Who has cluster-admin?
kubectl get clusterrolebindings -o json | \
  jq '.items[] | 
      select(.roleRef.name == "cluster-admin") | 
      {binding: .metadata.name, subjects: .subjects}'

# Part 2: What can the default service account in production do?
kubectl auth can-i --list \
  --as=system:serviceaccount:production:default \
  -n production

# Part 3: Find all ServiceAccounts with non-default permissions
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  for sa in $(kubectl get sa -n $ns -o jsonpath='{.items[*].metadata.name}'); do
    bindings=$(kubectl get rolebindings,clusterrolebindings \
      -n $ns --all-namespaces 2>/dev/null | \
      grep $sa | wc -l)
    if [ "$bindings" -gt 0 ]; then
      echo "SA with bindings: $ns/$sa ($bindings bindings)"
    fi
  done
done

# Part 4: Identify over-privileged namespaces (those without NetworkPolicy)
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  count=$(kubectl get networkpolicies -n $ns \
    --ignore-not-found 2>/dev/null | \
    grep -c "^" || echo 0)
  echo "$ns: $count network policies"
done
```

### Lab 3: ServiceAccount Access from Inside a Pod

```bash
# Objective: Use ServiceAccount token to call Kubernetes API from within a Pod

# Step 1: Create a ServiceAccount with pod-reading permissions
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pod-reader-sa
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: default
subjects:
  - kind: ServiceAccount
    name: pod-reader-sa
    namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EOF

# Step 2: Launch a Pod using the ServiceAccount
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: api-test-pod
  namespace: default
spec:
  serviceAccountName: pod-reader-sa
  containers:
    - name: test
      image: curlimages/curl:7.87.0
      command: ["sleep", "3600"]
EOF

# Step 3: From inside the Pod, call the API server
kubectl wait pod/api-test-pod --for=condition=Ready

kubectl exec -it api-test-pod -- sh << 'INNEREOF'
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
NAMESPACE=$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace)

# List pods in the same namespace (should work)
curl -s \
  --cacert $CACERT \
  -H "Authorization: Bearer $TOKEN" \
  "https://kubernetes.default.svc/api/v1/namespaces/$NAMESPACE/pods" | \
  python3 -c "import json,sys; data=json.load(sys.stdin); [print(p['metadata']['name']) for p in data['items']]"

# Try to list pods in kube-system (should fail with 403)
curl -s \
  --cacert $CACERT \
  -H "Authorization: Bearer $TOKEN" \
  "https://kubernetes.default.svc/api/v1/namespaces/kube-system/pods"
# Expected: {"kind":"Status","status":"Failure","reason":"Forbidden",...}
INNEREOF

# Cleanup
kubectl delete pod api-test-pod
kubectl delete sa pod-reader-sa
kubectl delete role pod-reader
kubectl delete rolebinding pod-reader-binding
```

### Lab 4: Multi-Context Production Safety

```bash
# Objective: Set up safe multi-cluster kubectl configuration

# Step 1: Install safety tools
kubectl krew install ctx ns

# Step 2: Use kubie for isolated contexts
# (prevents cross-context accidents)
export KUBECONFIG=/path/to/staging-kubeconfig.yaml
kubectl get pods  # Only can reach staging

# Or with kubie:
kubie ctx production  # Opens new shell limited to production
kubectl get pods      # Safe: can only reach production in this shell
exit                  # Returns to previous context

# Step 3: Color-code your prompt for production
# Add to ~/.bashrc or ~/.zshrc:
kube_ps1_context_color() {
  local ctx=$(kubectl config current-context 2>/dev/null)
  if [[ "$ctx" == *"production"* ]]; then
    echo "\\[\\033[0;31m\\]"  # Red for production
  else
    echo "\\[\\033[0;32m\\]"  # Green for non-production
  fi
}

# Step 4: Create an alias that requires confirmation for production commands
prod_kubectl() {
  echo "WARNING: You are about to run kubectl in PRODUCTION"
  echo "Command: kubectl $@"
  read -p "Confirm? [y/N] " confirm
  if [[ $confirm == "y" ]]; then
    kubectl --context=production "$@"
  fi
}
alias kprod='prod_kubectl'
```

### Lab 5: Audit Log Analysis for Security Review

```bash
# Objective: Analyze audit logs to understand cluster access patterns

# Setup: Ensure audit logging is enabled (kubeadm setup)
cat /etc/kubernetes/manifests/kube-apiserver.yaml | \
  grep -A 2 "audit-log"

# Analysis 1: Who made the most API calls today?
tail -10000 /var/log/kubernetes/audit.log | \
  jq -r '.user.username' | \
  sort | uniq -c | sort -rn | head -10

# Analysis 2: What secrets were accessed and by whom?
tail -10000 /var/log/kubernetes/audit.log | \
  jq -r 'select(.objectRef.resource == "secrets" and .verb == "get") | 
          "\(.user.username) accessed \(.objectRef.name) in \(.objectRef.namespace)"' | \
  sort | uniq -c | sort -rn

# Analysis 3: All 403 Forbidden events (access control violations)
tail -10000 /var/log/kubernetes/audit.log | \
  jq 'select(.responseStatus.code == 403) | 
       {user: .user.username,
        verb: .verb,
        resource: .objectRef.resource,
        namespace: .objectRef.namespace,
        time: .requestReceivedTimestamp}' | head -50

# Analysis 4: Pod exec events (potential security concern)
tail -10000 /var/log/kubernetes/audit.log | \
  jq 'select(.objectRef.subresource == "exec") | 
       {user: .user.username,
        pod: .objectRef.name,
        namespace: .objectRef.namespace,
        time: .requestReceivedTimestamp}'

# Analysis 5: RBAC changes
tail -10000 /var/log/kubernetes/audit.log | \
  jq 'select(
    (.objectRef.resource == "roles" or 
     .objectRef.resource == "rolebindings" or
     .objectRef.resource == "clusterroles" or
     .objectRef.resource == "clusterrolebindings") and
    (.verb == "create" or .verb == "update" or .verb == "delete")
  ) | {user: .user.username, verb: .verb, 
       resource: .objectRef.resource, 
       name: .objectRef.name,
       time: .requestReceivedTimestamp}'
```

---

## 28. Interview Questions — Beginner to Advanced

### Beginner Level

**Q1: What is the difference between Authentication and Authorization in Kubernetes?**

**A:** **Authentication (AuthN)** answers "Who are you?" — it verifies the identity of the requester. The API server validates credentials (X.509 certificate, bearer token, OIDC JWT) and extracts a username and group membership from them.

**Authorization (AuthZ)** answers "What are you allowed to do?" — it evaluates whether the authenticated identity has permission to perform the requested action. RBAC does this by checking if any Role/ClusterRole bound to the user's identity grants the requested verb on the requested resource.

You can be authenticated (valid credentials) but not authorized (no RBAC permission). The error for authentication failure is **401 Unauthorized**; for authorization failure, it's **403 Forbidden**.

---

**Q2: What is a ServiceAccount and how is it different from a user account in Kubernetes?**

**A:** A **ServiceAccount** is a Kubernetes-native identity for **workloads running inside the cluster** (Pods, controllers, CI/CD jobs). It exists as an actual Kubernetes resource (`kubectl get serviceaccounts`), is namespace-scoped, and automatically gets a token that Pods can use to call the API server.

A **user account** in Kubernetes is NOT a native Kubernetes resource — Kubernetes has no user database. User identities come from external systems: X.509 certificates (the CN field is the username), OIDC providers (the token's claim is the username), or static token files. You cannot list users with `kubectl get users`.

Key differences: ServiceAccounts can be created/deleted via kubectl; user accounts cannot. ServiceAccounts are namespaced; user accounts span the whole cluster. ServiceAccounts are used by Pods; user accounts are used by humans and external tools.

---

**Q3: What does `kubectl auth can-i` do and why is it useful?**

**A:** `kubectl auth can-i` performs a **SubjectAccessReview** — it asks the Kubernetes API server "can this subject perform this action?" and returns yes or no. It's useful for:

1. **Debugging permissions:** "Why am I getting 403?" — `kubectl auth can-i get deployments -n production` tells you immediately
2. **Testing before applying:** Verify a new ServiceAccount has the right permissions before deploying
3. **Security auditing:** With `--as=<user>` impersonation, test what various users can do without their credentials
4. **CI/CD validation:** Include `kubectl auth can-i` checks in deployment pipelines to verify service account permissions haven't changed

```bash
kubectl auth can-i create pods -n production
kubectl auth can-i --list -n production --as=system:serviceaccount:production:my-app
```

---

### Intermediate Level

**Q4: Explain how OIDC authentication works in Kubernetes and why it's preferred over static tokens.**

**A:** OIDC (OpenID Connect) authentication works as follows:

1. The user logs into an OIDC provider (Google, Okta, Keycloak, Azure AD)
2. The OIDC provider issues a signed **JWT ID token** containing the user's identity (email, groups)
3. The user configures kubectl to present this token as a Bearer token to the API server
4. The API server validates the JWT signature against the OIDC provider's public keys (JWKS endpoint specified by `--oidc-issuer-url`)
5. The API server extracts the username from the claim specified by `--oidc-username-claim` (typically `email`)
6. RBAC evaluates permissions for that username

**Why it's preferred:**
- **Short-lived tokens:** OIDC tokens expire in hours; static tokens never expire
- **Centralized management:** Add/remove users from the OIDC provider; access updates immediately across all clusters
- **No secret distribution:** No long-lived kubeconfig files with static tokens to distribute
- **SSO:** Users use the same identity as for all other corporate systems
- **Group membership:** Groups from the IdP map directly to Kubernetes RBAC groups
- **MFA:** OIDC providers support MFA; static tokens cannot

---

**Q5: A pod has `automountServiceAccountToken: false`. What happens when it tries to call the Kubernetes API?**

**A:** When `automountServiceAccountToken: false` is set, **no service account token is mounted into the Pod's filesystem**. The `/var/run/secrets/kubernetes.io/serviceaccount/` directory doesn't exist in the container.

If the application inside the Pod tries to call the Kubernetes API:
- Without any token → request arrives at API server without Authorization header
- API server checks anonymous authentication: if `--anonymous-auth=true` (not recommended in production), the request is processed as `system:anonymous`; if `--anonymous-auth=false`, it returns 401 Unauthorized
- Either way, with RBAC, anonymous requests will get 403 Forbidden for almost everything

**When to use this pattern:** For security-conscious deployments where the application doesn't need API access (web servers, databases, stateless microservices). This follows least-privilege: if you don't need API access, don't have it.

**If the application DOES need API access:** Enable `automountServiceAccountToken: true` and create an explicit RBAC binding with minimal permissions. Or better: use a projected volume with a short-lived bound token.

---

**Q6: How does Kubernetes prevent one kubelet from accessing another node's secrets?**

**A:** This is handled by the **Node authorization mode** combined with the **NodeRestriction admission controller**:

**Node authorization mode** (`--authorization-mode=Node`): When a kubelet (authenticated as `system:node:<node-name>` in the `system:nodes` group) makes API requests, the Node authorizer enforces that it can only:
- Read Secrets, ConfigMaps, and Services for Pods **scheduled on its own node**
- Update status only for **its own** Node and Pods running on it
- NOT read secrets for Pods on other nodes

**NodeRestriction admission controller**: Further restricts kubelets from modifying labels/taints on other nodes, preventing node impersonation.

**Certificate binding**: The kubelet authenticates with its own X.509 certificate (`system:node:worker-1`). This identity is what the Node authorizer uses to determine "which node is this?" and thus "which Pods' secrets can it access?"

Result: Even if a node is compromised, the attacker can only access secrets for Pods scheduled on THAT node — not the entire cluster.

---

### Advanced Level

**Q7: Design a production access model for a 50-person engineering organization using Kubernetes. Include authentication, authorization, and operational procedures.**

**A:** A production access model for 50 engineers:

**Authentication Layer (SSO via Okta/Azure AD):**
- Configure kube-apiserver with `--oidc-issuer-url`, `--oidc-username-claim=email`, `--oidc-groups-claim=groups`
- Teams in IdP: `k8s-platform-team`, `k8s-team-a`, `k8s-team-b`, `k8s-sre-oncall`, `k8s-security-auditors`
- kubectl configured with `oidc-login` plugin for browser-based SSO flow

**Authorization (RBAC):**
```
Platform team → ClusterRole: cluster-admin (break-glass only, via temporary binding)
Platform team → ClusterRole: platform-admin (create namespaces, manage cluster-scoped resources)
Team A/B engineers → ClusterRole: edit bound in their namespaces only
SRE on-call → ClusterRole: custom sre-oncall-reader (read + exec cluster-wide)
Security auditors → ClusterRole: view (read all resources including secrets)
CI/CD (GitHub Actions) → Role: deploy (update deployments/configmaps in specific ns)
```

**Operational Procedures:**
1. **Onboarding**: Add to IdP group → access works immediately (no kubectl needed)
2. **Offboarding**: Remove from IdP → access revoked immediately (no kubeconfig rotation needed)
3. **Production deployments**: Via GitOps (ArgoCD); humans have read-only access in production
4. **Emergency access**: Break-glass procedure: TechLead approves, temporary ClusterRoleBinding created with 4-hour TTL, action logged in ticketing system
5. **Quarterly RBAC audit**: Review all ClusterRoleBindings, ensure no stale bindings
6. **Certificate rotation**: Automated via cert-manager or kubeadm scheduled renewal

---

**Q8: Explain exactly what happens — step by step — when a Pod's ServiceAccount token is used to call the Kubernetes API, including how the token is validated.**

**A:** Full request lifecycle for a Pod calling the API:

1. **Token mounting**: kubelet requests a bound ServiceAccount token from the TokenRequest API (not a static Secret). The token contains: `iss` (issuer = cluster API server URL), `sub` (ServiceAccount), `exp` (expiry), `aud` (audience), `kubernetes.io/pod.name` (binding to specific Pod)

2. **Token storage**: kubelet writes the token to `/var/run/secrets/kubernetes.io/serviceaccount/token` inside the Pod container via a projected volume. The kubelet also handles token rotation before expiry.

3. **Pod uses token**: The application reads the token from the filesystem and adds `Authorization: Bearer <token>` to its HTTP request to `https://kubernetes.default.svc:443`.

4. **DNS resolution**: `kubernetes.default.svc` resolves to the kube-dns ClusterIP for the Kubernetes service, which forwards to the API server.

5. **TLS handshake**: API server presents its TLS certificate. The client verifies it against `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`.

6. **Authentication**: API server receives the Bearer token. The **TokenReview API** authenticates it:
   - Checks signature (signed with `--service-account-signing-key-file`)
   - Validates `exp` (not expired)
   - Validates `iss` (matches `--service-account-issuer`)
   - Validates `aud` (matches configured audiences)
   - Verifies Pod binding: is the Pod referenced in the token still running?
   - Extracts: `username=system:serviceaccount:<namespace>:<sa-name>`, `groups=["system:serviceaccounts", "system:serviceaccounts:<namespace>"]`

7. **Authorization**: RBAC authorizer checks: does `system:serviceaccount:<ns>:<sa>` have permission for the requested verb + resource + namespace?

8. **Response**: API server returns result; 403 if not authorized, 200 with data if authorized.

---

## 29. Cheat Sheet — Commands, Flags & Configs

### 29.1 kubeconfig Management

```bash
kubectl config current-context
kubectl config get-contexts
kubectl config use-context <n>
kubectl config set-context --current --namespace=<ns>
kubectl config view --raw
kubectl config view --minify       # Show only current context
KUBECONFIG=~/.kube/config1:~/.kube/config2 kubectl config view --flatten
```

### 29.2 Authentication Verification

```bash
# Who am I?
kubectl auth whoami                          # Shows current identity
kubectl config view --raw | grep user -A5

# Test a specific permission
kubectl auth can-i <verb> <resource> -n <ns>
kubectl auth can-i --list -n <ns>
kubectl auth can-i --list -n <ns> --as=<user>
kubectl auth can-i get secrets -n prod --as=system:serviceaccount:prod:my-app
```

### 29.3 RBAC Management

```bash
# View RBAC resources
kubectl get clusterroles,clusterrolebindings
kubectl get roles,rolebindings -n <ns>

# Create role/binding imperatively
kubectl create role <n> --verb=get,list --resource=pods -n <ns>
kubectl create rolebinding <n> --role=<n> --user=<user> -n <ns>
kubectl create clusterrolebinding <n> --clusterrole=<n> --user=<user>

# Describe for rule detail
kubectl describe clusterrole cluster-admin
kubectl describe rolebinding <n> -n <ns>

# Test as another user
kubectl get pods --as=<user> -n <ns>
kubectl get pods --as=system:serviceaccount:<ns>:<sa> -n <ns>

# Find who can do X
kubectl who-can delete pods -n production   # Requires krew plugin
```

### 29.4 ServiceAccount Management

```bash
kubectl get serviceaccounts --all-namespaces
kubectl create serviceaccount <n> -n <ns>
kubectl describe serviceaccount <n> -n <ns>

# Create short-lived token
kubectl create token <sa-name> -n <ns> --duration=1h

# Get token from Secret (legacy)
kubectl get secret <sa-token-secret> -n <ns> \
  -o jsonpath='{.data.token}' | base64 -d

# Disable auto-mount
kubectl patch serviceaccount default -n <ns> \
  -p '{"automountServiceAccountToken": false}'
```

### 29.5 Certificate Management

```bash
# Check certificate expiration
kubeadm certs check-expiration

# Renew all certificates
kubeadm certs renew all
kubeadm certs renew apiserver

# CertificateSigningRequest workflow
kubectl get csr
kubectl certificate approve <csr-name>
kubectl certificate deny <csr-name>
kubectl get csr <n> -o jsonpath='{.status.certificate}' | base64 -d > user.crt

# Inspect a certificate
openssl x509 -in cert.crt -noout -text | grep -E "Subject:|Validity|DNS:|IP"
openssl verify -CAfile /etc/kubernetes/pki/ca.crt cert.crt
```

### 29.6 Access Troubleshooting

```bash
# Verbose API calls (see exact request/response)
kubectl get pods -v=6    # URL + status code
kubectl get pods -v=8    # Full headers
kubectl get pods -v=9    # Full request + response body

# Check what API calls a command makes
kubectl get pods -v=6 2>&1 | grep "GET\|POST\|PUT\|PATCH\|DELETE"

# Audit log tail
tail -f /var/log/kubernetes/audit.log | \
  jq 'select(.responseStatus.code >= 400)'

# Force re-authentication (refresh OIDC token)
kubectl oidc-login get-token \
  --oidc-issuer-url=<url> \
  --oidc-client-id=<id>
```

### 29.7 Important kube-apiserver Flags for Access

```bash
# Authentication
--anonymous-auth=false
--oidc-issuer-url=<url>
--oidc-client-id=<id>
--oidc-username-claim=email
--oidc-groups-claim=groups

# Authorization  
--authorization-mode=Node,RBAC

# Admission
--enable-admission-plugins=NodeRestriction,PodSecurity,...

# Security
--tls-min-version=VersionTLS12
--service-account-max-token-expiration=86400s

# Audit
--audit-log-path=/var/log/kubernetes/audit.log
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
--audit-log-maxage=30
```

### 29.8 Emergency Access Procedure

```bash
# Last resort: use admin.conf on control plane node
kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes

# Create emergency admin binding
kubectl create clusterrolebinding emergency-admin \
  --clusterrole=cluster-admin \
  --user=admin@company.com

# Remove after incident
kubectl delete clusterrolebinding emergency-admin

# Certificate renewal if everything else fails
kubeadm certs renew all
kubectl --kubeconfig=/etc/kubernetes/admin.conf get nodes
```

---

## 30. Key Takeaways & Summary

### The 10 Immutable Laws of Production Kubernetes Access

1. **Authentication precedes Authorization.** You must be identified before your permissions are checked. A 401 means your credentials are invalid; a 403 means your credentials are valid but you lack permission.

2. **Controllers never access etcd directly.** Every access — from kubectl to in-cluster applications to controllers — flows through the API server, which enforces all security controls.

3. **RBAC is default-deny.** A subject with no bindings can do nothing. Design from minimal access upward, not from full access downward.

4. **ServiceAccount tokens are identities.** Treat them like passwords. Short-lived, scoped, rotated, never shared.

5. **SSO (OIDC) for humans; ServiceAccounts for machines.** This separation is the foundation of access governance at scale.

6. **Audit everything.** Without an audit log, you cannot answer "who did what and when?" — a requirement for security, compliance, and incident response.

7. **The API server is the universal enforcement point.** There is no way to access the cluster that bypasses it (in a correctly hardened cluster). This is the guarantee that makes RBAC meaningful.

8. **kubeconfig is a credential.** It deserves the same protection as a password: not in Git, not shared, permissions set to 600, rotated regularly.

9. **Network access control is the first line of defense.** Before AuthN even begins, ensure the API server is not reachable from the internet. VPN or bastion access is mandatory.

10. **Access is dynamic; design for change.** Engineers join and leave, roles evolve, incidents require temporary elevated access. Build your access model to handle these transitions gracefully and securely.

### Access Architecture in One Line

```
Network Gate → TLS → Authentication → RBAC Authorization → Admission → etcd
```

Everything you do in Kubernetes, every command you run, every Pod that calls the API, every controller that reconciles state — it all flows through this exact pipeline, enforced by the API server, with etcd as the ultimate source of truth that no one but the API server can directly access.

---

> **This guide covers Kubernetes v1.29+ access patterns. Always consult the official Kubernetes documentation at https://kubernetes.io/docs/reference/access-authn-authz/ for the most current security guidance.**

---

*End of: How to Access a Kubernetes Cluster in Production — Complete Guide*
