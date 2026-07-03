# 📌 Topic 5.8 — Pod Disruption Budgets (PDB)

> **Module 5 — Configuration & Secrets**
> **File:** `42-pod-disruption-budgets.md`
> **CKA Weight:** ~10% (Configuration) — PDB appears in cluster maintenance + drain tasks

---

## 🎯 What You'll Learn
- What Pod Disruption Budgets are and the problem they solve
- Voluntary vs involuntary disruptions
- minAvailable vs maxUnavailable — choosing the right field
- PDB with percentage vs absolute counts
- How PDB protects pods during `kubectl drain`
- PDB with Deployments, StatefulSets, and other controllers
- PDB status — current, desired, allowed disruptions
- Real-world PDB patterns for production
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Surgery Analogy 🏥
> Imagine a hospital with 5 surgeons on call:
>
> The hospital administrator needs to close Operating Room 2
> for maintenance (drain a node), but must ensure patient safety.
>
> **Without a PDB:**
> Administrator says "clear the room!" — all surgeons in OR2
> are told to leave immediately.
> If all 5 were in OR2, suddenly ZERO surgeons are available → crisis!
>
> **With a PDB (minAvailable: 3):**
> Administrator can only clear the room IF at least 3 surgeons
> remain available elsewhere.
> - If 4 surgeons are elsewhere → can move 1 from OR2 ✅
> - If 2 surgeons are elsewhere → BLOCKED: can't risk going below 3 ❌
>
> **PDB = a safety contract:**
> "No matter what maintenance you're doing,
> ALWAYS keep at least N pods (surgeons) available."

---

## 🔬 Voluntary vs Involuntary Disruptions

```
DISRUPTIONS COME IN TWO TYPES:

INVOLUNTARY DISRUPTIONS (PDB has NO control):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  → Hardware failures (node dies suddenly)
  → Kernel panics
  → Cloud provider preempting VM (spot/preemptible)
  → Power loss
  → Out of resources on node (OOM, disk full)
  → Network partition

  PDB can NOT prevent these — they're unplanned!
  But controllers (Deployment, StatefulSet) will recreate pods.

VOLUNTARY DISRUPTIONS (PDB CONTROLS these):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  → kubectl drain <node>        (cluster administrator)
  → kubectl delete pod <pod>    (accidental or deliberate)
  → Updating a Deployment       (rolling update)
  → kubectl rollout restart     (operator-initiated restart)
  → Node autoscaler scaling down nodes
  → Cluster upgrades
  → Any eviction via Eviction API

  PDB gates ALL voluntary evictions through the Eviction API!

HOW PDB WORKS:
  Tool wants to evict pod X
         │
         ▼
  Calls Eviction API (not direct pod delete!)
         │
         ▼
  API server checks: Does a PDB select pod X?
  YES → Check: Would this eviction violate PDB?
         │
    ┌────┴────┐
    ▼         ▼
  SAFE      VIOLATION
  ✅        ❌
  Allow     Return 429 Too Many Requests
  eviction  (retry later)
```

---

## 📄 PDB YAML — Full Anatomy

```yaml
apiVersion: policy/v1                   # v1 since K8s 1.21 (was v1beta1)
kind: PodDisruptionBudget
metadata:
  name: webapp-pdb
  namespace: production                 # PDB is NAMESPACED!

spec:
  # ── SELECTOR (which pods does this PDB protect?) ────────
  selector:
    matchLabels:
      app: webapp                       # protects pods with app=webapp
    # matchExpressions also supported

  # ── AVAILABILITY GUARANTEE (choose ONE of these) ────────

  # Option A: minAvailable — minimum pods that must remain UP
  minAvailable: 2                       # absolute count
  # OR
  minAvailable: "50%"                   # percentage of desired replicas
  # (rounded UP — e.g., 50% of 5 = 3 must stay available)

  # Option B: maxUnavailable — maximum pods allowed to be DOWN
  maxUnavailable: 1                     # absolute count
  # OR
  maxUnavailable: "25%"                 # percentage of desired
  # (rounded DOWN — e.g., 25% of 5 = 1 can be unavailable)

  # ⚠️  CANNOT set BOTH minAvailable AND maxUnavailable!
  # Choose exactly one.
```

---

## 🔢 minAvailable vs maxUnavailable

```
SCENARIO: Deployment with 5 replicas

┌──────────────────┬────────────────────────────────────────────────┐
│ PDB Setting      │ Effect                                         │
├──────────────────┼────────────────────────────────────────────────┤
│ minAvailable: 3  │ At least 3 pods must be available.             │
│                  │ Max 2 pods can be evicted simultaneously.       │
│                  │ If 3 pods available → allow eviction           │
│                  │ If only 3 available → BLOCK eviction           │
│                  │ ← USE when you know min healthy count needed   │
├──────────────────┼────────────────────────────────────────────────┤
│ minAvailable:"60%│ 60% of 5 = 3 (rounded up) must stay.          │
│                  │ Same effect as minAvailable: 3 for 5 replicas  │
│                  │ ← USE with percentage for flexible scaling     │
├──────────────────┼────────────────────────────────────────────────┤
│ maxUnavailable:1 │ At most 1 pod can be unavailable at any time.  │
│                  │ 5-1=4 pods must be available.                  │
│                  │ More restrictive than minAvailable: 3!         │
│                  │ ← USE when you know max tolerable outage       │
├──────────────────┼────────────────────────────────────────────────┤
│ maxUnavailable:  │ 25% of 5 = 1 (rounded down) can be gone.       │
│ "25%"            │ Same as maxUnavailable: 1 for 5 replicas       │
└──────────────────┴────────────────────────────────────────────────┘

CHOOSING BETWEEN THEM:

  minAvailable is better when:
  → You know the MINIMUM count needed for service (e.g., quorum)
  → You have DB clusters (MySQL: must keep 2 replicas minimum)
  → StatefulSets with leader election

  maxUnavailable is better when:
  → You want to limit the blast radius (max 1 pod down at once)
  → Rolling updates should not disrupt more than N pods
  → Mirrors Deployment rollingUpdate maxUnavailable

  minAvailable: 0 = PDB exists but allows everything (useful for auditing)
  maxUnavailable: 0 = ZERO tolerance (nothing can be evicted!) ← dangerous!
```

---

## 🛡️ PDB During kubectl drain

```
SCENARIO: Admin drains node1 for maintenance
  kubectl drain node1 --ignore-daemonsets

  Node1 has:
  webapp-pod-1 (selected by webapp-pdb, minAvailable: 2)
  webapp-pod-2 (selected by webapp-pdb, minAvailable: 2)
  webapp-pod-3 (selected by webapp-pdb, minAvailable: 2)
  nginx-pod    (not selected by any PDB)
  daemonset-pod (ignored due to --ignore-daemonsets)

  Node2 has:
  webapp-pod-4 ← available
  webapp-pod-5 ← available

  Currently available: pod-4 + pod-5 = 2 pods (meets minAvailable: 2)

  Drain process:
  Step 1: Try to evict webapp-pod-1
    Available without pod-1: pod-2 + pod-4 + pod-5 = 3 >= 2 ✅
    → Eviction ALLOWED, pod-1 evicted and rescheduled on node2

  Wait for pod-1 to reschedule...
  Now node2 has: pod-4, pod-5, pod-1 (new)

  Step 2: Try to evict webapp-pod-2
    Available without pod-2: pod-3 + pod-4 + pod-5 + pod-1 = 4 >= 2 ✅
    → Eviction ALLOWED

  Step 3: Try to evict webapp-pod-3
    Available without pod-3: pod-4 + pod-5 + pod-1 + pod-2 = 4 >= 2 ✅
    → Eviction ALLOWED

  Step 4: Evict nginx-pod (no PDB)
    → Eviction ALLOWED immediately (no PDB protection)

  Drain complete ✅ — zero downtime!

  BLOCKED SCENARIO:
  If only 2 pods were running across all nodes:
  Step 1: Try to evict webapp-pod-1
    Available without pod-1: pod-2 = 1 < minAvailable:2 ❌
    → 429 Too Many Requests → drain WAITS or FAILS
    kubectl drain output: "cannot evict pod ... : pod disruption budget allows 0 disruptions"
```

---

## 📊 PDB Status — Reading the Numbers

```bash
kubectl get pdb -n production
# NAME         MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# webapp-pdb   2               N/A               3                     5d
# db-pdb       N/A             1                 1                     5d

# ALLOWED DISRUPTIONS = how many pods can be evicted RIGHT NOW
# = current replicas - minAvailable (or based on maxUnavailable)

kubectl describe pdb webapp-pdb -n production
# Name:             webapp-pdb
# Namespace:        production
# Min available:    2
# Selector:         app=webapp
# Status:
#     Observed generation:  1
#     Disruptions allowed:  3      ← 5 running - 2 minAvailable = 3
#     Current pods:         5      ← total pods matching selector
#     Desired pods:         5      ← desired replicas
#     Total replicas:       5

# IF disruptions allowed = 0:
# → kubectl drain will BLOCK on this pod
# → Must fix: scale up replicas, fix unhealthy pods, or delete PDB

# Common scenario: disruptions allowed = 0
# Causes:
# 1. Already at minAvailable (some pods crashed)
# 2. maxUnavailable: 0 was set
# 3. Pods in non-Running state (not counted as available)
```

---

## 🌍 Real-World PDB Patterns

### Pattern 1: Web Application (High Availability)

```yaml
# 3+ replicas, max 1 down at a time
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: webapp-pdb
  namespace: production
spec:
  selector:
    matchLabels:
      app: webapp
  maxUnavailable: 1        # at most 1 pod down at any time
  # With 5 replicas: 4 always available
  # With 10 replicas: 9 always available (scales with deployment!)
```

### Pattern 2: Database Cluster (Quorum-Based)

```yaml
# MySQL with 3 replicas: need quorum (2 of 3) to function
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: mysql-pdb
  namespace: production
spec:
  selector:
    matchLabels:
      app: mysql
  minAvailable: 2          # always keep 2 for quorum
  # Even if 3rd crashes: 2 remain → reads still work
  # If drain would take below 2: BLOCKED
```

### Pattern 3: Kafka Brokers (Replication Factor)

```yaml
# Kafka with 5 brokers, replication factor 3
# Can survive losing up to 2 brokers (RF-1)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: kafka-pdb
  namespace: kafka
spec:
  selector:
    matchLabels:
      app: kafka
  minAvailable: 3          # keep at least 3 for replication safety
  # maxUnavailable: 2      # equivalent for 5 brokers
```

### Pattern 4: Critical Single Instance (Soft Protection)

```yaml
# Single-replica app that can tolerate brief downtime
# But protect against accidental mass deletion
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-app-pdb
  namespace: production
spec:
  selector:
    matchLabels:
      app: critical-scheduler
  minAvailable: 0          # allows 0 available (single pod can be drained)
  # Still useful: prevents SIMULTANEOUS disruption of multiple matching pods
  # if someone accidentally selects and deletes all
```

### Pattern 5: Stateless App (Percentage-Based)

```yaml
# Elastic auto-scaling app — 10-50 replicas
# Keep at least 80% available regardless of scale
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: api-pdb
  namespace: production
spec:
  selector:
    matchLabels:
      app: api-server
  minAvailable: "80%"      # works for any replica count!
  # 10 replicas: 8 must be available
  # 50 replicas: 40 must be available
  # Scales automatically with your Deployment! ✅
```

---

## 🔗 PDB Interaction with Cluster Operations

```
kubectl drain:
  → Uses Eviction API → checked against PDB
  → Respects PDB: waits or fails if violation would occur
  → --force flag BYPASSES PDB (⚠️ use with extreme caution!)
  → --disable-eviction flag BYPASSES PDB (very dangerous!)

Cluster Autoscaler:
  → Uses Eviction API → checked against PDB
  → Will NOT scale down a node if doing so would violate PDB
  → Pod annotation: cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
    (prevents autoscaler from evicting that specific pod)

kubectl delete pod:
  → Direct delete BYPASSES PDB (does not use Eviction API!)
  → PDB only protects against Eviction API-based removals
  → ⚠️  kubectl delete pod = NOT protected by PDB!

Rolling Updates (Deployment):
  → Controlled by Deployment's own maxUnavailable
  → PDB adds ADDITIONAL constraint (both must be satisfied)
  → If PDB is more restrictive: PDB wins
  → Best practice: align Deployment maxUnavailable with PDB settings

kubectl rollout restart:
  → Uses rolling update → respects both Deployment and PDB constraints
  → Safe for controlled restarts ✅
```

---

## 💻 Commands Reference

```bash
# ── CREATE ────────────────────────────────────────────────
kubectl apply -f pdb.yaml

# Quick imperative (limited options):
kubectl create poddisruptionbudget webapp-pdb \
  --selector=app=webapp \
  --min-available=2 \
  --namespace=production

# ── VIEW ──────────────────────────────────────────────────
kubectl get pdb -n production
kubectl get poddisruptionbudgets -n production  # long form
kubectl get pdb -A                              # all namespaces

# Output:
# NAME        MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS
# webapp-pdb  2               N/A               3
# db-pdb      N/A             1                 1

kubectl describe pdb webapp-pdb -n production
# Shows: selector, status, allowed disruptions, current/desired counts

# ── CHECK DISRUPTIONS ALLOWED ────────────────────────────
kubectl get pdb -n production -o wide
# ALLOWED DISRUPTIONS = how many pods can be evicted now
# 0 = drain will be blocked!

# ── TEST PDB (simulate eviction) ─────────────────────────
# Use the eviction API directly:
kubectl proxy &   # start proxy
curl -X POST http://localhost:8001/api/v1/namespaces/production/pods/webapp-abc123/eviction \
  -H "Content-Type: application/json" \
  -d '{"apiVersion":"policy/v1","kind":"Eviction","metadata":{"name":"webapp-abc123","namespace":"production"}}'
# 200 = eviction allowed
# 429 = eviction blocked by PDB

# ── DELETE ────────────────────────────────────────────────
kubectl delete pdb webapp-pdb -n production
# After deletion: no PDB protection — drain proceeds without check
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| PDB with no matching pods | PDB exists but protects nothing — check selector matches pod labels |
| minAvailable > replicas | ALWAYS blocked — impossible to drain any node! |
| maxUnavailable: 0 | Nothing can EVER be evicted — cluster maintenance impossible! |
| `kubectl delete pod` bypasses PDB | PDB only covers Eviction API — direct deletes ignore PDB |
| Single replica + minAvailable: 1 | ALWAYS blocked — can never drain the node running that pod |
| Not accounting for already-unavailable pods | If 1 pod is already crashed, disruptions allowed decreases |

---

## 🎓 CKA Exam Tips

```
✅ PDB apiVersion: policy/v1  (NOT v1beta1 — that's deprecated!)

✅ Create PDB quickly:
   kubectl create pdb webapp-pdb \
     --selector=app=webapp \
     --min-available=2 \
     -n production

✅ Check disruptions allowed (before draining!):
   kubectl get pdb -n <ns>
   → ALLOWED DISRUPTIONS column
   → 0 = drain will be blocked

✅ PDB blocks kubectl drain:
   kubectl drain node1 --ignore-daemonsets
   → If PDB violation: "cannot evict pod... pod disruption budget"
   → Fix: scale up replicas OR delete PDB temporarily

✅ minAvailable vs maxUnavailable:
   minAvailable → minimum pods that MUST be up
   maxUnavailable → maximum pods that CAN be down
   CANNOT set both! Choose one.

✅ Percentage works: "50%", "80%"
   → Scales automatically with replica count

✅ kubectl delete pod BYPASSES PDB
   Only Eviction API is checked against PDB

✅ PDB shorthand: pdb
   kubectl get pdb -n <ns>
   kubectl describe pdb <name> -n <ns>
```

---

## ❓ Interview & Scenario Questions

### Q1: What is a Pod Disruption Budget and what does it protect against?
**Answer:**
A Pod Disruption Budget (PDB) is a policy that limits the number of pods that can be voluntarily disrupted simultaneously. It protects against voluntary disruptions — those initiated by cluster administrators or automation, such as `kubectl drain` for node maintenance, cluster upgrades, node autoscaler scale-downs, and rolling updates. It does NOT protect against involuntary disruptions like hardware failures, kernel panics, or cloud provider preemptions. PDB works by gating the Kubernetes Eviction API — any eviction request is checked against applicable PDBs before being allowed, ensuring the service's availability guarantees are maintained during planned maintenance.

---

### Q2: What is the difference between `minAvailable` and `maxUnavailable` in a PDB?
**Answer:**
`minAvailable` specifies the minimum number (or percentage) of pods that must remain Available at all times. Evictions are blocked when they would drop the count below this minimum. Use when you know the minimum healthy count your service needs (e.g., quorum for a database cluster). `maxUnavailable` specifies the maximum number (or percentage) of pods that can be Unavailable at any time. Evictions are blocked when the unavailable count already equals or exceeds this limit. Use when you want to limit the blast radius of disruptions. They cannot both be set — choose one based on your availability model. Percentage values scale automatically with replica count changes.

---

### Q3: Scenario — `kubectl drain node1` gets stuck with "cannot evict pod ... pod disruption budget." How do you fix it?
**Answer:**
The drain is blocked because evicting a pod would violate a PDB. Diagnostic and fix:
```bash
# Step 1: Find which PDB is blocking
kubectl get pdb -n <namespace>
# Look for ALLOWED DISRUPTIONS = 0

# Step 2: Check why disruptions = 0
kubectl describe pdb <pdb-name> -n <namespace>
# Current pods: 2, Min available: 2, Allowed disruptions: 0
# → All pods at minimum!

# Fix options:
# Option A: Scale up the Deployment temporarily
kubectl scale deployment webapp --replicas=5 -n production
# Now disruptions allowed = 3, drain can proceed
kubectl drain node1 --ignore-daemonsets
# Then scale back down

# Option B: Delete the PDB temporarily (risky!)
kubectl delete pdb webapp-pdb -n production
kubectl drain node1 --ignore-daemonsets
kubectl apply -f webapp-pdb.yaml   # restore after drain

# Option C (DANGEROUS): Force bypass with --force
kubectl drain node1 --ignore-daemonsets --force --disable-eviction
# This COMPLETELY ignores PDB — may cause service outage!
# Only use in emergencies!
```

---

### Q4: Why does `kubectl delete pod` bypass PDB while `kubectl drain` respects it?
**Answer:**
PDB protection is implemented in the **Eviction API** (`/api/v1/namespaces/<ns>/pods/<pod>/eviction`), not in the pod deletion API. `kubectl drain` and the cluster autoscaler use the Eviction API specifically so PDB protection is applied. `kubectl delete pod` uses the direct pod deletion API (`DELETE /api/v1/namespaces/<ns>/pods/<pod>`) which does not check PDBs — it immediately deletes the pod. This is an important operational distinction: accidental `kubectl delete pod` commands can bypass your safety guarantees. For safety-critical pods, consider also using `kubectl` RBAC restrictions to prevent direct pod deletion, relying only on controller-managed rolling updates and drain for pod lifecycle management.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│         TOPIC 5.8 — POD DISRUPTION BUDGETS (PDB)             │
├──────────────────────────────────────────────────────────────┤
│  PURPOSE: Protect pods during VOLUNTARY disruptions          │
│  apiVersion: policy/v1                                       │
│  Scope: Namespaced   Shorthand: pdb                          │
├──────────────────────────────────────────────────────────────┤
│  VOLUNTARY DISRUPTIONS (PDB protects):                       │
│  kubectl drain, cluster autoscaler, rolling updates,         │
│  kubectl delete pod (NO! Direct delete bypasses PDB!)        │
│  ← Only Eviction API is checked against PDB!                 │
├──────────────────────────────────────────────────────────────┤
│  FIELDS (choose ONE):                                        │
│  minAvailable: N | "X%"  → min pods that must stay UP        │
│  maxUnavailable: N | "X%"→ max pods that can be DOWN         │
│  Cannot set BOTH!                                            │
├──────────────────────────────────────────────────────────────┤
│  PDB STATUS:                                                 │
│  kubectl get pdb → ALLOWED DISRUPTIONS column                │
│  0 = drain BLOCKED → scale up or temporarily delete PDB      │
├──────────────────────────────────────────────────────────────┤
│  COMMON PATTERNS:                                            │
│  Web app:       maxUnavailable: 1                            │
│  DB cluster:    minAvailable: 2 (quorum)                     │
│  Elastic scale: minAvailable: "80%" (scales with replicas)   │
│  Zero-downtime: minAvailable: replicas-1                     │
├──────────────────────────────────────────────────────────────┤
│  KEY COMMANDS:                                               │
│  kubectl create pdb <n> --selector=k=v --min-available=N     │
│  kubectl get pdb -n <ns>                                     │
│  kubectl describe pdb <n> -n <ns>  → disruptions allowed     │
└──────────────────────────────────────────────────────────────┘
```

---

> 🎉 **MODULE 5 COMPLETE!**
> All 8 Configuration & Secrets topics finished!
>
> **Next Module →** Module 6 — Security
> **First topic →** `43-kubernetes-security-overview.md`
> *4C security model: Cloud, Cluster, Container, Code*
