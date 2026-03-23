# 📌 Topic 2.3 — Deployments

> **Module 2 — Workloads & Scheduling**
> **File:** `11-deployments.md`
> **CKA Weight:** 🔥 HIGH — Deployments are the most used workload object in CKA

---

## 🎯 What You'll Learn
- What a Deployment is and why it exists
- Full Deployment YAML anatomy
- Rolling update strategy — how zero-downtime updates work
- Recreate strategy — when to use it
- How to update, rollback, and check rollout status
- Revision history and how rollbacks work internally
- Pausing and resuming rollouts
- Deployment scaling strategies
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Software Release Manager Analogy 🎬
> A Deployment is like a **professional release manager** for your application:
>
> - You hand the manager a **release plan** (Deployment YAML)
> - The manager handles ALL the details:
>   - Spinning up new instances of the new version
>   - Gradually shifting traffic to the new version
>   - Keeping the old version around until the new one is proven stable
>   - Rolling back INSTANTLY if something goes wrong
>   - Keeping a history of every past release
>
> Without a release manager (bare ReplicaSet):
> - You'd have to manually create new pods, manually delete old ones
> - One mistake = downtime
> - No rollback capability
>
> With a Deployment: one command → zero-downtime update → instant rollback if needed ✅

---

## 🔬 What is a Deployment?

A Deployment is a higher-level abstraction that manages ReplicaSets to provide:

```
┌─────────────────────────────────────────────────────────────┐
│                      DEPLOYMENT                             │
│                                                             │
│  ✅ Declarative updates (rolling updates)                   │
│  ✅ Zero-downtime deployments                               │
│  ✅ Automatic rollback on failure                           │
│  ✅ Revision history (configurable)                         │
│  ✅ Pause & resume rollouts                                 │
│  ✅ Scaling                                                 │
│                                                             │
│  Manages →  ReplicaSet v1 (old, replicas=0)                 │
│             ReplicaSet v2 (current, replicas=3)             │
│             ReplicaSet v3 (being rolled out...)             │
└─────────────────────────────────────────────────────────────┘
```

---

## 📄 Deployment YAML — Full Anatomy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: production
  labels:
    app: webapp
  annotations:
    kubernetes.io/change-cause: "Update nginx to 1.21"  # shows in rollout history

spec:
  # ── REPLICA COUNT ───────────────────────────────────────
  replicas: 3

  # ── POD SELECTOR ────────────────────────────────────────
  selector:
    matchLabels:
      app: webapp                  # must match template labels

  # ── UPDATE STRATEGY ─────────────────────────────────────
  strategy:
    type: RollingUpdate            # RollingUpdate | Recreate

    rollingUpdate:
      maxSurge: 1                  # max pods ABOVE desired during update
                                   # can be number or percentage: 25%
      maxUnavailable: 0            # max pods BELOW desired during update
                                   # can be number or percentage: 25%
                                   # 0 = no downtime (always have 3 ready)

  # ── REVISION HISTORY ────────────────────────────────────
  revisionHistoryLimit: 10         # how many old RS to keep (for rollback)
                                   # default: 10

  # ── PROGRESS DEADLINE ───────────────────────────────────
  progressDeadlineSeconds: 600     # fail if rollout takes longer than 10min

  # ── MIN READY SECONDS ───────────────────────────────────
  minReadySeconds: 10              # pod must be Ready for 10s before
                                   # counting as available (prevents flapping)

  # ── POD TEMPLATE ────────────────────────────────────────
  template:
    metadata:
      labels:
        app: webapp                # must match selector above
        version: "1.21"
      annotations:
        prometheus.io/scrape: "true"
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
          readinessProbe:          # IMPORTANT: RS waits for this before
            httpGet:               # marking pod as available during rollout
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 3
```

---

## 🔄 Update Strategies — Deep Dive

### Strategy 1: RollingUpdate (Default)

Zero-downtime updates. Gradually replaces old pods with new ones.

```
ROLLING UPDATE with maxSurge=1, maxUnavailable=0, replicas=3:

Initial state:
  [v1] [v1] [v1]   total=3, available=3

Step 1: Create 1 new pod (surge)
  [v1] [v1] [v1] [v2]   total=4, available=3 (v2 not ready yet)

Step 2: v2 becomes ready
  [v1] [v1] [v1] [v2✅]  total=4, available=4

Step 3: Delete 1 old pod
  [v1] [v1] [v2✅]        total=3, available=3

Step 4: Create another v2
  [v1] [v1] [v2✅] [v2]   total=4, available=3

Step 5: v2 becomes ready
  [v1] [v1] [v2✅] [v2✅]  total=4, available=4

Step 6: Delete 1 old pod
  [v1] [v2✅] [v2✅]        total=3, available=3

... continues until all pods are v2 ...

Final state:
  [v2] [v2] [v2]   total=3, available=3 ✅

Key: Always maintained 3 available pods (maxUnavailable=0)
     Never had more than 4 total pods (replicas + maxSurge=1)
```

### maxSurge and maxUnavailable Explained

```
replicas: 3

maxSurge: 1         → max pods during update = 3 + 1 = 4
maxUnavailable: 0   → min pods always available = 3 - 0 = 3
                    → SAFEST: no downtime, slightly more resources needed

maxSurge: 25%       → max pods during update = ceil(3 * 1.25) = 4
maxUnavailable: 25% → min pods available = floor(3 * 0.75) = 2
                    → BALANCED: some risk, uses fewer extra resources

maxSurge: 0         → never exceeds 3 pods total
maxUnavailable: 1   → 1 pod can be down at a time
                    → RESOURCE SAVING: no extra capacity needed

maxSurge: 3         → can go to 6 pods temporarily (fast update)
maxUnavailable: 3   → all pods can be replaced simultaneously
                    → FASTEST: essentially recreate but in one shot
```

### Strategy 2: Recreate

Kills ALL old pods, then creates ALL new pods. Causes downtime!

```yaml
strategy:
  type: Recreate
  # No rollingUpdate section needed
```

```
RECREATE UPDATE:

Initial:
  [v1] [v1] [v1]    all running

Step 1: Kill ALL v1 pods
  [   ] [   ] [   ]  ← DOWNTIME! ❌

Step 2: Create ALL v2 pods
  [v2] [v2] [v2]    all running again

When to use Recreate:
  → App cannot have 2 versions running simultaneously
  → Database schema changes that break old version
  → Stateful apps that can't run in parallel
  → Dev/test environments where downtime is acceptable
```

---

## 🚀 Deployment Operations — Complete Reference

### Create and Update

```bash
# ── CREATE ────────────────────────────────────────────────
# Imperative
kubectl create deployment webapp --image=nginx:1.20 --replicas=3

# Declarative
kubectl apply -f deployment.yaml

# Generate YAML
kubectl create deployment webapp \
  --image=nginx:1.20 \
  --replicas=3 \
  --dry-run=client -o yaml > deployment.yaml

# ── UPDATE IMAGE (triggers rolling update) ────────────────
kubectl set image deployment/webapp nginx=nginx:1.21
# Format: deployment/<name> <container-name>=<new-image>

# Multiple containers at once
kubectl set image deployment/webapp \
  nginx=nginx:1.21 \
  sidecar=fluentd:v1.14

# Via edit (interactive)
kubectl edit deployment webapp
# Change image field → save → rollout starts automatically

# Via apply (declarative — recommended)
# Edit image in deployment.yaml → kubectl apply -f deployment.yaml
```

### Monitor Rollout Status

```bash
# Watch rollout progress
kubectl rollout status deployment/webapp
# Output:
# Waiting for deployment "webapp" rollout to finish:
#   1 out of 3 new replicas have been updated...
#   2 out of 3 new replicas have been updated...
#   3 out of 3 new replicas have been updated...
# deployment "webapp" successfully rolled out

# Check rollout history
kubectl rollout history deployment/webapp
# REVISION  CHANGE-CAUSE
# 1         kubectl create deployment webapp --image=nginx:1.20
# 2         Update nginx to 1.21
# 3         Update nginx to 1.22

# See details of a specific revision
kubectl rollout history deployment/webapp --revision=2
# Shows the full pod template for that revision

# Watch pods during rollout
kubectl get pods -w
```

### Rollback

```bash
# Rollback to previous version
kubectl rollout undo deployment/webapp

# Rollback to specific revision
kubectl rollout undo deployment/webapp --to-revision=2

# Check status after rollback
kubectl rollout status deployment/webapp
kubectl get rs    # see which RS is now active
```

### Pause and Resume

```bash
# PAUSE a rollout (make multiple changes without triggering multiple rollouts)
kubectl rollout pause deployment/webapp

# Make changes while paused (no rollout triggered)
kubectl set image deployment/webapp nginx=nginx:1.22
kubectl set resources deployment/webapp \
  -c nginx --limits=cpu=500m,memory=256Mi
# ^ These changes accumulate but don't roll out yet

# RESUME — triggers ONE rollout with all accumulated changes
kubectl rollout resume deployment/webapp
```

### Scale

```bash
# Scale deployment
kubectl scale deployment webapp --replicas=5

# Autoscale (creates HPA)
kubectl autoscale deployment webapp --min=2 --max=10 --cpu-percent=70
```

---

## 🔍 How Rollouts Work Internally

```
DEPLOYMENT UPDATE INTERNALS:

Before update:
  Deployment
    └── ReplicaSet-v1 (nginx:1.20, replicas=3)
          ├── Pod-1 [v1]
          ├── Pod-2 [v1]
          └── Pod-3 [v1]

kubectl set image deployment/webapp nginx=nginx:1.21

  Deployment detects pod template changed
  (sha256 hash of template changed → new revision)
         │
         ▼
  Deployment creates NEW ReplicaSet-v2 (nginx:1.21, replicas=0)
         │
         ▼
  Rolling update begins:
  Deployment scales RS-v2 UP   (creates new pods)
  Deployment scales RS-v1 DOWN (deletes old pods)
  (respecting maxSurge and maxUnavailable)
         │
         ▼
After update:
  Deployment
    ├── ReplicaSet-v1 (nginx:1.20, replicas=0)  ← kept for rollback!
    └── ReplicaSet-v2 (nginx:1.21, replicas=3)  ← current
          ├── Pod-4 [v2]
          ├── Pod-5 [v2]
          └── Pod-6 [v2]

kubectl rollout undo:
  Deployment scales RS-v1 UP to 3
  Deployment scales RS-v2 DOWN to 0
  Instant rollback! ✅
```

---

## 🔖 Revision History & Change Cause

```bash
# Add change cause to rollout history:

# Method 1: kubectl annotation
kubectl annotate deployment webapp \
  kubernetes.io/change-cause="Upgrade nginx to 1.21 for security patch"

# Method 2: In YAML metadata
metadata:
  annotations:
    kubernetes.io/change-cause: "Upgrade nginx to 1.21 for security patch"

# Then apply changes
kubectl apply -f deployment.yaml

# Now history shows:
kubectl rollout history deployment/webapp
# REVISION  CHANGE-CAUSE
# 1         Initial deployment with nginx:1.20
# 2         Upgrade nginx to 1.21 for security patch
# 3         Add sidecar logging container

# Control how many revisions to keep:
spec:
  revisionHistoryLimit: 5    # default=10, set lower to save etcd space
```

---

## 📊 Deployment Status Conditions

```bash
kubectl describe deployment webapp

# Look for Conditions:
# Conditions:
#   Type           Status   Reason
#   Available      True     MinimumReplicasAvailable
#   Progressing    True     NewReplicaSetAvailable (or ReplicaSetUpdated)

# Bad conditions:
# Progressing    False    ProgressDeadlineExceeded
#   → Rollout took too long (> progressDeadlineSeconds)
#   → Fix: kubectl rollout undo deployment/webapp

# Available      False    MinimumReplicasUnavailable
#   → Fewer pods available than desired
#   → Check pod events for crash/image pull issues
```

---

## 🛡️ Deployment with Readiness Probe — Why It Matters

```
Without readiness probe:
  → K8s marks pod as "ready" as soon as container starts
  → Traffic sent to pod before app is actually ready
  → Users see errors during rollout! ❌

With readiness probe:
  → K8s waits until probe succeeds before marking pod ready
  → Pod NOT added to Service endpoints until truly ready
  → Old pods continue serving traffic until new pods are ready
  → True zero-downtime rollout ✅

  spec:
    containers:
      - name: nginx
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 10   # wait 10s before first check
          periodSeconds: 5          # check every 5s
          successThreshold: 1       # 1 success = ready
          failureThreshold: 3       # 3 failures = not ready

  Rolling update flow WITH readiness probe:
  1. Create new pod (v2)
  2. Wait for readiness probe to pass (pod is truly ready)
  3. ONLY THEN mark as available
  4. ONLY THEN delete an old pod (v1)
  → Zero downtime guaranteed ✅
```

---

## 💻 Full Deployment Workflow Example

```bash
# 1. Create deployment
kubectl create deployment webapp \
  --image=nginx:1.20 \
  --replicas=3 \
  --dry-run=client -o yaml > webapp-deploy.yaml

# 2. Add strategy and readiness probe, then apply
kubectl apply -f webapp-deploy.yaml

# 3. Check deployment
kubectl get deploy webapp
kubectl get pods -l app=webapp
kubectl get rs

# 4. Update image
kubectl set image deployment/webapp nginx=nginx:1.21
kubectl annotate deployment webapp \
  kubernetes.io/change-cause="Upgrade to nginx 1.21"

# 5. Watch the rollout
kubectl rollout status deployment/webapp

# 6. Verify new pods have new image
kubectl describe pods | grep Image

# 7. Oops — something is wrong! Rollback
kubectl rollout undo deployment/webapp

# 8. Check rollout history
kubectl rollout history deployment/webapp

# 9. Scale up for traffic spike
kubectl scale deployment webapp --replicas=10

# 10. Scale back down
kubectl scale deployment webapp --replicas=3
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| Editing pod directly in a Deployment | Controllers revert it — edit the Deployment spec instead |
| No readiness probe = not zero-downtime | Without readiness probe, K8s sends traffic to unready pods |
| `revisionHistoryLimit: 0` | No rollback possible! Keep at least 3-5 |
| `maxUnavailable: 0, maxSurge: 0` | Invalid — at least one must be non-zero |
| `kubectl apply` without `--record` | Use annotations for change-cause instead (--record is deprecated) |
| Deployment stuck in `Progressing` | Check pod events — usually ImagePullBackOff or failing readiness probe |
| Forgetting `progressDeadlineSeconds` | Default 600s — if rollout takes longer, deployment is marked Failed |

---

## 🎓 CKA Exam Tips

```
✅ Most common Deployment tasks in CKA:
   1. Create a Deployment with specific image + replicas
   2. Update (set image) and check rollout status
   3. Rollback to previous or specific revision
   4. Scale a deployment

✅ Quick create:
   kubectl create deployment <n> --image=<img> --replicas=<n>

✅ Update image:
   kubectl set image deployment/<n> <container>=<new-image>

✅ Check rollout:
   kubectl rollout status deployment/<n>

✅ Rollback:
   kubectl rollout undo deployment/<n>
   kubectl rollout undo deployment/<n> --to-revision=<n>

✅ History:
   kubectl rollout history deployment/<n>
   kubectl rollout history deployment/<n> --revision=<n>

✅ If rollout is stuck:
   kubectl describe deployment <n>   → check Conditions
   kubectl get pods                  → check pod status
   kubectl describe pod <pod>        → check Events

✅ Rolling vs Recreate:
   RollingUpdate → zero downtime (default)
   Recreate      → downtime, for apps that can't run 2 versions
```

---

## ❓ Interview & Scenario Questions

### Q1: How does a Deployment achieve zero-downtime updates?
**Answer:**
A Deployment achieves zero-downtime updates through the RollingUpdate strategy combined with readiness probes. It creates a new ReplicaSet for the new version and gradually scales it up while scaling down the old ReplicaSet. The `maxUnavailable` parameter (default 25%) controls how many pods can be unavailable at once, and `maxSurge` (default 25%) controls how many extra pods can exist above the desired count. With readiness probes configured, Kubernetes only marks a new pod as "available" after its readiness check passes — meaning it only removes an old pod after the new one is truly ready to serve traffic, guaranteeing zero downtime.

---

### Q2: What happens internally when you run `kubectl rollout undo`?
**Answer:**
Kubernetes doesn't recreate anything — it simply scales existing ReplicaSets. The previous ReplicaSet (which was scaled to 0 replicas) gets scaled back up to the desired count, and the current ReplicaSet gets scaled down to 0. This is why rollbacks are instant — the old pods were never deleted from etcd, their ReplicaSet was just scaled to 0. The number of kept old ReplicaSets is controlled by `revisionHistoryLimit` (default: 10).

---

### Q3: Scenario — Your deployment is stuck in a rolling update. How do you investigate and fix it?
**Answer:**
```bash
# Check deployment status
kubectl rollout status deployment/webapp
# "Waiting for deployment... 1 out of 3..."

# Check deployment conditions
kubectl describe deployment webapp
# Look for: ProgressDeadlineExceeded or False conditions

# Check pod status
kubectl get pods -l app=webapp
# Look for: ImagePullBackOff, CrashLoopBackOff, Pending

# Check specific pod events
kubectl describe pod <new-pod-name>
# Find root cause

# Fix options:
# 1. Fix the issue (correct image name, fix app bug, add resources)
kubectl set image deployment/webapp nginx=nginx:1.21-fixed

# 2. Rollback immediately
kubectl rollout undo deployment/webapp
```

---

### Q4: What is the difference between `maxSurge` and `maxUnavailable`?
**Answer:**
`maxSurge` defines how many pods ABOVE the desired replica count can exist during a rolling update (extra capacity). For example, with replicas=3 and maxSurge=1, up to 4 pods can exist at once. `maxUnavailable` defines how many pods BELOW the desired count can be unavailable during the update. With maxUnavailable=0, there are always at least 3 ready pods — no downtime. Setting both to 0 is invalid. The defaults (25% each) balance speed and availability. For zero-downtime: set maxUnavailable=0. For faster rollout: increase maxSurge.

---

### Q5: When would you use `Recreate` strategy instead of `RollingUpdate`?
**Answer:**
Use `Recreate` when your application cannot run two versions simultaneously. Common scenarios: database schema migrations that are not backward-compatible (old app version would break on new schema), apps that acquire exclusive locks or shared state, stateful apps that require a clean stop before restart, or situations where version conflicts cause data corruption. Recreate terminates ALL old pods before creating new ones — causing downtime but ensuring only one version ever runs at a time.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│               TOPIC 2.3 — DEPLOYMENTS                        │
├──────────────────────────────────────────────────────────────┤
│  PURPOSE: Managed rolling updates + rollback for pods        │
│  apiVersion: apps/v1                                         │
│  Manages: ReplicaSets (which manage Pods)                    │
├──────────────────────────────────────────────────────────────┤
│  STRATEGIES:                                                 │
│  RollingUpdate → zero downtime (default)                     │
│    maxSurge:       extra pods above desired (default 25%)    │
│    maxUnavailable: pods below desired allowed (default 25%)  │
│  Recreate      → all-at-once, causes downtime                │
├──────────────────────────────────────────────────────────────┤
│  KEY COMMANDS:                                               │
│  kubectl create deployment <n> --image=<i> --replicas=<r>    │
│  kubectl set image deployment/<n> <c>=<img>                  │
│  kubectl rollout status deployment/<n>                       │
│  kubectl rollout history deployment/<n>                      │
│  kubectl rollout undo deployment/<n>                         │
│  kubectl rollout undo deployment/<n> --to-revision=<r>       │
│  kubectl rollout pause/resume deployment/<n>                 │
│  kubectl scale deployment <n> --replicas=<r>                 │
├──────────────────────────────────────────────────────────────┤
│  ZERO DOWNTIME RECIPE:                                       │
│  1. Set maxUnavailable: 0                                    │
│  2. Add readinessProbe to containers                         │
│  3. Set minReadySeconds (e.g. 10)                            │
│  4. Keep revisionHistoryLimit ≥ 3                            │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `12-daemonsets.md`
> *One pod per node — logging agents, monitoring, networking use cases*
