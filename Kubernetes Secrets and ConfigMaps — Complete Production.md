## Table of Contents

1. [Introduction — Why Secrets and ConfigMaps Are Foundational](#1-introduction--why-secrets-and-configmaps-are-foundational)
2. [Core Identity Table — Secrets and ConfigMap Components](#2-core-identity-table--secrets-and-configmap-components)
3. [Understanding the Need for Secrets](#3-understanding-the-need-for-secrets)
4. [ConfigMaps — Decoupling Configuration from Code](#4-configmaps--decoupling-configuration-from-code)
5. [The Controller Pattern — Watch → Compare → Act → Loop](#5-the-controller-pattern--watch--compare--act--loop)
6. [Lab: Creating Secrets — All Methods](#6-lab-creating-secrets--all-methods)
7. [Lab: Using Secrets in Pods](#7-lab-using-secrets-in-pods)
8. [Lab: Creating ConfigMaps — All Methods](#8-lab-creating-configmaps--all-methods)
9. [Lab: Using ConfigMaps in Deployments](#9-lab-using-configmaps-in-deployments)
10. [Secret Types Deep Dive](#10-secret-types-deep-dive)
11. [Built-in Controllers and Their Interaction with Secrets/ConfigMaps](#11-built-in-controllers-and-their-interaction-with-secretsconfigmaps)
12. [Internal Working Concepts — Informers, Work Queues & Reconciliation](#12-internal-working-concepts--informers-work-queues--reconciliation)
13. [API Server and etcd Interaction — The Storage Layer](#13-api-server-and-etcd-interaction--the-storage-layer)
14. [Leader Election and HA for Secret/ConfigMap Management](#14-leader-election-and-ha-for-secretconfigmap-management)
15. [Encryption at Rest — Protecting Secrets in etcd](#15-encryption-at-rest--protecting-secrets-in-etcd)
16. [Dynamic Configuration Updates — Secrets and ConfigMaps Without Restart](#16-dynamic-configuration-updates--secrets-and-configmaps-without-restart)
17. [External Secret Management Integration](#17-external-secret-management-integration)
18. [Performance Tuning for Secret/ConfigMap Usage](#18-performance-tuning-for-secretconfigmap-usage)
19. [Security Hardening Practices](#19-security-hardening-practices)
20. [Monitoring and Observability](#20-monitoring-and-observability)
21. [Troubleshooting Secrets and ConfigMap Issues](#21-troubleshooting-secrets-and-configmap-issues)
22. [Disaster Recovery Concepts](#22-disaster-recovery-concepts)
23. [Comparison — Secrets vs ConfigMap vs Environment Variables vs Volume Files](#23-comparison--secrets-vs-configmap-vs-environment-variables-vs-volume-files)
24. [Comparison — kube-apiserver vs kube-scheduler vs kube-controller-manager (in context)](#24-comparison--kube-apiserver-vs-kube-scheduler-vs-kube-controller-manager-in-context)
25. [ASCII Architecture Diagram](#25-ascii-architecture-diagram)
26. [Real-World Production Use Cases](#26-real-world-production-use-cases)
27. [Best Practices for Production Environments](#27-best-practices-for-production-environments)
28. [Common Mistakes and Pitfalls](#28-common-mistakes-and-pitfalls)
29. [Interview Questions — Beginner to Advanced](#29-interview-questions--beginner-to-advanced)
30. [Cheat Sheet — Commands, Flags & Manifests](#30-cheat-sheet--commands-flags--manifests)
31. [Key Takeaways & Summary](#31-key-takeaways--summary)

---

## 1. Introduction — Why Secrets and ConfigMaps Are Foundational

Modern applications are not static. They need configuration — database connection strings, API keys, feature flags, TLS certificates, service endpoints, log levels — and this configuration varies across environments. Hardcoding configuration into container images creates brittleness: a single image cannot serve development, staging, and production without modification.

Kubernetes solves this with two complementary primitives:

**ConfigMap**: Stores non-sensitive configuration data as key-value pairs. Think environment-specific application settings, configuration files, command arguments.

**Secret**: Stores sensitive data — passwords, tokens, certificates — with additional security controls: base64 encoding in etcd, ability to encrypt at rest, restricted RBAC, and in-memory tmpfs mount delivery.

Together, these two objects enable the **12-Factor App** principle of separating configuration from code. They allow the same container image to run in dev, staging, and production with only the Kubernetes objects around it changing.

### Why This Is Critical for Production

| Problem Without Secrets/ConfigMaps | Solution |
|---|---|
| Credentials hardcoded in images | Secrets injected at runtime; image has no sensitive data |
| Different images per environment | Same image, different ConfigMap per environment |
| Leaked credentials in Git | Secrets managed in Kubernetes (or Vault), not in source control |
| Application restart needed for config change | ConfigMap/Secret volumes update automatically |
| Credentials visible in docker inspect | Secret data in tmpfs; not in container filesystem permanently |
| No audit trail for secret access | Kubernetes audit logs capture every Secret read |

### The Separation Architecture

```
Container Image (immutable, public-safe):
  → Application binary
  → Libraries
  → OS utilities
  → NO credentials, NO env-specific config

ConfigMap (mutable, non-sensitive):
  → Database hostname
  → Log level
  → Feature flags
  → Configuration files

Secret (mutable, sensitive):
  → Database password
  → API keys
  → TLS certificates
  → Service account tokens
```

---

## 2. Core Identity Table — Secrets and ConfigMap Components

| Component | Kind / Binary | API Group | Namespace Scoped | Storage | Encoding | Role |
|---|---|---|---|---|---|---|
| **ConfigMap** | `configmap` | `core/v1` | Yes | etcd plaintext | None | Non-sensitive configuration key-value store |
| **Secret** | `secret` | `core/v1` | Yes | etcd (base64) | base64 | Sensitive data store with access controls |
| **kube-apiserver** | `kube-apiserver` | N/A | N/A | etcd | N/A | Validates, stores, and serves ConfigMaps/Secrets |
| **etcd** | `etcd` | N/A | N/A | On-disk | Optional encryption | Physical storage for all cluster state |
| **kubelet** | `kubelet` | N/A | N/A | Node tmpfs / env | N/A | Mounts Secrets/ConfigMaps into Pods |
| **kube-controller-manager** | `kube-controller-manager` | N/A | N/A | N/A | N/A | ServiceAccount token controller manages Secret lifecycle |
| **ServiceAccount Token Secret** | `secret` (type: SA-token) | `core/v1` | Yes | etcd | base64 JWT | Auto-created token for in-cluster ServiceAccount auth |
| **Docker Registry Secret** | `secret` (type: docker) | `core/v1` | Yes | etcd | base64 JSON | Pull credentials for private container registries |
| **TLS Secret** | `secret` (type: tls) | `core/v1` | Yes | etcd | base64 PEM | TLS certificate and private key pair |
| **EncryptionConfiguration** | Config file | N/A | N/A | API server disk | Encryption config | Configures at-rest encryption for Secrets |
| **RBAC Role/Binding** | `role`, `rolebinding` | `rbac.authorization.k8s.io` | Yes | etcd | N/A | Controls who can read/write Secrets |

---

## 3. Understanding the Need for Secrets

### 3.1 The Problem: Credentials in Images

Before Kubernetes Secrets, many teams would embed credentials directly in container images or pass them as plain-text environment variables in Docker Compose. This creates multiple security vulnerabilities:

```
PROBLEMATIC PATTERNS:
─────────────────────────────────────────────────────────────────────
# Pattern 1: Credentials in Dockerfile (terrible)
ENV DB_PASSWORD=supersecret123

# Pattern 2: Plain text in docker run (visible in process list)
docker run -e DB_PASSWORD=supersecret123 myapp

# Pattern 3: Credentials in docker-compose.yml committed to Git
environment:
  - DB_PASSWORD=supersecret123

# All of these can be discovered by:
# - docker inspect <container>
# - ps aux (visible in /proc/*/environ)
# - git log (if committed)
# - docker history <image>
```

### 3.2 How Kubernetes Secrets Address This

```
SECURE PATTERN WITH KUBERNETES:
─────────────────────────────────────────────────────────────────────
Container Image:
  → application binary
  → NO credentials

Kubernetes Secret:
  → DB_PASSWORD: <base64-encoded>
  → Encrypted at rest in etcd (if configured)
  → RBAC controls who can read
  → Audit-logged every access
  → Delivered via:
     A) Environment variable (simple but visible in process list)
     B) Mounted volume (tmpfs, not written to disk, preferred)
```

### 3.3 What Secrets Protect Against

| Threat | Protection Mechanism |
|---|---|
| **Container image scanning** | Credentials not in the image |
| **Git repository exposure** | Credentials not committed to Git |
| **Docker inspect exposure** | Volume-mounted secrets invisible to inspect |
| **Process list exposure (for env vars)** | Use volume mounts instead of env vars |
| **Node disk exposure** | tmpfs mount — not written to disk |
| **Unauthorized access** | RBAC: only specific ServiceAccounts can read |
| **etcd data breach** | Encryption at rest via AES-CBC or AES-GCM |
| **API server compromise** | Audit logs track every Secret access |

### 3.4 Kubernetes Secret Limitations (Know These!)

Kubernetes Secrets are NOT perfect out of the box:

```
BASE64 ≠ ENCRYPTION:
  → kubectl get secret my-secret -o yaml
  → base64 decode → plaintext
  → Base64 is encoding, not encryption!
  → Anyone with kubectl get secret access can read the value

DEFAULT etcd STORAGE:
  → Without encryption-at-rest configuration:
  → Secrets stored in plaintext in etcd
  → Anyone with etcd access sees plaintext data

API SERVER ACCESS:
  → By default, any Pod can read Secrets in its namespace
  → Must configure RBAC to restrict

MEMORY EXPOSURE:
  → Secrets loaded as env vars remain in process memory
  → Volume mounts are more secure (can be read and discarded)
```

This is why production clusters use:
- **Encryption at rest** (AES-GCM in etcd)
- **Restrictive RBAC** (no wildcard Secret access)
- **External secret managers** (Vault, AWS Secrets Manager)
- **Volume mounts** over environment variables

---

## 4. ConfigMaps — Decoupling Configuration from Code

### 4.1 What ConfigMaps Store

ConfigMaps are designed for non-sensitive, application-tuning data:

```yaml
# ConfigMap can store:
# 1. Simple key-value pairs
data:
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"
  FEATURE_FLAG_X: "enabled"

# 2. Entire configuration files (multi-line values)
data:
  nginx.conf: |
    server {
        listen 80;
        server_name example.com;
        location / {
            proxy_pass http://backend:8080;
        }
    }
  app.properties: |
    spring.datasource.url=jdbc:postgresql://db:5432/mydb
    spring.jpa.show-sql=false
    server.port=8080

# 3. Binary data (base64-encoded)
binaryData:
  logo.png: <base64-encoded-binary>
```

### 4.2 ConfigMap vs Secret Decision Matrix

| Data Type | ConfigMap | Secret |
|---|---|---|
| **Database hostname** | ✅ | ❌ (not sensitive) |
| **Database password** | ❌ | ✅ |
| **Log level** | ✅ | ❌ |
| **API endpoint URL** | ✅ | ❌ |
| **API key** | ❌ | ✅ |
| **TLS certificate** | Cert file (✅) | Private key (✅ Secret) |
| **Feature flags** | ✅ | ❌ |
| **Application config file** | ✅ | ❌ (unless contains secrets) |
| **JWT signing key** | ❌ | ✅ |
| **Service names** | ✅ | ❌ |
| **OAuth tokens** | ❌ | ✅ |
| **SMTP host** | ✅ | ❌ |
| **SMTP password** | ❌ | ✅ |

---

## 5. The Controller Pattern — Watch → Compare → Act → Loop

Understanding the controller pattern explains how Secrets and ConfigMaps behave when they're used by Pods and Deployments.

### 5.1 How the Pattern Applies to Secrets/ConfigMaps

```
┌─────────────────────────────────────────────────────────────────────┐
│          CONTROLLER PATTERN FOR SECRETS/CONFIGMAPS                  │
│                                                                     │
│   WATCH:  Kubelet informer watches Pods assigned to this node       │
│           Tracks which Pods reference which ConfigMaps/Secrets      │
│                                                                     │
│   COMPARE: Pod expected to have ConfigMap mounted at /etc/config    │
│            Currently: ConfigMap not mounted (Pod just created)      │
│                                                                     │
│   ACT:    Kubelet fetches ConfigMap from API server                 │
│           Creates volume mount in container's filesystem             │
│           Injects env vars from Secret into container env           │
│                                                                     │
│   LOOP:   Periodic resync checks for Secret/ConfigMap updates       │
│           If ConfigMap changes → update files in mounted volume      │
│           (Env vars NOT automatically updated — require restart)    │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 Secret/ConfigMap Reconciliation by kubelet

```bash
# When a ConfigMap is updated:
kubectl edit configmap app-config

# What happens:
# 1. API server stores updated ConfigMap in etcd
# 2. kubelet's informer detects MODIFIED event
# 3. For volume-mounted ConfigMap:
#    → Files in mount path updated within ~1-2 minutes (configmap.sync-period)
#    → Application must watch for file changes to reload
# 4. For env-var-injected ConfigMap:
#    → NO automatic update
#    → Pod must be restarted to pick up new values
```

### 5.3 ServiceAccount Token Controller

The kube-controller-manager's **ServiceAccount controller** follows the controller pattern to manage Secret lifecycle:

```
WATCH:   ServiceAccount created/updated
COMPARE: Does a token Secret exist for this SA? No.
ACT:     Create Secret of type kubernetes.io/service-account-token
         Sign JWT with cluster CA key
         Store in etcd via API server
LOOP:    Watch for SA deletion → delete associated Secrets
         Watch for token expiry → rotate if configured
```

---

## 6. Lab: Creating Secrets — All Methods

### 6.1 Method 1: kubectl create secret generic (Literal Values)

```bash
# ── SIMPLE KEY-VALUE SECRETS ─────────────────────────────────────────────

# Create from literal values
kubectl create secret generic db-credentials \
  --from-literal=username=dbadmin \
  --from-literal=password='MyP@ssw0rd!Secure' \
  --from-literal=host=postgres.production.svc.cluster.local

# Verify creation (note: values are base64 encoded)
kubectl get secret db-credentials
# NAME             TYPE     DATA   AGE
# db-credentials   Opaque   3      5s

# View the encoded data
kubectl get secret db-credentials -o yaml
# data:
#   host: cG9zdGdyZXMucHJvZHVjdGlvbi5zdmMuY2x1c3Rlci5sb2NhbA==
#   password: TXlQQHNzdzByZCFTZWN1cmU=
#   username: ZGJhZG1pbg==

# Decode a specific value to verify
kubectl get secret db-credentials \
  -o jsonpath='{.data.password}' | base64 -d
# Output: MyP@ssw0rd!Secure

# Alternative: extract all values
kubectl get secret db-credentials -o json | \
  jq '.data | map_values(@base64d)'
```

### 6.2 Method 2: From Files

```bash
# Create files with sensitive content (don't commit these to Git!)
echo -n "dbadmin" > /tmp/username.txt
echo -n 'MyP@ssw0rd!Secure' > /tmp/password.txt

# Create Secret from files (key = filename, value = file contents)
kubectl create secret generic db-from-files \
  --from-file=username=/tmp/username.txt \
  --from-file=password=/tmp/password.txt

# Create with custom key names
kubectl create secret generic db-custom-keys \
  --from-file=DB_USER=/tmp/username.txt \
  --from-file=DB_PASS=/tmp/password.txt

# From entire directory (all files become keys)
mkdir /tmp/db-secrets
echo -n "dbadmin" > /tmp/db-secrets/username
echo -n 'MyP@ssw0rd!Secure' > /tmp/db-secrets/password
kubectl create secret generic db-from-dir \
  --from-file=/tmp/db-secrets/

# Cleanup temp files
rm -rf /tmp/username.txt /tmp/password.txt /tmp/db-secrets
```

### 6.3 Method 3: YAML Manifest (with base64 encoding)

```bash
# Create the base64-encoded values
echo -n "dbadmin" | base64
# ZGJhZG1pbg==
echo -n "MyP@ssw0rd!Secure" | base64
# TXlQQHNzdzByZCFTZWN1cmU=

# Create Secret via YAML
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: db-yaml-secret
  namespace: default
  labels:
    app: my-app
    tier: backend
  annotations:
    secret.company.com/owner: "platform-team"
    secret.company.com/rotation-due: "2026-01-01"
type: Opaque
# IMPORTANT: data values MUST be base64 encoded
data:
  username: ZGJhZG1pbg==           # dbadmin
  password: TXlQQHNzdzByZCFTZWN1cmU=  # MyP@ssw0rd!Secure
EOF

# Alternative: Use stringData (plain text — Kubernetes auto-encodes)
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: db-stringdata-secret
type: Opaque
# stringData: plain text input (auto base64-encoded before storage)
stringData:
  username: dbadmin
  password: "MyP@ssw0rd!Secure"
  connection-string: "postgres://dbadmin:MyP@ssw0rd!Secure@db:5432/myapp"
EOF

# Note: After creation, stringData is converted to data (base64)
kubectl get secret db-stringdata-secret -o yaml | grep -A 10 "^data:"
```

### 6.4 Method 4: TLS Secret

```bash
# Generate a self-signed TLS certificate (for testing)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /tmp/tls.key \
  -out /tmp/tls.crt \
  -subj "/CN=my-app.example.com/O=MyOrg"

# Create TLS secret
kubectl create secret tls my-app-tls \
  --cert=/tmp/tls.crt \
  --key=/tmp/tls.key

# View the TLS secret structure
kubectl get secret my-app-tls -o yaml
# type: kubernetes.io/tls
# data:
#   tls.crt: <base64-encoded certificate>
#   tls.key: <base64-encoded private key>

# Cleanup
rm /tmp/tls.crt /tmp/tls.key
```

### 6.5 Method 5: Docker Registry Secret

```bash
# Create a secret for pulling from a private registry
kubectl create secret docker-registry registry-credentials \
  --docker-server=registry.example.com \
  --docker-username=robot-user \
  --docker-password=robot-token-abc123 \
  --docker-email=deploy@example.com

# View the structure
kubectl get secret registry-credentials -o yaml
# type: kubernetes.io/dockerconfigjson
# data:
#   .dockerconfigjson: <base64-encoded JSON auth config>

# Decode to see the auth structure
kubectl get secret registry-credentials \
  -o jsonpath='{.data.\.dockerconfigjson}' | \
  base64 -d | jq
```

### 6.6 Inspecting and Managing Secrets

```bash
# List all secrets
kubectl get secrets
kubectl get secrets --all-namespaces
kubectl get secrets -n production

# Describe (shows type and keys but NOT values)
kubectl describe secret db-credentials
# Name:         db-credentials
# Type:         Opaque
# Data
# ====
# host:     50 bytes   ← key names and data sizes, not values
# password: 18 bytes
# username: 7 bytes

# Edit a secret (opens in editor with base64 values)
kubectl edit secret db-credentials

# Delete secrets
kubectl delete secret db-credentials db-from-files
```

---

## 7. Lab: Using Secrets in Pods

### 7.1 Method 1: Secrets as Environment Variables

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
    - name: app
      image: nginx:1.25
      # Method A: Map specific keys to specific env var names
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials    # Secret name
              key: username            # Key within the secret
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
              optional: false         # Pod fails if key missing (default: false)
        # Method B: All keys in a secret as env vars
      envFrom:
        - secretRef:
            name: db-credentials      # All keys become env vars
            optional: true            # Pod starts even if secret doesn't exist
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 200m
          memory: 256Mi
EOF

# Verify env vars are injected
kubectl exec secret-env-pod -- env | grep -E "DB_|USERNAME|PASSWORD|host"
# DB_USERNAME=dbadmin
# DB_PASSWORD=MyP@ssw0rd!Secure
# host=postgres.production.svc.cluster.local

# IMPORTANT: Env vars from secrets are visible in process environment
# Check: kubectl exec secret-env-pod -- cat /proc/1/environ | tr '\0' '\n'
# This shows all env vars including secrets — a security concern

kubectl delete pod secret-env-pod
```

### 7.2 Method 2: Secrets as Volume Mounts (Recommended)

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  volumes:
    # Define the volume from the secret
    - name: db-creds-volume
      secret:
        secretName: db-credentials
        # Optional: set file permissions (octal)
        defaultMode: 0400           # Owner read-only
        # Optional: mount only specific keys
        items:
          - key: username
            path: db-username       # File path within the volume
          - key: password
            path: db-password       # Stored as separate files
            mode: 0400              # Individual file permission

  containers:
    - name: app
      image: nginx:1.25
      volumeMounts:
        - name: db-creds-volume
          mountPath: /etc/db-credentials    # Directory where files are created
          readOnly: true                      # Always mount secrets read-only
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 200m
          memory: 256Mi
EOF

# Verify the mounted files
kubectl exec secret-volume-pod -- ls -la /etc/db-credentials/
# total 0
# lrwxrwxrwx 1 root root 19 Jan 1 10:00 db-password -> ..data/db-password
# lrwxrwxrwx 1 root root 19 Jan 1 10:00 db-username -> ..data/db-username
# Note: Symlinks point to ..data/ — this enables atomic updates!

kubectl exec secret-volume-pod -- cat /etc/db-credentials/db-username
# dbadmin
kubectl exec secret-volume-pod -- cat /etc/db-credentials/db-password
# MyP@ssw0rd!Secure

# Check that it's tmpfs (not written to disk)
kubectl exec secret-volume-pod -- mount | grep db-credentials
# tmpfs on /etc/db-credentials type tmpfs (ro,relatime)
# ← tmpfs: in-memory, not persisted to node disk!

kubectl delete pod secret-volume-pod
```

### 7.3 Method 3: Image Pull Secret

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: private-registry-pod
spec:
  # Use the registry secret for image pulling
  imagePullSecrets:
    - name: registry-credentials
  containers:
    - name: app
      image: registry.example.com/my-private-app:v1.0   # Private image
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
EOF

# Or add the secret to a ServiceAccount so ALL pods using it can pull
kubectl patch serviceaccount default \
  -p '{"imagePullSecrets": [{"name": "registry-credentials"}]}'
```

### 7.4 Complete Production Pod with Secrets

```bash
# First, create the required secrets
kubectl create secret generic app-secrets \
  --from-literal=db_password='SuperSecret123!' \
  --from-literal=api_key='sk-live-abc123xyz' \
  --from-literal=jwt_secret='jwt-signing-key-very-long-random-string'

kubectl create secret generic oauth-secrets \
  --from-literal=client_id='oauth-client-123' \
  --from-literal=client_secret='oauth-client-secret-456'

# Deploy with comprehensive secret usage
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: production-app
  template:
    metadata:
      labels:
        app: production-app
    spec:
      # Use dedicated ServiceAccount (not default)
      serviceAccountName: production-app-sa
      automountServiceAccountToken: false

      volumes:
        # Credentials as files (preferred for sensitive values)
        - name: db-credentials
          secret:
            secretName: app-secrets
            items:
              - key: db_password
                path: db_password
            defaultMode: 0400
        # OAuth credentials as files
        - name: oauth-creds
          secret:
            secretName: oauth-secrets
            defaultMode: 0400

      containers:
        - name: api
          image: my-api:v2.0
          # Simple config values as env vars (non-sensitive)
          env:
            - name: APP_ENV
              value: "production"
            - name: LOG_LEVEL
              value: "warn"
            # Inject specific API key from secret
            - name: EXTERNAL_API_KEY
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: api_key
          # Sensitive values mounted as files
          volumeMounts:
            - name: db-credentials
              mountPath: /run/secrets/db
              readOnly: true
            - name: oauth-creds
              mountPath: /run/secrets/oauth
              readOnly: true
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          securityContext:
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1000
EOF

# In the application code, read from files:
# db_password = open('/run/secrets/db/db_password').read()
# client_id = open('/run/secrets/oauth/client_id').read()

# Cleanup
kubectl delete deployment production-app
kubectl delete secret app-secrets oauth-secrets
```

---

## 8. Lab: Creating ConfigMaps — All Methods

### 8.1 Method 1: From Literal Values

```bash
# Create ConfigMap with simple key-value pairs
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=MAX_CONNECTIONS=100 \
  --from-literal=CACHE_TTL=300 \
  --from-literal=ENABLE_METRICS=true \
  --from-literal=DB_HOST=postgres.production.svc.cluster.local \
  --from-literal=DB_PORT=5432 \
  --from-literal=DB_NAME=myapp_production

# Verify
kubectl get configmap app-config -o yaml
kubectl describe configmap app-config
```

### 8.2 Method 2: From Configuration Files

```bash
# Create configuration files
mkdir /tmp/app-config

cat > /tmp/app-config/app.properties << 'EOF'
spring.datasource.url=jdbc:postgresql://postgres:5432/myapp
spring.datasource.driver-class-name=org.postgresql.Driver
spring.jpa.show-sql=false
spring.jpa.hibernate.ddl-auto=validate
server.port=8080
management.endpoints.web.exposure.include=health,metrics,prometheus
management.endpoint.health.show-details=always
logging.level.root=WARN
logging.level.com.myapp=INFO
EOF

cat > /tmp/app-config/nginx.conf << 'EOF'
server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 10s;
        proxy_read_timeout 30s;
    }

    location /health {
        proxy_pass http://localhost:8080/actuator/health;
        access_log off;
    }
}
EOF

cat > /tmp/app-config/log4j2.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{ISO8601} [%t] %-5level %logger{36} - %msg%n"/>
        </Console>
    </Appenders>
    <Loggers>
        <Root level="info">
            <AppenderRef ref="Console"/>
        </Root>
    </Loggers>
</Configuration>
EOF

# Create ConfigMap from entire directory
kubectl create configmap app-config-files \
  --from-file=/tmp/app-config/

# Verify all files are present as keys
kubectl describe configmap app-config-files
# Name:         app-config-files
# Data
# ====
# app.properties: ----   (entire file content as value)
# nginx.conf:     ----
# log4j2.xml:     ----

# Create ConfigMap from single file with custom key name
kubectl create configmap nginx-config \
  --from-file=nginx.conf=/tmp/app-config/nginx.conf

# Cleanup
rm -rf /tmp/app-config
```

### 8.3 Method 3: YAML Manifest (Declarative)

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: application-config
  namespace: default
  labels:
    app: my-api
    environment: production
    managed-by: helm
  annotations:
    description: "Main application configuration for my-api"
    last-updated-by: "platform-team"
data:
  # Simple key-value pairs (used as env vars)
  LOG_LEVEL: "warn"
  MAX_CONNECTIONS: "200"
  CACHE_TTL_SECONDS: "600"
  ENABLE_TRACING: "true"
  SERVICE_PORT: "8080"
  METRICS_PORT: "9090"

  # Configuration files (used as volume-mounted files)
  application.yml: |
    server:
      port: 8080
      shutdown: graceful

    spring:
      lifecycle:
        timeout-per-shutdown-phase: 30s
      datasource:
        url: jdbc:postgresql://postgres:5432/myapp
        hikari:
          maximum-pool-size: 20
          minimum-idle: 5

    management:
      endpoints:
        web:
          exposure:
            include: health,metrics,prometheus
      endpoint:
        health:
          show-details: always

    logging:
      level:
        root: WARN
        com.myapp: INFO
        org.springframework: WARN

  nginx.conf: |
    worker_processes auto;
    events {
        worker_connections 1024;
    }
    http {
        include /etc/nginx/mime.types;
        server {
            listen 80;
            location / {
                proxy_pass http://127.0.0.1:8080;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
            }
            location /health {
                proxy_pass http://127.0.0.1:8080/actuator/health;
                access_log off;
            }
        }
    }
EOF

# Verify
kubectl get configmap application-config -o yaml
kubectl describe configmap application-config
```

### 8.4 Environment-Specific ConfigMaps

```bash
# Pattern: Different ConfigMaps per environment with same structure

# Development ConfigMap
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: development
data:
  LOG_LEVEL: "debug"
  DB_HOST: "postgres-dev.development.svc.cluster.local"
  REPLICA_COUNT: "1"
  CACHE_ENABLED: "false"
  ENABLE_DEBUG_ENDPOINTS: "true"
EOF

# Production ConfigMap (same name, different namespace)
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  LOG_LEVEL: "warn"
  DB_HOST: "postgres.production.svc.cluster.local"
  REPLICA_COUNT: "3"
  CACHE_ENABLED: "true"
  ENABLE_DEBUG_ENDPOINTS: "false"
EOF

# The Deployment YAML is identical in both namespaces
# Only the ConfigMap values differ — true config/code separation
```

---

## 9. Lab: Using ConfigMaps in Deployments

### 9.1 Method 1: ConfigMap Keys as Environment Variables

```bash
# Ensure configmap exists
kubectl create configmap app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=MAX_CONNECTIONS=100 \
  --from-literal=DB_HOST=postgres:5432 \
  --from-literal=CACHE_TTL=300

# Deploy using ConfigMap for environment variables
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: configmap-env-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: configmap-env-demo
  template:
    metadata:
      labels:
        app: configmap-env-demo
    spec:
      containers:
        - name: app
          image: nginx:1.25
          # Method A: Load ALL keys as environment variables
          envFrom:
            - configMapRef:
                name: app-config
                optional: false    # Fail if ConfigMap missing
          # Method B: Load SPECIFIC keys with custom env var names
          env:
            - name: DATABASE_HOST    # Custom env var name
              valueFrom:
                configMapKeyRef:
                  name: app-config  # ConfigMap name
                  key: DB_HOST      # Key within ConfigMap
            - name: CONNECTION_POOL_SIZE
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: MAX_CONNECTIONS
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
EOF

# Verify environment variables are injected
kubectl exec deployment/configmap-env-demo -- env | grep -E "LOG|MAX|DB|CACHE|DATABASE|POOL"
# LOG_LEVEL=info
# MAX_CONNECTIONS=100
# DB_HOST=postgres:5432
# CACHE_TTL=300
# DATABASE_HOST=postgres:5432     ← Custom name mapping
# CONNECTION_POOL_SIZE=100         ← Custom name mapping

kubectl delete deployment configmap-env-demo
```

### 9.2 Method 2: ConfigMap as Volume (Recommended for Config Files)

```bash
# Create ConfigMap with config files
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    server {
        listen 80;
        server_name _;
        root /usr/share/nginx/html;
        index index.html;

        location / {
            try_files $uri $uri/ /index.html;
        }

        location /health {
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }

  default.conf: |
    server_tokens off;
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-XSS-Protection "1; mode=block";
EOF

# Deploy with ConfigMap as volume mount
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-with-config
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx-configured
  template:
    metadata:
      labels:
        app: nginx-configured
    spec:
      volumes:
        # Volume backed by the ConfigMap
        - name: nginx-config-vol
          configMap:
            name: nginx-config
            # Optional: set permissions on config files
            defaultMode: 0644
            # Optional: mount only specific keys as files
            items:
              - key: nginx.conf
                path: nginx.conf    # File path within the volume
              - key: default.conf
                path: conf.d/default.conf   # Subdirectory path

      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          volumeMounts:
            # Mount ConfigMap files into container
            - name: nginx-config-vol
              mountPath: /etc/nginx   # Replace default nginx config dir
              readOnly: true
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
          livenessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
EOF

# Wait for pods to be ready
kubectl rollout status deployment/nginx-with-config

# Verify the config files are mounted
kubectl exec deployment/nginx-with-config -- cat /etc/nginx/nginx.conf

# Test nginx is using the custom config
kubectl port-forward deployment/nginx-with-config 8080:80 &
curl http://localhost:8080/health
# Response: healthy

kill %1  # Kill port-forward
```

### 9.3 Method 3: Combined ConfigMap + Secret in One Deployment

```bash
# Create required objects
kubectl create configmap api-app-config \
  --from-literal=LOG_LEVEL=info \
  --from-literal=DB_HOST=postgres.default.svc.cluster.local \
  --from-literal=DB_PORT=5432 \
  --from-literal=DB_NAME=production_db \
  --from-literal=REDIS_HOST=redis.default.svc.cluster.local \
  --from-literal=REDIS_PORT=6379

kubectl create secret generic api-app-secrets \
  --from-literal=DB_PASSWORD='SecurePass123!' \
  --from-literal=REDIS_PASSWORD='RedisPass456!' \
  --from-literal=JWT_SECRET='very-long-random-jwt-signing-key-abc123xyz'

# Production deployment combining both
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-server
  annotations:
    kubernetes.io/change-cause: "v1.2.0 - Added Redis caching support"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api-server
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: api-server
        version: "1.2.0"
    spec:
      serviceAccountName: api-server-sa
      automountServiceAccountToken: false

      volumes:
        # Config files from ConfigMap
        - name: app-config-files
          configMap:
            name: api-app-config
            items:
              - key: DB_HOST
                path: db_host
        # Sensitive files from Secret
        - name: app-secrets-vol
          secret:
            secretName: api-app-secrets
            defaultMode: 0400
            items:
              - key: DB_PASSWORD
                path: db_password
              - key: JWT_SECRET
                path: jwt_secret

      containers:
        - name: api
          image: my-api:v1.2.0
          ports:
            - name: http
              containerPort: 8080
            - name: metrics
              containerPort: 9090

          # Non-sensitive config from ConfigMap
          envFrom:
            - configMapRef:
                name: api-app-config

          # Additional secret values as env vars (for simple string values)
          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: api-app-secrets
                  key: REDIS_PASSWORD

          # Sensitive values as files
          volumeMounts:
            - name: app-secrets-vol
              mountPath: /run/secrets/app
              readOnly: true

          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi

          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1000
            capabilities:
              drop:
                - ALL

          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
EOF

kubectl rollout status deployment/api-server

# Cleanup
kubectl delete deployment api-server
kubectl delete configmap api-app-config
kubectl delete secret api-app-secrets
kubectl delete deployment nginx-with-config
kubectl delete configmap nginx-config
```

### 9.4 Updating ConfigMap and Rolling Update

```bash
# Demonstrate ConfigMap update → Deployment rollout
kubectl create configmap webapp-config \
  --from-literal=FEATURE_FLAG=disabled \
  --from-literal=MAX_WORKERS=4

cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: feature-flag-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: feature-flag-demo
  template:
    metadata:
      labels:
        app: feature-flag-demo
      annotations:
        # This annotation forces a rollout when ConfigMap changes
        # Update it to force restart
        configmap-version: "v1"
    spec:
      containers:
        - name: app
          image: nginx:1.25
          envFrom:
            - configMapRef:
                name: webapp-config
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
EOF

# Verify initial config
kubectl exec deployment/feature-flag-demo -- env | grep FEATURE
# FEATURE_FLAG=disabled

# Update the ConfigMap
kubectl patch configmap webapp-config \
  --type merge \
  -p '{"data":{"FEATURE_FLAG":"enabled","MAX_WORKERS":"8"}}'

# Env vars are NOT automatically updated in running pods
# Must trigger a rollout to pick up new env var values
kubectl rollout restart deployment/feature-flag-demo

# Monitor the rollout
kubectl rollout status deployment/feature-flag-demo

# Verify new config is active
kubectl exec deployment/feature-flag-demo -- env | grep FEATURE
# FEATURE_FLAG=enabled

# For volume-mounted ConfigMaps: files update automatically
# (within kubelet's sync period, ~1-2 minutes)
# No rollout needed for volume-mounted config

# Cleanup
kubectl delete deployment feature-flag-demo
kubectl delete configmap webapp-config
```

---

## 10. Secret Types Deep Dive

### 10.1 Built-in Secret Types

| Type | Key | Use Case | Created By |
|---|---|---|---|
| `Opaque` | Any | Generic secrets (passwords, tokens) | Manual |
| `kubernetes.io/service-account-token` | `token`, `ca.crt`, `namespace` | In-cluster API authentication | ServiceAccount controller |
| `kubernetes.io/dockercfg` | `.dockercfg` | Docker config v1 registry auth | Manual (legacy) |
| `kubernetes.io/dockerconfigjson` | `.dockerconfigjson` | Docker config v2 registry auth | Manual |
| `kubernetes.io/basic-auth` | `username`, `password` | Basic auth credentials | Manual |
| `kubernetes.io/ssh-auth` | `ssh-privatekey` | SSH private key | Manual |
| `kubernetes.io/tls` | `tls.crt`, `tls.key` | TLS certificate + key | Manual / cert-manager |
| `bootstrap.kubernetes.io/token` | Various | Node bootstrap tokens | Bootstrap controller |

### 10.2 Secret Size Limits

```bash
# Maximum Secret size: 1 MiB (enforced by etcd)
# This applies to the raw data (before base64 encoding)
# base64 adds ~33% overhead → effective limit ~0.75 MiB of raw data

# For large files (SSL certificates, etc.) use:
# 1. Volumes with external secret stores (Vault)
# 2. Multiple Secrets for large data
# 3. ConfigMap for non-sensitive large configs

# Check total etcd storage used by secrets
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl get /registry/secrets --prefix --keys-only \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key | wc -l
```

---

## 11. Built-in Controllers and Their Interaction with Secrets/ConfigMaps

### 11.1 ServiceAccount Controller — Token Management

The ServiceAccount controller watches ServiceAccount objects and automatically creates token Secrets:

```bash
# Create a ServiceAccount
kubectl create serviceaccount my-app-sa -n production

# Controller automatically creates a token Secret (legacy behavior <1.24)
kubectl get secrets -n production | grep my-app-sa
# my-app-sa-token-xxxxx   kubernetes.io/service-account-token

# In 1.24+: Tokens are created on-demand via TokenRequest API (not as persistent Secrets)
kubectl create token my-app-sa -n production --duration=1h

# The token Secret contains:
kubectl describe secret my-app-sa-token-xxxxx -n production
# ca.crt:     The cluster CA certificate
# namespace:  The namespace this token is valid for  
# token:      JWT token signed by cluster CA
```

### 11.2 Deployment Controller and ConfigMap/Secret Dependencies

```bash
# Deployment controller creates Pods with ConfigMap/Secret references
# If a referenced Secret doesn't exist:
kubectl create deployment broken-deploy \
  --image=nginx:1.25 \
  -- env-from-secret non-existent-secret

# The Pod will fail with:
# Error: secret "non-existent-secret" not found
# This prevents Pod from starting → Deployment shows 0/1 available

# Fix: Create the secret first, then create the deployment
# OR use optional: true for non-critical secrets

# Verify dependency error:
kubectl describe pod <broken-pod-name>
# Events:
#   Warning  Failed     5s    kubelet  Error: secret "non-existent-secret" not found
```

### 11.3 ReplicaSet Controller

The ReplicaSet controller creates and maintains Pod replicas. Each Pod it creates has access to Secrets and ConfigMaps as defined in the Pod template.

```bash
# RS controller behavior with Secrets:
# 1. RS creates Pods from template
# 2. Template specifies volume mounts and env refs
# 3. Each Pod independently reads its ConfigMap/Secret
# 4. If Secret is deleted WHILE Pods are running:
#    → Existing Pods continue with cached data
#    → New Pods fail to start (Secret not found)

# Demonstrate:
kubectl create secret generic temp-secret \
  --from-literal=key=value

kubectl create deployment secret-test --image=nginx:1.25
kubectl patch deployment secret-test \
  -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","env":[{"name":"KEY","valueFrom":{"secretKeyRef":{"name":"temp-secret","key":"key"}}}]}]}}}}'

kubectl get pods -l app=secret-test  # Running

kubectl delete secret temp-secret    # Delete while pod is running

# Existing pod: continues running with the env var already set
# kubectl exec deployment/secret-test -- env | grep KEY
# KEY=value ← Still there from initial injection

# New pods (after restart): fail to start
kubectl rollout restart deployment/secret-test
kubectl get pods -l app=secret-test
# CrashLoopBackOff or Error:  secret "temp-secret" not found

kubectl delete deployment secret-test
```

### 11.4 StatefulSet Controller

StatefulSets often use Secrets for database credentials and ConfigMaps for database configuration:

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
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
      volumes:
        - name: postgres-config
          configMap:
            name: postgres-config     # PG configuration
        - name: postgres-init
          configMap:
            name: postgres-init-sql   # Initialization SQL
      containers:
        - name: postgres
          image: postgres:14-alpine
          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: username
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: POSTGRES_DB
              valueFrom:
                configMapKeyRef:
                  name: postgres-config
                  key: DB_NAME
          volumeMounts:
            - name: postgres-config
              mountPath: /etc/postgresql
              readOnly: true
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 1Gi
EOF
```

### 11.5 DaemonSet Controller

DaemonSets for monitoring/logging agents use ConfigMaps for agent configuration:

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      tolerations:
        - operator: Exists
          effect: NoSchedule
      volumes:
        - name: fluentd-config
          configMap:
            name: fluentd-config     # Fluentd configuration
        - name: output-credentials
          secret:
            secretName: elasticsearch-credentials  # Output credentials
      containers:
        - name: fluentd
          image: fluent/fluentd:v1.16
          volumeMounts:
            - name: fluentd-config
              mountPath: /fluentd/etc
              readOnly: true
            - name: output-credentials
              mountPath: /run/secrets/output
              readOnly: true
          resources:
            requests:
              cpu: 100m
              memory: 200Mi
            limits:
              cpu: 500m
              memory: 500Mi
EOF
```

### 11.6 Job Controller

Jobs use ConfigMaps for job parameters and Secrets for credentials:

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: data-migration
spec:
  template:
    spec:
      restartPolicy: OnFailure
      volumes:
        - name: migration-config
          configMap:
            name: migration-params
        - name: db-credentials
          secret:
            secretName: migration-db-secret
      containers:
        - name: migrator
          image: my-migrator:v1.0
          envFrom:
            - configMapRef:
                name: migration-params
          volumeMounts:
            - name: db-credentials
              mountPath: /run/secrets
              readOnly: true
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: "2"
              memory: 2Gi
EOF
```

### 11.7 Node Controller

The Node controller applies system taints when nodes become unhealthy. This can cause Pods to be evicted, including Pods with Secrets/ConfigMaps mounted. When evicted Pods are rescheduled on a new node, kubelet re-mounts the Secrets and ConfigMaps on the new node.

### 11.8 Namespace Controller

When a Namespace is deleted, the Namespace controller cascades deletion to ALL resources in that namespace, including ConfigMaps and Secrets. Plan namespace deletion carefully.

### 11.9 Garbage Collector

The Garbage Collector uses `ownerReferences` to clean up dependent resources. ServiceAccount token Secrets have the ServiceAccount as their owner — deleting the ServiceAccount triggers the GC to delete its token Secrets.

### 11.10 PersistentVolume Controller

PV controller doesn't directly interact with Secrets but is often combined with Secrets via Storage Classes that reference Secret credentials (e.g., for NFS, Ceph, or AWS EFS credentials).

---

## 12. Internal Working Concepts — Informers, Work Queues & Reconciliation

### 12.1 How kubelet Watches for ConfigMap/Secret Updates

```
┌─────────────────────────────────────────────────────────────────────┐
│          KUBELET CONFIGMAP/SECRET INFORMER                          │
│                                                                     │
│  kubelet maintains a CACHE of all ConfigMaps and Secrets            │
│  referenced by Pods on this node.                                   │
│                                                                     │
│  INFORMER WATCH:                                                    │
│  GET /api/v1/namespaces/default/configmaps?watch=true               │
│  GET /api/v1/namespaces/default/secrets?watch=true                  │
│  (Only objects referenced by local Pods — not ALL objects)          │
│                                                                     │
│  WORK QUEUE:                                                        │
│  ConfigMap MODIFIED event → queue item: "default/app-config"        │
│  Worker picks up item → reconcile                                   │
│                                                                     │
│  RECONCILE:                                                         │
│  For each Pod using this ConfigMap as a VOLUME:                     │
│    → Re-read ConfigMap data from cache                              │
│    → Update files in the mounted volume (via ..data/ symlink swap) │
│    → Atomic: symlink swap prevents partial reads                    │
│                                                                     │
│  For Pods using ConfigMap as ENV VARS:                              │
│    → Nothing: env vars only injected at pod start                  │
│    → Only a pod restart updates env-var-injected values            │
└─────────────────────────────────────────────────────────────────────┘
```

### 12.2 Volume Mount Update Mechanism — Atomic Symlinks

```bash
# Kubernetes uses a clever symlink mechanism for atomic ConfigMap updates
# Look at the actual mount structure:
kubectl exec <pod-name> -- ls -la /etc/config/
# total 0
# lrwxrwxrwx 1 root root 13 ..data
# lrwxrwxrwx 1 root root 17 ..data/key1 -> ..2025_01_01_10_00_00.123456/key1
# lrwxrwxrwx 1 root root 17 ..data/key2 -> ..2025_01_01_10_00_00.123456/key2
# lrwxrwxrwx 1 root root 9  key1 -> ..data/key1
# lrwxrwxrwx 1 root root 9  key2 -> ..data/key2

# When ConfigMap is updated:
# 1. New directory created: ..2025_01_01_10_05_00.123456/
# 2. New files written to that directory
# 3. ..data symlink atomically updated to point to new directory
# 4. Application now reads new files via the stable key1/key2 symlinks
# 5. Old directory removed

# This atomic swap prevents files from being read partially during update
```

### 12.3 Reconciliation Loop for Secret Changes

```bash
# The kubelet sync period for ConfigMaps and Secrets:
# Default: --sync-frequency=1m (how often kubelet syncs with API server)
# Volume content updates: within configmap.sync-period (default: 1m)
# Wait up to ~2 minutes for volume-mounted updates

# Observe the update happening
kubectl create configmap update-test --from-literal=version=v1

cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: config-update-test
spec:
  volumes:
    - name: config
      configMap:
        name: update-test
  containers:
    - name: watcher
      image: busybox:1.36
      command: ["sh", "-c", "while true; do cat /config/version; echo ''; sleep 5; done"]
      volumeMounts:
        - name: config
          mountPath: /config
      resources:
        requests:
          cpu: 10m
          memory: 16Mi
EOF

# Watch the version in real-time
kubectl logs config-update-test -f &

# Update the ConfigMap
kubectl patch configmap update-test \
  --type merge -p '{"data":{"version":"v2"}}'

# Wait ~60 seconds and observe the log change from v1 to v2
# WITHOUT restarting the pod!

kill %1  # Stop log following
kubectl delete pod config-update-test
kubectl delete configmap update-test
```

---

## 13. API Server and etcd Interaction — The Storage Layer

### 13.1 The Golden Rule

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  CONTROLLERS NEVER ACCESS ETCD DIRECTLY.                             ║
║                                                                      ║
║  All Secret and ConfigMap operations flow:                           ║
║  Client → kube-apiserver → etcd                                     ║
║                                                                      ║
║  kube-apiserver provides:                                            ║
║  1. Authentication: Who is requesting?                               ║
║  2. Authorization (RBAC): Can they read/write Secrets?               ║
║  3. Admission: Is the object valid?                                  ║
║  4. Audit: Log every Secret read/write with user/time/resource       ║
║  5. Watch cache: Efficiently fan-out to all watchers (kubelet, etc.) ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 13.2 How Secrets Are Stored in etcd

```bash
# Secret storage path in etcd:
# /registry/secrets/<namespace>/<secret-name>

# Without encryption at rest:
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl get /registry/secrets/default/db-credentials \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key | strings
# You'll see base64-encoded values BUT the full JSON is visible

# With encryption at rest (AES-GCM):
# /registry/secrets/default/db-credentials
# → k8s:enc:aesgcm:v1:key1:... (encrypted binary)
# → Cannot decode without the encryption key
```

### 13.3 Secret Watch by API Server

```bash
# The API server maintains an in-memory watch cache for Secrets
# This means kubelet doesn't poll etcd — it gets pushed updates

# API server watch cache for secrets
kubectl get --raw /metrics 2>/dev/null | \
  grep 'watch_cache_capacity{resource="secrets"}'

# Direct etcd key for a specific secret
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl get /registry/secrets --prefix --keys-only \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key 2>/dev/null | head -20
```

---

## 14. Leader Election and HA for Secret/ConfigMap Management

### 14.1 Why Leader Election Matters for Secrets

The `kube-controller-manager` runs the ServiceAccount controller that manages token Secrets. In HA setups, only one instance is the leader:

```bash
# View current leader
kubectl get lease kube-controller-manager -n kube-system -o yaml
# spec.holderIdentity: "k8s-cp-1_abc-123"

# If leader crashes:
# 1. Existing Secrets/ConfigMaps continue to work (stored in etcd)
# 2. New ServiceAccount token creation pauses (~30s for leader election)
# 3. New leader takes over and resumes token management

# Secrets and ConfigMaps are FULLY operational during controller manager failure
# → They're stored in etcd and served directly by API server
# → No controller is needed to serve existing Secrets/ConfigMaps
```

### 14.2 API Server HA for Secret Access

```bash
# API server is Active-Active — multiple instances serve requests
# Secrets/ConfigMaps served from the API server's watch cache
# If one API server fails, others continue serving

# Check API server count
kubectl get pods -n kube-system | grep kube-apiserver

# For high availability:
# - All API servers read from the same etcd cluster
# - Watch caches are rebuilt from etcd if an API server restarts
# - Typical rebuild time: < 30 seconds for most clusters
```

### 14.3 Leader Election Flags

| Flag | Default | Component | Description |
|---|---|---|---|
| `--leader-elect` | `true` | controller-manager | Enable leader election |
| `--leader-elect-lease-duration` | `15s` | controller-manager | Lease validity period |
| `--leader-elect-renew-deadline` | `10s` | controller-manager | Must renew before this |
| `--leader-elect-retry-period` | `2s` | controller-manager | Standby retry interval |

---

## 15. Encryption at Rest — Protecting Secrets in etcd

### 15.1 Why Encryption at Rest Is Critical

Without encryption at rest, Secrets in etcd are only base64-encoded — easily decoded. Anyone with physical or network access to etcd can read all cluster Secrets.

### 15.2 Configuring AES-GCM Encryption

```bash
# Step 1: Generate a 32-byte encryption key
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
echo "Encryption key: $ENCRYPTION_KEY"

# Step 2: Create the EncryptionConfiguration file (on control plane)
cat > /etc/kubernetes/encryption-config.yaml << EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets           # Encrypt secrets
      - configmaps        # Optionally encrypt configmaps too
    providers:
      # PRIMARY: AES-GCM (recommended - authenticated encryption)
      - aesgcm:
          keys:
            - name: key1
              secret: ${ENCRYPTION_KEY}
      # FALLBACK: identity = no encryption (for reading old unencrypted data)
      - identity: {}
EOF

chmod 600 /etc/kubernetes/encryption-config.yaml

# Step 3: Configure API server to use it
# Add to kube-apiserver manifest:
# --encryption-provider-config=/etc/kubernetes/encryption-config.yaml

# Step 4: Restart API server
# (In kubeadm clusters, edit the static pod manifest)
cat /etc/kubernetes/manifests/kube-apiserver.yaml | \
  grep "encryption-provider-config"
# Should show the flag

# Step 5: Encrypt existing secrets (re-write them through API server)
# This forces all existing secrets through the new encryption
kubectl get secrets --all-namespaces -o json | \
  kubectl replace -f -

# Step 6: Verify encryption is working
kubectl create secret generic test-encryption-secret \
  --from-literal=key=mysecretvalue

# Read directly from etcd (should show encrypted data)
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl get /registry/secrets/default/test-encryption-secret \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key | \
  strings | head -5
# Should show: k8s:enc:aesgcm:v1:key1:...
# NOT the plain text value!

kubectl delete secret test-encryption-secret
```

### 15.3 Key Rotation

```bash
# Step 1: Add new key at the TOP of providers list
cat > /etc/kubernetes/encryption-config.yaml << EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aesgcm:
          keys:
            - name: key2   # NEW KEY (first = used for encryption)
              secret: ${NEW_ENCRYPTION_KEY}
            - name: key1   # OLD KEY (still can decrypt)
              secret: ${ENCRYPTION_KEY}
      - identity: {}
EOF

# Step 2: Restart API server (new secrets encrypted with key2, old still readable)

# Step 3: Re-encrypt all secrets with new key
kubectl get secrets --all-namespaces -o json | \
  kubectl replace -f -

# Step 4: Remove old key (after verifying all secrets re-encrypted)
# Remove key1 from encryption-config.yaml
# Restart API server
```

---

## 16. Dynamic Configuration Updates — Secrets and ConfigMaps Without Restart

### 16.1 What Updates Automatically vs What Requires Restart

| Injection Method | ConfigMap Update | Secret Update |
|---|---|---|
| **Volume mount** | ✅ Auto-updates in ~1min | ✅ Auto-updates in ~1min |
| **Environment variable** | ❌ Requires Pod restart | ❌ Requires Pod restart |
| **envFrom** | ❌ Requires Pod restart | ❌ Requires Pod restart |

### 16.2 Building Applications for Dynamic Config

```bash
# Pattern: Application watches for file changes in mounted volumes
# Using inotify in Python:

cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: dynamic-config
data:
  config.json: |
    {
      "feature_flags": {
        "new_checkout": false,
        "beta_ui": false
      },
      "rate_limits": {
        "api": 1000,
        "auth": 100
      }
    }
EOF

# Python app watching for config file changes:
# import json
# import time
# import os
# 
# CONFIG_PATH = '/etc/config/config.json'
# last_mtime = 0
# config = {}
# 
# while True:
#     mtime = os.path.getmtime(CONFIG_PATH)
#     if mtime != last_mtime:
#         with open(CONFIG_PATH) as f:
#             config = json.load(f)
#         last_mtime = mtime
#         print(f"Config reloaded: {config}")
#     time.sleep(5)

# Trigger update without pod restart:
kubectl patch configmap dynamic-config \
  --type merge \
  -p '{"data":{"config.json":"{\"feature_flags\":{\"new_checkout\":true,\"beta_ui\":false},\"rate_limits\":{\"api\":1000,\"auth\":100}}"}}'

# App detects file change and reloads within ~60s
kubectl delete configmap dynamic-config
```

### 16.3 Immutable Secrets and ConfigMaps (v1.21+)

```bash
# Mark as immutable to:
# 1. Prevent accidental changes
# 2. Improve performance (kubelet stops watching immutable objects)
# 3. Protect from unintended rollouts

cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: immutable-app-config
  labels:
    version: v1.2.0
data:
  APP_VERSION: "1.2.0"
  SCHEMA_VERSION: "42"
immutable: true    # Cannot be changed after creation
EOF

# Try to update (should fail)
kubectl patch configmap immutable-app-config \
  --type merge -p '{"data":{"APP_VERSION":"1.3.0"}}'
# Error: configmap "immutable-app-config" is immutable

# Must delete and recreate for updates
kubectl delete configmap immutable-app-config

cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: immutable-db-secret
type: Opaque
stringData:
  password: "SecurePass123!"
immutable: true    # Works for Secrets too
EOF

kubectl delete secret immutable-db-secret
```

---

## 17. External Secret Management Integration

### 17.1 The Case for External Secret Managers

Native Kubernetes Secrets have limitations:

```
Native K8s Secrets limitations:
  → Only base64 encoded (not encrypted by default)
  → No built-in rotation
  → No approval workflows
  → No audit trail (beyond basic K8s audit logs)
  → No integration with enterprise IAM
  → No cross-cluster secret sharing
  → No secret versioning/history

External secret managers solve all of this:
  → HashiCorp Vault: PKI, dynamic secrets, fine-grained policies
  → AWS Secrets Manager: IAM integration, auto-rotation
  → Azure Key Vault: Azure AD integration
  → GCP Secret Manager: GCP IAM, versioning
```

### 17.2 External Secrets Operator (ESO)

```bash
# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets \
  external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace

# Configure AWS Secrets Manager as the backend
cat << 'EOF' | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-store
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        secretRef:
          accessKeyIDSecretRef:
            name: aws-credentials
            key: access-key-id
          secretAccessKeySecretRef:
            name: aws-credentials
            key: secret-access-key
---
# Define which AWS secret to sync
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: production
spec:
  refreshInterval: 1h    # Re-sync every hour
  secretStoreRef:
    name: aws-secrets-store
    kind: SecretStore
  target:
    name: database-credentials    # K8s Secret name to create
    creationPolicy: Owner
  data:
    - secretKey: password         # Key in K8s Secret
      remoteRef:
        key: prod/database        # AWS Secrets Manager path
        property: password        # JSON property within the secret
    - secretKey: username
      remoteRef:
        key: prod/database
        property: username
EOF
```

### 17.3 HashiCorp Vault Agent Injector

```bash
# With Vault Agent Injector, secrets are injected via annotations
# No changes to the application code or deployment spec needed

cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vault-injected-app
spec:
  template:
    metadata:
      annotations:
        # Enable Vault injection
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "my-app-role"
        # Inject this secret as /vault/secrets/database.txt
        vault.hashicorp.com/agent-inject-secret-database.txt: "secret/data/database"
        # Template for the secret file
        vault.hashicorp.com/agent-inject-template-database.txt: |
          {{- with secret "secret/data/database" -}}
          DB_PASSWORD={{ .Data.data.password }}
          DB_USERNAME={{ .Data.data.username }}
          {{- end -}}
    spec:
      serviceAccountName: my-app-vault-sa
      containers:
        - name: app
          image: my-app:v1.0
          # Application reads /vault/secrets/database.txt
EOF
```

---

## 18. Performance Tuning for Secret/ConfigMap Usage

### 18.1 API Server Cache Configuration

```yaml
# Increase watch cache sizes for high-throughput clusters
# In kube-apiserver manifest:
--watch-cache-sizes=secret#2000,configmap#5000
# secret#2000: Cache 2000 most recent Secret versions
# configmap#5000: Cache 5000 most recent ConfigMap versions

# For very large clusters with many rolling deployments:
--watch-cache-sizes=secret#5000,configmap#10000
```

### 18.2 kubelet Sync Performance

```yaml
# KubeletConfiguration for Secret/ConfigMap sync tuning
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

# How often kubelet syncs with API server
syncFrequency: 1m0s

# Reduces redundant API calls for immutable objects
# Mark frequently-accessed but unchanging ConfigMaps as immutable: true
# kubelet stops watching them after initial load
```

### 18.3 Avoiding Common Performance Anti-Patterns

```bash
# ANTI-PATTERN 1: Single large ConfigMap with all config
# Problem: Every pod restart re-reads the entire large object
# Solution: Split into smaller, purpose-specific ConfigMaps

# ANTI-PATTERN 2: Secrets with binary data > 512KB
# Problem: Large objects slow API server watch cache
# Solution: Use external storage (S3, persistent volumes) for large blobs

# ANTI-PATTERN 3: Frequent Secret updates
# Problem: Many watch events generated, all kubelets notified
# Solution: Use immutable Secrets; create new Secret on rotation

# ANTI-PATTERN 4: All Pods watching all Secrets
# Problem: Namespace-wide watch for Secrets is expensive
# Solution: RBAC restricts which pods watch which secrets

# Check current Secret count (indicator of cleanup needed)
kubectl get secrets --all-namespaces | wc -l
# Large numbers (>1000) may indicate leftover job/deployment secrets
```

### 18.4 Immutable Objects for Performance

```bash
# Immutable ConfigMaps/Secrets: kubelet stops watching
# → Significant performance improvement for large clusters
# → Especially for ConfigMaps that only change on deployment

# Measure the impact:
# Without immutable: kubelet watches all ConfigMaps = N watch connections per namespace
# With immutable: kubelet watches 0 connections for immutable objects

# Create immutable ConfigMap for production:
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: production-config-v1-2-0
  labels:
    version: "1.2.0"
    immutable: "true"
data:
  config.yaml: |
    version: 1.2.0
    database:
      pool_size: 20
immutable: true
EOF
```

---

## 19. Security Hardening Practices

### 19.1 RBAC for Secrets — Minimal Access Model

```yaml
# PRINCIPLE: No workload should read Secrets it doesn't own

# BAD: Default RBAC allows all pods to list secrets
# kubectl auth can-i list secrets --as=system:serviceaccount:default:default
# → yes (if no restrictive RBAC in place)

# GOOD: Create specific roles for specific secrets

# Allow an app to read ONLY its own specific secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: payment-service-secret-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]
    resourceNames:
      - "payment-db-credentials"   # SPECIFIC secrets only
      - "payment-api-keys"
      - "payment-tls-cert"
    # NO list, NO watch, NO create, NO delete
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payment-service-secret-binding
  namespace: production
subjects:
  - kind: ServiceAccount
    name: payment-service-sa
    namespace: production
roleRef:
  kind: Role
  name: payment-service-secret-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
# AUDIT: Find overly-broad Secret access
kubectl get clusterrolebindings -o json | \
  jq '.items[] | 
      select(.roleRef.name == "cluster-admin" or 
             .roleRef.name == "admin" or 
             .roleRef.name == "edit") |
      {name: .metadata.name, subjects: .subjects}'

# Check if default SA has secret access (should not in production)
kubectl auth can-i list secrets \
  --as=system:serviceaccount:production:default \
  -n production
# Expected: no

# Restrict default ServiceAccount
kubectl patch serviceaccount default \
  -n production \
  -p '{"automountServiceAccountToken": false}'
```

### 19.2 TLS Configuration for API Server

```bash
# API server uses TLS for all communication
# Verify TLS configuration
kubectl get pod kube-apiserver-$(hostname) -n kube-system \
  -o jsonpath='{.spec.containers[0].command}' | \
  tr ' ' '\n' | grep 'tls\|cert\|min-version'

# Check minimum TLS version (should be TLS 1.2 minimum)
# --tls-min-version=VersionTLS12

# For etcd at-rest encryption (see Section 15)
# Ensure AES-GCM or AES-CBC encryption is configured
```

### 19.3 ServiceAccount Security

```yaml
# Create dedicated ServiceAccounts per application
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: production
  annotations:
    kubernetes.io/description: "ServiceAccount for my-app Deployment"
automountServiceAccountToken: false  # Disable auto-mount; use explicit projection

---
# Use projected service account tokens (short-lived, audience-specific)
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: my-app-sa
  volumes:
    - name: token
      projected:
        sources:
          - serviceAccountToken:
              audience: my-app-service    # Scoped to specific audience
              expirationSeconds: 3600     # 1 hour expiry (auto-rotated)
              path: token
  containers:
    - name: app
      volumeMounts:
        - name: token
          mountPath: /run/secrets/kubernetes.io/serviceaccount
          readOnly: true
```

### 19.4 Audit Policy for Secrets

```yaml
# Comprehensive audit policy targeting Secrets
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all Secret access at RequestResponse level (full body)
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
    verbs: ["get", "list", "watch"]
    # Log which user accessed which secret when
  
  # Log Secret creation/deletion
  - level: RequestResponse
    resources:
      - group: ""
        resources: ["secrets"]
    verbs: ["create", "update", "patch", "delete"]
  
  # Minimal logging for ConfigMaps
  - level: Metadata
    resources:
      - group: ""
        resources: ["configmaps"]
  
  # Don't log health checks (noise)
  - level: None
    nonResourceURLs: ["/healthz", "/readyz", "/livez"]
```

### 19.5 Pod Security for Secret Access

```yaml
# Enforce readOnlyRootFilesystem to prevent secret data from leaking to disk
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: app
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true     # Prevents writing secrets to disk
        capabilities:
          drop:
            - ALL
      # writable temp dirs if needed
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: secrets
          mountPath: /run/secrets
          readOnly: true  # Always mount secrets read-only
  volumes:
    - name: tmp
      emptyDir: {}
    - name: secrets
      secret:
        secretName: my-app-secret
```

---

## 20. Monitoring and Observability

### 20.1 Key Metrics for Secrets and ConfigMaps

| Metric | Description | Alert Condition |
|---|---|---|
| `apiserver_request_total{resource="secrets",verb="get"}` | Secret read rate | Unusually high rate may indicate scraping |
| `apiserver_request_total{resource="configmaps"}` | ConfigMap operation rate | High rate during rolling updates |
| `etcd_object_counts{resource="secrets"}` | Total secrets in etcd | Unbounded growth → cleanup needed |
| `etcd_object_counts{resource="configmaps"}` | Total ConfigMaps | Monitor for leaks |
| `apiserver_storage_size_bytes` | etcd storage usage | > 80% of quota |
| `workqueue_depth{name="serviceaccount"}` | SA token queue depth | High backlog → controller issue |
| `apiserver_audit_requests_rejected_total` | Audit log failures | > 0 |

### 20.2 Audit Log Analysis

```bash
# Find all Secret access events
tail -1000 /var/log/kubernetes/audit.log | \
  jq 'select(.objectRef.resource == "secrets") | 
      {user: .user.username, 
       verb: .verb, 
       secret: .objectRef.name,
       ns: .objectRef.namespace,
       time: .requestReceivedTimestamp}'

# Find unexpected Secret reads
tail -1000 /var/log/kubernetes/audit.log | \
  jq 'select(.objectRef.resource == "secrets" and .verb == "get") |
      {user: .user.username, 
       secret: .objectRef.name}' | \
  sort | uniq -c | sort -rn | head -20

# Track Secret modifications
tail -1000 /var/log/kubernetes/audit.log | \
  jq 'select(.objectRef.resource == "secrets" and
             (.verb == "create" or .verb == "delete" or .verb == "patch"))'
```

### 20.3 Prometheus Alerting Rules

```yaml
groups:
  - name: kubernetes-secrets-configmaps
    rules:
      # Alert on high Secret access rate (possible data exfiltration)
      - alert: KubeSecretHighAccessRate
        expr: |
          sum by (user) (
            rate(apiserver_request_total{resource="secrets",verb="get"}[5m])
          ) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Unusually high Kubernetes Secret access rate"

      # Alert on etcd storage growth
      - alert: KubeEtcdStorageGrowing
        expr: |
          etcd_mvcc_db_total_size_in_bytes > 4 * 1024 * 1024 * 1024  # 4 GB
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "etcd database size exceeds 4 GB"

      # Alert on many orphaned Secrets (leaked from Jobs, etc.)
      - alert: KubeManyOrphanedSecrets
        expr: |
          count(kube_secret_info{type="Opaque"}) > 1000
        for: 30m
        labels:
          severity: info
        annotations:
          summary: "More than 1000 Opaque secrets - review for cleanup"

      # Alert on failed Secret access (possible misconfiguration)
      - alert: KubeSecretAccessForbidden
        expr: |
          sum(rate(apiserver_request_total{
            resource="secrets",
            code="403"
          }[5m])) > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High rate of forbidden Secret access - RBAC misconfiguration?"
```

---

## 21. Troubleshooting Secrets and ConfigMap Issues

### 21.1 Pod Not Starting Due to Missing Secret/ConfigMap

```bash
# ── DIAGNOSIS ─────────────────────────────────────────────────────────────

# Step 1: Check pod events
kubectl describe pod <pod-name> -n <namespace>
# Look for:
# Warning  Failed     5s    kubelet  Error: secret "db-credentials" not found
# Warning  Failed     5s    kubelet  configmap "app-config" not found

# Step 2: Verify the secret/configmap exists
kubectl get secret db-credentials -n <namespace>
kubectl get configmap app-config -n <namespace>

# Step 3: Check namespace (common mistake — wrong namespace)
kubectl get secrets -n <wrong-namespace>  # Exists here but not in pod's namespace
kubectl get secrets -n <correct-namespace>

# Step 4: Check the exact key names match
kubectl describe secret db-credentials
# Look at the "Data" section for key names
kubectl get pod <n> -o yaml | grep secretKeyRef -A 3
# Compare key names

# Step 5: Fix
# If secret doesn't exist: create it
kubectl create secret generic db-credentials \
  --from-literal=username=dbadmin \
  --from-literal=password=secretpass \
  -n <namespace>

# If key name mismatch: edit the pod spec or rename the key
kubectl patch secret db-credentials \
  --type merge \
  -p '{"stringData":{"DB_USER":"dbadmin"}}'
```

### 21.2 Deployment Stuck After ConfigMap Update

```bash
# ConfigMap update but pods not restarting

# Check if the deployment is using immutable ConfigMap
kubectl get configmap <n> -o yaml | grep immutable
# If immutable: true → cannot update; must delete and recreate

# Force rolling restart to pick up new env-var-based config
kubectl rollout restart deployment/<n> -n <namespace>
kubectl rollout status deployment/<n>

# For volume-mounted config: wait for kubelet sync (up to 2 minutes)
# Check if files are updated in the pod
kubectl exec deployment/<n> -- cat /etc/config/<key-name>
# Should show new value

# Check kubelet sync status
kubectl describe node <node-name> | grep "Kubelet Version"
# Check kubelet logs if sync seems delayed:
# journalctl -u kubelet -n 100 | grep "configmap\|syncFrequency"
```

### 21.3 Pods Not Creating — Deployment Stuck

```bash
# General deployment debugging
kubectl rollout status deployment/<n> -n <namespace>
kubectl describe deployment <n> -n <namespace>

# Check ReplicaSet events
kubectl get replicasets -l app=<n>
kubectl describe replicaset <rs-name>

# Check pod events
kubectl get pods -l app=<n>
kubectl describe pod <failing-pod-name>

# Common causes related to Secrets/ConfigMaps:
# 1. Secret/ConfigMap doesn't exist → pod fails to start
# 2. Wrong key name in secretKeyRef → pod fails
# 3. RBAC: ServiceAccount can't read the Secret → mount fails
# 4. Secret too large for tmpfs → rare but possible

# Check RBAC issue
kubectl auth can-i get secret db-credentials \
  --as=system:serviceaccount:<namespace>:<sa-name> \
  -n <namespace>
```

### 21.4 Node NotReady and Secret Impact

```bash
# When a node goes NotReady:
# - Pods on that node continue running
# - Secret/ConfigMap volumes continue serving from tmpfs (in-memory)
# - No new pods scheduled to this node

# If node is drained (pod evicted and rescheduled):
# - New pod on different node mounts Secrets/ConfigMaps fresh
# - Should be fine as long as Secrets/ConfigMaps exist

# Check if replacement pods have mounted secrets correctly
kubectl get pods -l app=<n> -o wide
kubectl exec <new-pod-name> -- ls /run/secrets/

# Verify Node status
kubectl get nodes
kubectl describe node <notready-node> | grep "Conditions:" -A 10
```

### 21.5 Debugging Secret Values

```bash
# Decode and inspect a Secret value
# Method 1: jsonpath
kubectl get secret db-credentials \
  -o jsonpath='{.data.password}' | base64 -d

# Method 2: jq
kubectl get secret db-credentials -o json | \
  jq '.data | map_values(@base64d)'

# Method 3: Print all key-value pairs
for key in $(kubectl get secret db-credentials -o json | \
  jq -r '.data | keys[]'); do
  echo -n "$key: "
  kubectl get secret db-credentials \
    -o jsonpath="{.data.$key}" | base64 -d
  echo ""
done

# Method 4: Check from inside the pod
kubectl exec <pod-name> -- env | grep -i password
kubectl exec <pod-name> -- cat /run/secrets/password
```

---

## 22. Disaster Recovery Concepts

### 22.1 Stateless Controllers and Secret Management

```
The kube-controller-manager is COMPLETELY STATELESS.
All Secret and ConfigMap data lives in etcd.

If kube-controller-manager crashes:
  → Existing Secrets/ConfigMaps: FULLY ACCESSIBLE
    - They're in etcd, served by API server
    - Running pods continue using them
  
  → New ServiceAccount token creation: PAUSED
    - ~30 seconds until new leader elected
    - New tokens not issued during this window
  
  → After recovery:
    - Controller re-lists all ServiceAccounts
    - Creates missing tokens
    - Fully operational within seconds of leader election
```

### 22.2 etcd Backup — The Source of Truth for Secrets

```bash
# etcd backup includes ALL Secrets and ConfigMaps
# This is the only backup that matters for Kubernetes data

# Backup etcd
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify the backup
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl snapshot status /tmp/etcd-backup.db \
  --write-out=table

# Copy backup to safe location
kubectl cp kube-system/etcd-$(hostname):/tmp/etcd-backup.db \
  ~/etcd-backup-$(date +%Y%m%d).db

# IMPORTANT: Backup the encryption key too!
# /etc/kubernetes/encryption-config.yaml must be backed up separately
# WITHOUT this file, encrypted secrets CANNOT be decrypted from etcd backup
```

### 22.3 GitOps as DR for Secrets

```bash
# BEST PRACTICE: Store Secrets in a git-friendly way using:
# 1. Sealed Secrets (encrypted CRDs safe for Git)
# 2. External Secrets Operator (source of truth is external)
# 3. Vault with Kubernetes auth

# Sealed Secrets approach:
# Install Sealed Secrets controller
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml

# Seal a secret (creates encrypted SealedSecret CRD)
kubectl create secret generic my-secret \
  --from-literal=password=MySecretPass \
  --dry-run=client -o yaml | \
  kubeseal \
    --controller-name=sealed-secrets-controller \
    --controller-namespace=kube-system \
    --format yaml > my-sealed-secret.yaml

# This file is SAFE to commit to Git
# Only the cluster with the matching private key can decrypt it

# On a new cluster: apply the sealed secret
kubectl apply -f my-sealed-secret.yaml
# Sealed Secrets controller decrypts and creates a regular Secret
```

### 22.4 Secret Rotation Strategy

```bash
# Production secret rotation procedure:
# 1. Create new Secret version
kubectl create secret generic db-credentials-v2 \
  --from-literal=password='NewSecurePass456!'

# 2. Update Deployment to use new Secret
kubectl patch deployment my-app \
  -p '{"spec":{"template":{"spec":{"volumes":[{"name":"db-creds","secret":{"secretName":"db-credentials-v2"}}]}}}}'

# 3. Monitor rolling update
kubectl rollout status deployment/my-app

# 4. Verify all pods are using new secret
kubectl get pods -l app=my-app -o wide
kubectl exec deployment/my-app -- \
  cat /run/secrets/password
# Should show: NewSecurePass456!

# 5. Delete old secret (after confirming all pods use new one)
kubectl delete secret db-credentials-v1
```

---

## 23. Comparison — Secrets vs ConfigMap vs Environment Variables vs Volume Files

### 23.1 Full Feature Comparison

| Feature | ConfigMap | Secret | Plain Env Var | Volume File (Secret) |
|---|---|---|---|---|
| **Sensitive data** | ❌ Not suitable | ✅ | ❌ Visible everywhere | ✅ Best choice |
| **Non-sensitive config** | ✅ Best choice | ❌ Overkill | ✅ Simple | ✅ Works |
| **RBAC protection** | Limited | ✅ Restricted | ❌ None | ✅ (via Secret) |
| **Encryption at rest** | ❌ Plaintext | ✅ Configurable | ❌ | ✅ (via Secret) |
| **Audit logging** | Basic | ✅ Detailed | ❌ | ✅ (via Secret) |
| **Dynamic updates** | ✅ (volumes) | ✅ (volumes) | ❌ Restart needed | ✅ Auto-update |
| **Disk persistence on node** | RAM cache | ❌ tmpfs only | In /proc | ❌ tmpfs |
| **Visible in process list** | If env var | If env var | ✅ Visible in ps | ❌ Not visible |
| **Max size** | 1 MiB | 1 MiB | OS limit (~32KB) | 1 MiB |
| **Git storage** | ✅ Safe | ❌ Never plain | ❌ Never plain | ❌ Never plain |

---

## 24. Comparison — kube-apiserver vs kube-scheduler vs kube-controller-manager (in context)

| Dimension | kube-apiserver | kube-scheduler | kube-controller-manager |
|---|---|---|---|
| **Secret/ConfigMap role** | Stores, validates, serves, enforces RBAC | Not involved in S/CM | ServiceAccount token controller creates Secrets |
| **etcd access** | YES — directly reads/writes | NO | NO — via API server |
| **ConfigMap watch** | Maintains watch cache | N/A | Watches for ServiceAccount changes |
| **Security enforcement** | AuthN/AuthZ/Audit for every request | N/A | Creates token Secrets with proper RBAC |
| **HA model** | Active-Active | Active-Passive (leader) | Active-Passive (leader) |
| **Failure impact on S/CM** | Catastrophic — no new reads/writes | No impact | Token creation paused only |
| **Audit logging** | Every Secret read/write logged | N/A | N/A |
| **Encryption** | Applies encryption via EncryptionConfiguration | N/A | N/A |

---

## 25. ASCII Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║           KUBERNETES SECRETS AND CONFIGMAPS — COMPLETE ARCHITECTURE             ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║                                                                                  ║
║  DEVELOPER / CI-CD                                                               ║
║  kubectl create secret generic db-creds --from-literal=pass=secret              ║
║  kubectl apply -f configmap.yaml                                                 ║
║           │                                                                      ║
║           │ HTTPS :6443                                                          ║
║           ▼                                                                      ║
║  ┌──────────────────────────────────────────────────────────────────────────┐    ║
║  │                    CONTROL PLANE                                         │    ║
║  │                                                                          │    ║
║  │  ┌───────────────────────────────────────────────────────────────────┐  │    ║
║  │  │  kube-apiserver :6443                                             │  │    ║
║  │  │                                                                   │  │    ║
║  │  │  1. AuthN: Who is the caller?                                    │  │    ║
║  │  │  2. AuthZ: RBAC check: can they read/write secrets?             │  │    ║
║  │  │  3. Admission: Valid Secret/ConfigMap object?                   │  │    ║
║  │  │  4. Encryption: AES-GCM encrypt before writing to etcd         │  │    ║
║  │  │  5. Audit: Log every Secret access with user/time/verb         │  │    ║
║  │  │  6. Watch cache: Fan-out updates to kubelet watchers           │  │    ║
║  │  └──────────────────────────────┬────────────────────────────────┘  │    ║
║  │                                  │                                    │    ║
║  │         ┌────────────────────────▼──────────────────────────────┐   │    ║
║  │         │   etcd :2379                                          │   │    ║
║  │         │   /registry/secrets/<ns>/<name>  → Encrypted!        │   │    ║
║  │         │   /registry/configmaps/<ns>/<n> → Plaintext          │   │    ║
║  │         │   (Only API server accesses etcd directly)           │   │    ║
║  │         └────────────────────────────────────────────────────┘   │    ║
║  │                                                                    │    ║
║  │  ┌────────────────────────────────────────────────────────────┐   │    ║
║  │  │  kube-controller-manager :10257                            │   │    ║
║  │  │  ServiceAccount Controller:                                │   │    ║
║  │  │    WATCH ServiceAccounts → CREATE token Secrets            │   │    ║
║  │  │    WATCH ServiceAccounts → DELETE token Secrets on SA del  │   │    ║
║  │  │  (Leader elected - only one instance active)               │   │    ║
║  │  └────────────────────────────────────────────────────────────┘   │    ║
║  └──────────────────────────────────────────────────────────────────────┘    ║
║                              │ API server Watch stream                          ║
║              ┌───────────────┴────────────────────────┐                        ║
║              │                                        │                        ║
║  ┌───────────▼────────────────┐          ┌────────────▼────────────────────┐   ║
║  │  WORKER NODE 1             │          │  WORKER NODE 2                  │   ║
║  │                            │          │                                  │   ║
║  │  kubelet :10250            │          │  kubelet :10250                  │   ║
║  │  ┌─────────────────────┐   │          │  ┌─────────────────────────────┐ │   ║
║  │  │ Informer: watches   │   │          │  │ Informer: watches Secrets   │ │   ║
║  │  │ Secrets/ConfigMaps  │   │          │  │ ConfigMaps for local Pods   │ │   ║
║  │  │ for local Pods only │   │          │  └─────────────────────────────┘ │   ║
║  │  └────────┬────────────┘   │          │                                  │   ║
║  │           │                │          │  ┌───────────────────────────┐   │   ║
║  │  ┌────────▼────────────┐   │          │  │  POD (nginx)              │   │   ║
║  │  │  POD (my-api)       │   │          │  │                           │   │   ║
║  │  │                     │   │          │  │  ConfigMap as env vars:   │   │   ║
║  │  │  ConfigMap as files:│   │          │  │  LOG_LEVEL=info           │   │   ║
║  │  │  /etc/config/       │   │          │  │  MAX_CONN=100             │   │   ║
║  │  │  ├── nginx.conf     │   │          │  │                           │   │   ║
║  │  │  └── app.yml        │   │          │  │  Secret as env var:       │   │   ║
║  │  │                     │   │          │  │  DB_PASS=MyP@ss           │   │   ║
║  │  │  Secret as tmpfs:   │   │          │  │  (visible in /proc/env)   │   │   ║
║  │  │  /run/secrets/      │   │          │  └───────────────────────────┘   │   ║
║  │  │  ├── db_password    │   │          └──────────────────────────────────┘   ║
║  │  │  └── jwt_secret     │   │                                                 ║
║  │  │  (tmpfs - not disk) │   │  ┌─────────────────────────────────────────┐   ║
║  │  └─────────────────────┘   │  │  EXTERNAL SECRET MANAGERS (Optional)    │   ║
║  └────────────────────────────┘  │  HashiCorp Vault / AWS Secrets Manager  │   ║
║                                   │  Azure Key Vault / GCP Secret Manager  │   ║
║  SECURITY LAYERS:                 │  External Secrets Operator syncs →     │   ║
║  1. etcd encrypted at rest        │  K8s Secrets from external store        │   ║
║  2. tmpfs: secrets not on disk    └─────────────────────────────────────────┘   ║
║  3. RBAC: only authed SA can read                                                ║
║  4. Audit: every access logged                                                   ║
║  5. Immutable: protect critical config                                           ║
╚══════════════════════════════════════════════════════════════════════════════════╝
```

---

## 26. Real-World Production Use Cases

### 26.1 Database Credentials Management

```yaml
# Pattern: Separate Secrets per database
# Each microservice gets only what it needs

# Payment service Secret (payment-service-sa can read)
apiVersion: v1
kind: Secret
metadata:
  name: payment-db-credentials
  namespace: payment-service
type: Opaque
stringData:
  host: "postgres.payment.svc.cluster.local"
  port: "5432"
  database: "payments"
  username: "payment_app"
  password: "payment-secure-pass-XYZ"
  ssl-mode: "require"

---
# Analytics service Secret (analytics-sa can read)
apiVersion: v1
kind: Secret
metadata:
  name: analytics-db-credentials
  namespace: analytics
type: Opaque
stringData:
  host: "postgres-analytics.analytics.svc.cluster.local"
  username: "analytics_readonly"
  password: "analytics-read-only-pass"
```

### 26.2 Multi-Environment Configuration

```yaml
# Same app, 3 environments, different ConfigMaps
# Namespace dev:
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: development
data:
  LOG_LEVEL: "debug"
  REPLICA_COUNT: "1"
  CACHE_ENABLED: "false"
  DEBUG_PROFILING: "enabled"
  EXTERNAL_API: "https://api-sandbox.payment.com"

---
# Namespace staging:
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: staging
data:
  LOG_LEVEL: "info"
  REPLICA_COUNT: "2"
  CACHE_ENABLED: "true"
  DEBUG_PROFILING: "disabled"
  EXTERNAL_API: "https://api-staging.payment.com"

---
# Namespace production:
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
data:
  LOG_LEVEL: "warn"
  REPLICA_COUNT: "5"
  CACHE_ENABLED: "true"
  DEBUG_PROFILING: "disabled"
  EXTERNAL_API: "https://api.payment.com"
```

### 26.3 TLS Certificate Management with cert-manager

```bash
# cert-manager automatically manages TLS Secrets

# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Create a Certificate (cert-manager creates the Secret automatically)
cat << 'EOF' | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: my-app-tls
  namespace: production
spec:
  secretName: my-app-tls-secret   # Secret created by cert-manager
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - api.myapp.example.com
  duration: 2160h      # 90 days
  renewBefore: 360h    # Renew 15 days before expiry
EOF

# cert-manager creates and manages:
kubectl get secret my-app-tls-secret -n production
# type: kubernetes.io/tls
# data:
#   tls.crt: <auto-rotated certificate>
#   tls.key: <auto-rotated private key>
```

### 26.4 Feature Flag Management

```bash
# ConfigMap as a live feature flag store
# Application watches for file changes and updates behavior dynamically

kubectl create configmap feature-flags \
  --from-literal=enable_new_checkout=false \
  --from-literal=enable_beta_search=false \
  --from-literal=ab_test_percentage=10

# Deploy app that reads feature flags from volume
# ... (deployment uses volume-mounted configmap)

# Enable feature flag without restart
kubectl patch configmap feature-flags \
  --type merge \
  -p '{"data":{"enable_new_checkout":"true"}}'

# Application detects file change within ~60s
# New checkout flow activated without deployment!
```

---

## 27. Best Practices for Production Environments

### 27.1 Security

- **Enable encryption at rest** for Secrets using AES-GCM; never run production with Secrets in plaintext etcd
- **Use volume mounts, not env vars for sensitive data** — env vars are visible in `ps aux` and `/proc/<pid>/environ`
- **Apply restrictive RBAC** — ServiceAccounts should only have `get` on their specific Secrets (never `list` or `watch` across all secrets)
- **Disable automountServiceAccountToken** on Pods that don't need API access
- **Use immutable Secrets** for credentials that don't change — prevents accidental modification and improves kubelet performance
- **Integrate with external secret managers** (Vault, AWS Secrets Manager) for enterprise-grade secret lifecycle management

### 27.2 Configuration Management

- **Use ConfigMaps for non-sensitive config** — never store passwords, tokens, or keys in ConfigMaps
- **Version your ConfigMaps** using labels or name suffixes (`app-config-v1.2.0`) for rollback capability
- **Use immutable ConfigMaps** for versioned configs; create new versions on updates rather than editing
- **Separate ConfigMaps by purpose** — don't put all config in one massive ConfigMap
- **Test ConfigMap updates in staging** before production — a bad nginx.conf crashes all nginx pods

### 27.3 Operational

- **Back up etcd AND the encryption key** — losing either makes Secret recovery impossible
- **Document Secret ownership** — use annotations to track which team owns each Secret, rotation schedule, and purpose
- **Set up Secret rotation** — credentials should rotate regularly; automate with cert-manager (TLS) or external tools
- **Monitor Secret access via audit logs** — unexpected reads may indicate a breach or misconfiguration
- **Use Sealed Secrets or External Secrets Operator** for GitOps-friendly secret management

---

## 28. Common Mistakes and Pitfalls

### 28.1 Storing Secrets in ConfigMaps

**Mistake:** Using ConfigMap for passwords because "it's easier."
**Impact:** No RBAC restriction, no encryption consideration, visible to all pods in namespace.
**Fix:** Always use Secret for sensitive data, ConfigMap for configuration.

---

### 28.2 Forgetting base64 Encoding in YAML

**Mistake:**
```yaml
data:
  password: MyP@ssword!   # WRONG: Must be base64 encoded in data:
```
**Impact:** `kubectl apply` fails with validation error.
**Fix:** Use `stringData:` for plain text input, or pre-encode: `echo -n "MyP@ssword!" | base64`

---

### 28.3 Using Optional: false for Critical Secrets That Might Not Exist

**Mistake:** Deploying a new application where the secret is created after the deployment.
**Impact:** Pod fails to start with `secret "xxx" not found`; Deployment shows 0 available.
**Fix:** Create Secrets BEFORE deploying the application. Use `optional: true` only for genuinely optional secrets.

---

### 28.4 Expecting Env-Var Injected Config to Update Automatically

**Mistake:** Updating a ConfigMap and expecting running pods to pick up the change immediately.
**Impact:** Confusion when application continues with old config despite ConfigMap being updated.
**Fix:** Use volume mounts for dynamic config. For env vars, always restart pods after ConfigMap update: `kubectl rollout restart deployment/<n>`.

---

### 28.5 RBAC That Allows listing All Secrets

**Mistake:**
```yaml
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]  # list allows seeing all secret NAMES
```
**Impact:** Any pod with this role can discover ALL secret names in the namespace.
**Fix:** Use `resourceNames` to restrict to specific secrets; avoid `list` and `watch` on Secrets.

---

### 28.6 Committing Secret YAML to Git

**Mistake:** Putting a Secret manifest with `stringData` into a Git repository.
**Impact:** Credentials are visible to everyone with repo access; credentials in git history.
**Fix:** Use Sealed Secrets, External Secrets Operator, or GitOps with Vault.

---

### 28.7 Large ConfigMaps Causing Performance Issues

**Mistake:** Storing large binaries or multi-megabyte configurations in a ConfigMap.
**Impact:** API server watch cache pressure; slow kubelet syncs; approaching the 1MiB limit.
**Fix:** Store large files in PersistentVolumes or object storage (S3). ConfigMaps are for lightweight configuration.

---

## 29. Interview Questions — Beginner to Advanced

### Beginner Level

**Q1: What is the difference between a ConfigMap and a Secret in Kubernetes?**

**A:** Both are API objects that store configuration data as key-value pairs. The key differences are:
- **Purpose**: ConfigMap stores non-sensitive data (log levels, hostnames, config files). Secret stores sensitive data (passwords, API keys, TLS certs).
- **Encoding**: ConfigMap values are plain text. Secret values are base64-encoded.
- **Security**: Secrets have additional protections — restricted RBAC by default, ability to encrypt at rest in etcd, delivered via tmpfs (not disk).
- **Pod delivery**: Both can be injected as env vars or volume-mounted files.

**Important**: base64 is NOT encryption — it's just encoding. Secrets require encryption at rest configuration to be truly protected.

---

**Q2: What are two ways to inject a ConfigMap into a Pod?**

**A:**
1. **Environment variables**: `envFrom.configMapRef` (loads all keys) or `env.valueFrom.configMapKeyRef` (loads specific keys). Simple but env vars are fixed at pod start — they don't update if the ConfigMap changes.

2. **Volume mount**: Mount ConfigMap as a directory of files. Each key becomes a file; the value is the file content. Volume-mounted ConfigMaps **auto-update** in ~1-2 minutes when the ConfigMap is changed, without restarting the pod.

Volume mounts are generally preferred for configuration files (nginx.conf, app.properties) while env vars work for simple string values.

---

**Q3: Why is base64 not the same as encryption for Secrets?**

**A:** base64 is a reversible encoding scheme — anyone who sees the base64 string can trivially decode it with `echo "ZGJhZG1pbg==" | base64 -d` to get `dbadmin`. It has no key, no complexity, no security benefit.

Encryption requires a key to both encrypt and decrypt. Without the key, the encrypted data is computationally infeasible to reverse.

Kubernetes Secrets use base64 for storage format consistency, NOT for security. To actually protect Secret data, you must:
1. Configure encryption at rest (etcd AES-GCM encryption)
2. Restrict RBAC (only authorized service accounts can read Secrets)
3. Enable audit logging to track all Secret access

---

### Intermediate Level

**Q4: How does a pod receive ConfigMap updates when the ConfigMap is mounted as a volume?**

**A:** Kubernetes uses an atomic symlink mechanism:

When a ConfigMap is updated:
1. kubelet's informer detects the MODIFIED event via its watch connection to the API server
2. kubelet creates a new timestamped directory (e.g., `..2025_01_01_10_05_00.123456/`) inside the volume
3. New file contents are written to this new directory
4. The `..data` symlink is atomically updated to point to the new directory
5. Application code reads files through the stable symlinks (e.g., `config.yaml → ..data/config.yaml`)
6. The old directory is cleaned up

The atomic symlink swap prevents partial reads during the transition. The update typically takes 1-2 minutes (kubelet sync period).

**Note**: Env vars injected from ConfigMaps are NOT automatically updated. Only volume-mounted ConfigMaps update automatically.

---

**Q5: Explain the security implications of using environment variables vs volume mounts for Secrets.**

**A:**

**Environment Variables:**
- Visible in `/proc/<pid>/environ` — any process on the same node (with sufficient permissions) can read them
- Visible in `kubectl exec <pod> -- env` — any user with exec access sees all env vars
- Fixed at pod start; not automatically updated
- Can appear in crash dumps, core dumps, and bug reports
- Easier to accidentally log (e.g., `print(os.environ)` in debug code)

**Volume Mounts (preferred for secrets):**
- Delivered via tmpfs filesystem — data is in memory, not written to node disk
- Not visible in process environment list
- Can be read by application and immediately discarded from memory
- Auto-updated when Secret changes (within ~1-2 min)
- Can be granularly controlled with file permissions (chmod 0400)
- Files not visible in `kubectl exec -- env`

**Best practice**: Use volume mounts for all sensitive data (passwords, keys, certificates). Use env vars only for non-sensitive configuration values.

---

**Q6: What is the controller pattern for the ServiceAccount token controller, and what does it do?**

**A:** The ServiceAccount token controller runs inside `kube-controller-manager` and follows the Watch → Compare → Act → Loop pattern:

**WATCH**: Informer watches ServiceAccount objects for ADDED/DELETED events.

**COMPARE**: For each ServiceAccount, check if a corresponding `kubernetes.io/service-account-token` Secret exists.

**ACT**: 
- If SA created without a token Secret → create a Secret with: a JWT token signed by the cluster CA, the cluster CA certificate, and the namespace name
- If SA deleted → delete the associated token Secret
- In Kubernetes 1.24+: tokens are bound to specific Pods via the TokenRequest API (short-lived, not stored as Secrets)

**LOOP**: Continuously watches for SA changes.

This controller never talks directly to etcd — it makes API calls to create/delete Secrets, which the API server then stores in etcd.

---

### Advanced Level

**Q7: How does encryption at rest for Secrets work in Kubernetes, and what are the risks if the encryption key is lost?**

**A:** Encryption at rest is configured via an `EncryptionConfiguration` file passed to kube-apiserver with `--encryption-provider-config`.

**How it works:**
1. Developer creates a Secret via kubectl
2. API server receives the Secret and calls the encryption provider (e.g., AES-GCM)
3. Encryption provider encrypts the Secret data using the configured key
4. Encrypted bytes stored in etcd as: `k8s:enc:aesgcm:v1:key1:<encrypted-data>`
5. On read: API server fetches encrypted bytes, decrypts, returns plaintext to caller

**Key rotation:**
- New key added first in the providers list (used for encryption)
- Old key kept second (used for decryption of old data)
- Re-encrypt all secrets: `kubectl get secrets -A -o json | kubectl replace -f -`
- Remove old key after all secrets are re-encrypted

**Risk if key is lost:**
- All Secrets stored with that key become permanently unreadable
- etcd backup is useless without the matching encryption key
- **Critical**: You must back up the encryption config file separately from etcd
- Without the key: cluster is effectively bricked from a secrets perspective — all credentials (database passwords, API keys, TLS certs) are permanently inaccessible

**Production requirement**: Store the encryption key in a Hardware Security Module (HSM) or key management service (AWS KMS, Google Cloud KMS) for envelope encryption — the etcd key encrypts local keys, which the KMS manages.

---

**Q8: A developer updates a ConfigMap used by a 3-replica Deployment. Some pods show old config, some show new. Explain what's happening and how to ensure consistency.**

**A:** This is a **race condition** in the kubelet ConfigMap sync:

**What's happening:**
1. ConfigMap is updated in etcd
2. Each node's kubelet independently watches for ConfigMap changes
3. kubelet syncs occur on a schedule (default: ~1 minute)
4. Node 1's kubelet syncs first → Pod A sees new config
5. Node 2's kubelet syncs 30s later → Pod B sees old config
6. Node 3 syncs last → Pod C sees old config

**Why this happens**: Kubernetes doesn't guarantee all pods see ConfigMap updates simultaneously. Each kubelet syncs independently.

**Solutions:**

1. **For volume-mounted config**: Accept eventual consistency and design the application to tolerate a brief transition period. All pods will sync within ~2 minutes.

2. **For immediate consistency**: Trigger a rolling restart after ConfigMap update:
   ```bash
   kubectl rollout restart deployment/<n>
   ```
   This creates new pods that start fresh with the latest ConfigMap, and removes old pods in a controlled rolling fashion.

3. **For zero-downtime config changes with consistency**: Use immutable ConfigMaps with versioned names:
   - Update Deployment to reference `app-config-v2` instead of `app-config-v1`
   - Rolling update ensures all new pods use `v2`
   - `kubectl rollout undo` can revert to `v1`

4. **For truly atomic config changes**: Build the application to re-read config from files and check version consistency before serving traffic, using a readiness probe that validates config version.

---

## 30. Cheat Sheet — Commands, Flags & Manifests

### 30.1 Secret Commands

```bash
# ── CREATE ────────────────────────────────────────────────────────────────
kubectl create secret generic <n> --from-literal=key=value
kubectl create secret generic <n> --from-literal=k1=v1 --from-literal=k2=v2
kubectl create secret generic <n> --from-file=/path/to/file
kubectl create secret generic <n> --from-file=key=/path/to/file
kubectl create secret generic <n> --from-file=/path/to/dir/
kubectl create secret tls <n> --cert=cert.crt --key=cert.key
kubectl create secret docker-registry <n> \
  --docker-server=X --docker-username=Y --docker-password=Z

# ── READ ──────────────────────────────────────────────────────────────────
kubectl get secrets
kubectl get secrets -n <namespace>
kubectl get secret <n> -o yaml
kubectl get secret <n> -o jsonpath='{.data.key}' | base64 -d
kubectl describe secret <n>

# ── UPDATE ────────────────────────────────────────────────────────────────
kubectl patch secret <n> --type merge -p '{"stringData":{"key":"value"}}'
kubectl edit secret <n>

# ── DELETE ────────────────────────────────────────────────────────────────
kubectl delete secret <n>
kubectl delete secret -l app=<label>
```

### 30.2 ConfigMap Commands

```bash
# ── CREATE ────────────────────────────────────────────────────────────────
kubectl create configmap <n> --from-literal=key=value
kubectl create configmap <n> --from-literal=k1=v1 --from-literal=k2=v2
kubectl create configmap <n> --from-file=/path/to/file
kubectl create configmap <n> --from-file=key=/path/to/file
kubectl create configmap <n> --from-file=/path/to/dir/
kubectl create configmap <n> --from-env-file=.env

# ── READ ──────────────────────────────────────────────────────────────────
kubectl get configmap <n>
kubectl get configmap <n> -o yaml
kubectl describe configmap <n>

# ── UPDATE ────────────────────────────────────────────────────────────────
kubectl patch configmap <n> --type merge -p '{"data":{"key":"value"}}'
kubectl edit configmap <n>

# ── DELETE ────────────────────────────────────────────────────────────────
kubectl delete configmap <n>
```

### 30.3 Quick YAML Templates

```yaml
# ── SECRET (stringData input) ────────────────────────────────────────────
apiVersion: v1
kind: Secret
metadata:
  name: my-secret
  namespace: default
type: Opaque
stringData:                # Plain text input (auto base64)
  username: admin
  password: "MySecurePass!"
immutable: false           # Set true to prevent changes

# ── CONFIGMAP ────────────────────────────────────────────────────────────
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-config
  namespace: default
data:
  LOG_LEVEL: "info"
  config.yaml: |
    key: value
    nested:
      key: value
immutable: false

# ── SECRET AS ENV VARS IN POD ────────────────────────────────────────────
env:
  - name: DB_PASS
    valueFrom:
      secretKeyRef:
        name: my-secret
        key: password
        optional: false

# ── SECRET AS VOLUME ────────────────────────────────────────────────────
volumes:
  - name: secret-vol
    secret:
      secretName: my-secret
      defaultMode: 0400
volumeMounts:
  - name: secret-vol
    mountPath: /run/secrets
    readOnly: true

# ── CONFIGMAP AS ENV VARS ────────────────────────────────────────────────
envFrom:
  - configMapRef:
      name: my-config

# ── CONFIGMAP AS VOLUME ──────────────────────────────────────────────────
volumes:
  - name: config-vol
    configMap:
      name: my-config
      defaultMode: 0644
volumeMounts:
  - name: config-vol
    mountPath: /etc/config
    readOnly: true
```

### 30.4 Security Commands

```bash
# Check if Secret is encrypted at rest
kubectl exec -n kube-system etcd-$(hostname) -- \
  etcdctl get /registry/secrets/default/<secret-name> \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key | \
  strings | grep 'k8s:enc'

# Verify RBAC for Secret access
kubectl auth can-i get secret <n> -n <ns> --as=system:serviceaccount:<ns>:<sa>

# Audit recent Secret access
tail -100 /var/log/kubernetes/audit.log | \
  jq 'select(.objectRef.resource=="secrets")'

# Find Secrets without RBAC restrictions
kubectl get secrets --all-namespaces -o wide
```

### 30.5 Rolling Update After ConfigMap Change

```bash
# Force pod restart to pick up new env var values
kubectl rollout restart deployment/<n>
kubectl rollout status deployment/<n>

# OR: Use annotation to trigger update
kubectl patch deployment <n> \
  -p "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"restart-timestamp\":\"$(date +%s)\"}}}}}"

# Verify new config in running pod
kubectl exec deployment/<n> -- env | grep <CONFIG_KEY>
kubectl exec deployment/<n> -- cat /etc/config/<config-file>
```

---

## 31. Key Takeaways & Summary

### The Mental Model for Secrets and ConfigMaps

```
SEPARATION OF CONCERNS:
─────────────────────────────────────────────────────────────────────
Container Image       → Application code (immutable, portable)
ConfigMap             → Environment-specific, non-sensitive config
Secret                → Sensitive credentials (encrypted, RBAC-protected)

The same container image runs in dev, staging, and production.
Only the ConfigMap and Secret values differ per environment.
─────────────────────────────────────────────────────────────────────

DELIVERY METHODS:
─────────────────────────────────────────────────────────────────────
Environment Variables:
  → Simple, convenient
  → Fixed at pod start (no auto-update)
  → Visible in /proc/environ (security risk for sensitive data)
  → Best for: non-sensitive config values

Volume Mounts:
  → Files in tmpfs (not on disk for Secrets)
  → Auto-update for changes (volume-mounted only)
  → Not visible in process environment
  → Best for: sensitive data, large config files, dynamic config
─────────────────────────────────────────────────────────────────────

SECURITY LAYERS:
─────────────────────────────────────────────────────────────────────
1. RBAC: Only authorized ServiceAccounts can read specific Secrets
2. Encryption at rest: AES-GCM in etcd (requires explicit configuration)
3. tmpfs delivery: Secret files in memory, not on node disk
4. Audit logging: Every Secret access tracked with user/time
5. Immutable objects: Prevent accidental modification
6. External secrets: Vault/AWS for enterprise-grade management
─────────────────────────────────────────────────────────────────────
```

### The 10 Rules of Production Secret/ConfigMap Management

1. **Secrets store sensitive data; ConfigMaps store everything else.** Never mix them.
2. **base64 is encoding, not encryption.** Enable etcd encryption at rest.
3. **Volume mounts are safer than env vars for Secrets.** Use them for passwords and keys.
4. **RBAC must restrict Secret access.** Applications should only read their own Secrets.
5. **Controllers never access etcd directly.** All operations flow through the API server.
6. **Volume-mounted ConfigMaps auto-update; env vars don't.** Design accordingly.
7. **Back up etcd AND the encryption key.** Losing either is catastrophic.
8. **Never commit Secrets to Git.** Use Sealed Secrets, External Secrets, or Vault.
9. **Immutable Secrets and ConfigMaps improve performance and safety.** Use them for versioned config.
10. **Audit Secret access continuously.** Unexpected reads may indicate a breach.

---

> **This guide covers Kubernetes v1.29+ Secrets and ConfigMap management. Always consult the official documentation at https://kubernetes.io/docs/concepts/configuration/ for the most current information.**

---

*End of: Kubernetes Secrets and ConfigMaps — Complete Production-Grade Guide*
