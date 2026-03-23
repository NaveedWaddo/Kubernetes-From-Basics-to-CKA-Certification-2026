# 📌 Topic 2.8 — Node Selectors & Affinity

> **Module 2 — Workloads & Scheduling**
> **File:** `16-node-selectors-affinity.md`
> **CKA Weight:** ~15% (Workloads/Scheduling) — Affinity rules appear in CKA scheduling tasks

---

## 🎯 What You'll Learn
- What nodeSelector is and its limitations
- nodeAffinity — required vs preferred rules
- podAffinity — schedule pods NEAR other pods
- podAntiAffinity — spread pods AWAY from each other
- Topology keys — zones, nodes, regions
- Real-world use cases for each type
- How affinity combines with taints/tolerations
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Office Seating Analogy 💼
> Think of scheduling pods like assigning employees to desks:
>
> - **nodeSelector** = "This employee MUST sit on Floor 3 (GPU floor)"
>   Simple rule — floor must have the exact label
>
> - **nodeAffinity (required)** = "This employee MUST sit in a room
>   that has a window AND is on floors 2-5"
>   More expressive — multiple conditions, set operators
>
> - **nodeAffinity (preferred)** = "This employee PREFERS a window seat,
>   but will sit anywhere if none available"
>   Soft rule — scheduler tries its best but won't block
>
> - **podAffinity** = "This employee wants to sit NEAR the Marketing team"
>   Co-locate pods on the same node/zone as other pods
>
> - **podAntiAffinity** = "These employees should NOT sit next to each other"
>   Spread pods across nodes/zones for high availability

---

## 📍 nodeSelector — Simple Node Selection

The simplest way to constrain pods to specific nodes.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  # ── NODE SELECTOR ───────────────────────────────────────
  nodeSelector:
    accelerator: nvidia-tesla-v100   # only nodes with this label
    kubernetes.io/os: linux          # and this label

  containers:
    - name: ml-trainer
      image: tensorflow:2.9-gpu
```

```bash
# Label nodes first
kubectl label node node1 accelerator=nvidia-tesla-v100
kubectl label node node2 accelerator=nvidia-tesla-v100
kubectl label node node3 disk=ssd

# Verify labels
kubectl get nodes --show-labels
kubectl get nodes -l accelerator=nvidia-tesla-v100

# Remove label
kubectl label node node1 accelerator-    # trailing dash removes label

# Built-in node labels (always present):
# kubernetes.io/hostname=node1
# kubernetes.io/os=linux
# kubernetes.io/arch=amd64
# node.kubernetes.io/instance-type=t3.large   (cloud)
# topology.kubernetes.io/zone=us-east-1a      (cloud)
# topology.kubernetes.io/region=us-east-1     (cloud)
```

### nodeSelector Limitations

```
nodeSelector problems:
  ❌ Only equality matching (label=value)
  ❌ No "OR" conditions (must match ALL labels = AND)
  ❌ No "NOT" conditions
  ❌ No preference — it's all-or-nothing (pod stays Pending if no match)

nodeAffinity solves all of this ✅
```

---

## 🎯 nodeAffinity — Advanced Node Selection

nodeAffinity supports powerful expressions for selecting nodes.

```yaml
spec:
  affinity:
    nodeAffinity:

      # ── REQUIRED (hard requirement — pod won't schedule if not met) ──
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: accelerator
                operator: In
                values: ["nvidia-v100", "nvidia-a100"]  # OR within values
              - key: kubernetes.io/os
                operator: In
                values: ["linux"]
          # Multiple matchExpressions in same term = AND
          # Multiple nodeSelectorTerms = OR

      # ── PREFERRED (soft preference — scheduler tries but won't block) ──
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80             # weight 1-100 (higher = more preferred)
          preference:
            matchExpressions:
              - key: zone
                operator: In
                values: ["us-east-1a"]
        - weight: 20
          preference:
            matchExpressions:
              - key: instance-type
                operator: In
                values: ["m5.xlarge"]
```

---

## 🔧 nodeAffinity Operators

```
┌──────────────────────────────────────────────────────────────┐
│  Operator       │ Description                                │
├──────────────────────────────────────────────────────────────┤
│  In             │ label value must be in the list            │
│                 │ key: zone, In: [us-east-1a, us-east-1b]    │
├──────────────────────────────────────────────────────────────┤
│  NotIn          │ label value must NOT be in the list        │
│                 │ key: env, NotIn: [test, dev]               │
├──────────────────────────────────────────────────────────────┤
│  Exists         │ label key must exist (any value)           │
│                 │ key: gpu (any gpu label)                   │
├──────────────────────────────────────────────────────────────┤
│  DoesNotExist   │ label key must NOT exist                   │
│                 │ key: spot-instance (not a spot node)       │
├──────────────────────────────────────────────────────────────┤
│  Gt             │ label value must be greater than (numeric) │
│                 │ key: storage-size, Gt: ["100"]             │
├──────────────────────────────────────────────────────────────┤
│  Lt             │ label value must be less than (numeric)    │
│                 │ key: generation, Lt: ["5"]                 │
└──────────────────────────────────────────────────────────────┘
```

---

## 🗺️ nodeAffinity — AND / OR Logic

```
UNDERSTANDING LOGIC:

nodeSelectorTerms:        ← multiple terms = OR
  - matchExpressions:     ← multiple expressions = AND
      - key: zone
        operator: In
        values: [us-east-1a]
      - key: gpu           ← AND this
        operator: Exists
  - matchExpressions:     ← OR this entire block
      - key: zone
        operator: In
        values: [us-west-2a]

Translation:
  ( zone=us-east-1a AND gpu exists )
  OR
  ( zone=us-west-2a )

EXAMPLE — Multi-term:
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:       # TERM 1
              - key: disk
                operator: In
                values: ["ssd"]
              - key: region
                operator: In
                values: ["us-east"]
          - matchExpressions:       # TERM 2 (OR)
              - key: disk
                operator: In
                values: ["nvme"]

# Schedule on: (disk=ssd AND region=us-east) OR (disk=nvme)
```

---

## 🔄 IgnoredDuringExecution

```
The long suffix "IgnoredDuringExecution" means:
→ Once a pod is scheduled, affinity rules are NOT re-evaluated
→ If node labels change AFTER scheduling, pod keeps running

There is a "RequiredDuringExecution" variant (alpha/beta):
→ Would evict pods if node labels change and rules no longer match
→ Not commonly used in production yet

Current behavior summary:
  requiredDuringSchedulingIgnoredDuringExecution:
    → MUST match at schedule time
    → NOT re-checked after pod is running

  preferredDuringSchedulingIgnoredDuringExecution:
    → TRIES to match at schedule time
    → NOT re-checked after pod is running
```

---

## 🤝 podAffinity — Co-locate Pods Together

podAffinity schedules a pod ON THE SAME node/zone as other matching pods.

```
Use case: A web frontend needs low latency to its cache
  → Schedule frontend pods ON THE SAME NODE as redis cache pods

┌─────────────────────────────────────────────────────────────┐
│  Node 1                  │  Node 2                         │
│  [redis-cache] ←─────────│──────────────────────────────── │
│  [frontend-1]  co-located│  [frontend-2] (no cache here)   │
└─────────────────────────────────────────────────────────────┘
```

```yaml
# frontend pod that wants to be near redis
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values: ["redis"]
          topologyKey: kubernetes.io/hostname  # same NODE

      # Or prefer (soft):
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: redis
            topologyKey: topology.kubernetes.io/zone  # same ZONE
```

---

## 🔀 podAntiAffinity — Spread Pods Apart

podAntiAffinity schedules a pod AWAY FROM nodes/zones that already have matching pods.

```
Use case: Spread 3 replicas of webapp across different nodes
  → If node goes down, other replicas survive on different nodes

WITHOUT antiAffinity:              WITH antiAffinity:
┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│ Node 1 │  │ Node 2 │  │ Node 3 │  │ Node 1 │  │ Node 2 │  │ Node 3 │
│[web-1] │  │[web-2] │  │        │  │[web-1] │  │[web-2] │  │[web-3] │
│[web-3] │  │        │  │        │  │        │  │        │  │        │
└────────┘  └────────┘  └────────┘  └────────┘  └────────┘  └────────┘
Node 1 dies → web-1 AND web-3 gone!   Node 1 dies → only web-1 gone ✅
```

```yaml
# webapp deployment - spread across nodes
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:

          # HARD: never put 2 webapp pods on same node
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values: ["webapp"]
              topologyKey: kubernetes.io/hostname  # per-node

          # SOFT: try to spread across zones
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: webapp
                topologyKey: topology.kubernetes.io/zone  # per-zone
```

---

## 🌍 Topology Keys — The Spread Dimension

The `topologyKey` defines the "boundary" for affinity rules:

```
topologyKey: kubernetes.io/hostname
  → The domain is individual NODES
  → "same node" or "different node"

topologyKey: topology.kubernetes.io/zone
  → The domain is availability ZONES (us-east-1a, us-east-1b)
  → "same zone" or "different zone"

topologyKey: topology.kubernetes.io/region
  → The domain is REGIONS (us-east-1, eu-west-1)
  → "same region" or "different region"

topologyKey: my-custom-label
  → Any label you define on nodes!
  → e.g., rack=rack-1 for physical rack topology

EXAMPLES:

podAntiAffinity + topologyKey: hostname
  → Each pod goes to a different NODE (max 1 per node)

podAntiAffinity + topologyKey: zone
  → Spread pods across ZONES (HA across AZs)

podAffinity + topologyKey: hostname
  → All pods go to SAME NODE (co-location)

podAffinity + topologyKey: zone
  → All pods go to SAME ZONE (same-zone latency)
```

---

## 📄 Complete Real-World Example

### High-Availability Web Application

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 6
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      affinity:
        # 1. HARD: Schedule on nodes with SSD storage
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: disk
                    operator: In
                    values: ["ssd"]

          # 2. SOFT: Prefer nodes in us-east-1a
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 50
              preference:
                matchExpressions:
                  - key: topology.kubernetes.io/zone
                    operator: In
                    values: ["us-east-1a"]

        # 3. SOFT: Spread pods across different zones
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: webapp
                topologyKey: topology.kubernetes.io/zone

          # 4. HARD: Never put 2 webapp pods on same node
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: webapp
              topologyKey: kubernetes.io/hostname

      containers:
        - name: webapp
          image: my-webapp:v1
```

---

## 🆚 Comparison: All Scheduling Controls

```
┌──────────────────┬─────────────────────────────────────────┐
│ Mechanism        │ Use For                                  │
├──────────────────┼─────────────────────────────────────────┤
│ nodeSelector     │ Simple label match on nodes (basic)     │
│ nodeAffinity     │ Advanced node selection with operators  │
│                  │ (In, NotIn, Exists, Gt, Lt)             │
│ podAffinity      │ Schedule pods NEAR other pods           │
│                  │ (same node/zone as specific pods)       │
│ podAntiAffinity  │ Schedule pods AWAY from other pods      │
│                  │ (spread replicas for HA)                │
├──────────────────┼─────────────────────────────────────────┤
│ required...      │ HARD — pod won't schedule if not met    │
│ preferred...     │ SOFT — scheduler tries but won't block  │
├──────────────────┼─────────────────────────────────────────┤
│ Taints           │ Node repels pods (push pods away)       │
│ Tolerations      │ Pod tolerates a node taint (allow in)   │
│                  │ (covered in next topic!)                │
└──────────────────┴─────────────────────────────────────────┘
```

---

## 💻 Commands Reference

```bash
# ── NODE LABELS ───────────────────────────────────────────
# Add label to node
kubectl label node node1 disk=ssd
kubectl label node node2 accelerator=gpu
kubectl label node node3 zone=us-east-1a

# View node labels
kubectl get nodes --show-labels
kubectl get nodes -l disk=ssd          # filter by label
kubectl get nodes -l 'disk in (ssd,nvme)'

# Remove label
kubectl label node node1 disk-         # trailing - removes label

# Update label
kubectl label node node1 disk=nvme --overwrite

# ── CHECK SCHEDULING ──────────────────────────────────────
# Why is pod pending?
kubectl describe pod my-pod
# Events:
# Warning  FailedScheduling  0/3 nodes are available:
#   1 node(s) didn't match pod's node affinity/selector

# Which nodes match my affinity?
kubectl get nodes -l disk=ssd,accelerator=gpu

# ── VERIFY PLACEMENT ──────────────────────────────────────
kubectl get pods -o wide          # shows which node each pod is on
kubectl get pods --field-selector spec.nodeName=node1
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| Required podAntiAffinity with replicas > nodes | Pod stays Pending — can't place more pods than nodes |
| topologyKey not matching any node label | All nodes treated as same topology → antiAffinity has no effect |
| Forgetting `required` blocks all if no match | Use `preferred` unless you truly need hard enforcement |
| nodeSelector AND nodeAffinity both set | BOTH must be satisfied (AND logic) |
| podAffinity labels match the pod itself | Pod won't schedule — it tries to find ANOTHER pod with same label |
| High weight values not properly balanced | Multiple preferred rules with same weight split evenly |

---

## 🎓 CKA Exam Tips

```
✅ nodeSelector = simplest, use when label match is exact

✅ nodeAffinity required = hard requirement:
   requiredDuringSchedulingIgnoredDuringExecution

✅ nodeAffinity preferred = soft preference (with weight 1-100):
   preferredDuringSchedulingIgnoredDuringExecution

✅ podAntiAffinity = spread pods for HA:
   Most common use: spread replicas across nodes or zones

✅ topologyKey for most cases:
   Per-node:  kubernetes.io/hostname
   Per-zone:  topology.kubernetes.io/zone

✅ If pod is Pending due to affinity:
   kubectl describe pod → Events section says exactly which rule failed

✅ Use kubectl explain for fields:
   kubectl explain pod.spec.affinity.nodeAffinity
   kubectl explain pod.spec.affinity.podAntiAffinity

✅ Label a node quickly for exam:
   kubectl label node <node-name> <key>=<value>
```

---

## ❓ Interview & Scenario Questions

### Q1: What is the difference between nodeSelector and nodeAffinity?
**Answer:**
`nodeSelector` is a simple key-value map that only supports exact equality matching — the node must have ALL the specified labels. It's easy to use but limited. `nodeAffinity` is more expressive with `matchExpressions` supporting operators like `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`. It also supports: (1) **required** rules (hard — pod won't schedule if not met) vs **preferred** rules (soft — scheduler tries but won't block), (2) OR logic between multiple `nodeSelectorTerms`, and (3) AND logic within a single term. If both nodeSelector and nodeAffinity are specified, both must be satisfied.

---

### Q2: What is the difference between podAffinity and podAntiAffinity?
**Answer:**
`podAffinity` schedules a pod **on the same** node/zone/topology domain as pods matching the selector — used for co-location to reduce latency (e.g., a frontend with its Redis cache on the same node). `podAntiAffinity` schedules a pod **away from** nodes/zones that already have pods matching the selector — used to spread replicas for high availability (e.g., ensuring 3 webapp replicas land on 3 different nodes so a single node failure doesn't kill all replicas). Both support `required` (hard) and `preferred` (soft) variants.

---

### Q3: Scenario — You have 3 replicas of a critical service and want to ensure they NEVER run on the same node. How do you implement this?
**Answer:**
Use `podAntiAffinity` with `required` rule and `topologyKey: kubernetes.io/hostname`:
```yaml
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: critical-service
              topologyKey: kubernetes.io/hostname
```
This hard rule ensures no two pods with label `app=critical-service` land on the same node. **Important caveat**: if you have more replicas than nodes, the extra pods will stay `Pending` — the scheduler cannot place them without violating the rule. For 3 replicas, you need at least 3 nodes.

---

### Q4: What is the `topologyKey` field and why is it important?
**Answer:**
`topologyKey` defines the **granularity of the domain** for affinity/anti-affinity rules. It references a node label, and all nodes with the same value for that label are considered the same "topology domain." For example: with `topologyKey: kubernetes.io/hostname`, each node is its own domain (since hostname is unique per node), so anti-affinity means "different nodes." With `topologyKey: topology.kubernetes.io/zone`, nodes in the same availability zone share a domain, so anti-affinity means "different zones." You can use any node label as a topology key — including custom ones for rack-aware scheduling.

---

### Q5: Scenario — Your ML training pods must run on GPU nodes, but you also want them to prefer nodes in the us-west zone. How do you configure this?
**Answer:**
Combine `required` for GPU (hard) with `preferred` for zone (soft):
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: accelerator
                operator: Exists        # must have any GPU label
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values: ["us-west-2a", "us-west-2b"]
```
The pod will ONLY schedule on GPU nodes (hard requirement), but among those, it will try to pick ones in us-west zones first (soft preference, weight=80).

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│        TOPIC 2.8 — NODE SELECTORS & AFFINITY                 │
├──────────────────────────────────────────────────────────────┤
│  nodeSelector    → simple label match (all = AND)            │
│  nodeAffinity    → advanced: In/NotIn/Exists operators       │
│                    required (hard) or preferred (soft)       │
│  podAffinity     → co-locate near matching pods              │
│  podAntiAffinity → spread away from matching pods            │
├──────────────────────────────────────────────────────────────┤
│  RULE TYPES:                                                 │
│  required...  → pod won't schedule if not met (HARD)         │
│  preferred... → scheduler tries but won't block (SOFT)       │
│                 has "weight" field (1-100)                   │
├──────────────────────────────────────────────────────────────┤
│  TOPOLOGY KEYS:                                              │
│  kubernetes.io/hostname          → per-node                  │
│  topology.kubernetes.io/zone     → per-AZ                    │
│  topology.kubernetes.io/region   → per-region                │
├──────────────────────────────────────────────────────────────┤
│  LOGIC:                                                      │
│  Multiple nodeSelectorTerms  → OR                            │
│  Multiple matchExpressions   → AND                           │
├──────────────────────────────────────────────────────────────┤
│  OPERATORS:                                                  │
│  In | NotIn | Exists | DoesNotExist | Gt | Lt                │
├──────────────────────────────────────────────────────────────┤
│  COMMON USE CASES:                                           │
│  GPU workloads   → nodeAffinity required: accelerator=gpu    │
│  HA replicas     → podAntiAffinity: spread across nodes/AZs  │
│  Low latency     → podAffinity: co-locate with cache/db      │
│  Env separation  → nodeAffinity: env=production nodes only   │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `17-taints-and-tolerations.md`
> *Node taints, pod tolerations, NoSchedule/NoExecute — the other side of scheduling control*
