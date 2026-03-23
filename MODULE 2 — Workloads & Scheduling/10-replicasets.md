# 📌 Topic 2.2 — ReplicaSets

> **Module 2 — Workloads & Scheduling**
> **File:** `10-replicasets.md`
> **CKA Weight:** ~15% (as part of Workloads) — Understanding RS is key for Deployments

---

## 🎯 What You'll Learn
- What a ReplicaSet is and the problem it solves
- How ReplicaSets use label selectors to own pods
- Creating, scaling, and deleting ReplicaSets
- How the ReplicaSet controller reconciliation loop works
- ReplicaSet vs ReplicationController (legacy)
- Why you almost never create ReplicaSets directly
- ReplicaSet relationship with Deployments
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Security Guard Analogy 🛡️
> A ReplicaSet is like a **security guard** at a venue with a strict headcount:
>
> - The guard has **orders**: "There must ALWAYS be exactly 3 people in this room"
> - The guard **watches the room constantly** (the reconciliation loop)
> - If someone **leaves** (pod crashes) → guard immediately calls for a replacement
> - If someone **sneaks in** (extra pod with matching label) → guard asks them to leave
> - The guard doesn't care WHO the people are — just that the COUNT matches
>   and they have the **right badge** (matching labels)
>
> This is ReplicaSet: it ensures exactly N replicas of a pod are always running,
> healing the cluster automatically when pods fail.

---

## 🔬 What is a ReplicaSet?

A ReplicaSet is a Kubernetes controller that **ensures a specified number of pod replicas are running at any given time**. It is the successor to ReplicationController and provides:

```
┌─────────────────────────────────────────────────────────────┐
│                      ReplicaSet                             │
│                                                             │
│  Desired replicas: 3                                        │
│  Selector: app=webapp                                       │
│                                                             │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐               │
│  │  Pod 1   │   │  Pod 2   │   │  Pod 3   │               │
│  │app=webapp│   │app=webapp│   │app=webapp│               │
│  │ Running  │   │ Running  │   │ Running  │               │
│  └──────────┘   └──────────┘   └──────────┘               │
│                                                             │
│  ReplicaSet Controller watches constantly:                  │
│  → Count running pods with selector: app=webapp             │
│  → If count < 3: CREATE new pods                            │
│  → If count > 3: DELETE excess pods                         │
│  → If count = 3: Do nothing ✅                              │
└─────────────────────────────────────────────────────────────┘
```

---

## 📄 ReplicaSet YAML — Full Anatomy

```yaml
apiVersion: apps/v1           # ← apps/v1 (NOT v1!)
kind: ReplicaSet
metadata:
  name: webapp-rs
  namespace: default
  labels:
    app: webapp               # labels on the ReplicaSet itself

spec:
  # ── HOW MANY REPLICAS ──────────────────────────────────
  replicas: 3                 # desired number of pods

  # ── WHICH PODS TO MANAGE ───────────────────────────────
  selector:
    matchLabels:
      app: webapp             # owns pods with THIS label
    # matchExpressions:       # alternative — more powerful
    #   - key: app
    #     operator: In
    #     values: [webapp, web-app]

  # ── POD TEMPLATE ───────────────────────────────────────
  # This is what new pods look like when RS creates them
  template:
    metadata:
      labels:
        app: webapp           # ⚠️ MUST match selector above!
        version: "1.0"        # extra labels OK
    spec:
      containers:
        - name: nginx
          image: nginx:1.21
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
```

### The CRITICAL Rule: Selector Must Match Template Labels

```
⚠️  IF selector does NOT match template labels → ReplicaSet creation FAILS

  selector:
    matchLabels:
      app: webapp          ← selector looks for this

  template:
    metadata:
      labels:
        app: webapp        ← pod must have this (MUST MATCH!)

  ✅ Valid: selector = template label (exact or subset)
  ❌ Invalid: selector has label not in template → rejected by apiserver
```

---

## ⚙️ ReplicaSet Controller — Reconciliation Loop

```
┌─────────────────────────────────────────────────────────────┐
│              RECONCILIATION LOOP (runs forever)             │
│                                                             │
│  1. OBSERVE                                                 │
│     Count pods matching selector: app=webapp                │
│     Current count = X                                       │
│                                                             │
│  2. COMPARE                                                 │
│     Desired = 3                                             │
│     Actual  = X                                             │
│                                                             │
│  3. ACT                                                     │
│     If X < 3: Create (3-X) new pods from template          │
│     If X > 3: Delete (X-3) excess pods                     │
│     If X = 3: No action needed ✅                           │
│                                                             │
│  4. REPEAT                                                  │
│     Go back to step 1                                       │
└─────────────────────────────────────────────────────────────┘

EXAMPLE SCENARIOS:

Scenario A: Pod crashes
  Actual: 3 → pod-1 crashes → Actual: 2
  RS detects: 2 < 3 → creates 1 new pod
  Actual: 3 ✅

Scenario B: Node goes down (2 pods on it)
  Actual: 3 → node dies → Actual: 1
  RS detects: 1 < 3 → creates 2 new pods on healthy nodes
  Actual: 3 ✅

Scenario C: Manual extra pod created with matching label
  Actual: 3 → someone creates pod with app=webapp → Actual: 4
  RS detects: 4 > 3 → DELETES 1 pod
  Actual: 3 ✅ (careful with manually created pods!)

Scenario D: Scale down to 1
  kubectl scale rs webapp-rs --replicas=1
  Actual: 3 → RS deletes 2 pods
  Actual: 1 ✅
```

---

## 🏷️ Label Selectors — How RS Owns Pods

This is the most important concept to understand about ReplicaSets:

```
OWNERSHIP IS DETERMINED BY LABELS — NOT by who created the pod!

  ReplicaSet selector: app=webapp

  These pods are ALL owned by this RS:
  ┌────────────────────┐  ┌────────────────────┐
  │ labels:            │  │ labels:             │
  │   app: webapp  ✅  │  │   app: webapp   ✅  │
  │   version: v1      │  │   version: v2       │
  └────────────────────┘  └────────────────────┘

  These pods are NOT owned:
  ┌────────────────────┐  ┌────────────────────┐
  │ labels:            │  │ labels:             │
  │   app: api     ❌  │  │   env: prod     ❌  │
  │   version: v1      │  │   (no app label)    │
  └────────────────────┘  └────────────────────┘
```

### matchExpressions — Advanced Selectors

```yaml
selector:
  matchExpressions:
    - key: app
      operator: In              # pod label must be IN this list
      values: [webapp, web-app]

    - key: env
      operator: NotIn           # pod label must NOT be in this list
      values: [test, dev]

    - key: tier
      operator: Exists          # label key must exist (any value)

    - key: legacy
      operator: DoesNotExist    # label key must NOT exist

# Operators: In | NotIn | Exists | DoesNotExist
```

---

## 💻 ReplicaSet Operations

```bash
# ── CREATE ────────────────────────────────────────────────
kubectl apply -f replicaset.yaml

# ── VIEW ──────────────────────────────────────────────────
kubectl get replicasets
kubectl get rs                    # shorthand

# Output:
# NAME        DESIRED   CURRENT   READY   AGE
# webapp-rs   3         3         3       5m

# Detailed view
kubectl describe rs webapp-rs

# See pods owned by this RS
kubectl get pods -l app=webapp
kubectl get pods --show-labels

# ── SCALE ─────────────────────────────────────────────────
# Imperative scale
kubectl scale rs webapp-rs --replicas=5

# Edit directly
kubectl edit rs webapp-rs
# → change spec.replicas value

# Patch (scriptable)
kubectl patch rs webapp-rs -p '{"spec":{"replicas":2}}'

# ── UPDATE IMAGE ──────────────────────────────────────────
# ⚠️  Changing the image in RS template does NOT update existing pods!
# Only NEW pods created after the change get the new image.
# This is why Deployments exist — they handle rolling updates!
kubectl edit rs webapp-rs
# change image → only affects future pods, NOT running ones

# ── DELETE ────────────────────────────────────────────────
# Delete RS AND its pods (default — cascade)
kubectl delete rs webapp-rs

# Delete RS but KEEP pods running (orphan)
kubectl delete rs webapp-rs --cascade=orphan
# Pods continue running but are no longer managed
```

---

## 🆚 ReplicaSet vs ReplicationController

ReplicationController (RC) is the OLD way — replaced by ReplicaSet.

```
┌────────────────────────────────────────────────────────────┐
│          ReplicationController (LEGACY — avoid!)           │
│                                                            │
│  apiVersion: v1                                            │
│  kind: ReplicationController                               │
│                                                            │
│  Selector: equality-based ONLY                             │
│  → selector: {app: webapp}                                 │
│  → Can only match exact key=value pairs                    │
│  → Cannot use In, NotIn, Exists operators                  │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│              ReplicaSet (CURRENT — use this!)              │
│                                                            │
│  apiVersion: apps/v1                                       │
│  kind: ReplicaSet                                          │
│                                                            │
│  Selector: set-based                                       │
│  → matchLabels (equality)                                  │
│  → matchExpressions (In, NotIn, Exists, DoesNotExist)      │
│  → More powerful and flexible                              │
└────────────────────────────────────────────────────────────┘

Key differences:
  RC:  apiVersion: v1           (core group)
  RS:  apiVersion: apps/v1      (apps group)

  RC:  selector: {app: webapp}  (simple equality only)
  RS:  selector: matchLabels + matchExpressions (powerful)

  RC:  OLD — still works but not recommended
  RS:  CURRENT — what Deployments use internally
```

---

## 🔗 ReplicaSet & Deployment Relationship

```
In practice: YOU create Deployments, Deployments create ReplicaSets

  kubectl apply -f deployment.yaml
         │
         ▼
    Deployment
    (revision 1)
         │
         │ creates
         ▼
    ReplicaSet v1        ← old RS (kept for rollback history)
    (image: nginx:1.20)
    replicas: 0          ← scaled to 0 when new RS takes over

    ReplicaSet v2        ← current active RS
    (image: nginx:1.21)
    replicas: 3

  When you do kubectl rollout undo:
  → Deployment scales RS v1 back to 3
  → Scales RS v2 to 0
  → Instant rollback! ✅

# See RSes created by a Deployment
kubectl get rs
# NAME                    DESIRED   CURRENT   READY   AGE
# webapp-deploy-7d9f8b   3         3         3       5m   ← active
# webapp-deploy-5c4d2a   0         0         0       2d   ← old (for rollback)

# RS name = deployment-name + pod-template-hash
# The hash changes whenever the pod template changes
```

---

## 🔍 Inspecting ReplicaSet Ownership

```bash
# See which RS owns a pod
kubectl get pod my-pod -o yaml | grep -A5 ownerReferences
# ownerReferences:
# - apiVersion: apps/v1
#   kind: ReplicaSet
#   name: webapp-rs
#   uid: abc123...
#   controller: true
#   blockOwnerDeletion: true

# See which Deployment owns a RS
kubectl get rs webapp-rs-7d9f8b -o yaml | grep -A5 ownerReferences
# ownerReferences:
# - apiVersion: apps/v1
#   kind: Deployment
#   name: webapp
#   uid: xyz456...
#   controller: true

# Full ownership chain:
# Deployment → ReplicaSet → Pod → Container
```

---

## ⚠️ The Pod Adoption Problem — Dangerous!

```
⚠️  WARNING: RS will ADOPT any pod matching its selector!

Scenario:
  1. You have a standalone pod with label: app=webapp (pod-manual)
  2. You create a RS with selector: app=webapp and replicas: 3
  3. RS sees: 1 pod already matches (pod-manual)
  4. RS only creates 2 new pods (not 3)
  5. pod-manual is now OWNED by the RS
  6. kubectl delete rs webapp-rs → deletes pod-manual too! 😱

Even worse:
  1. RS has replicas: 3 and is running fine
  2. You manually create a 4th pod with label: app=webapp
  3. RS sees 4 matching pods → 4 > 3
  4. RS DELETES one pod (maybe your manually created one)

Lesson: Never manually create pods with labels matching an RS selector!

# Fix an accidentally adopted pod:
# Change the pod's label to break the match:
kubectl label pod pod-manual app=webapp-orphaned --overwrite
# Now RS doesn't count this pod → creates a new one
# The old pod runs free, not managed by anyone
```

---

## 🎓 CKA Exam Tips

```
✅ ReplicaSet apiVersion = apps/v1 (not v1!)
   kubectl api-resources | grep ReplicaSet

✅ selector.matchLabels MUST match template.metadata.labels
   Missing or mismatched → apiserver rejects with error

✅ RS doesn't do rolling updates — Deployment does
   If asked to update an image: use Deployment, not RS directly

✅ Scaling:
   kubectl scale rs <name> --replicas=<n>

✅ Quickly check what pods a RS manages:
   kubectl get pods -l <selector-key>=<selector-value>

✅ If a pod has the same labels as an RS selector:
   The RS will adopt it — be careful!

✅ RS is created automatically by Deployments
   In CKA you often don't create RS directly — you'll use Deployments
   But you need to READ and understand RS when debugging

✅ Delete RS without deleting pods:
   kubectl delete rs <name> --cascade=orphan
```

---

## ❓ Interview & Scenario Questions

### Q1: What is a ReplicaSet and how does it differ from a ReplicationController?
**Answer:**
A ReplicaSet (RS) is a Kubernetes controller that ensures a specified number of pod replicas are running at all times. It uses label selectors to identify which pods it manages and the reconciliation loop to create/delete pods to match the desired count. The key difference from ReplicationController (the legacy equivalent) is the selector: ReplicationController only supports equality-based selectors (`app: webapp`), while ReplicaSet supports set-based selectors with `matchLabels` and `matchExpressions` (using operators like In, NotIn, Exists, DoesNotExist). ReplicaSet is also the mechanism Deployments use internally, making it the current standard.

---

### Q2: If you change the image in a ReplicaSet template, what happens to existing pods?
**Answer:**
Nothing happens to existing pods. The template change only affects NEW pods that the ReplicaSet creates in the future. If a current pod crashes and the RS replaces it, the new pod will use the updated image. But running pods are NOT restarted or updated. This is the fundamental limitation of ReplicaSets and the main reason Deployments exist — Deployments manage multiple ReplicaSets to perform rolling updates where all pods gradually get the new image.

---

### Q3: Scenario — You have a RS with replicas=3. You manually create a 4th pod with a matching label. What happens?
**Answer:**
The ReplicaSet controller sees 4 pods matching its selector, but desired is 3 — so it will delete 1 pod. It picks a pod to delete (usually the most recently created one, but not guaranteed). This is why you should never manually create pods with labels matching an active ReplicaSet selector. The RS considers label-matching pods as its own regardless of who created them.

---

### Q4: How does a ReplicaSet know which pods it "owns"?
**Answer:**
Through two mechanisms working together: first, the label selector — the RS controller watches all pods matching its `selector.matchLabels` (or `matchExpressions`). Second, `ownerReferences` — when the RS creates a pod, it sets an `ownerReferences` entry in the pod's metadata pointing to the RS. This creates a bidirectional link: the RS can find its pods via selector, and each pod knows its owner via ownerReferences. The garbage collector uses ownerReferences to clean up pods when their RS is deleted.

---

### Q5: When would you ever create a ReplicaSet directly instead of using a Deployment?
**Answer:**
Almost never in practice. Deployments are almost always preferred because they provide rolling updates, rollback history, and revision management — all of which require managing multiple ReplicaSets. The main rare cases for a standalone RS: when you need a specific pod count guaranteed without any rolling update capability, or when using a custom update mechanism. Even then, most engineers use Deployments with `strategy.type: Recreate` instead of bare ReplicaSets. The CKA exam may ask you to CREATE and SCALE a ReplicaSet directly, so you must know the YAML, but in production Deployments are the standard.

---

### Q6: Scenario — You delete a Deployment. Do its ReplicaSets and pods get deleted too?
**Answer:**
Yes, by default. Kubernetes uses cascade deletion — deleting a Deployment deletes its owned ReplicaSets, which deletes their owned Pods. This chain works through `ownerReferences`. To keep pods running while deleting the Deployment use `--cascade=orphan`:
```bash
kubectl delete deployment my-app --cascade=orphan
```
The RS and pods continue running but are now "orphaned" — no controller manages them. If a pod crashes, it won't be replaced. You can also orphan just the RS from the Deployment, leaving pods still managed by the RS.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│               TOPIC 2.2 — REPLICASETS                        │
├──────────────────────────────────────────────────────────────┤
│  PURPOSE: Ensure exactly N pod replicas run at all times     │
│  apiVersion: apps/v1                                         │
│  Shorthand: rs                                               │
├──────────────────────────────────────────────────────────────┤
│  SELECTOR TYPES:                                             │
│  matchLabels       → simple key=value equality               │
│  matchExpressions  → In | NotIn | Exists | DoesNotExist      │
├──────────────────────────────────────────────────────────────┤
│  CRITICAL RULE:                                              │
│  selector.matchLabels MUST match template.metadata.labels    │
├──────────────────────────────────────────────────────────────┤
│  RECONCILIATION:                                             │
│  count < desired → create pods                               │
│  count > desired → delete pods                               │
│  count = desired → do nothing                                │
├──────────────────────────────────────────────────────────────┤
│  KEY COMMANDS:                                               │
│  kubectl get rs                                              │
│  kubectl describe rs <name>                                  │
│  kubectl scale rs <name> --replicas=<n>                      │
│  kubectl delete rs <name> --cascade=orphan  (keep pods)      │
├──────────────────────────────────────────────────────────────┤
│  RS vs RC:                                                   │
│  RC (legacy)  → v1, equality selectors only                  │
│  RS (current) → apps/v1, set-based selectors                 │
├──────────────────────────────────────────────────────────────┤
│  ⚠️  GOTCHAS:                                                │
│  → Changing RS image template doesn't update running pods    │
│  → RS ADOPTS any pod with matching labels (careful!)         │
│  → In production always use Deployment, not bare RS          │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `11-deployments.md`
> *Rolling updates, rollback strategies, revision history — the most used workload object*
