# 📌 Topic 2.1 — Pods Deep Dive

> **Module 2 — Workloads & Scheduling**
> **File:** `09-pods-deep-dive.md`
> **CKA Weight:** 🔥 HIGH — Pods are the foundation of every CKA task

---

## 🎯 What You'll Learn
- What a Pod is and why it's the smallest deployable unit
- Pod lifecycle — phases, conditions, and container states
- Single-container vs multi-container pods (sidecar, ambassador, adapter)
- Init containers — purpose, ordering, and use cases
- Static pods — what they are and how kubelet manages them
- Pod YAML anatomy — every important field explained
- Resource requests and limits at the pod level
- How to debug pods effectively
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Apartment Analogy 🏠
> A **Pod** is like an **apartment unit** in a building:
>
> - The **building** = the Worker Node
> - Each **apartment** = a Pod
> - **Rooms inside the apartment** = containers
> - **Shared utilities** (electricity, plumbing, address) = shared network namespace, volumes, IPC
>
> Key insight:
> - All rooms (containers) in the same apartment share the SAME front door (IP address)
> - Rooms can communicate via intercom (localhost)
> - Each apartment has its OWN address (Pod IP) — unique in the building
> - If the apartment burns down (pod crashes), a new one is built elsewhere (by a controller)
> - A bare pod (no Deployment) = no one rebuilds the apartment if it burns down ❌

---

## 🔬 What is a Pod?

A Pod is the **smallest deployable unit** in Kubernetes. It is a wrapper around one or more containers that:
- Share the same **network namespace** (same IP, same ports)
- Share the same **IPC namespace** (can communicate via shared memory)
- Can share **volumes** (same storage mounts)
- Are always **co-located** on the same node
- Are **co-scheduled** — they start and stop together

```
┌─────────────────────────────────────────────────────────┐
│                        POD                              │
│                  IP: 10.244.1.15                        │
│                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌────────────┐  │
│  │   pause     │    │    nginx    │    │  log-ship  │  │
│  │ (infra/     │    │ (app        │    │ (sidecar   │  │
│  │  sandbox)   │    │  container) │    │  container)│  │
│  │             │    │             │    │            │  │
│  │ Holds:      │    │ Port: 80    │    │ Port: 9090 │  │
│  │ - Network NS│    │             │    │            │  │
│  │ - IPC NS    │    │             │    │            │  │
│  └─────────────┘    └─────────────┘    └────────────┘  │
│                                                         │
│  Shared between ALL containers:                         │
│  ✅ Same IP (10.244.1.15)                               │
│  ✅ Same localhost (127.0.0.1)                          │
│  ✅ Shared volumes (if mounted)                         │
│  ✅ Same node                                           │
└─────────────────────────────────────────────────────────┘
```

---

## 📄 Pod YAML — Complete Anatomy

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-webapp                      # unique name in namespace
  namespace: production                # which namespace
  labels:
    app: webapp                        # for Service selectors
    tier: frontend
    version: "1.2"
  annotations:
    team: "frontend-team"

spec:
  # ── NODE SELECTION ──────────────────────────────────────
  nodeName: node1                      # force to specific node (bypasses scheduler)
  nodeSelector:                        # schedule on nodes with these labels
    disk: ssd
  
  # ── RESTART POLICY ──────────────────────────────────────
  restartPolicy: Always                # Always | OnFailure | Never
  # Always     → restart on any failure (default, good for apps)
  # OnFailure  → restart only on non-zero exit (good for Jobs)
  # Never      → never restart (good for one-off tasks)

  # ── INIT CONTAINERS ─────────────────────────────────────
  initContainers:
    - name: init-db-check
      image: busybox:1.35
      command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']
    - name: init-migrations
      image: my-app:migrate
      command: ['python', 'manage.py', 'migrate']

  # ── MAIN CONTAINERS ─────────────────────────────────────
  containers:
    - name: webapp                     # container name (unique in pod)
      image: nginx:1.21                # image:tag
      imagePullPolicy: IfNotPresent    # Always | Never | IfNotPresent

      # Ports (informational only — doesn't actually expose them)
      ports:
        - containerPort: 80
          protocol: TCP
          name: http

      # Environment Variables
      env:
        - name: ENV
          value: "production"
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database_host
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: db_password

      # All env from ConfigMap/Secret
      envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: app-secret

      # Resource Management
      resources:
        requests:                      # MINIMUM guaranteed
          cpu: "100m"                  # 0.1 CPU core
          memory: "128Mi"
        limits:                        # MAXIMUM allowed
          cpu: "500m"
          memory: "512Mi"

      # Volume Mounts
      volumeMounts:
        - name: config-volume
          mountPath: /etc/app/config
          readOnly: true
        - name: data-volume
          mountPath: /var/data

      # Health Probes
      livenessProbe:                   # Is container alive?
        httpGet:
          path: /healthz
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 5
        failureThreshold: 3

      readinessProbe:                  # Is container ready for traffic?
        httpGet:
          path: /ready
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 3

      startupProbe:                    # Has container started properly?
        httpGet:
          path: /startup
          port: 80
        failureThreshold: 30
        periodSeconds: 10

      # Lifecycle hooks
      lifecycle:
        postStart:
          exec:
            command: ["/bin/sh", "-c", "echo started > /tmp/started"]
        preStop:
          exec:
            command: ["/bin/sh", "-c", "sleep 5"]

      # Security Context (container level)
      securityContext:
        runAsUser: 1000
        runAsNonRoot: true
        readOnlyRootFilesystem: true
        allowPrivilegeEscalation: false

  # ── POD SECURITY CONTEXT ────────────────────────────────
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000                      # group for volumes

  # ── VOLUMES ─────────────────────────────────────────────
  volumes:
    - name: config-volume
      configMap:
        name: app-config
    - name: secret-volume
      secret:
        secretName: app-secret
    - name: data-volume
      emptyDir: {}                     # ephemeral, deleted with pod
    - name: host-volume
      hostPath:
        path: /var/log/app
        type: DirectoryOrCreate

  # ── SERVICE ACCOUNT ─────────────────────────────────────
  serviceAccountName: webapp-sa

  # ── DNS CONFIG ──────────────────────────────────────────
  dnsPolicy: ClusterFirst              # ClusterFirst | Default | None

  # ── IMAGE PULL SECRETS ──────────────────────────────────
  imagePullSecrets:
    - name: registry-secret

  # ── TERMINATION ─────────────────────────────────────────
  terminationGracePeriodSeconds: 30    # how long to wait before SIGKILL
```

---

## 🔄 Pod Lifecycle — Phases

```
POD PHASES:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Pending   → Pod accepted by cluster, not yet running
              Common reasons:
              • Waiting for scheduler to assign a node
              • Pulling container image
              • Init containers running
              • Waiting for PVC to bind

  Running   → At least one container is running
              (doesn't mean all containers are healthy!)

  Succeeded → All containers exited with code 0
              Applies to Jobs — app containers don't usually "succeed"

  Failed    → At least one container exited non-zero
              AND restartPolicy = Never or OnFailure (exhausted)

  Unknown   → Node lost contact with apiserver
              Pod state cannot be determined

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

FLOW DIAGRAM:

  kubectl apply pod.yaml
         │
         ▼
     [Pending]  ← scheduler assigns node
         │        kubelet pulls image
         │        init containers run
         │
         ▼
     [Running]  ← app containers started
         │        probes running
         │
    ┌────┴────┐
    ▼         ▼
[Succeeded] [Failed]
(exit 0)   (exit ≠ 0)
```

---

## 📊 Container States (inside a Running Pod)

```
Even when a Pod is "Running", individual containers have their own state:

  Waiting    → Container is not running yet
               Reason: PullImage | ContainerCreating | PodInitializing

  Running    → Container is executing normally

  Terminated → Container has exited
               Reason: Completed | Error | OOMKilled | StartError

# Check container states:
kubectl describe pod my-pod
# Look for:
# Containers:
#   nginx:
#     State:  Running
#       Started: Mon, 01 Jan 2026 10:00:00 +0000
#     Ready: True
#     Restart Count: 2   ← how many times it restarted
#     Last State: Terminated
#       Reason: OOMKilled    ← was killed for using too much memory
#       Exit Code: 137
```

---

## 🔄 Pod Conditions

```
Conditions tell you MORE than just phase:

kubectl describe pod my-pod | grep -A10 Conditions

Conditions:
  Type              Status
  Initialized       True    ← init containers done
  Ready             True    ← all containers ready, in Service endpoints
  ContainersReady   True    ← all containers running
  PodScheduled      True    ← assigned to a node

If Ready = False → pod is NOT receiving traffic from Services
If PodScheduled = False → scheduler couldn't find a node
```

---

## 🎭 Multi-Container Pod Patterns

### Pattern 1: Sidecar
```
The sidecar ENHANCES the main container.

┌─────────────────────────────────────────┐
│                  POD                    │
│                                         │
│  ┌────────────┐    ┌─────────────────┐  │
│  │   nginx    │    │  log-shipper    │  │
│  │ (main app) │    │  (sidecar)      │  │
│  │            │───►│                 │  │
│  │ writes to  │    │ reads from      │  │
│  │ /var/log   │    │ /var/log →      │  │
│  │            │    │ sends to        │  │
│  └────────────┘    │ Elasticsearch   │  │
│                    └─────────────────┘  │
│          Shared volume: /var/log        │
└─────────────────────────────────────────┘

Use cases:
→ Log shipping (Filebeat, Fluentd alongside your app)
→ Metrics collection (Prometheus exporter sidecar)
→ Service mesh proxy (Istio/Envoy injected sidecar)
→ Secret syncing (Vault agent alongside app)
```

### Pattern 2: Ambassador
```
The ambassador PROXIES outbound connections for the main container.
Main container talks to localhost, ambassador handles external routing.

┌─────────────────────────────────────────┐
│                  POD                    │
│                                         │
│  ┌────────────┐    ┌─────────────────┐  │
│  │   app      │    │   ambassador    │  │
│  │            │───►│   (proxy)       │───►  External DB
│  │ connects   │    │                 │    (with connection
│  │ localhost  │    │ handles:        │     pooling, retries,
│  │ :5432      │    │ - routing       │     TLS, etc.)
│  └────────────┘    │ - TLS           │  │
│                    │ - retries       │  │
│                    └─────────────────┘  │
└─────────────────────────────────────────┘

Use cases:
→ Database connection pooling proxy (pgBouncer)
→ Cloud SQL Proxy (GCP)
→ Smart routing to different DB replicas
```

### Pattern 3: Adapter
```
The adapter TRANSFORMS output of main container to standard format.
Used when your app has non-standard logging/metrics format.

┌─────────────────────────────────────────┐
│                  POD                    │
│                                         │
│  ┌────────────┐    ┌─────────────────┐  │
│  │ legacy-app │    │   adapter       │  │
│  │            │───►│                 │───► Monitoring system
│  │ outputs    │    │ transforms      │    (Prometheus,
│  │ custom     │    │ custom format   │     Datadog, etc.)
│  │ log format │    │ → standard      │  │
│  └────────────┘    │   format        │  │
│                    └─────────────────┘  │
└─────────────────────────────────────────┘

Use cases:
→ Log format transformation
→ Metrics format conversion
→ Protocol translation
```

---

## 🏁 Init Containers — Deep Dive

Init containers run **before** app containers. They must complete successfully (exit 0) before app containers start.

```
INIT CONTAINER FLOW:

  Pod starts
     │
     ▼
  init-container-1 runs
     │ (must exit 0)
     ▼
  init-container-2 runs
     │ (must exit 0)
     ▼
  init-container-N runs
     │ (must exit 0)
     ▼
  app containers start (ALL at once)
     │
     ▼
  Pod = Running

Key properties:
  ✅ Run sequentially (one at a time, in order)
  ✅ Must complete successfully before next starts
  ✅ If init container fails → restarts until success (or pod fails)
  ✅ Can have different images from app containers
  ✅ Share volumes with app containers
  ❌ Do NOT support livenessProbe / readinessProbe
  ❌ NOT included in pod Ready condition check
```

### Init Container Use Cases
```yaml
# Use Case 1: Wait for a dependency to be ready
initContainers:
  - name: wait-for-db
    image: busybox:1.35
    command:
      - sh
      - -c
      - |
        until nc -z -v -w30 postgres-service 5432; do
          echo "Waiting for PostgreSQL..."
          sleep 2
        done
        echo "PostgreSQL is ready!"

# Use Case 2: Run database migrations before app starts
  - name: run-migrations
    image: my-app:v2.0
    command: ["python", "manage.py", "migrate"]
    env:
      - name: DB_URL
        valueFrom:
          secretKeyRef:
            name: db-secret
            key: url

# Use Case 3: Clone git repo into shared volume
  - name: git-clone
    image: alpine/git
    command:
      - git
      - clone
      - https://github.com/myorg/config.git
      - /config
    volumeMounts:
      - name: config-volume
        mountPath: /config
```

---

## 🔒 Static Pods — Deep Dive

Static pods are managed **directly by kubelet**, not by the API server or scheduler.

```
STATIC POD vs NORMAL POD:

Normal Pod:
  kubectl apply → apiserver → etcd → scheduler → kubelet → container
  ↑ Managed by K8s control plane

Static Pod:
  kubelet reads YAML from /etc/kubernetes/manifests/ → container
  ↑ Managed directly by kubelet on that specific node

                    STATIC POD PROPERTIES:
  ┌──────────────────────────────────────────────────────┐
  │ ✅ kubelet creates them automatically on startup      │
  │ ✅ kubelet restarts them if they crash                │
  │ ✅ kubelet creates "mirror pods" in apiserver         │
  │    (you can SEE them via kubectl but not manage them) │
  │ ❌ Cannot be moved to another node                    │
  │ ❌ Cannot be managed by Deployments/ReplicaSets       │
  │ ❌ kubectl delete pod <static-pod> just recreates it! │
  │    (to truly delete: remove the YAML file from disk)  │
  └──────────────────────────────────────────────────────┘
```

### How to Create a Static Pod

```bash
# Find the static pod directory
cat /var/lib/kubelet/config.yaml | grep staticPodPath
# staticPodPath: /etc/kubernetes/manifests

# Create a static pod by placing YAML in the directory
cat <<EOF > /etc/kubernetes/manifests/my-static-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-static-pod
  namespace: default
spec:
  containers:
    - name: nginx
      image: nginx:1.21
      ports:
        - containerPort: 80
EOF

# kubelet picks it up automatically (no kubectl apply needed!)
# Verify:
kubectl get pods   # shows: my-static-pod-<nodename>
#                            ↑ node name is appended to static pod names

# To DELETE a static pod:
rm /etc/kubernetes/manifests/my-static-pod.yaml
# kubelet stops and removes it automatically
```

### Why Static Pods Matter

```
Control plane components ARE static pods:
  ls /etc/kubernetes/manifests/
  → etcd.yaml
  → kube-apiserver.yaml
  → kube-controller-manager.yaml
  → kube-scheduler.yaml

This solves the bootstrap problem:
  How does apiserver start before apiserver is running?
  → kubelet starts these as static pods directly!
  → No apiserver needed to start the apiserver itself ✅
```

---

## 🛠️ Pod Debugging — Full Toolkit

```bash
# ── Status Overview ───────────────────────────────────────
kubectl get pod my-pod
kubectl get pod my-pod -o wide         # which node, IP
kubectl get pods --field-selector=status.phase=Pending

# ── Detailed Info + Events (most useful!) ─────────────────
kubectl describe pod my-pod
# Always look at:
# → State: (Waiting/Running/Terminated + Reason)
# → Last State: (previous container state)
# → Restart Count:
# → Events: (at the bottom — errors appear here)

# ── Logs ──────────────────────────────────────────────────
kubectl logs my-pod                    # current container logs
kubectl logs my-pod -c container-name  # specific container
kubectl logs my-pod --previous         # PREVIOUS crashed container
kubectl logs my-pod -f                 # follow live
kubectl logs my-pod --tail=100         # last 100 lines
kubectl logs my-pod --since=1h         # last hour

# ── Execute Commands ──────────────────────────────────────
kubectl exec my-pod -- ls /app
kubectl exec my-pod -- env
kubectl exec -it my-pod -- /bin/bash   # interactive shell
kubectl exec -it my-pod -c sidecar -- /bin/sh  # specific container

# ── Debug with ephemeral container (K8s 1.23+) ────────────
# When container has no shell (distroless images):
kubectl debug -it my-pod --image=busybox --target=my-container

# ── Copy files ────────────────────────────────────────────
kubectl cp my-pod:/var/log/app.log ./local-app.log
kubectl cp ./config.yaml my-pod:/etc/app/config.yaml

# ── Resource Usage ────────────────────────────────────────
kubectl top pod my-pod
kubectl top pod my-pod --containers    # per container

# ── Events ────────────────────────────────────────────────
kubectl get events --field-selector involvedObject.name=my-pod
kubectl get events --sort-by='.lastTimestamp' -n default
```

---

## 🚨 Common Pod Issues & Fixes

```
STATUS: Pending
  → kubectl describe pod → check Events
  → "Insufficient cpu/memory" → node resources full, add nodes or reduce requests
  → "0/3 nodes available" → no node satisfies scheduling constraints
  → "pod has unbound PVC" → persistent volume not provisioned

STATUS: ImagePullBackOff / ErrImagePull
  → Wrong image name or tag
  → Private registry without imagePullSecrets
  → No internet access from node
  Fix: kubectl describe pod → check Events for exact error

STATUS: CrashLoopBackOff
  → Container starts then crashes repeatedly
  → kubectl logs my-pod --previous → see why it crashed
  → Common causes: wrong command, missing env vars, failed health probe

STATUS: OOMKilled
  → Container exceeded memory limit → kernel killed it
  → kubectl describe pod → Last State: Terminated, Reason: OOMKilled
  → Fix: increase memory limits or fix memory leak

STATUS: Terminating (stuck)
  → Pod stuck in terminating state
  → kubectl delete pod my-pod --force --grace-period=0
  → Or check for finalizers: kubectl get pod my-pod -o yaml | grep finalizer

STATUS: Error
  → Container exited with non-zero code
  → kubectl logs my-pod --previous to see error output
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| Creating bare pods in production | Always use Deployment — bare pods don't self-heal |
| `containerPort` opens the port | `containerPort` is just documentation — the port is already open regardless |
| Init containers run in parallel | Init containers run SEQUENTIALLY — one at a time |
| `kubectl delete pod <static-pod>` deletes it | Static pods are immediately recreated — delete the YAML file instead |
| `restartPolicy: Never` for long-running apps | Use `Always` for apps — `Never` is for one-time tasks |
| Pod `Running` = all containers healthy | Running means at least one container is running — check `Ready` condition |

---

## 🎓 CKA Exam Tips

```
✅ Generate pod YAML quickly:
   kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

✅ Add labels at create time:
   kubectl run nginx --image=nginx --labels="app=web,env=prod"

✅ Run a one-off debug pod (auto-deletes when done):
   kubectl run debug --image=busybox --rm -it -- /bin/sh

✅ Static pod location: /etc/kubernetes/manifests/
   Know this for control plane troubleshooting!

✅ CrashLoopBackOff = check logs of PREVIOUS container:
   kubectl logs <pod> --previous

✅ If you can't exec into container (no shell):
   kubectl debug -it <pod> --image=busybox --target=<container>

✅ Pod status flow to remember:
   Pending → Running → (Succeeded | Failed)

✅ restartPolicy cheat sheet:
   Long-running app → Always
   Batch job        → OnFailure
   One-time task    → Never

✅ Multi-container pod — containers share:
   Same IP, same localhost, same volumes (if mounted)
   They do NOT share filesystem by default
```

---

## ❓ Interview & Scenario Questions

### Q1: What is a Pod and why is it the smallest deployable unit in Kubernetes?
**Answer:**
A Pod is a Kubernetes abstraction that wraps one or more containers into a single deployable unit. It's the smallest unit because Kubernetes doesn't manage containers directly — it manages pods. Containers within a pod share a network namespace (same IP and localhost), can share volumes, and are always co-located and co-scheduled on the same node. A Pod represents a single instance of a running process. You can't schedule individual containers without a pod wrapper.

---

### Q2: What is the difference between init containers and regular containers?
**Answer:**
Init containers run sequentially before any app containers start, and each must exit with code 0 before the next begins. They're used to set up preconditions — waiting for dependencies, running migrations, or populating volumes. Key differences: init containers run sequentially (app containers run in parallel), they don't support liveness/readiness probes, they can use different images, and they're not counted in the pod's Ready condition. If an init container fails, Kubernetes restarts it repeatedly until it succeeds (or the pod fails).

---

### Q3: Scenario — A pod is in CrashLoopBackOff. How do you debug it?
**Answer:**
```bash
# Step 1: Check pod status and events
kubectl describe pod my-pod
# Look at: State, Last State, Exit Code, Events section

# Step 2: Check logs of the crashed container
kubectl logs my-pod --previous
# --previous gets logs from the LAST crashed instance

# Step 3: Common causes and fixes:
# Exit Code 1    → Application error — check app logs
# Exit Code 137  → OOMKilled — increase memory limit
# Exit Code 139  → Segfault — application bug
# "exec failed"  → Wrong command/entrypoint in YAML
# "no such file" → Wrong image or missing dependency

# Step 4: If no shell, use ephemeral debug container
kubectl debug -it my-pod --image=busybox --target=my-container
```

---

### Q4: What is a static pod and when would you use one?
**Answer:**
A static pod is a pod that kubelet manages directly by reading YAML files from a specific directory on disk (`/etc/kubernetes/manifests/` by default), without going through the API server or scheduler. kubelet creates "mirror pods" so they appear in `kubectl get pods` but they can't be truly deleted via kubectl — you must remove the YAML file. Use cases: Kubernetes control plane components (apiserver, etcd, scheduler, controller-manager) use static pods to solve the bootstrap problem — how do you start apiserver if apiserver isn't running? kubelet starts them directly.

---

### Q5: Two containers in a pod both need to use port 80. Is this possible?
**Answer:**
No. All containers in a pod share the same network namespace and therefore the same IP address. Two containers can't both listen on port 80 — they'd conflict. You'd need to run them in separate pods, or configure one container to use a different port (e.g., one on 80, another on 8080). This is also why the sidecar pattern typically has the sidecar on a different port than the main app.

---

### Q6: Scenario — You need a pod that waits for a database to be ready before starting the main application. How do you implement this?
**Answer:**
Use an init container:
```yaml
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:1.35
      command:
        - sh
        - -c
        - |
          until nc -z postgres-service 5432; do
            echo "Waiting for DB..."
            sleep 3
          done
          echo "DB is ready!"
  containers:
    - name: my-app
      image: my-app:v1
```
The init container polls the database port until it's open. Only after it exits with code 0 does Kubernetes start the main `my-app` container.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│                TOPIC 2.1 — PODS DEEP DIVE                    │
├──────────────────────────────────────────────────────────────┤
│  POD = wrapper for 1+ containers sharing:                    │
│  → Same IP, same localhost, optional shared volumes          │
│  → Always co-located on same node                            │
├──────────────────────────────────────────────────────────────┤
│  POD PHASES:                                                 │
│  Pending → Running → Succeeded | Failed | Unknown            │
├──────────────────────────────────────────────────────────────┤
│  MULTI-CONTAINER PATTERNS:                                   │
│  Sidecar    → enhances main app (logging, metrics)           │
│  Ambassador → proxies outbound connections                   │
│  Adapter    → transforms output to standard format           │
├──────────────────────────────────────────────────────────────┤
│  INIT CONTAINERS:                                            │
│  → Run SEQUENTIALLY before app containers                    │
│  → Must exit 0 before next starts                            │
│  → Use: wait for deps, run migrations, populate volumes      │
├──────────────────────────────────────────────────────────────┤
│  STATIC PODS:                                                │
│  → kubelet manages from /etc/kubernetes/manifests/           │
│  → No scheduler needed — kubelet places directly             │
│  → Delete by removing YAML file (not kubectl delete)         │
│  → Control plane components are static pods!                 │
├──────────────────────────────────────────────────────────────┤
│  RESTART POLICIES:                                           │
│  Always     → long-running apps (default)                    │
│  OnFailure  → batch jobs                                     │
│  Never      → one-time tasks                                 │
├──────────────────────────────────────────────────────────────┤
│  KEY DEBUG COMMANDS:                                         │
│  kubectl describe pod <n>         → events + state        │
│  kubectl logs <pod> --previous    → crashed container logs   │
│  kubectl exec -it <pod> -- bash   → shell inside             │
│  kubectl debug -it <pod> \        → debug distroless         │
│    --image=busybox --target=<c>                              │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `10-replicasets.md`
> *ReplicaSet purpose, label selectors, scaling, ownership — deep dive*
