# 📌 Topic 2.7 — Resource Requests & Limits

> **Module 2 — Workloads & Scheduling**
> **File:** `15-resource-requests-limits.md`
> **CKA Weight:** ~15% (Workloads) — Resources appear in scheduling + troubleshooting tasks

---

## 🎯 What You'll Learn
- What resource requests and limits are
- How requests affect scheduling decisions
- How limits enforce resource boundaries at runtime
- CPU vs Memory — how they behave differently
- QoS Classes — Guaranteed, Burstable, BestEffort
- OOMKilled — what it is and how to fix it
- CPU throttling vs memory killing
- LimitRange — default limits per namespace
- ResourceQuota — total limits per namespace
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Hotel Room Analogy 🏨
> Think of cluster resources like **hotel rooms**:
>
> - **Request** = the room you **reserve** (guaranteed minimum)
>   - When you book, the hotel holds it for you
>   - The scheduler uses this to decide if a node has "room"
>   - Even if you don't use all the space, it's still "yours"
>
> - **Limit** = the **maximum space** you can use
>   - You can use more than your booking (burst)
>   - But if you exceed the max? You get kicked out (OOMKilled)
>     or throttled (CPU)
>
> - **No request/limit** = walk-in guest with no booking
>   - You get whatever's left over
>   - First to be evicted when the hotel gets full
>   - Can use as much as you want (dangerous!)

---

## 🔬 Requests vs Limits — Core Difference

```
┌─────────────────────────────────────────────────────────────┐
│                    REQUESTS                                 │
│                                                             │
│  → Used by the SCHEDULER to pick a node                     │
│  → "I need at least this much to run"                       │
│  → Node must have this much FREE to schedule the pod        │
│  → Guaranteed minimum for the container                     │
│  → Does NOT limit actual usage                              │
│                                                             │
│  Scheduler checks: node.allocatable - sum(pod.requests) ≥ 0 │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     LIMITS                                  │
│                                                             │
│  → Used by the KUBELET/KERNEL at runtime                    │
│  → "This container can never use more than this"            │
│  → Enforced by Linux kernel cgroups                         │
│  → Exceeding CPU limit → throttled (slowed down)            │
│  → Exceeding Memory limit → OOMKilled (killed immediately)  │
│                                                             │
│  Limit is enforced REGARDLESS of node available resources   │
└─────────────────────────────────────────────────────────────┘

VISUAL:

  Available node capacity:  8 CPU,  16Gi RAM

  Pod A:  request=2CPU/4Gi   limit=4CPU/8Gi
  Pod B:  request=2CPU/4Gi   limit=4CPU/8Gi
  Pod C:  request=2CPU/4Gi   limit=4CPU/8Gi
                             ↑
  Total requests: 6CPU/12Gi  → fits on node (6<8, 12<16) ✅
  Pod D: request=3CPU/6Gi    → doesn't fit! (6+3=9 > 8) ❌
                               Pod D stays Pending
```

---

## 📐 CPU Units

```
CPU is measured in "millicores" (m):

  1000m  = 1 CPU core = 1 vCPU = 1 AWS vCPU = 1 GCP Core
  500m   = 0.5 CPU
  250m   = 0.25 CPU
  100m   = 0.1 CPU  (minimum recommended)
  1m     = 0.001 CPU (minimum possible)

Examples in YAML:
  cpu: "1"      → 1 full core
  cpu: "0.5"    → 500m (half core)
  cpu: "500m"   → 500m (same as 0.5)
  cpu: "100m"   → 100 millicores

CPU behavior:
  → CPU is COMPRESSIBLE
  → If container exceeds limit → throttled (runs slower)
  → Container is NOT killed for CPU overuse
  → Pod stays alive, just runs slower
```

---

## 📦 Memory Units

```
Memory is measured in bytes with SI suffixes:

  Ki  = Kibibyte = 1024 bytes
  Mi  = Mebibyte = 1024 Ki  = 1,048,576 bytes
  Gi  = Gibibyte = 1024 Mi  = 1,073,741,824 bytes

  K   = Kilobyte = 1000 bytes
  M   = Megabyte = 1000 K
  G   = Gigabyte = 1000 M

  (Note: Ki/Mi/Gi ≠ K/M/G — use Mi/Gi for RAM)

Examples in YAML:
  memory: "128Mi"    → 128 mebibytes (recommended)
  memory: "1Gi"      → 1 gibibyte
  memory: "512Mi"    → 512 mebibytes
  memory: "256M"     → 256 megabytes (slightly less than 256Mi)

Memory behavior:
  → Memory is INCOMPRESSIBLE
  → If container exceeds limit → OOMKilled IMMEDIATELY
  → Kernel sends SIGKILL (no graceful shutdown!)
  → Exit code: 137 (128 + 9 for SIGKILL)
  → Container restarts (if restartPolicy allows)
```

---

## 📄 Resource YAML — Full Examples

### Basic Container Resources

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
    - name: app
      image: nginx:1.21
      resources:
        requests:
          cpu: "100m"       # scheduler needs 0.1 CPU free on node
          memory: "128Mi"   # scheduler needs 128Mi free on node
        limits:
          cpu: "500m"       # max 0.5 CPU (throttled if exceeded)
          memory: "256Mi"   # max 256Mi (OOMKilled if exceeded)
```

### Resources with Multiple Containers

```yaml
spec:
  containers:
    - name: app
      image: nginx:1.21
      resources:
        requests:
          cpu: "200m"
          memory: "256Mi"
        limits:
          cpu: "1000m"
          memory: "512Mi"

    - name: sidecar
      image: fluentd:v1.14
      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          cpu: "100m"
          memory: "128Mi"

# Pod total request = sum of all containers:
# CPU request:    200m + 50m  = 250m
# Memory request: 256Mi + 64Mi = 320Mi
# Scheduler needs 250m CPU and 320Mi RAM free on node
```

---

## 🎭 QoS Classes — Quality of Service

Kubernetes assigns every pod a QoS class based on its resource configuration. This determines **eviction priority** when a node runs low on resources.

```
┌─────────────────────────────────────────────────────────────┐
│  QoS Class 1: GUARANTEED (highest priority — evicted last)  │
│                                                             │
│  Condition: ALL containers must have:                        │
│    requests.cpu    = limits.cpu                             │
│    requests.memory = limits.memory                          │
│    (AND both must be set)                                   │
│                                                             │
│  Example:                                                   │
│    resources:                                               │
│      requests:                                              │
│        cpu: "500m"                                          │
│        memory: "256Mi"                                      │
│      limits:                                                │
│        cpu: "500m"      ← same as request                   │
│        memory: "256Mi"  ← same as request                   │
│                                                             │
│  Behavior: Gets exactly what it asks for                    │
│            Never evicted unless node is completely full     │
│            Ideal for: production databases, critical apps   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  QoS Class 2: BURSTABLE (medium priority)                   │
│                                                             │
│  Condition: At least one container has:                      │
│    request != limit (OR only limit set, no request)         │
│    BUT not all containers qualify for Guaranteed            │
│                                                             │
│  Example:                                                   │
│    resources:                                               │
│      requests:                                              │
│        cpu: "100m"                                          │
│        memory: "128Mi"                                      │
│      limits:                                                │
│        cpu: "500m"      ← different from request            │
│        memory: "256Mi"  ← different from request            │
│                                                             │
│  Behavior: Can burst up to limit when resources available   │
│            Evicted if node is under pressure                │
│            AND pod is using more than its request           │
│            Most common class in practice                    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  QoS Class 3: BESTEFFORT (lowest priority — evicted first)  │
│                                                             │
│  Condition: NO containers have any requests OR limits set   │
│                                                             │
│  Example:                                                   │
│    resources: {}   ← no resources section at all           │
│                                                             │
│  Behavior: Gets whatever is left over                       │
│            First to be evicted under ANY pressure           │
│            Can use unlimited resources (dangerous!)         │
│            Only for: dev/test, truly non-critical work      │
└─────────────────────────────────────────────────────────────┘

EVICTION ORDER (when node is under pressure):
  BestEffort → evicted first
  Burstable  → evicted second (if using > request)
  Guaranteed → evicted last
```

### Check QoS Class

```bash
kubectl get pod my-pod -o jsonpath='{.status.qosClass}'
# Output: Guaranteed | Burstable | BestEffort

kubectl describe pod my-pod | grep "QoS Class"
# QoS Class: Burstable
```

---

## 💥 OOMKilled — Deep Dive

OOMKilled (Out Of Memory Killed) happens when a container tries to use more memory than its limit.

```
WHAT HAPPENS:

  Container memory limit: 256Mi
  Container starts using more and more memory...
  Memory usage hits 256Mi
         │
         ▼
  Linux kernel OOM killer activates
         │
         ▼
  SIGKILL sent to container process (no warning, no cleanup!)
         │
         ▼
  Container exits with code 137 (128 + 9)
         │
         ▼
  kubelet sees container exited
         │
         ▼
  If restartPolicy = Always/OnFailure:
    Container restarts
    If memory leak present: OOMKilled again → CrashLoopBackOff

HOW TO DETECT:
  kubectl describe pod my-pod
  → State: Terminated
    Reason: OOMKilled
    Exit Code: 137

  kubectl get pod my-pod -o jsonpath=\
    '{.status.containerStatuses[0].lastState.terminated.reason}'
  # OOMKilled

HOW TO FIX:
  Option 1: Increase memory limit
    resources:
      limits:
        memory: "512Mi"   ← doubled

  Option 2: Fix memory leak in application code

  Option 3: If app legitimately needs more:
    Optimize application memory usage
    Consider splitting into multiple smaller services
```

---

## 🔄 CPU Throttling — Deep Dive

```
WHAT HAPPENS when CPU exceeds limit:

  Container CPU limit: 500m
  Container tries to use 800m CPU
         │
         ▼
  Linux kernel CFS (Completely Fair Scheduler) activates
         │
         ▼
  Container gets THROTTLED (runs at 500m max)
  → Computation slows down
  → Requests take longer to process
  → Container NOT killed

HOW TO DETECT CPU THROTTLING:
  # Install metrics-server first, then:
  kubectl top pod my-pod --containers

  # In Prometheus/Grafana:
  # container_cpu_cfs_throttled_seconds_total metric
  # If this is high → container is CPU throttled

HOW TO FIX:
  Option 1: Increase CPU limit
    limits:
      cpu: "1000m"

  Option 2: Remove CPU limit (container uses available CPU freely)
    resources:
      requests:
        cpu: "200m"
      limits:
        memory: "256Mi"   ← keep memory limit, remove CPU limit

  Option 3: Optimize application code for lower CPU usage
```

---

## 📏 LimitRange — Namespace Defaults

LimitRange sets default requests/limits for containers that don't specify their own.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
    # ── CONTAINER DEFAULTS ──────────────────────────────
    - type: Container
      default:              # default LIMIT if not specified
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:       # default REQUEST if not specified
        cpu: "100m"
        memory: "128Mi"
      max:                  # max allowed limit
        cpu: "2"
        memory: "2Gi"
      min:                  # min allowed request
        cpu: "50m"
        memory: "64Mi"

    # ── POD DEFAULTS ────────────────────────────────────
    - type: Pod
      max:                  # max total across all containers
        cpu: "4"
        memory: "4Gi"

    # ── PVC DEFAULTS ────────────────────────────────────
    - type: PersistentVolumeClaim
      max:
        storage: 10Gi
      min:
        storage: 1Gi
```

```bash
# Apply LimitRange
kubectl apply -f limitrange.yaml

# View LimitRange
kubectl get limitrange -n production
kubectl describe limitrange default-limits -n production

# Effect: Any container without resources gets:
# requests: cpu=100m, memory=128Mi
# limits:   cpu=500m, memory=256Mi
# automatically injected by admission controller
```

---

## 📊 ResourceQuota — Namespace Total Limits

ResourceQuota limits the TOTAL resources a namespace can consume.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: production-quota
  namespace: production
spec:
  hard:
    # Compute
    requests.cpu: "10"          # total CPU requests across all pods
    requests.memory: 20Gi       # total memory requests
    limits.cpu: "20"            # total CPU limits
    limits.memory: 40Gi         # total memory limits

    # Object counts
    pods: "50"                  # max 50 pods
    services: "20"
    secrets: "50"
    configmaps: "50"
    persistentvolumeclaims: "10"
    services.loadbalancers: "2"  # max 2 LoadBalancer services
    services.nodeports: "5"
```

```bash
# View quota usage
kubectl describe resourcequota production-quota -n production

# Output:
# Resource              Used    Hard
# --------              ----    ----
# limits.cpu            4500m   20
# limits.memory         8Gi     40Gi
# pods                  12      50
# requests.cpu          2       10
# requests.memory       4Gi     20Gi

# IMPORTANT: If ResourceQuota exists in namespace,
# ALL pods MUST have resource requests and limits set!
# Otherwise: Error from server (Forbidden):
# "must specify limits.cpu, requests.cpu"
```

---

## 🔍 Node Allocatable vs Capacity

```
Node Capacity:
  → Total physical resources on the node
  → CPU: 8 cores, Memory: 16Gi

Node Allocatable:
  → Capacity minus reserved for system components
  → Reserved for: kubelet, OS, kube-proxy, system daemons

  kubectl describe node node1
  # Capacity:
  #   cpu:     8
  #   memory:  16398Mi
  # Allocatable:
  #   cpu:     7800m     ← reserved 200m for system
  #   memory:  15360Mi   ← reserved ~1Gi for system

  Scheduler uses ALLOCATABLE, not Capacity!
  Sum of pod requests must be < Allocatable

# Check node resource usage
kubectl describe node node1 | grep -A10 "Allocated resources"
# Allocated resources:
#   (Total limits may be over 100 percent, ...)
#   Resource           Requests    Limits
#   --------           --------    ------
#   cpu                3500m/7800m  8000m/7800m
#   memory             6Gi/15360Mi  12Gi/15360Mi
```

---

## 💻 Resource Commands Reference

```bash
# ── VIEW RESOURCE USAGE ───────────────────────────────────
# Requires metrics-server installed
kubectl top nodes                    # node CPU/memory usage
kubectl top pods                     # pod CPU/memory usage
kubectl top pods -n kube-system      # system pods
kubectl top pods --containers        # per-container breakdown
kubectl top pods --sort-by=cpu       # sorted by CPU
kubectl top pods --sort-by=memory    # sorted by memory

# ── VIEW REQUESTS/LIMITS ──────────────────────────────────
kubectl describe pod my-pod | grep -A10 "Limits\|Requests"

kubectl get pod my-pod -o jsonpath=\
  '{.spec.containers[0].resources}'

# ── VIEW NODE CAPACITY ────────────────────────────────────
kubectl describe nodes | grep -A10 "Allocated resources"
kubectl get nodes -o jsonpath=\
  '{range .items[*]}{.metadata.name}{"\t"}{.status.allocatable}{"\n"}{end}'

# ── CHECK QoS CLASS ───────────────────────────────────────
kubectl get pod my-pod -o jsonpath='{.status.qosClass}'

# ── CHECK OOMKilled ───────────────────────────────────────
kubectl describe pod my-pod | grep -A5 "Last State"
kubectl get pod my-pod -o jsonpath=\
  '{.status.containerStatuses[*].lastState.terminated.reason}'

# ── LIMITRANGE & RESOURCEQUOTA ────────────────────────────
kubectl get limitrange -n production
kubectl describe limitrange -n production
kubectl get resourcequota -n production
kubectl describe resourcequota -n production
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| No resource requests set | BestEffort QoS — first to be evicted; can starve other pods |
| limit < request | Invalid — apiserver rejects: "limit must be >= request" |
| CPU limit too low | CPU throttling → slow app, hard to diagnose |
| Memory limit too low | OOMKilled → CrashLoopBackOff |
| Setting limit without request | Request defaults to = limit (Guaranteed QoS accidentally) |
| Ignoring ResourceQuota | If quota exists, pods without resources are REJECTED |
| Memory in M instead of Mi | 256M ≠ 256Mi (256M = ~244Mi) — use Mi/Gi always |

---

## 🎓 CKA Exam Tips

```
✅ Know all 3 QoS classes and their conditions:
   Guaranteed: requests == limits (all containers)
   Burstable:  requests != limits (at least one container)
   BestEffort: no requests or limits at all

✅ OOMKilled identification:
   kubectl describe pod → Last State: Terminated, Reason: OOMKilled
   Exit Code: 137

✅ If pod is Pending with "Insufficient cpu/memory":
   → Node doesn't have enough to satisfy requests
   → Either reduce requests OR add more nodes

✅ ResourceQuota requires all pods to have resources set:
   If quota exists and pod has no resources → REJECTED

✅ CPU is compressible (throttled), Memory is not (killed)

✅ LimitRange injects defaults → pods without resources
   get default requests/limits from LimitRange

✅ kubectl top requires metrics-server:
   kubectl apply -f https://github.com/kubernetes-sigs/
     metrics-server/releases/latest/download/components.yaml

✅ Check node allocatable (what scheduler sees):
   kubectl describe node <n> | grep Allocatable
```

---

## ❓ Interview & Scenario Questions

### Q1: What is the difference between resource requests and resource limits?
**Answer:**
**Requests** are used by the Kubernetes scheduler to decide which node can run a pod — the scheduler only places a pod on a node that has enough free allocatable resources to satisfy the request. Requests represent the guaranteed minimum the container will get. **Limits** are enforced at runtime by the Linux kernel's cgroup mechanism — they set the hard maximum a container can use. Exceeding the CPU limit causes throttling (slowed down), while exceeding the memory limit causes the container to be OOMKilled immediately with exit code 137.

---

### Q2: Explain the three QoS classes in Kubernetes.
**Answer:**
- **Guaranteed**: Every container in the pod has both requests and limits set, AND requests equal limits for both CPU and memory. These pods are never evicted unless the node is completely exhausted. Best for production critical workloads.
- **Burstable**: At least one container has requests different from limits, or has only limits set. These can burst above their request when resources are available, but are evicted when the node is under pressure and they're using above their requested amount.
- **BestEffort**: No container has any requests or limits set. These use whatever's available and are evicted first under any resource pressure. Only suitable for non-critical dev/test workloads.

---

### Q3: Scenario — A pod keeps getting OOMKilled every few minutes. How do you diagnose and fix it?
**Answer:**
```bash
# Step 1: Confirm OOMKilled
kubectl describe pod my-pod
# Look for: Last State: Terminated, Reason: OOMKilled, Exit Code: 137

# Step 2: Check current memory limit
kubectl get pod my-pod -o jsonpath=\
  '{.spec.containers[0].resources.limits.memory}'
# e.g., 256Mi

# Step 3: Check actual memory usage
kubectl top pod my-pod --containers
# e.g., using 240Mi → approaching limit → spikes over 256Mi

# Step 4: Fix options:
# A) Increase the limit in deployment:
kubectl set resources deployment/my-app \
  -c app --limits=memory=512Mi

# B) Fix memory leak in app code
# C) Add JVM heap limits (for Java apps):
# env: JAVA_OPTS="-Xmx200m"  (keep below container limit)
```

---

### Q4: Why is CPU overuse handled differently than memory overuse?
**Answer:**
CPU is a **compressible resource** — if a container uses too much, the kernel throttles it (allocates less CPU time), so it just runs slower. The container and its data are not harmed. Memory is an **incompressible resource** — once allocated, the kernel can't just "take it back" without corrupting the application's state. If a container exceeds its memory limit, the only safe option is to kill it (OOMKill) and restart from a clean state. This fundamental difference in how CPU and memory work at the OS level drives Kubernetes' different handling: throttle for CPU, kill for memory.

---

### Q5: Scenario — A namespace has a ResourceQuota, and your new pod is rejected. What's the likely cause?
**Answer:**
Two common causes: (1) The namespace has exhausted one of its quota limits — check with `kubectl describe resourcequota -n <ns>` and look for any resource where `Used >= Hard`. (2) The pod doesn't have resource requests/limits set — when a ResourceQuota exists for compute resources, ALL pods in that namespace MUST have CPU and memory requests/limits defined, otherwise the admission controller rejects them with "must specify requests.cpu" etc. Fix: add resources to the pod spec, or increase the quota if the namespace is full.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│         TOPIC 2.7 — RESOURCE REQUESTS & LIMITS               │
├──────────────────────────────────────────────────────────────┤
│  REQUESTS → used by SCHEDULER (node selection)               │
│  LIMITS   → enforced by KERNEL at runtime                    │
├──────────────────────────────────────────────────────────────┤
│  CPU:    compressible → throttled if exceeded (not killed)   │
│  Memory: incompressible → OOMKilled if exceeded (exit 137)   │
├──────────────────────────────────────────────────────────────┤
│  UNITS:                                                      │
│  CPU:    100m = 0.1 core, 1000m = 1 core                     │
│  Memory: 128Mi = mebibytes (use Mi/Gi not M/G)               │
├──────────────────────────────────────────────────────────────┤
│  QoS CLASSES:                                                │
│  Guaranteed → req == limit (all containers) → evicted last   │
│  Burstable  → req != limit (any container)  → evicted second │
│  BestEffort → no req or limit at all        → evicted first  │
├──────────────────────────────────────────────────────────────┤
│  OOMKilled: Reason=OOMKilled, ExitCode=137                   │
│  Fix: increase memory limit or fix memory leak               │
├──────────────────────────────────────────────────────────────┤
│  LimitRange  → defaults/min/max per container in namespace   │
│  ResourceQuota → total limits per namespace                  │
│  (If quota exists, pods MUST have resources set)             │
├──────────────────────────────────────────────────────────────┤
│  KEY COMMANDS:                                               │
│  kubectl top nodes/pods                                      │
│  kubectl describe node <n> | grep Allocated                  │
│  kubectl get pod <p> -o jsonpath='{.status.qosClass}'        │
│  kubectl describe resourcequota -n <ns>                      │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `16-node-selectors-affinity.md`
> *nodeSelector, nodeAffinity, podAffinity, podAntiAffinity — control where pods land*
