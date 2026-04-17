## Table of Contents

1. [Introduction — Why Application Health Management Is Critical](#1-introduction--why-application-health-management-is-critical)
2. [Core Identity Table — Health Management Components](#2-core-identity-table--health-management-components)
3. [The Complete Probe Architecture — How kubelet Manages Health](#3-the-complete-probe-architecture--how-kubelet-manages-health)
4. [The Controller Pattern — Watch → Compare → Act → Loop](#4-the-controller-pattern--watch--compare--act--loop)
5. [Understanding Startup Probe — Protecting Slow-Starting Containers](#5-understanding-startup-probe--protecting-slow-starting-containers)
6. [Lab: Designing a Deployment with Startup Probe](#6-lab-designing-a-deployment-with-startup-probe)
7. [Understanding Readiness Probe — Traffic Gate for Containers](#7-understanding-readiness-probe--traffic-gate-for-containers)
8. [Lab: Configuring Deployment with Readiness Probe](#8-lab-configuring-deployment-with-readiness-probe)
9. [Understanding Liveness Probe — Self-Healing Restart Mechanism](#9-understanding-liveness-probe--self-healing-restart-mechanism)
10. [Lab: Configuring Deployment with Liveness Probe](#10-lab-configuring-deployment-with-liveness-probe)
11. [Probe Methods Deep Dive — HTTP, TCP, Exec, gRPC](#11-probe-methods-deep-dive--http-tcp-exec-grpc)
12. [Combining All Three Probes — The Full Production Pattern](#12-combining-all-three-probes--the-full-production-pattern)
13. [Attaching Lifecycle Handlers to Containers](#13-attaching-lifecycle-handlers-to-containers)
14. [Lab: Adding postStart and preStop Events](#14-lab-adding-poststart-and-prestop-events)
15. [Built-in Controllers and Health Management Interaction](#15-built-in-controllers-and-health-management-interaction)
16. [Internal Working Concepts — Informers, Work Queues & Reconciliation](#16-internal-working-concepts--informers-work-queues--reconciliation)
17. [API Server and etcd Interaction in Health Management](#17-api-server-and-etcd-interaction-in-health-management)
18. [Leader Election — HA for Health-Aware Controllers](#18-leader-election--ha-for-health-aware-controllers)
19. [Performance Tuning for Probes and Health Checks](#19-performance-tuning-for-probes-and-health-checks)
20. [Security Hardening for Health Endpoints](#20-security-hardening-for-health-endpoints)
21. [Monitoring and Observability for Application Health](#21-monitoring-and-observability-for-application-health)
22. [Troubleshooting Health Probe Issues](#22-troubleshooting-health-probe-issues)
23. [Disaster Recovery — Health Checks and Resilience](#23-disaster-recovery--health-checks-and-resilience)
24. [Comparison — kube-apiserver vs kube-scheduler vs kubelet in Health Context](#24-comparison--kube-apiserver-vs-kube-scheduler-vs-kubelet-in-health-context)
25. [ASCII Architecture Diagram](#25-ascii-architecture-diagram)
26. [Real-World Production Use Cases](#26-real-world-production-use-cases)
27. [Best Practices for Production Health Management](#27-best-practices-for-production-health-management)
28. [Common Mistakes and Pitfalls](#28-common-mistakes-and-pitfalls)
29. [Interview Questions — Beginner to Advanced](#29-interview-questions--beginner-to-advanced)
30. [Cheat Sheet — Commands, Configs & Patterns](#30-cheat-sheet--commands-configs--patterns)
31. [Key Takeaways & Summary](#31-key-takeaways--summary)

---

## 1. Introduction — Why Application Health Management Is Critical

In a Kubernetes cluster, applications are continuously deployed, updated, scaled, and rescheduled across nodes. The cluster itself is a dynamic environment where networking changes, nodes go down, and application processes can silently deadlock or run out of memory. Without a structured health management system, Kubernetes would:

- Route traffic to Pods that are still initializing and not ready to handle requests
- Route traffic to Pods that have deadlocked or are experiencing errors
- Continue running zombie processes that have lost database connections
- Terminate applications mid-deployment without waiting for in-flight requests to complete
- Fail to detect JVM or Python applications still loading large class files or datasets

**Kubernetes solves all of these through three probes and two lifecycle hooks:**

| Component | Function | Managed By |
|---|---|---|
| **Startup Probe** | Delays other probes until app is fully initialized | kubelet |
| **Readiness Probe** | Gates traffic: only routes to Ready pods | kubelet + EndpointSlice controller |
| **Liveness Probe** | Detects and kills deadlocked/unhealthy containers | kubelet |
| **postStart hook** | Executes logic immediately after container starts | kubelet |
| **preStop hook** | Executes cleanup logic before container receives SIGTERM | kubelet |

Together, these mechanisms form a **contract between Kubernetes and your application**. You define what "healthy" means for your application; Kubernetes enforces that contract continuously.

### The Production Cost of Missing Probes

| Scenario | Without Probes | With Proper Probes |
|---|---|---|
| Rolling update of slow-starting JVM app | Traffic errors for 30-60 seconds | Zero-downtime; old pods removed only after new ones are ready |
| Application deadlock (all goroutines blocked) | Pod runs forever in broken state; SLAs violated | Container restarted within `failureThreshold × periodSeconds` |
| Node OOM kills application process | kubelet restarts container; no traffic during restart | Readiness probe removes pod from LB immediately; liveness restarts container |
| Deployment rollout | New pods receive traffic before initialization | Startup probe delays readiness probe; no premature traffic |
| Pod termination | SIGTERM sent immediately; in-flight requests dropped | preStop hook drains connections; terminationGracePeriodSeconds ensured |

---

## 2. Core Identity Table — Health Management Components

| Component | Kind / Binary | Port | Role in Health Management |
|---|---|---|---|
| **kubelet** | `kubelet` | 10250 | Executes all probes; manages container lifecycle hooks; reports status to API server |
| **kube-apiserver** | `kube-apiserver` | 6443 | Stores Pod health status; exposes watch for EndpointSlice controller |
| **kube-controller-manager** | `kube-controller-manager` | 10257 | EndpointSlice controller reacts to Pod Ready condition changes |
| **kube-proxy** | `kube-proxy` | 10249 | Programs iptables/IPVS based on EndpointSlice updates |
| **Pod** | `pod` | App ports | Target of probes; receives lifecycle hooks |
| **Container** | CRI container | App ports | Responds to probe checks; executes hook commands |
| **StartupProbe** | Config in pod spec | App port or exec | Delays liveness/readiness until app initialized |
| **ReadinessProbe** | Config in pod spec | App port or exec | Determines inclusion in Service Endpoints |
| **LivenessProbe** | Config in pod spec | App port or exec | Triggers container restart when application is unhealthy |
| **postStart hook** | `lifecycle.postStart` | N/A | Runs immediately after container start |
| **preStop hook** | `lifecycle.preStop` | N/A | Runs before container receives SIGTERM |
| **EndpointSlice** | `endpointslice` | N/A | Tracks IPs of Ready pods for Service routing |
| **HPA** | `horizontalpodautoscaler` | N/A | Scales based on metrics; respects readiness for accurate calculation |

---

## 3. The Complete Probe Architecture — How kubelet Manages Health

### 3.1 The Probe Execution Timeline

```
Container lifecycle with all probes:
─────────────────────────────────────────────────────────────────────────────────

t=0:    Container STARTS
        postStart hook executes (async with container start)

t=0+:   STARTUP PROBE begins
        periodSeconds interval, up to failureThreshold failures
        Liveness + Readiness probes BLOCKED while startup probe active
        Pod.status.conditions[Ready] = False
        Pod NOT in Service Endpoints

t=?:    STARTUP PROBE SUCCEEDS
        Container considered "started"
        Liveness probe begins
        Readiness probe begins

t=?+:   READINESS PROBE first success
        Pod.status.conditions[Ready] = True
        Pod.status.containerStatuses[].ready = true
        EndpointSlice controller adds Pod IP to Service Endpoints
        kube-proxy updates iptables/IPVS rules
        TRAFFIC NOW FLOWS TO POD

t=∞:    LIVENESS PROBE runs continuously
        READINESS PROBE runs continuously

        If liveness fails failureThreshold times:
          → Container killed with exit code 137 (SIGKILL)
          → kubelet respects restartPolicy (Always → restart)
          → Pod NOT removed (same IP retained during restart)

        If readiness fails failureThreshold times:
          → Pod.status.conditions[Ready] = False
          → EndpointSlice controller removes Pod IP from Service
          → kube-proxy removes pod from load balancing
          → No traffic to pod (but pod keeps running)

t=termination: SIGTERM received (after preStop hook completes)
        preStop hook executes (synchronous, before SIGTERM)
        Pod removed from Service Endpoints immediately
        Container receives SIGTERM after preStop hook
        terminationGracePeriodSeconds countdown starts
        Container receives SIGKILL after grace period if still running
```

### 3.2 Probe Execution Model

kubelet runs probe checks **locally** on the node — it does NOT make these checks through the API server:

```
kubelet (on node)
    │
    │ Directly executes probe based on type:
    ├── HTTPGet → HTTP request to container's IP:port/path
    ├── TCPSocket → TCP dial to container's IP:port
    ├── Exec → executes command inside container (via CRI)
    └── gRPC → gRPC health check call

Results:
  Success (HTTP 2xx, TCP connect OK, exit code 0, gRPC SERVING)
  Failure (HTTP !2xx, TCP refused, exit code !=0, gRPC not SERVING)
  Unknown (probe couldn't execute; treated as failure for liveness)
```

---

## 4. The Controller Pattern — Watch → Compare → Act → Loop

The controller pattern underpins how probe results affect the rest of the cluster. Understanding this helps you reason about why traffic routing changes when a probe fails.

### 4.1 The Health-Aware Reconciliation Loop

```
┌──────────────────────────────────────────────────────────────────────┐
│          ENDPOINTSLICE CONTROLLER — HEALTH-AWARE RECONCILIATION      │
│                                                                      │
│  WATCH:                                                              │
│  kubelet updates Pod.status.conditions[Ready] = True/False           │
│  EndpointSlice controller informer detects Pod MODIFIED event        │
│  Event queued: "default/my-service"                                  │
│                                                                      │
│  COMPARE:                                                            │
│  Current EndpointSlice: contains pods A, B, C                       │
│  Desired EndpointSlice: pod B's Ready=False → should be removed      │
│  Delta: remove pod B's IP from EndpointSlice                        │
│                                                                      │
│  ACT:                                                                │
│  PATCH EndpointSlice → remove pod B entry                           │
│  kube-proxy informer detects EndpointSlice MODIFIED                 │
│  kube-proxy updates iptables/IPVS → removes pod B from routing      │
│                                                                      │
│  LOOP:                                                               │
│  Continue watching for Pod condition changes                         │
│  When pod B recovers (Ready=True) → add back to EndpointSlice       │
└──────────────────────────────────────────────────────────────────────┘
```

### 4.2 kubelet's Internal Health Loop

```
For each container with probes configured:
  Every periodSeconds:
    Execute probe
    IF success:
      consecutiveSuccesses++
      IF consecutiveSuccesses >= successThreshold:
        Mark container as healthy for this probe type
    IF failure:
      consecutiveFailures++
      IF consecutiveFailures >= failureThreshold:
        Take action:
          Liveness fail: kill container → restart
          Readiness fail: set condition Ready=False
          Startup fail: kill container (as liveness)
```

---

## 5. Understanding Startup Probe — Protecting Slow-Starting Containers

### 5.1 The Problem Startup Probe Solves

Before startup probes (introduced in v1.16, GA in v1.20), operators faced a dilemma:
- **Short `initialDelaySeconds` on liveness**: Fast apps recover quickly, but slow-starting apps (JVM, Python ML models, databases) get killed before they finish initializing
- **Long `initialDelaySeconds` on liveness**: Slow apps start safely, but fast apps take forever to detect deadlocks after startup

**The startup probe cleanly separates the startup phase from the runtime health phase.**

```
WITHOUT startup probe:
─────────────────────────────────────────────────────────────────────
  t=0:  Container starts
  t=5s: liveness probe starts (initialDelaySeconds=5)
        App still loading (JVM startup, DB connection pool init)
        Probe fails!
  t=15s: Probe fails 3rd time → Container killed!
  t=15s: Container restarts → CrashLoopBackOff
  Result: App never starts

WITH startup probe:
─────────────────────────────────────────────────────────────────────
  t=0:   Container starts
  t=0-300s: Startup probe runs every 10s
             Liveness/Readiness blocked during this time
  t=45s: App finishes loading → startup probe succeeds
  t=45s: Liveness probe begins
         Readiness probe begins
  Result: App starts successfully; liveness works as expected
```

### 5.2 Startup Probe Configuration Reference

```yaml
startupProbe:
  # Probe method (same options as liveness/readiness)
  httpGet:
    path: /health/startup
    port: 8080

  # How often to check (seconds)
  periodSeconds: 10

  # Failures before action (kill container for startup)
  failureThreshold: 30

  # Maximum startup window:
  # failureThreshold × periodSeconds = 30 × 10 = 300 seconds (5 minutes)
  # This is the maximum time the app has to start

  # Number of successes to consider started (must be 1 for startup probe)
  successThreshold: 1

  # Per-probe timeout
  timeoutSeconds: 5

  # Note: initialDelaySeconds is less useful with startupProbe
  # Use failureThreshold × periodSeconds for total startup budget
```

### 5.3 Startup Probe Calculation Table

| Application Type | Typical Start Time | Recommended Configuration |
|---|---|---|
| Go/Node.js (fast) | 1-5 seconds | `failureThreshold: 3, periodSeconds: 5` = 15s budget |
| Spring Boot (JVM) | 20-60 seconds | `failureThreshold: 12, periodSeconds: 10` = 120s budget |
| Python ML model | 60-120 seconds | `failureThreshold: 20, periodSeconds: 10` = 200s budget |
| Database (warm) | 30-120 seconds | `failureThreshold: 15, periodSeconds: 10` = 150s budget |
| Database (cold recovery) | 120-600 seconds | `failureThreshold: 60, periodSeconds: 10` = 600s budget |

---

## 6. Lab: Designing a Deployment with Startup Probe

### 6.1 Simulating a Slow-Starting Application

```bash
# Create a configmap that simulates slow startup
cat << 'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: startup-simulator
  namespace: default
data:
  # Script that takes 30 seconds to "start", then serves requests
  server.sh: |
    #!/bin/sh
    READY_FILE="/tmp/app-ready"
    START_TIME=$(date +%s)
    STARTUP_DURATION=30  # seconds to simulate startup

    echo "Application starting... (${STARTUP_DURATION}s startup time)"

    # Background process: simulate initialization
    (
      sleep $STARTUP_DURATION
      echo "started" > $READY_FILE
      echo "Application ready!"
    ) &

    # Serve HTTP requests using netcat
    while true; do
      NOW=$(date +%s)
      ELAPSED=$((NOW - START_TIME))

      if [ -f "$READY_FILE" ]; then
        # App is ready
        echo -e "HTTP/1.1 200 OK\r\nContent-Type: text/plain\r\n\r\nOK - started in ${ELAPSED}s" | \
          nc -l -p 8080 -q 1 2>/dev/null || true
      else
        # App is still starting
        echo -e "HTTP/1.1 503 Service Unavailable\r\nContent-Type: text/plain\r\n\r\nStarting... elapsed: ${ELAPSED}s" | \
          nc -l -p 8080 -q 1 2>/dev/null || true
      fi
    done
EOF
```

### 6.2 Deploy with Startup Probe

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: startup-probe-demo
  labels:
    app: startup-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: startup-demo
  template:
    metadata:
      labels:
        app: startup-demo
    spec:
      volumes:
        - name: scripts
          configMap:
            name: startup-simulator
            defaultMode: 0755

      containers:
        - name: slow-app
          image: busybox:1.36
          command: ["/scripts/server.sh"]
          ports:
            - name: http
              containerPort: 8080

          volumeMounts:
            - name: scripts
              mountPath: /scripts

          # ── STARTUP PROBE ─────────────────────────────────────────────
          startupProbe:
            httpGet:
              path: /            # Must return 2xx to indicate started
              port: http
            # Budget: 20 × 5s = 100 seconds to start
            periodSeconds: 5
            failureThreshold: 20    # 100 seconds total startup budget
            timeoutSeconds: 3
            successThreshold: 1     # Must be 1 for startup probes

          # ── LIVENESS PROBE ────────────────────────────────────────────
          # Only starts AFTER startup probe succeeds
          livenessProbe:
            httpGet:
              path: /
              port: http
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5

          # ── READINESS PROBE ───────────────────────────────────────────
          # Only starts AFTER startup probe succeeds
          readinessProbe:
            httpGet:
              path: /
              port: http
            periodSeconds: 5
            failureThreshold: 3
            successThreshold: 1

          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 100m
              memory: 128Mi
EOF

# Watch the startup sequence
kubectl rollout status deployment/startup-probe-demo

# Observe probe status during startup
kubectl get pods -l app=startup-demo -w &

# Watch events showing startup probe activity
kubectl get events \
  --field-selector involvedObject.name=$(kubectl get pods -l app=startup-demo -o name | head -1 | cut -d/ -f2) \
  --sort-by='.lastTimestamp' -w &

sleep 60
kill %1 %2

# Check pod conditions
kubectl describe pod $(kubectl get pods -l app=startup-demo -o name | head -1) | \
  grep -A 5 "Conditions:\|Startup Probe:\|Liveness Probe:\|Readiness Probe:"
```

### 6.3 Demonstrating Startup Probe Protection

```bash
# Deploy WITHOUT startup probe (liveness probe only with short delay)
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: startup-probe-missing
spec:
  replicas: 1
  selector:
    matchLabels:
      app: startup-missing
  template:
    metadata:
      labels:
        app: startup-missing
    spec:
      volumes:
        - name: scripts
          configMap:
            name: startup-simulator
            defaultMode: 0755
      containers:
        - name: slow-app
          image: busybox:1.36
          command: ["/scripts/server.sh"]
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: scripts
              mountPath: /scripts
          # Liveness with 10s delay — too short for 30s startup!
          livenessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 10   # Won't help — app needs 30s
            periodSeconds: 5
            failureThreshold: 3       # Fails at 10+(3×5) = 25s — before app ready!
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
EOF

# Watch it fail and restart (CrashLoopBackOff)
kubectl get pods -l app=startup-missing -w

# Compare with the version that has startup probe
kubectl get pods -l app=startup-demo -o wide
kubectl get pods -l app=startup-missing -o wide
# startup-demo: Running (healthy, 0 restarts)
# startup-missing: CrashLoopBackOff (killed by liveness before ready)

kubectl delete deployment startup-probe-demo startup-probe-missing
kubectl delete configmap startup-simulator
```

---

## 7. Understanding Readiness Probe — Traffic Gate for Containers

### 7.1 What Readiness Probe Controls

The readiness probe determines whether a Pod should receive traffic from Services. This is the **only probe that affects Service Endpoints** — liveness and startup probes do NOT affect routing.

```
READINESS PROBE RESULT → Pod.status.conditions[Ready]
                       → EndpointSlice controller reacts
                       → kube-proxy updates iptables/IPVS
                       → Traffic included/excluded from Service

KEY DISTINCTION:
  Readiness fails → Pod STAYS RUNNING, just removed from Service routing
  Liveness fails  → Pod RESTARTED (container killed and recreated)
```

### 7.2 When Readiness Probe Should Fail

Design your readiness probe to return a failure when the application:
- Is still loading data or building caches
- Has lost connectivity to a dependency (database, cache, external API)
- Is under too much load to handle additional requests
- Is in a degraded state that would produce errors for users
- Is performing a maintenance operation

```
READINESS: "Am I ready to accept and SUCCESSFULLY PROCESS new user requests?"
LIVENESS:  "Am I still alive and capable of doing MEANINGFUL work?"
```

### 7.3 Readiness Probe in Rolling Updates

The readiness probe is the mechanism that enables **zero-downtime rolling updates**:

```
Rolling update: old pods → new pods
─────────────────────────────────────────────────────────────────────
1. New pod created (image updated)
2. startupProbe runs (if configured)
3. readinessProbe runs:
   - Returns failure (503) while new app loads
   - Pod.status.conditions[Ready] = False
   - Pod NOT in Service Endpoints
   - OLD pods still in endpoints → traffic continues normally
4. readinessProbe succeeds:
   - Pod.status.conditions[Ready] = True
   - Pod added to Service Endpoints
   - New pod receives traffic
5. Deployment controller: N new pods ready → scale down old pods
6. Old pods removed from endpoints when their preStop hooks finish
```

### 7.4 Readiness vs Startup vs Liveness Comparison

| Aspect | Startup Probe | Readiness Probe | Liveness Probe |
|---|---|---|---|
| **Purpose** | Guard slow initialization | Gate traffic routing | Detect runtime deadlock/corruption |
| **Blocks other probes** | Yes (liveness + readiness) | No | No |
| **Action on failure** | Kill container (restart) | Remove from Service | Kill container (restart) |
| **Runs after startup** | N/A (it IS startup) | Yes (after startup) | Yes (after startup) |
| **Affects Service routing** | No | **YES** | No |
| **successThreshold** | Must be 1 | Configurable (default 1) | Must be 1 |
| **Multiple failures needed** | Yes (failureThreshold) | Yes (failureThreshold) | Yes (failureThreshold) |
| **When to use** | Slow-starting apps | All apps with Services | Apps that can deadlock |

---

## 8. Lab: Configuring Deployment with Readiness Probe

### 8.1 Create a Service to Demonstrate Endpoint Behavior

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: readiness-demo
  labels:
    app: readiness-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: readiness-demo
  template:
    metadata:
      labels:
        app: readiness-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - name: http
              containerPort: 80

          # ── READINESS PROBE ───────────────────────────────────────────
          readinessProbe:
            httpGet:
              path: /ready     # Custom readiness endpoint
              port: http
              httpHeaders:
                - name: X-Health-Check
                  value: readiness
            # Time before first probe attempt
            initialDelaySeconds: 5
            # How often to probe
            periodSeconds: 5
            # Consecutive failures before removing from endpoints
            failureThreshold: 3
            # Consecutive successes required to add back to endpoints
            successThreshold: 1
            # Per-probe timeout
            timeoutSeconds: 3

          # ── LIVENESS PROBE ────────────────────────────────────────────
          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5

          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: readiness-demo-svc
spec:
  selector:
    app: readiness-demo
  ports:
    - port: 80
      targetPort: http
EOF

kubectl rollout status deployment/readiness-demo

# Check that all pods are in Service endpoints
kubectl get endpoints readiness-demo-svc
# ENDPOINTS: 10.244.x.x:80,10.244.x.x:80,10.244.x.x:80
```

### 8.2 Demonstrate Readiness Probe Removing Pod from Traffic

```bash
# Watch endpoints in real time (terminal 1)
kubectl get endpoints readiness-demo-svc -w &

# Simulate readiness failure on one pod
# Create a file that makes nginx return 503 on /ready path
# (normally this would be your app's logic)
POD1=$(kubectl get pods -l app=readiness-demo -o name | head -1 | cut -d/ -f2)

# Simulate the readiness probe endpoint returning 503
# Add a custom nginx config that returns 503 for /ready
kubectl exec $POD1 -- sh -c \
  'echo "location /ready { return 503; }" > /etc/nginx/conf.d/health.conf && nginx -s reload'

# Watch: after failureThreshold × periodSeconds, pod removed from endpoints
# failureThreshold=3, periodSeconds=5 → removed after ~15 seconds

sleep 20

# Verify the pod is removed from endpoints
kubectl get endpoints readiness-demo-svc
# ENDPOINTS: only 2 IPs (pod1 removed)

kubectl describe endpoints readiness-demo-svc
# Addresses:  2 pod IPs (ready)
# NotReadyAddresses: pod1 IP (not ready)

# The pod is still running! Just not in load balancing
kubectl get pod $POD1
# STATUS: Running, but READY: 0/1

# Restore readiness
kubectl exec $POD1 -- sh -c \
  'rm /etc/nginx/conf.d/health.conf && nginx -s reload'

sleep 15

# Pod re-added to endpoints after successThreshold successes
kubectl get endpoints readiness-demo-svc
# All 3 IPs back

kill %1  # Stop watching endpoints
kubectl delete deployment readiness-demo
kubectl delete service readiness-demo-svc
```

### 8.3 Readiness Probe with Dependency Check

```bash
# Real-world pattern: check dependencies before marking ready
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-with-dependency-check
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-dep-check
  template:
    metadata:
      labels:
        app: api-dep-check
    spec:
      containers:
        - name: api
          image: nginx:1.25
          ports:
            - containerPort: 80

          # Readiness that checks application health deeply
          readinessProbe:
            # This exec probe could check:
            # - Database connectivity
            # - Required files exist
            # - Cache is populated
            exec:
              command:
                - sh
                - -c
                - |
                  # Check if nginx is running
                  nginx -t 2>/dev/null || exit 1
                  # Check if config file exists
                  [ -f /etc/nginx/nginx.conf ] || exit 1
                  # Check if we can reach the web root
                  curl -f -s http://localhost/ > /dev/null 2>&1 || exit 1
                  exit 0
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5

          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 15
            periodSeconds: 10

          resources:
            requests:
              cpu: 100m
              memory: 128Mi
EOF

kubectl rollout status deployment/api-with-dependency-check
kubectl get pods -l app=api-dep-check
kubectl delete deployment api-with-dependency-check
```

---

## 9. Understanding Liveness Probe — Self-Healing Restart Mechanism

### 9.1 What Liveness Probe Controls

The liveness probe answers the question: **"Is this container still worth running, or should it be restarted?"**

It detects runtime failures that a container can't recover from on its own:
- Thread pool exhaustion (all goroutines/threads blocked on deadlocks)
- Memory corruption or stuck garbage collection
- Lost connections to critical dependencies with no auto-reconnect
- Infinite loops consuming CPU without progress
- Application process still running but not processing requests

**Unlike readiness failure, liveness failure triggers a container restart.**

### 9.2 The Restart Process

```
Liveness probe fails failureThreshold times:
  1. kubelet marks container as unhealthy
  2. kubelet sends SIGTERM to container process
  3. Container has terminationGracePeriodSeconds to exit
  4. If not exited: kubelet sends SIGKILL
  5. kubelet creates NEW container in same pod (same node, same IP)
  6. restartCount incremented
  7. Pod.status.containerStatuses[].state = Waiting (Reason: CrashLoopBackOff if recurring)

IMPORTANT: The Pod itself is NOT rescheduled (keeps same node, same IP)
           Only the container restarts
           The Pod object remains the same
```

### 9.3 Restart Policy and Backoff

```yaml
# Pod-level restartPolicy controls what kubelet does after liveness failure
spec:
  restartPolicy: Always    # Default: Always restart (appropriate for web services)
  # Options:
  # Always:    Always restart → appropriate for Deployments (stateless services)
  # OnFailure: Restart only if non-zero exit → appropriate for Jobs
  # Never:     Never restart → appropriate for batch one-shots
```

**CrashLoopBackOff**: If a container keeps restarting, kubelet applies exponential backoff:
```
1st restart: immediate
2nd restart: 10 seconds
3rd restart: 20 seconds
4th restart: 40 seconds
...
Max backoff: 5 minutes
```

### 9.4 When NOT to Use Liveness Probes

Liveness probes can cause more harm than good if misconfigured:

```
BAD LIVENESS PROBE PATTERN:
  A service under heavy load has high response times
  Liveness probe times out (timeoutSeconds: 1s but load causes >1s response)
  probe fails → container restarted → ALL existing connections dropped
  → Cascading restarts under load → "thundering herd" restart loop

BETTER APPROACH:
  Use generous timeoutSeconds for liveness (5-10s)
  Use readiness probe for traffic gating under load
  Liveness probe only detects total deadlocks (not slow responses)
  
  Or: Don't use liveness probe at all for well-designed services
  that restart cleanly on any failure.
```

---

## 10. Lab: Configuring Deployment with Liveness Probe

### 10.1 Liveness Probe with HTTP Endpoint

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: liveness-demo
  labels:
    app: liveness-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: liveness-demo
  template:
    metadata:
      labels:
        app: liveness-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - name: http
              containerPort: 80

          # ── LIVENESS PROBE ────────────────────────────────────────────
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
              scheme: HTTP
            # IMPORTANT: For liveness, initialDelaySeconds should cover
            # startup time if not using startupProbe
            initialDelaySeconds: 15
            # How often to check
            periodSeconds: 10
            # Failures before restart
            failureThreshold: 3
            # Timeout per probe
            timeoutSeconds: 5
            # Must be 1 for liveness (cannot be > 1)
            successThreshold: 1

          # ── READINESS PROBE ───────────────────────────────────────────
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3

          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
EOF

kubectl rollout status deployment/liveness-demo
kubectl get pods -l app=liveness-demo
```

### 10.2 Demonstrate Liveness Probe Triggering Restart

```bash
# Get a pod name
POD=$(kubectl get pods -l app=liveness-demo -o name | head -1 | cut -d/ -f2)

# Watch the pod (terminal 1)
kubectl get pod $POD -w &

# Watch events (terminal 2)
kubectl get events --field-selector involvedObject.name=$POD \
  --sort-by='.lastTimestamp' -w &

# Simulate liveness failure: make nginx serve 500 errors on /healthz
kubectl exec $POD -- sh -c \
  'mkdir -p /etc/nginx/conf.d && \
   echo "location /healthz { return 500; }" > /etc/nginx/conf.d/healthz.conf && \
   nginx -s reload'

# After failureThreshold × periodSeconds = 3 × 10 = 30 seconds:
# Container restarted!

sleep 45

# Check restart count
kubectl get pod $POD -o jsonpath='{.status.containerStatuses[0].restartCount}'
# Should be 1

kubectl describe pod $POD | grep -A 5 "Last State:\|Restart Count:"
# Last State: Terminated
#   Reason:  Completed (or Error depending on nginx)
#   Exit Code: ...
# Restart Count: 1

kill %1 %2

kubectl delete deployment liveness-demo
```

### 10.3 Liveness Probe with TCP Check

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tcp-liveness-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tcp-liveness
  template:
    metadata:
      labels:
        app: tcp-liveness
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports:
            - name: redis
              containerPort: 6379

          # TCP liveness: checks if port is listening
          livenessProbe:
            tcpSocket:
              port: redis
            initialDelaySeconds: 10
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5

          # TCP readiness: same check for routing
          readinessProbe:
            tcpSocket:
              port: redis
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3

          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi
EOF

kubectl rollout status deployment/tcp-liveness-demo
kubectl get pods -l app=tcp-liveness
kubectl delete deployment tcp-liveness-demo
```

### 10.4 Liveness Probe with Exec Command

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: exec-liveness-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: exec-liveness
  template:
    metadata:
      labels:
        app: exec-liveness
    spec:
      containers:
        - name: app
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              # Create health file on start
              touch /tmp/healthy
              # Sleep to simulate a running process
              sleep 3600
          ports: []

          # Exec-based liveness: checks if file exists
          livenessProbe:
            exec:
              command:
                - cat
                - /tmp/healthy
            initialDelaySeconds: 5
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 3

          resources:
            requests:
              cpu: 50m
              memory: 32Mi
EOF

POD=$(kubectl get pods -l app=exec-liveness -o name | head -1 | cut -d/ -f2)

# Wait for pod to be running
kubectl wait pod/$POD --for=condition=Ready --timeout=30s

# Check healthy
kubectl exec $POD -- cat /tmp/healthy && echo "Probe endpoint OK"

# Simulate failure: remove the health file
kubectl exec $POD -- rm /tmp/healthy

# Watch restart
kubectl get pod $POD -w
# Will restart after failureThreshold × periodSeconds = 30 seconds

kubectl delete deployment exec-liveness-demo
```

---

## 11. Probe Methods Deep Dive — HTTP, TCP, Exec, gRPC

### 11.1 HTTP GET Probe

```yaml
httpGet:
  scheme: HTTP           # HTTP (default) or HTTPS
  host: ""               # Default: pod IP; avoid using; set to localhost if needed
  port: 8080             # Integer port or named port (from ports[].name)
  path: /health/live     # URL path
  httpHeaders:           # Optional custom headers
    - name: Authorization
      value: Bearer healthcheck-token
    - name: X-Custom-Header
      value: k8s-probe

# When to use HTTP:
# - Web applications with HTTP health endpoints
# - REST APIs
# - Any application that serves HTTP
# - Most common probe type in production
```

### 11.2 TCP Socket Probe

```yaml
tcpSocket:
  port: 6379           # Port number or named port
  host: ""             # Default: pod IP

# When to use TCP:
# - Non-HTTP protocols (Redis, MySQL, PostgreSQL, Kafka)
# - Simple connectivity checks
# - Applications that don't have HTTP health endpoints
# - When you only need to verify the port is listening

# Limitations:
# - Only checks if port is listening (doesn't verify application health)
# - A zombie process listening on the port would pass this probe
```

### 11.3 Exec Command Probe

```yaml
exec:
  command:
    - sh
    - -c
    - |
      # Check multiple conditions
      # Return 0 for success, non-zero for failure
      pg_isready -U postgres -d mydb || exit 1
      [ $(df -k /data | tail -1 | awk '{print $5}' | tr -d '%') -lt 90 ] || exit 1
      exit 0

# When to use Exec:
# - Custom health logic not expressible in HTTP/TCP
# - Database readiness checks (pg_isready, mysql ping)
# - File system checks
# - Multi-step health validations

# Performance considerations:
# - Creates a process inside the container on every check
# - Can be expensive for high-frequency probes
# - timeout matters: process killed if exceeds timeoutSeconds
```

### 11.4 gRPC Probe (v1.27 GA)

```yaml
grpc:
  port: 50051            # gRPC port
  service: ""            # Optional: specific gRPC service name

# Requires container to implement:
# grpc.health.v1.Health/Check
# Response: SERVING = success
# Response: NOT_SERVING or other = failure

# When to use gRPC:
# - Applications using gRPC (not HTTP/REST)
# - Microservices with gRPC health protocol
# - Avoids adding HTTP server just for health checks

# Example gRPC health server (Go):
# import "google.golang.org/grpc/health/grpc_health_v1"
# grpc_health_v1.RegisterHealthServer(s, health.NewServer())
```

### 11.5 Probe Parameters Reference Table

| Parameter | Default | Type | Description |
|---|---|---|---|
| `initialDelaySeconds` | 0 | int | Delay before first probe (use startupProbe instead) |
| `periodSeconds` | 10 | int | How often to probe (minimum: 1) |
| `timeoutSeconds` | 1 | int | Probe timeout in seconds (minimum: 1) |
| `successThreshold` | 1 | int | Consecutive successes to mark healthy (startup/liveness must be 1) |
| `failureThreshold` | 3 | int | Consecutive failures before action (minimum: 1) |
| `terminationGracePeriodSeconds` | Pod default | int | Override pod-level grace period for this probe |

---

## 12. Combining All Three Probes — The Full Production Pattern

### 12.1 The Three-Probe Pattern Explained

```yaml
# Production pattern: all three probes working together
containers:
  - name: app
    image: my-service:v2.0

    # ── STARTUP PROBE ──────────────────────────────────────────────────
    # Purpose: Give the app time to start without triggering restarts
    # Blocks liveness and readiness until succeeded
    startupProbe:
      httpGet:
        path: /actuator/health/startup   # Spring Boot startup indicator
        port: 8080
      # Budget: 24 × 5s = 120 seconds maximum startup time
      failureThreshold: 24
      periodSeconds: 5
      timeoutSeconds: 3

    # ── READINESS PROBE ────────────────────────────────────────────────
    # Purpose: Gate traffic during startup, loading, and dependency failures
    # Checks: App logic + dependencies (DB, cache connectivity)
    readinessProbe:
      httpGet:
        path: /actuator/health/readiness  # Spring Boot readiness includes DB check
        port: 8080
      initialDelaySeconds: 0    # startupProbe handles the delay
      periodSeconds: 5
      failureThreshold: 3
      successThreshold: 1
      timeoutSeconds: 3

    # ── LIVENESS PROBE ─────────────────────────────────────────────────
    # Purpose: Restart completely deadlocked containers
    # Checks: Application is processing requests (not just alive)
    livenessProbe:
      httpGet:
        path: /actuator/health/liveness   # Spring Boot liveness (self-only, no deps)
        port: 8080
      initialDelaySeconds: 0    # startupProbe handles the delay
      periodSeconds: 10
      failureThreshold: 3       # Wait 30s before restart
      timeoutSeconds: 5         # Generous timeout for liveness
      successThreshold: 1
```

### 12.2 What Each Probe Endpoint Should Do

```
STARTUP PROBE endpoint (/health/startup):
  ✓ Returns 200 ONLY when initialization is complete
  ✓ Check: all required files loaded, DB schema migrated, caches warmed
  ✓ Returns 503 during initialization

READINESS PROBE endpoint (/health/ready):
  ✓ Returns 200 ONLY when ready to handle production traffic
  ✓ Check: DB connection pool healthy, cache connectivity, required config loaded
  ✓ Check: current load below threshold (optional)
  ✓ Returns 503 if any dependency is unavailable
  ✗ Do NOT check external services you don't own (would cascade failures)
  ✗ Do NOT make the check too expensive (called every periodSeconds)

LIVENESS PROBE endpoint (/health/live):
  ✓ Returns 200 ONLY if the application is not completely broken
  ✓ Check: event loops running, thread pools not exhausted
  ✓ Keep it SIMPLE and fast
  ✗ Do NOT check external dependencies (would restart on dep failure)
  ✗ Do NOT have complex logic (probe itself should never deadlock)
```

---

## 13. Attaching Lifecycle Handlers to Containers

### 13.1 Understanding Container Lifecycle Hooks

Kubernetes provides two lifecycle hooks that allow containers to execute custom code at specific points in their lifecycle:

```
CONTAINER LIFECYCLE:
─────────────────────────────────────────────────────────────────────

  CREATE: Container created by kubelet via CRI
      │
      ▼
  START: Container process starts (CMD/ENTRYPOINT runs)
      │
      ▼── postStart hook executes (async, concurrent with container start)
      │   (Container receives SIGKILL if hook fails)
      │
      ▼
  RUNNING: Container is running normally
      │   Probes execute during this phase
      │
      ▼── preStop hook executes (sync, BEFORE SIGTERM is sent)
      │   (kubelet waits for hook to complete, up to terminationGracePeriod)
      │
      ▼── SIGTERM sent to container process
      │
      ▼── terminationGracePeriodSeconds countdown
      │
      ▼── SIGKILL if still running after grace period
      │
      ▼
  TERMINATED
```

### 13.2 postStart Hook

```yaml
lifecycle:
  postStart:
    # Method 1: HTTP POST (does NOT wait for response by default)
    httpGet:
      host: "localhost"
      path: /api/init
      port: 8080

    # Method 2: Execute a command in the container
    exec:
      command:
        - sh
        - -c
        - |
          # Register with service discovery
          curl -X POST http://consul:8500/v1/agent/service/register \
            -d '{"ID":"'$POD_NAME'","Name":"my-service","Address":"'$POD_IP'"}'

    # Method 3: TCP (checks if connection can be established)
    tcpSocket:
      port: 8080
```

**CRITICAL postStart characteristics:**
- **Runs concurrently with the container's ENTRYPOINT** — there's no guarantee of ordering relative to the main process
- **Container is NOT marked as ready until postStart completes** — even if readiness probe passes
- **If postStart fails → container is killed** (kubelet sends SIGKILL, restartPolicy applies)
- **No separate timeout** — bounded by `terminationGracePeriodSeconds`

### 13.3 preStop Hook

```yaml
lifecycle:
  preStop:
    # Method 1: Sleep to drain connections (most common in production)
    exec:
      command:
        - sh
        - -c
        - |
          # Wait for in-flight requests to complete
          sleep 5
          # Optionally: trigger graceful shutdown
          kill -QUIT 1

    # Method 2: HTTP call to trigger graceful shutdown
    httpGet:
      host: "localhost"
      path: /shutdown
      port: 8080

    # Method 3: Deregister from service discovery
    exec:
      command:
        - sh
        - -c
        - |
          curl -X PUT http://consul:8500/v1/agent/service/deregister/$POD_NAME || true
```

**CRITICAL preStop characteristics:**
- **Runs BEFORE SIGTERM is sent** — gives app time to prepare for shutdown
- **Synchronous** — kubelet WAITS for hook to complete before sending SIGTERM
- **Counts against terminationGracePeriodSeconds** — total time budget = preStop + app shutdown
- **If preStop takes too long → SIGKILL after grace period**
- **If preStop fails → SIGTERM sent anyway** (failure doesn't block termination)

### 13.4 The Complete Termination Sequence

```
kubectl delete pod my-pod (or node drain or rolling update removes pod)
│
│ t=0: Pod removed from Service Endpoints IMMEDIATELY
│      (kube-proxy removes from routing — no new requests)
│
│ t=0: preStop hook starts
│
│ t=5: preStop hook completes (e.g., sleep 5)
│
│ t=5: SIGTERM sent to container PID 1
│      App begins graceful shutdown
│
│ t=30: terminationGracePeriodSeconds expires
│       SIGKILL sent if process still running
│
└ Pod terminated and object deleted
```

---

## 14. Lab: Adding postStart and preStop Events

### 14.1 postStart Hook — Service Registration Pattern

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lifecycle-hooks-demo
  labels:
    app: lifecycle-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: lifecycle-demo
  template:
    metadata:
      labels:
        app: lifecycle-demo
    spec:
      terminationGracePeriodSeconds: 60   # Total budget for preStop + shutdown

      containers:
        - name: app
          image: nginx:1.25
          ports:
            - name: http
              containerPort: 80

          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP

          lifecycle:
            # ── postStart ───────────────────────────────────────────────
            postStart:
              exec:
                command:
                  - sh
                  - -c
                  - |
                    echo "$(date): postStart executing for pod $POD_NAME" >> /var/log/nginx/lifecycle.log
                    # Warm up the application cache
                    curl -s http://localhost/ > /dev/null 2>&1 || true
                    # Create a registration marker
                    echo "$POD_NAME registered at $(date)" > /tmp/registered
                    echo "$(date): postStart completed" >> /var/log/nginx/lifecycle.log

            # ── preStop ─────────────────────────────────────────────────
            preStop:
              exec:
                command:
                  - sh
                  - -c
                  - |
                    echo "$(date): preStop executing for pod $POD_NAME" >> /var/log/nginx/lifecycle.log
                    # Allow time for:
                    # 1. Load balancer to stop sending new requests
                    # 2. In-flight requests to complete
                    # Typical recommendation: 5-15 seconds depending on your load balancer
                    sleep 10
                    echo "$(date): Sending nginx graceful shutdown signal" >> /var/log/nginx/lifecycle.log
                    # Send nginx QUIT signal for graceful shutdown
                    nginx -s quit 2>/dev/null || true
                    echo "$(date): preStop completed" >> /var/log/nginx/lifecycle.log

          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5

          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 15
            periodSeconds: 10

          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 200m
              memory: 256Mi

          volumeMounts:
            - name: logs
              mountPath: /var/log/nginx

      volumes:
        - name: logs
          emptyDir: {}
EOF

kubectl rollout status deployment/lifecycle-hooks-demo

# Verify postStart ran
POD=$(kubectl get pods -l app=lifecycle-demo -o name | head -1 | cut -d/ -f2)
kubectl exec $POD -- cat /var/log/nginx/lifecycle.log
# timestamp: postStart executing for pod lifecycle-hooks-demo-xxx
# timestamp: postStart completed

kubectl exec $POD -- cat /tmp/registered
# lifecycle-hooks-demo-xxx registered at timestamp
```

### 14.2 Observing preStop During Rolling Update

```bash
# Set up a long-running request to observe graceful shutdown
kubectl port-forward deployment/lifecycle-hooks-demo 8080:80 &

# Watch the graceful termination during rolling update
kubectl get pods -l app=lifecycle-demo -w &

# Trigger rolling update (will trigger preStop on old pods)
kubectl set image deployment/lifecycle-hooks-demo app=nginx:1.26

# Meanwhile observe: pod removed from endpoints immediately
# preStop sleeps 10 seconds (allowing in-flight requests to complete)
# SIGTERM sent after preStop
# Pod terminates gracefully

sleep 45
kill %1 %2

# Check the logs from a terminated pod
# (logs still accessible until pod object deleted)
kubectl logs \
  $(kubectl get pods -l app=lifecycle-demo -o name | tail -1) \
  2>/dev/null || echo "Pod already terminated"
```

### 14.3 Complete Production Lifecycle Example

```bash
cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-lifecycle-example
  annotations:
    kubernetes.io/change-cause: "Production lifecycle patterns demo"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: prod-lifecycle
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0   # Zero downtime requires proper lifecycle
  template:
    metadata:
      labels:
        app: prod-lifecycle
    spec:
      # CRITICAL: Must be longer than preStop + app shutdown time
      terminationGracePeriodSeconds: 90

      containers:
        - name: web
          image: nginx:1.25
          ports:
            - name: http
              containerPort: 80

          lifecycle:
            postStart:
              exec:
                command: ["/bin/sh", "-c",
                  "echo 'Container started' > /tmp/health && echo 'postStart done'"]

            preStop:
              exec:
                command: ["/bin/sh", "-c",
                  # Step 1: Sleep to allow load balancer to drain (15s)
                  # Step 2: Send graceful shutdown to nginx
                  "sleep 15 && nginx -s quit"]

          # Startup probe: max 120 seconds for startup
          startupProbe:
            httpGet:
              path: /
              port: http
            failureThreshold: 24
            periodSeconds: 5
            timeoutSeconds: 3

          # Readiness: tight probe for quick endpoint management
          readinessProbe:
            httpGet:
              path: /
              port: http
            periodSeconds: 5
            failureThreshold: 3
            successThreshold: 1
            timeoutSeconds: 3

          # Liveness: generous probe to avoid false restarts
          livenessProbe:
            httpGet:
              path: /
              port: http
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5

          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false  # nginx needs writable temp dirs

      # Pod-level security
      securityContext:
        runAsNonRoot: false  # nginx requires root for port 80
EOF

kubectl rollout status deployment/production-lifecycle-example
kubectl describe deployment production-lifecycle-example | grep -A 30 "Pod Template:"
kubectl delete deployment production-lifecycle-example lifecycle-hooks-demo
```

---

## 15. Built-in Controllers and Health Management Interaction

### 15.1 EndpointSlice Controller — The Critical Link

This is the controller that translates probe results into actual traffic routing decisions:

```
kubelet executes readiness probe
    │
    │ Probe result: FAIL
    ▼
kubelet updates Pod.status.conditions[Ready] = False
    │
    │ API server watch event: Pod MODIFIED
    ▼
EndpointSlice Controller informer detects change
    │
    │ Compare: desired (Ready pods) vs current EndpointSlice
    ▼
EndpointSlice Controller patches EndpointSlice
    │ Removes pod IP from addresses
    │ Adds pod IP to notReadyAddresses
    ▼
kube-proxy informer detects EndpointSlice change
    │
    ▼
kube-proxy updates iptables/IPVS rules
    │ Pod IP removed from Service routing
    ▼
No new traffic goes to unhealthy pod
```

### 15.2 Deployment Controller and Probes

The Deployment controller uses Pod readiness (from readiness probes) to determine rollout progress:

```bash
# During rolling update:
# Deployment controller waits for new pods to be Ready before removing old pods
# "Ready" = readiness probe passing

# minReadySeconds adds additional wait time after readiness passes:
spec:
  minReadySeconds: 10  # Wait 10s after pod becomes Ready before counting it
                       # Prevents flapping pods from being counted as "done"

# Deployment condition:
# Available: readyReplicas >= minReadySeconds of being Ready
# Progressing: rolling update in progress
# ProgressDeadlineExceeded: update stalled
```

### 15.3 HPA Controller and Readiness

```bash
# HPA excludes not-Ready pods from its calculations
# This prevents HPA from scaling based on pods that are warming up

# Example: 
# 3 pods: 2 running at 80% CPU, 1 starting (readiness failing)
# HPA calculates: (2 pods × 80%) / 2 pods = 80% avg (not 3 pods)
# This is correct behavior: don't count warming pods in scaling decision

# startupProbe ensures pods don't affect HPA during startup either
# (Pod not Ready = not counted in HPA calculations)
```

### 15.4 ReplicaSet Controller and Pod Recovery

```bash
# ReplicaSet controller doesn't directly react to probe failures
# It reacts to Pod DELETION or Pod going to Failed phase

# Liveness probe failure:
# 1. kubelet kills container (not pod)
# 2. Pod status changes to ContainerCreating (not Failed)
# 3. Pod IP stays same; ReplicaSet count unchanged
# 4. RS controller not triggered (pod still exists)

# Pod failure (crash, eviction):
# 1. Pod goes to Failed phase
# 2. RS controller detects Pod count < desired
# 3. RS controller creates new Pod
# 4. New Pod scheduled by scheduler
# 5. New Pod gets different IP
```

### 15.5 StatefulSet Controller

StatefulSets interact with probes differently than Deployments:
- Ordered rolling updates: next pod only updated when previous pod is Ready
- This makes readiness probes even more critical in StatefulSets
- A stuck readiness probe can halt the entire StatefulSet rollout

```bash
# StatefulSet rollout respects readiness
kubectl rollout status statefulset my-db
# Waiting for 1 pods to be ready...
# waiting for statefulset rolling update to complete 1 pods at 'my-db' to be ready...
```

### 15.6 DaemonSet Controller

DaemonSet rolling updates also respect readiness probes per node:
- Updates one node at a time (by default)
- Waits for the DaemonSet pod to be Ready before moving to next node
- Critical for log collectors, monitoring agents, network plugins

### 15.7 Node Controller

The Node controller adds system taints when nodes become unhealthy. This triggers pod eviction, which creates new pods on healthy nodes. The new pods go through the full probe sequence (startup → readiness → liveness). The readiness probe ensures the replacement pods are actually serving traffic before being counted.

### 15.8 Job Controller

Jobs don't use readiness probes for routing (they don't have Services typically), but liveness probes are valuable for detecting deadlocked batch jobs:

```yaml
# Job with liveness probe to detect stuck processing
spec:
  template:
    spec:
      containers:
        - name: batch-processor
          livenessProbe:
            exec:
              command: ["check-processing-progress.sh"]  # Returns 1 if no progress in 5 min
            periodSeconds: 300   # Check every 5 minutes
            failureThreshold: 1  # Restart immediately if stuck
```

### 15.9 Garbage Collector

The Garbage Collector doesn't directly interact with probes but is affected by them:
- When a container is restarted due to liveness probe failure, old container resources are cleaned up by kubelet (not GC)
- Pod-level ownerReferences cascades deletion of associated resources when Pods are deleted by RS controller

### 15.10 PersistentVolume Controller

PV operations can affect probe behavior:
- If a volume mount fails during pod startup, the container may not start properly
- Startup probe + liveness probe should be configured to give time for volume mounts
- A PVC stuck in Pending state prevents the pod from starting → startup probe never runs

---

## 16. Internal Working Concepts — Informers, Work Queues & Reconciliation

### 16.1 How kubelet Processes Probe Results

```
kubelet internal probe manager:
─────────────────────────────────────────────────────────────────────
For each container:
  probeManager.AddPod(pod)
    → Creates goroutine per probe type (startup, liveness, readiness)

Each goroutine:
  ticker := time.NewTicker(periodSeconds)
  for range ticker.C:
    result := executeProbe(probe)
    resultsManager.Set(podUID, containerID, probeType, result)
    
    if result == Failure:
      failureCount++
      if failureCount >= failureThreshold:
        if probeType == Liveness || Startup:
          eventManager.Record("Killing container due to failed probe")
          containerManager.KillContainer(containerID)
        if probeType == Readiness:
          statusManager.SetContainerReadiness(containerID, false)
          // This triggers Pod.status.conditions[Ready] update
          // Which triggers API server notification
          // Which triggers EndpointSlice controller
```

### 16.2 The Informer Chain for Health Events

```
kubelet                    API server                 Controller
   │                           │                         │
   │ Pod.status update         │                         │
   │ (Ready condition)         │                         │
   ├──── PATCH /pods/status ──►│                         │
   │                           │ Write to etcd           │
   │                           ├── etcd.Put() ──────────►│
   │                           │                         │
   │                           │ Watch event MODIFIED    │
   │                           │ (Pod Ready = False)     │
   │                           ├──── Watch event ───────►│ EndpointSlice
   │                           │                         │ controller
   │                           │                         │ informer
   │                           │                         │
   │                           │                         │ Compare:
   │                           │                         │ Pod IP in
   │                           │                         │ EndpointSlice?
   │                           │                         │
   │                           │                         │ ACT:
   │                           │ PATCH EndpointSlice     │
   │                           │◄── Remove pod IP ───────┤
   │                           │                         │
   │                           │ Watch event MODIFIED    │
   │                           │ (EndpointSlice changed) │
   │                           ├──── Watch event ───────►│ kube-proxy
   │                           │                         │ informer
   │                           │                         │
   │                           │                         │ Update iptables
   │                           │                         │ Remove pod IP
   │                           │                         │ from routing
```

### 16.3 Work Queue Behavior for Health Events

```bash
# When many pods fail health probes simultaneously (e.g., bad deployment):
# EndpointSlice controller receives many MODIFIED events
# Work queue deduplicates: "default/my-service" → single reconcile
# Single reconcile updates all affected endpoints at once

# This is more efficient than per-pod processing
# and prevents thundering herd on the API server

# Monitor EndpointSlice controller queue depth
curl -sk https://127.0.0.1:10257/metrics | \
  grep 'workqueue_depth{name="endpointslice"}' | grep -v "^#"
```

---

## 17. API Server and etcd Interaction in Health Management

### 17.1 The Golden Rule

```
╔══════════════════════════════════════════════════════════════════════╗
║                                                                      ║
║  kubelet executes probes LOCALLY on the node.                        ║
║  It does NOT go through the API server to check pod health.          ║
║                                                                      ║
║  However, kubelet DOES go through the API server to:                 ║
║  1. Update Pod.status.conditions[Ready] on probe result change       ║
║  2. Update Pod.status.containerStatuses[].restartCount on restart    ║
║  3. Record events (FailedProbe, Unhealthy, etc.)                     ║
║                                                                      ║
║  The EndpointSlice controller NEVER accesses etcd directly:          ║
║  It watches Pod changes via API server informer                      ║
║  And writes EndpointSlice changes through the API server             ║
║                                                                      ║
╚══════════════════════════════════════════════════════════════════════╝
```

### 17.2 etcd Storage of Health State

```bash
# Pod health state stored in etcd:
# /registry/pods/<namespace>/<pod-name>
# → Pod.status.conditions (includes Ready condition)
# → Pod.status.containerStatuses (includes ready, restartCount)

# EndpointSlice stored in etcd:
# /registry/endpointslices/<namespace>/<slice-name>
# → endpoints[].conditions.ready
# → endpoints[].addresses

# View current pod ready condition
kubectl get pod <n> \
  -o jsonpath='{.status.conditions[?(@.type=="Ready")]}' | jq

# View EndpointSlice status
kubectl get endpointslices -l kubernetes.io/service-name=<svc-name> -o yaml | \
  grep -A 5 "conditions:"
```

---

## 18. Leader Election — HA for Health-Aware Controllers

### 18.1 EndpointSlice Controller Leader Election

```bash
# EndpointSlice controller runs inside kube-controller-manager
# Multiple controller-manager instances → only one leader
# Leader election via Lease object

kubectl get lease kube-controller-manager -n kube-system -o yaml
# spec.holderIdentity: "k8s-cp-1_abc123"   ← Current leader
# spec.leaseDurationSeconds: 15

# If leader fails:
# - Lease expires after 15 seconds
# - Standby wins new lease
# - New leader re-lists all Services and Pods
# - Reconciles EndpointSlices (may add/remove endpoints missed during gap)

# During the ~15 second gap:
# - Existing pods continue serving (iptables rules intact)
# - NEW probe failures don't update endpoints (leader not running)
# - NEW readiness recoveries don't add pods back (leader not running)
# - After recovery: all pending updates processed via reconciliation
```

### 18.2 HA Best Practices for Health Management

```bash
# 1. Run 3 control-plane nodes (ensures EndpointSlice controller HA)
kubectl get pods -n kube-system | grep kube-controller-manager
# Should show 3 pods (one per control-plane node)

# 2. Monitor lease transitions
kubectl get lease kube-controller-manager -n kube-system \
  -o jsonpath='{.spec.leaseTransitions}'
# High count = instability; investigate control plane health

# 3. Don't rely on instantaneous endpoint updates
# Design applications to handle brief periods of stale routing
# → Readiness probe + preStop hook is the correct solution
```

### 18.3 Leader Election Flags

| Flag | Default | Description |
|---|---|---|
| `--leader-elect` | `true` | Enable leader election |
| `--leader-elect-lease-duration` | `15s` | Lease validity period |
| `--leader-elect-renew-deadline` | `10s` | Must renew before expiry |
| `--leader-elect-retry-period` | `2s` | Standby check interval |

---

## 19. Performance Tuning for Probes and Health Checks

### 19.1 Probe Frequency Impact

```
Each probe type adds load:
  HTTP probe:  kubelet → HTTP request to pod → response
  TCP probe:   kubelet → TCP dial to pod → response  
  Exec probe:  kubelet → fork process inside container → collect result
  gRPC probe:  kubelet → gRPC dial → health check → response

Impact on control plane:
  Each probe result update (Pod.status) → API server PATCH
  With 1000 pods × 2 probes × every 10s = 200 PATCH requests/second
  → Monitor: apiserver_request_total{verb="PATCH",resource="pods"}
```

### 19.2 Probe Tuning for Performance

```yaml
# CONSERVATIVE (less load, slower detection):
readinessProbe:
  periodSeconds: 30        # Check every 30 seconds
  failureThreshold: 3      # Takes 90 seconds to remove from endpoints
  timeoutSeconds: 10       # Generous timeout

# AGGRESSIVE (more load, faster detection):
readinessProbe:
  periodSeconds: 5         # Check every 5 seconds
  failureThreshold: 2      # Takes 10 seconds to remove from endpoints
  timeoutSeconds: 2        # Tight timeout

# RECOMMENDED BALANCE FOR PRODUCTION:
readinessProbe:
  periodSeconds: 5-10      # Balance between freshness and load
  failureThreshold: 3      # 15-30 second detection time
  timeoutSeconds: 3-5      # Not too tight (avoids false positives under load)

livenessProbe:
  periodSeconds: 10-30     # Less critical to catch fast; reduce API load
  failureThreshold: 3      # 30-90 seconds before restart
  timeoutSeconds: 5-10     # Very generous (avoid restart loop under load)
```

### 19.3 Startup Probe Performance

```yaml
# Startup probe runs until success, then STOPS
# After startup probe succeeds: no more startup probe checks
# This means startup probe adds load only during the startup window

# Configure to stop quickly after success:
startupProbe:
  periodSeconds: 5          # Check frequently during startup
  failureThreshold: 60      # 300s budget for slow apps
  timeoutSeconds: 3
  successThreshold: 1       # Stop after first success
```

### 19.4 preStop Performance Optimization

```yaml
# preStop with sleep reduces cascading failures during deployments
# But adds to deployment time: each pod termination takes longer

# Balance:
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]  # 5 seconds is usually enough
                                               # Longer = safer but slower deployments

# With 3 replicas and maxUnavailable=1:
# Each pod removal takes 5s (preStop) + shutdown time
# Total rolling update: 3 × (5s + app_shutdown) = minimum 15s
```

### 19.5 Important kubelet Configuration Flags

| Flag | Default | Description |
|---|---|---|
| `--container-runtime-endpoint` | CRI socket | CRI socket for exec probes |
| `--event-qps` | 5 | Rate limit for probe events |
| `--event-burst` | 10 | Burst for probe events |
| `--kube-api-qps` | 5 | Rate limit for status updates |
| `--kube-api-burst` | 10 | Burst for status updates |

---

## 20. Security Hardening for Health Endpoints

### 20.1 Securing Health Probe Endpoints

```yaml
# ISSUE: Exposing detailed health information via probes

# BAD: Health endpoint reveals internal state
# GET /health returns:
# {
#   "db_connection": "postgres://user:pass@host:5432/db",
#   "redis_key_count": 15000,
#   "version": "2.1.0-internal"
# }

# GOOD: Health endpoint reveals only status
# GET /health returns:
# {"status": "UP"}
# or: HTTP 200 (body not checked)

# Best practice: Separate health endpoints by audience
health:
  liveness:
    path: /actuator/health/liveness    # Returns: {"status":"UP"}
  readiness:
    path: /actuator/health/readiness   # Returns: {"status":"UP"}
  info:                                # NOT used for probes
    path: /actuator/info               # Returns: version, build info (for humans only)
```

### 20.2 RBAC for Health-Related Resources

```yaml
# Restrict who can modify pod status (which affects probe results)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-health-manager
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods/status"]
    verbs: ["patch", "update"]    # Only kubelet should need this
  - apiGroups: ["discovery.k8s.io"]
    resources: ["endpointslices"]
    verbs: ["get", "list", "watch"]  # Monitor health routing

# Restrict exec access (which can interfere with probe-executing containers)
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-exec-restricted
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["pods/exec"]
    verbs: ["create"]  # Restrict to specific service accounts only
```

### 20.3 Network Policy for Health Probes

```yaml
# Network policy must allow kubelet to reach probe endpoints
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-kubelet-probes
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
    - Ingress
  ingress:
    # Allow health probes from kubelet (node IP range)
    - from:
        - ipBlock:
            cidr: 10.0.0.0/8    # Node network CIDR
      ports:
        - port: 8080              # Health check port only
          protocol: TCP
    # Allow normal application traffic from other pods
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - port: 8080
```

### 20.4 Service Account for Application Health

```yaml
# Application pods should use dedicated ServiceAccounts
# With minimal permissions
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-app-sa
  namespace: production
automountServiceAccountToken: false   # Only enable if app needs API access
---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      serviceAccountName: web-app-sa
      automountServiceAccountToken: false
      containers:
        - name: web
          # Health endpoints don't need special permissions
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
```

---

## 21. Monitoring and Observability for Application Health

### 21.1 Key Prometheus Metrics

| Metric | Description | Alert Threshold |
|---|---|---|
| `kube_pod_container_status_ready` | Container ready status | 0 for > 5 min |
| `kube_pod_container_status_restarts_total` | Container restart count | Rate > 1/hour |
| `kube_pod_status_phase{phase="Failed"}` | Failed pods | > 0 |
| `kube_pod_status_unschedulable` | Unschedulable pods | > 0 for > 5 min |
| `kube_endpoint_address_available` | Available endpoint count | < minReplicas |
| `kube_endpoint_address_not_ready` | Not-ready endpoint count | > 0 for > 5 min |
| `container_last_seen` | Container last seen timestamp | Gap > 60s |
| `kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}` | CrashLoopBackOff | > 0 |
| `kube_pod_container_status_waiting_reason{reason="OOMKilled"}` | OOMKilled | > 0 |

### 21.2 Probe Event Monitoring

```bash
# Find all probe-related events
kubectl get events --all-namespaces \
  --field-selector reason=Unhealthy \
  --sort-by='.lastTimestamp' | tail -20

kubectl get events --all-namespaces \
  --field-selector reason=ProbeError \
  --sort-by='.lastTimestamp' | tail -20

# Find all container restarts
kubectl get pods --all-namespaces \
  -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}: restarts={.status.containerStatuses[0].restartCount}{"\n"}{end}' | \
  grep -v ": restarts=0" | sort -t= -k2 -rn | head -20
```

### 21.3 Prometheus Alert Rules

```yaml
groups:
  - name: kubernetes-health-probes
    rules:
      # Container restarting frequently (CrashLoopBackOff early warning)
      - alert: KubePodRepeatedlyRestarting
        expr: |
          increase(kube_pod_container_status_restarts_total[1h]) > 3
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} container {{ $labels.container }} restarting frequently"

      # Pod not ready (readiness probe failing)
      - alert: KubePodNotReady
        expr: |
          sum by (namespace, pod) (
            kube_pod_status_ready{condition="true"} == 0
          ) > 0
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} has been not ready for 15 minutes"

      # Service has no ready endpoints
      - alert: KubeServiceNoEndpoints
        expr: |
          kube_endpoint_address_available == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.namespace }}/{{ $labels.endpoint }} has no ready endpoints"

      # High readiness probe failure rate
      - alert: KubeHighReadinessProbeFailureRate
        expr: |
          sum(kube_pod_status_ready{condition="false"}) 
          / sum(kube_pod_status_ready) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "More than 10% of pods are not ready"
```

### 21.4 Grafana Dashboard Queries

```promql
# Pod restart rate over time
rate(kube_pod_container_status_restarts_total[5m]) * 60 * 5

# Service endpoint availability percentage
kube_endpoint_address_available /
(kube_endpoint_address_available + kube_endpoint_address_not_ready) * 100

# Pod readiness by namespace
sum by (namespace) (kube_pod_status_ready{condition="true"})
/ sum by (namespace) (kube_pod_status_ready) * 100
```

---

## 22. Troubleshooting Health Probe Issues

### 22.1 Diagnosing CrashLoopBackOff

```bash
# Step 1: Identify the problem pod
kubectl get pods --all-namespaces | grep CrashLoopBackOff

# Step 2: Check restart count and last state
kubectl describe pod <pod-name> -n <namespace>
# Focus on:
# - "Last State": shows why previous container died (exit code, reason)
# - "Restart Count": how many times it's been restarted
# - "Events": probe failures logged here

# Step 3: Get logs from current container
kubectl logs <pod-name> -n <namespace>

# Step 4: Get logs from PREVIOUS container (after crash)
kubectl logs <pod-name> -n <namespace> --previous

# Step 5: Common causes:
# Exit code 137: OOMKilled (memory limit exceeded)
# → Fix: Increase memory limit or fix memory leak

# Exit code 1/2: Application error
# → Fix: Check application logs for startup errors

# Liveness probe failed (no OOMKill):
# → Exit code matches what your app returns on failure
# → Check if initialDelaySeconds is long enough
# → Consider adding startupProbe

# Step 6: Debug directly
kubectl exec <pod-name> -n <namespace> -- <probe-command>
# Run the exact probe command manually to see result
```

### 22.2 Diagnosing Deployment Stuck

```bash
# Deployment not completing rollout
kubectl rollout status deployment/<n> -n <namespace> --timeout=60s

# Check conditions
kubectl describe deployment <n> -n <namespace> | grep -A 5 "Conditions:"

# Check new pods
kubectl get pods -l app=<label> -n <namespace>
kubectl describe pod <new-pod> -n <namespace>

# COMMON CAUSE: Readiness probe never succeeds
# Check: Is the probe endpoint correct?
kubectl exec <new-pod> -- curl -v http://localhost:8080/health/ready

# Check: Is there an initialDelaySeconds or startupProbe configured correctly?
kubectl get pod <new-pod> -o yaml | grep -A 20 "readinessProbe:"

# Check: Does the application need more time?
# Add startupProbe with larger failureThreshold
kubectl rollout undo deployment/<n>   # Rollback while investigating

# COMMON CAUSE: New pods failing liveness before startup completes
kubectl get pod <n> -o jsonpath='{.status.containerStatuses[0].state}'
# Shows "waiting" with reason "CrashLoopBackOff"
# Fix: Add startupProbe or increase initialDelaySeconds on liveness
```

### 22.3 Diagnosing Node NotReady — Impact on Probes

```bash
# When node goes NotReady:
# kubelet stops running
# Probes stop executing
# Pods become "unknown" state

# Check node status
kubectl get nodes
kubectl describe node <notready-node> | grep -A 10 "Conditions:"

# Check if pods on that node have correct health state
kubectl get pods --all-namespaces \
  --field-selector spec.nodeName=<notready-node>

# Node NotReady → pods transition to Unknown phase
# After pod-eviction-timeout (5 min by default):
# Node controller adds NoExecute taint
# Pods without toleration are evicted
# ReplicaSet controller creates replacement pods
# New pods go through full probe sequence on new node

# Monitor replacement pods
kubectl get pods -l app=<n> -w
# Should see new pods appear and become Ready
```

### 22.4 Probe Debugging Commands

```bash
# Get exact probe configuration for a pod
kubectl get pod <n> \
  -o jsonpath='{.spec.containers[0].livenessProbe}' | jq

kubectl get pod <n> \
  -o jsonpath='{.spec.containers[0].readinessProbe}' | jq

kubectl get pod <n> \
  -o jsonpath='{.spec.containers[0].startupProbe}' | jq

# Check current ready conditions
kubectl get pod <n> \
  -o jsonpath='{.status.conditions}' | jq

# Watch probe events in real time
kubectl get events \
  --field-selector involvedObject.name=<pod-name> \
  --sort-by='.lastTimestamp' -w

# Check container status (ready flag)
kubectl get pod <n> \
  -o jsonpath='{.status.containerStatuses[0].ready}'
# true = passing readiness probe

# Test probe manually from inside the pod
kubectl exec <pod> -- wget -qO- http://localhost:8080/health
kubectl exec <pod> -- nc -z localhost 6379
kubectl exec <pod> -- cat /tmp/healthy   # For exec probe

# Test from another pod in same namespace
kubectl run probe-test --image=curlimages/curl --restart=Never -it --rm \
  -- curl http://<pod-ip>:8080/health/ready
```

---

## 23. Disaster Recovery — Health Checks and Resilience

### 23.1 Probes as Automatic DR

Probes ARE the Kubernetes disaster recovery mechanism for application health:

```
Liveness probe = automatic container restart on deadlock
  → No human intervention needed for stuck processes
  → Self-healing within failureThreshold × periodSeconds

Readiness probe = automatic traffic diversion on unhealthy pods
  → No human intervention needed for degraded instances
  → Traffic automatically routed to healthy pods

StartupProbe = protection during restart recovery
  → Slow-starting containers don't get killed during recovery
  → Application has full startup window on every restart

Together: Self-healing without operator intervention
```

### 23.2 etcd Backup and Probe Configuration

```bash
# Probe configurations are stored in etcd as part of Pod specs
# etcd backup captures all probe configs

# After cluster restore from backup:
# All probe configurations restored
# kubelet on each node reads pod specs from API server
# Probes resume executing immediately
# Health state (Ready conditions) re-established by probes

# GitOps: Probe configs in deployment manifests = always recoverable
# git clone my-deployment-repo && kubectl apply -f .
# → Probes configured correctly from first apply
```

### 23.3 PodDisruptionBudget and Health Probes

```yaml
# PDB protects against too many pods being unhealthy simultaneously
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-server-pdb
  namespace: production
spec:
  # Always keep at least 2 pods ready (passing readiness probe)
  minAvailable: 2
  selector:
    matchLabels:
      app: api-server

# PDB + probes:
# 1. Node drain: PDB prevents draining if it would violate minAvailable
# 2. Rolling update: Deployment waits for ready before removing old
# 3. Voluntary disruption: blocked by PDB if too many pods unhealthy
```

---

## 24. Comparison — kube-apiserver vs kube-scheduler vs kubelet in Health Context

| Dimension | kube-apiserver | kube-scheduler | kubelet |
|---|---|---|---|
| **Role in health** | Stores health state (Pod.status); serves to all watchers | Assigns Pods to nodes; uses readiness for scheduling | Executes probes; runs lifecycle hooks; updates API server |
| **Who executes probes** | No | No | **YES — always kubelet** |
| **Who runs lifecycle hooks** | No | No | **YES — always kubelet** |
| **Who updates endpoints** | Receives updates; notifies EndpointSlice controller | N/A | Indirectly (via readiness → Pod.status → API server → EndpointSlice controller) |
| **etcd access** | YES (direct) | NO (via API server) | NO (via API server) |
| **Port** | 6443 | 10259 | 10250 |
| **Failure impact on probes** | Health state not updateable; stale endpoint state | New pods not placed | Probes stop; containers not health-checked; no restart on failure |
| **HA model** | Active-Active | Active-Passive | One per node (no leader election) |
| **Health endpoint** | `/healthz`, `/readyz`, `/livez` | `/healthz` | `/healthz` |

---

## 25. ASCII Architecture Diagram

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║          KUBERNETES APPLICATION HEALTH MANAGEMENT — ARCHITECTURE                 ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║                                                                                  ║
║  CONTROL PLANE                                                                   ║
║  ┌──────────────────────────────────────────────────────────────────────────┐   ║
║  │  kube-apiserver :6443                                                    │   ║
║  │  Stores: Pod.status.conditions[Ready]                                    │   ║
║  │          Pod.status.containerStatuses[].restartCount                     │   ║
║  │  Watch: EndpointSlice controller gets notified of Pod changes           │   ║
║  └───────────────────────────┬──────────────────────────────────────────────┘   ║
║                              │ Watch event: Pod.status.conditions[Ready] changed║
║  ┌───────────────────────────▼──────────────────────────────────────────────┐   ║
║  │  kube-controller-manager :10257 (Leader Elected)                        │   ║
║  │  EndpointSlice Controller                                                │   ║
║  │  ┌──────────────────────────────────────────────────────────────────┐   │   ║
║  │  │  WATCH Pod status → COMPARE with EndpointSlice → ACT            │   │   ║
║  │  │  Ready=True  → Add pod IP to EndpointSlice addresses            │   │   ║
║  │  │  Ready=False → Move to notReadyAddresses                        │   │   ║
║  │  └──────────────────────────────────────────────────────────────────┘   │   ║
║  └────────────────────────────────────────────────────────────────────────┘   ║
║                                                                                  ║
║  WORKER NODE                                                                     ║
║  ┌──────────────────────────────────────────────────────────────────────────┐   ║
║  │  kubelet :10250                                                          │   ║
║  │                                                                          │   ║
║  │  ┌─────────────────────────────────────────────────────────────────┐   │   ║
║  │  │  PROBE MANAGER                                                  │   │   ║
║  │  │                                                                 │   │   ║
║  │  │  Container lifecycle timeline:                                  │   │   ║
║  │  │  START → postStart hook → STARTUP PROBE (blocks others)        │   │   ║
║  │  │               ↓                                                 │   │   ║
║  │  │  STARTUP SUCCESS → READINESS PROBE + LIVENESS PROBE (parallel) │   │   ║
║  │  │               ↓                                                 │   │   ║
║  │  │  READINESS FAIL → Pod.status.conditions[Ready]=False           │   │   ║
║  │  │                   → EndpointSlice removes pod IP               │   │   ║
║  │  │                   → kube-proxy removes from routing            │   │   ║
║  │  │               ↓                                                 │   │   ║
║  │  │  LIVENESS FAIL  → Container RESTARTED (same pod, same IP)      │   │   ║
║  │  │  (failureThreshold × periodSeconds delay)                       │   │   ║
║  │  │               ↓                                                 │   │   ║
║  │  │  TERMINATION → preStop hook → SIGTERM → gracePeriod → SIGKILL  │   │   ║
║  │  └─────────────────────────────────────────────────────────────────┘   │   ║
║  │                                                                          │   ║
║  │  ┌─────────────────────────────────────────────────────────────────┐   │   ║
║  │  │  CONTAINER (your application)                                   │   │   ║
║  │  │                                                                 │   │   ║
║  │  │  HTTP probe endpoints:                                          │   │   ║
║  │  │  /health/startup  → 200 when initialization complete           │   │   ║
║  │  │  /health/ready    → 200 when ready for traffic                 │   │   ║
║  │  │  /health/live     → 200 when not deadlocked                    │   │   ║
║  │  │                                                                 │   │   ║
║  │  │  Lifecycle hooks:                                               │   │   ║
║  │  │  postStart exec: registration, cache warmup                     │   │   ║
║  │  │  preStop exec: sleep 5 (drain), graceful shutdown signal        │   │   ║
║  │  └─────────────────────────────────────────────────────────────────┘   │   ║
║  │                                                                          │   ║
║  │  kube-proxy (iptables/IPVS)                                             │   ║
║  │  ClusterIP:80 → Ready Pod IPs only                                      │   ║
║  └──────────────────────────────────────────────────────────────────────────┘   ║
║                                                                                  ║
║  PROBE TYPES:                                                                    ║
║  httpGet: HTTP GET to path/port | tcpSocket: TCP dial                           ║
║  exec: command in container     | grpc: gRPC health check                       ║
╚══════════════════════════════════════════════════════════════════════════════════╝
```

---

## 26. Real-World Production Use Cases

### 26.1 Spring Boot Microservice

```yaml
# Spring Boot with Actuator health endpoints
# Handles: JVM startup (30-60s), DB connection pool, cache warmup
spec:
  containers:
    - name: payment-service
      image: payment-service:v2.1.0

      startupProbe:
        httpGet:
          path: /actuator/health/startup
          port: 8080
        failureThreshold: 30      # 30 × 5s = 150s startup budget
        periodSeconds: 5
        timeoutSeconds: 3

      readinessProbe:
        httpGet:
          path: /actuator/health/readiness
          port: 8080
        # Spring Boot readiness includes DB connection pool check
        periodSeconds: 10
        failureThreshold: 3
        timeoutSeconds: 3

      livenessProbe:
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        # Spring Boot liveness: only self (no external deps)
        periodSeconds: 30        # Less frequent for liveness
        failureThreshold: 3
        timeoutSeconds: 10       # Generous timeout

      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 10"]
```

### 26.2 Python ML Model Serving

```yaml
# ML model: large model file loading (60-120s), GPU initialization
spec:
  containers:
    - name: model-server
      image: ml-model-server:v3.0

      startupProbe:
        httpGet:
          path: /health/startup
          port: 8080
        failureThreshold: 60      # 60 × 5s = 300s (5 min) budget
        periodSeconds: 5
        timeoutSeconds: 5

      readinessProbe:
        httpGet:
          path: /health/ready
          port: 8080
        periodSeconds: 10
        failureThreshold: 3
        timeoutSeconds: 5

      livenessProbe:
        exec:
          command: ["/bin/sh", "-c",
            # Check if model is loaded and GPU is responsive
            "python3 -c 'import requests; r=requests.get(\"http://localhost:8080/health/live\"); exit(0 if r.status_code==200 else 1)'"]
        periodSeconds: 30
        failureThreshold: 3
        timeoutSeconds: 15        # Python subprocess takes time

      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c",
              # Allow current inference requests to complete
              "sleep 30"]

  terminationGracePeriodSeconds: 90
```

### 26.3 Database (PostgreSQL)

```yaml
# PostgreSQL: startup probe for initial data loading, TCP liveness
spec:
  containers:
    - name: postgres
      image: postgres:14

      startupProbe:
        exec:
          command: ["pg_isready", "-U", "postgres", "-d", "myapp"]
        failureThreshold: 30    # 30 × 5s = 150s startup budget
        periodSeconds: 5

      readinessProbe:
        exec:
          command: ["pg_isready", "-U", "postgres", "-d", "myapp"]
        periodSeconds: 10
        failureThreshold: 3

      livenessProbe:
        tcpSocket:
          port: 5432
        periodSeconds: 30       # Infrequent; TCP check is simple
        failureThreshold: 3

      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c",
              # Wait for current transactions to complete
              "sleep 5 && su postgres -c 'pg_ctl stop -m smart'"]

  terminationGracePeriodSeconds: 60
```

### 26.4 Node.js/Go Fast-Starting Service

```yaml
# Fast-starting service: no startup probe needed
# But still needs readiness for rolling updates
spec:
  containers:
    - name: api
      image: go-api:v1.5

      # NO startup probe: Go starts in < 1 second
      # (No need for startup probe for fast-starting apps)

      readinessProbe:
        httpGet:
          path: /healthz/ready
          port: 8080
        initialDelaySeconds: 5   # Short delay for Go
        periodSeconds: 5
        failureThreshold: 3

      livenessProbe:
        httpGet:
          path: /healthz/live
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 15       # Go rarely deadlocks; infrequent check
        failureThreshold: 3
        timeoutSeconds: 5

      lifecycle:
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]
```

---

## 27. Best Practices for Production Health Management

### 27.1 Startup Probe

- **Always use startupProbe for JVM, Python ML, database applications** — never use `initialDelaySeconds` as the sole startup strategy
- **Set startup budget generously** — it's better to give too much time than restart a healthy app
- **Budget formula:** `failureThreshold × periodSeconds ≥ 2× average startup time` (safety margin)
- **startupProbe endpoint should check actual initialization** — not just "process is running"

### 27.2 Readiness Probe

- **Every Pod that receives Service traffic MUST have a readiness probe** — without it, rolling updates aren't zero-downtime
- **Readiness endpoint should check dependencies** (DB pool, cache connectivity) relevant to serving traffic
- **Don't check external services you don't control** — this cascades failures
- **Set successThreshold: 1** for fastest recovery when readiness is restored
- **Use minReadySeconds** (Deployment spec, not probe spec) to add stability window after readiness passes

### 27.3 Liveness Probe

- **Be conservative with liveness probes** — a misconfigured liveness probe causes restart loops under load
- **Liveness endpoint should NOT check dependencies** — dependency failure should affect readiness, not cause restarts
- **Set generous timeoutSeconds for liveness** (5-10s) — network blips or GC pauses shouldn't cause unnecessary restarts
- **Consider NOT using liveness probes** for well-designed services that crash cleanly on errors
- **Never use successThreshold > 1 for liveness** (it must be exactly 1)

### 27.4 Lifecycle Hooks

- **Always configure preStop with sleep 5-15** for services behind load balancers — the LB needs time to remove the pod from rotation before SIGTERM arrives
- **Set `terminationGracePeriodSeconds` = preStop duration + max request time + shutdown buffer**
- **postStart hook should not do slow operations** — delays container Ready marking
- **Make preStop hooks idempotent** — they may run multiple times in edge cases
- **Test preStop hooks manually** — `kubectl exec pod -- /path/to/prestop-script.sh`

---

## 28. Common Mistakes and Pitfalls

### 28.1 Setting initialDelaySeconds Instead of Using startupProbe

**Mistake:** `livenessProbe.initialDelaySeconds: 120` for a slow-starting app.  
**Problem:** Liveness now also takes 120s to kick in for ANY container restart during normal operation — even after startup phase. A deadlock at t=200s would take 120s + (3×10s) = 150s to detect instead of just 30s.  
**Fix:** Use `startupProbe` for startup delay; keep `livenessProbe.initialDelaySeconds: 0`.

---

### 28.2 Sharing the Same Endpoint for All Three Probes

**Mistake:** All probes point to the same `/health` endpoint.  
**Problem:** The endpoint has to satisfy the requirements of all three probes simultaneously, which is often impossible. A readiness check that checks DB connectivity would cause container restarts (via liveness) when DB is temporarily unavailable.  
**Fix:** Use separate endpoints: `/health/startup`, `/health/ready`, `/health/live`.

---

### 28.3 Making Liveness Probe Check External Dependencies

**Mistake:**
```yaml
livenessProbe:
  httpGet:
    path: /health   # This checks DB connectivity!
    port: 8080
```
**Problem:** DB goes down → liveness probe fails → container restarted → DB still down → container keeps restarting → CrashLoopBackOff → total outage.  
**Fix:** Liveness probe checks only the process itself. Readiness probe checks dependencies.

---

### 28.4 Not Setting preStop Hook Before Pod Removal

**Mistake:** No preStop hook, defaulting to immediate SIGTERM.  
**Problem:** Load balancer continues routing traffic for ~5-15 seconds after pod is removed from endpoints. SIGTERM is sent immediately → active connections dropped with errors.  
**Fix:** Add `preStop: exec: sleep 10` to give load balancer time to drain.

---

### 28.5 terminationGracePeriodSeconds Shorter Than preStop + Shutdown Time

**Mistake:**
```yaml
terminationGracePeriodSeconds: 15
lifecycle:
  preStop: sleep 20  # 20s preStop > 15s grace period!
```
**Problem:** preStop runs for 20s but total budget is 15s → SIGKILL sent after 15s mid-preStop.  
**Fix:** `terminationGracePeriodSeconds` = preStop time + application shutdown time + safety buffer.

---

### 28.6 Setting successThreshold > 1 for Liveness or Startup Probes

**Mistake:**
```yaml
livenessProbe:
  successThreshold: 3  # INVALID for liveness probes!
```
**Problem:** `successThreshold` must be 1 for liveness and startup probes. Kubernetes will reject this (validation error) or behave unexpectedly.  
**Fix:** Only `readinessProbe.successThreshold` can be > 1.

---

### 28.7 Probe Timeout Too Short Under Load

**Mistake:**
```yaml
livenessProbe:
  timeoutSeconds: 1  # Very tight timeout
```
**Problem:** Under heavy CPU load or during GC pauses, the probe endpoint might take >1s to respond → liveness fails → container restarted under load → cascading failure loop.  
**Fix:** Set `timeoutSeconds: 5-10` for liveness. Under load, use readiness probe to reduce traffic before liveness kills the container.

---

## 29. Interview Questions — Beginner to Advanced

### Beginner Level

**Q1: What are the three types of probes in Kubernetes and what does each do?**

**A:** The three probe types are:
- **Startup Probe**: Runs at container start, blocking liveness and readiness probes until it succeeds. Gives slow-starting applications (JVM, ML models, databases) time to initialize without being killed. On failure (beyond failureThreshold), it kills the container. Use when your app takes > 30 seconds to start.
- **Readiness Probe**: Determines if a Pod should receive traffic from Services. When it fails, the Pod is removed from the Service's Endpoints so no new traffic is routed to it. The Pod continues running. Use this for every application that receives Service traffic.
- **Liveness Probe**: Detects containers that are running but broken (deadlocked, goroutines all blocked, etc.). When it fails failureThreshold times, kubelet kills the container and restarts it. Use this to detect runtime failures the application can't recover from on its own.

---

**Q2: What is the difference between a readiness probe failing and a liveness probe failing?**

**A:** 
- **Readiness probe failure**: Pod STAYS RUNNING but is REMOVED from Service Endpoints. Traffic stops being routed to this pod. The pod continues running and the probe keeps checking. When readiness recovers, the pod is added back to endpoints. Use case: database is temporarily unavailable; pod shouldn't receive traffic but shouldn't be restarted.
- **Liveness probe failure**: Container is KILLED and RESTARTED (following restartPolicy). The pod object stays the same (same IP, same node), but the container process is killed and a new one started. Use case: application is completely deadlocked; killing and restarting is the only way to recover.

Key distinction: **readiness controls traffic routing; liveness controls container restarts**.

---

**Q3: What is a postStart hook and when does it execute?**

**A:** The postStart hook is a lifecycle hook that executes a command (or HTTP call) immediately after a container's main process starts. It runs **concurrently** with the container's main process — there's no guarantee it runs before the main process is ready.

The container is not marked Ready (even if readiness probe passes) until the postStart hook completes successfully. If the postStart hook fails (non-zero exit code), the container is killed.

Common uses: registering with service discovery, warming up caches, creating initialization files, or performing any setup that needs to happen before traffic arrives.

---

### Intermediate Level

**Q4: Explain the zero-downtime rolling update flow with readiness probes.**

**A:** The sequence is:
1. Deployment updates Pod template (e.g., new image)
2. Deployment controller creates new Pod (new image)
3. startupProbe runs (if configured) — blocks readiness/liveness
4. After startup: readinessProbe starts running
5. readinessProbe FAILS initially (app still loading) → Pod NOT in Endpoints
6. OLD pods still in Endpoints → ALL traffic goes to old pods
7. readinessProbe SUCCEEDS → Pod.status.conditions[Ready] = True
8. EndpointSlice controller adds new Pod IP to Endpoints
9. kube-proxy updates iptables → new pod receives traffic
10. Deployment controller: new pod is Ready → scales down old ReplicaSet by 1
11. Old pod receives SIGTERM (after preStop hook runs)
12. Old pod removed from Endpoints BEFORE receiving SIGTERM
13. Process repeats until all old pods replaced

Result: At no point is there a gap in service — old pods always serve traffic until new pods are confirmed ready.

---

**Q5: You have a Spring Boot application that takes 60 seconds to start. How would you configure all three probes?**

**A:**
```yaml
# Startup probe: 60s startup budget + 50% safety margin = 90s
startupProbe:
  httpGet:
    path: /actuator/health/startup
    port: 8080
  failureThreshold: 18    # 18 × 5s = 90 seconds
  periodSeconds: 5
  timeoutSeconds: 3

# Readiness: check Spring Boot's readiness including DB
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  periodSeconds: 10
  failureThreshold: 3     # Remove from LB after 30s of failure
  timeoutSeconds: 3

# Liveness: check only self (no DB dependency)
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  periodSeconds: 30       # Less frequent — we don't want false restarts
  failureThreshold: 3     # Restart after 90s of failure
  timeoutSeconds: 10      # Generous — Spring can be slow under GC
```

Reasoning:
- Startup probe: Spring Boot's built-in `/actuator/health/startup` reports UP when ApplicationReadyEvent fires. Budget covers 60s startup with margin.
- Readiness: Spring Boot's `/actuator/health/readiness` includes DB connection pool. Fails gracefully when DB is temporarily unavailable.
- Liveness: `/actuator/health/liveness` only checks Spring internals (no dependencies). Generous timeout prevents false restarts during GC pauses.

---

### Advanced Level

**Q6: Explain how a readiness probe failure propagates through the Kubernetes system to affect traffic routing.**

**A:** The propagation follows this chain:

1. **kubelet** executes the readiness probe (HTTP call or exec) on the node — directly to the container's IP, not through the API server
2. If the probe fails `failureThreshold` times: kubelet updates `Pod.status.conditions[Ready] = False` via PATCH request to the kube-apiserver
3. **kube-apiserver** writes this update to etcd and notifies watchers
4. **EndpointSlice controller** (in kube-controller-manager) has an informer watching Pod changes. It detects the MODIFIED event showing `Ready = False`
5. EndpointSlice controller reconciles: the desired state (only Ready pods in EndpointSlice) differs from current state (this pod is still in EndpointSlice)
6. EndpointSlice controller sends PATCH to kube-apiserver to update the EndpointSlice — moves this pod's IP from `addresses` to `notReadyAddresses`
7. kube-apiserver writes EndpointSlice update to etcd, notifies watchers
8. **kube-proxy** (on every node) has an informer watching EndpointSlice changes. It detects the MODIFIED event
9. kube-proxy updates its iptables/IPVS rules — removes the pod IP from the Service's NAT rules
10. New traffic arriving at the Service ClusterIP is no longer DNAT'd to the failing pod

**Total propagation time**: typically 5-20 seconds from probe failure to traffic stopping, depending on: probe periodSeconds, failureThreshold, EndpointSlice controller sync period, kube-proxy sync period.

---

**Q7: Describe a scenario where a preStop hook is critical for zero-downtime deployments, and explain exactly what happens during pod termination without it.**

**A:**

**The Problem (without preStop):**

When a pod is removed from a Deployment (during rolling update or scale-down):
1. Kubernetes immediately removes the pod from Endpoints (`Ready = False` set or pod deletion begins)
2. kube-proxy updates iptables rules — no NEW connections go to the pod
3. BUT: most cloud load balancers (AWS ALB, nginx Ingress) have their OWN connection tables. They don't instantly know the pod is gone. The load balancer may continue sending traffic to the pod's IP for 5-15 seconds (until its own health check detects the pod is gone)
4. Meanwhile, SIGTERM is sent to the container process immediately
5. Application starts shutting down, refusing new connections
6. Requests in-flight from the load balancer hit the shutting-down application → connection reset errors → 502/503 errors for users

**With preStop hook:**
```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]
```

1. Pod deletion initiated (rolling update removes pod)
2. Pod removed from Kubernetes Endpoints immediately
3. **preStop hook runs: sleeps 10 seconds**
4. During these 10 seconds: kube-proxy has updated iptables, load balancer has detected pod is unhealthy and stopped routing to it, in-flight requests complete
5. **After preStop completes: SIGTERM sent**
6. Application now gracefully shuts down with no active connections remaining

This ensures zero 502/503 errors during rolling updates. The 10-second sleep bridges the gap between Kubernetes removing the pod from endpoints and the load balancer stopping its traffic.

---

## 30. Cheat Sheet — Commands, Configs & Patterns

### 30.1 Probe Commands

```bash
# Check probe configuration for a pod
kubectl get pod <n> -o jsonpath='{.spec.containers[0].startupProbe}' | jq
kubectl get pod <n> -o jsonpath='{.spec.containers[0].readinessProbe}' | jq
kubectl get pod <n> -o jsonpath='{.spec.containers[0].livenessProbe}' | jq

# Check pod readiness status
kubectl get pod <n> -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'

# Check container restart count
kubectl get pod <n> -o jsonpath='{.status.containerStatuses[0].restartCount}'

# Watch probe events
kubectl get events --field-selector involvedObject.name=<pod> -w

# Check all pods with restarts
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}: {.status.containerStatuses[0].restartCount}{"\n"}{end}' | grep -v ": 0"

# Get pods not ready
kubectl get pods --all-namespaces --field-selector=status.phase=Running | \
  awk 'NR>1 && $3!~/^[0-9]+\/[0-9]+$/ {print}'
# Or:
kubectl get pods -A | grep -v "Running\|Completed" | grep -v NAMESPACE

# Test probe endpoint manually
kubectl exec <pod> -- curl -v http://localhost:8080/health/ready
kubectl exec <pod> -- nc -z localhost 6379  # TCP probe
kubectl exec <pod> -- cat /tmp/healthy        # Exec probe

# Force readiness failure (for testing)
kubectl exec <pod> -- sh -c 'echo fail > /tmp/readiness'

# Debug probe timing
kubectl describe pod <n> | grep -A 5 "Liveness\|Readiness\|Startup"
```

### 30.2 Lifecycle Hook Commands

```bash
# Check lifecycle hook configuration
kubectl get pod <n> -o jsonpath='{.spec.containers[0].lifecycle}' | jq

# Watch termination sequence
kubectl get pod <n> -w
# Running → Terminating → (gone)

# Check termination grace period
kubectl get pod <n> -o jsonpath='{.spec.terminationGracePeriodSeconds}'

# Force immediate deletion (skips preStop and grace period)
kubectl delete pod <n> --grace-period=0 --force
# WARNING: This skips preStop hook; use only for stuck pods

# Normal deletion (respects grace period and preStop)
kubectl delete pod <n>

# Check if pod was properly terminated
kubectl describe pod <n>
# State: Terminated
# Exit Code: 0 (clean) or 143 (SIGTERM received)
```

### 30.3 Quick Probe Templates

```yaml
# FAST-STARTING APPS (< 10s startup):
readinessProbe:
  httpGet: {path: /health, port: 8080}
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
livenessProbe:
  httpGet: {path: /health, port: 8080}
  initialDelaySeconds: 10
  periodSeconds: 15
  failureThreshold: 3

# SLOW-STARTING APPS (30-120s startup):
startupProbe:
  httpGet: {path: /health, port: 8080}
  failureThreshold: 30
  periodSeconds: 5
readinessProbe:
  httpGet: {path: /health/ready, port: 8080}
  periodSeconds: 10
  failureThreshold: 3
livenessProbe:
  httpGet: {path: /health/live, port: 8080}
  periodSeconds: 30
  failureThreshold: 3

# TCP SERVICES (Redis, PostgreSQL, etc.):
startupProbe:
  tcpSocket: {port: 6379}
  failureThreshold: 15
  periodSeconds: 5
readinessProbe:
  tcpSocket: {port: 6379}
  periodSeconds: 10
  failureThreshold: 3
livenessProbe:
  tcpSocket: {port: 6379}
  periodSeconds: 30
  failureThreshold: 3

# LIFECYCLE HOOKS (production default):
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]
terminationGracePeriodSeconds: 90
```

### 30.4 Probe Parameter Reference

| Parameter | Startup | Readiness | Liveness | Notes |
|---|---|---|---|---|
| `initialDelaySeconds` | Optional | Optional | Optional | Use startupProbe instead |
| `periodSeconds` | 5-10 | 5-10 | 10-30 | Readiness: tight; liveness: generous |
| `timeoutSeconds` | 3-5 | 3-5 | 5-10 | Liveness must be generous |
| `successThreshold` | **Must be 1** | 1-3 | **Must be 1** | Readiness can be > 1 |
| `failureThreshold` | 20-60 | 3-5 | 3-5 | Startup: large; others: small |
| Budget (max time) | threshold×period | threshold×period | threshold×period | Startup: cover startup time |

### 30.5 Troubleshooting Quick Reference

```bash
# Pod stuck in Init phase → init container issue
kubectl describe pod <n> | grep "Init Containers:" -A 20

# CrashLoopBackOff → container keeps failing
kubectl logs <n> --previous
kubectl describe pod <n> | grep "Last State:"

# 0/1 Ready → readiness probe failing
kubectl get events --field-selector involvedObject.name=<pod>
kubectl exec <pod> -- curl http://localhost:8080/health

# Deployment stuck → new pods not becoming Ready
kubectl rollout status deployment/<n>
kubectl describe pod <new-pod> | grep Events -A 10
```

---

## 31. Key Takeaways & Summary

### The Application Health Contract

```
YOUR APPLICATION PROMISES:
  /health/startup → 200 when fully initialized
  /health/ready   → 200 when ready to process requests
  /health/live    → 200 when not deadlocked

KUBERNETES PROMISES IN RETURN:
  → No traffic during startup (startup probe protects you)
  → No traffic when /health/ready returns non-200
  → Container restart when /health/live returns non-200 repeatedly
  → Connection draining before SIGTERM (via preStop hook)
  → Sufficient time to shut down gracefully (terminationGracePeriodSeconds)
```

### Decision Framework

```
When to use STARTUP PROBE:
  App starts in > 30 seconds? → YES, use startupProbe
  JVM, Python, database, ML model? → YES
  Go, Node.js, simple binary? → Probably not needed

When to use READINESS PROBE:
  Pod receives traffic from a Service? → ALWAYS use readinessProbe
  No exceptions for production workloads

When to use LIVENESS PROBE:
  App can deadlock/zombie? → YES
  App crashes on failure (clean exit)? → Maybe not needed
  Safety: YES, but configure conservatively

When to use preStop:
  Behind a load balancer or Ingress? → ALWAYS
  Service has incoming connections? → YES
  Minimum: sleep 5; typical: sleep 10-15
```

### The 10 Rules of Production Health Management

1. **Every pod with a Service needs a readiness probe** — no exceptions
2. **Startup and liveness probes are separate concerns** — use startupProbe for slow starts
3. **Liveness probe must NOT check external dependencies** — DB down ≠ restart container
4. **Readiness probe SHOULD check critical dependencies** — DB down = stop routing traffic
5. **preStop sleep is mandatory behind load balancers** — prevents request drops during rolling updates
6. **terminationGracePeriodSeconds > preStop time + shutdown time** — prevent premature SIGKILL
7. **Generous timeoutSeconds for liveness** — GC pauses and load spikes cause false positives
8. **successThreshold for liveness and startup must be 1** — Kubernetes enforces this
9. **kubelet executes probes locally** — probes don't go through API server
10. **Controllers never touch etcd directly** — probe results → API server → EndpointSlice controller → kube-proxy

---

> **This guide covers Kubernetes v1.29+ health management. Consult the official documentation at https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/ for current information.**

---

*End of: Kubernetes — Managing Health of Applications: Complete Production-Grade Guide*
