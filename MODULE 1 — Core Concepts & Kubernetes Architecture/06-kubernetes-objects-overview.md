# 📌 Topic 1.6 — Kubernetes Objects Overview

> **Module 1 — Core Concepts & Kubernetes Architecture**
> **File:** `06-kubernetes-objects-overview.md`
> **CKA Weight:** ~10% — Foundation for ALL workload topics in Module 2+

---

## 🎯 What You'll Learn
- What a Kubernetes Object is (the core abstraction)
- Desired State vs Actual State — the K8s philosophy
- The complete map of Kubernetes object types
- Pod, ReplicaSet, Deployment, Service — how they relate
- Object anatomy — apiVersion, kind, metadata, spec, status
- Labels, Annotations, and Selectors — how objects connect
- Namespaced vs Cluster-scoped resources
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Blueprint Analogy 🏗️
> Kubernetes objects are like **construction blueprints**:
>
> - A **blueprint** (object spec) describes what you WANT built
> - The **construction company** (Kubernetes controllers) reads it and builds it
> - Once built, the **inspector** (controllers again) constantly checks
>   the real building matches the blueprint
> - If the real building drifts from the blueprint (a pod crashes),
>   the construction company fixes it automatically
>
> You never say "go pour concrete now" — you say
> "I want a 3-floor building" and K8s figures out the how.

---

## 🧱 What is a Kubernetes Object?

A Kubernetes object is a **persistent record** in the cluster that represents a piece of desired state. When you create an object, you're telling the cluster: *"This is what I want to exist."* Kubernetes then works continuously to make reality match that desired state.

```
┌────────────────────────────────────────────────────────────┐
│                  KUBERNETES OBJECT                         │
│                                                            │
│  Every object has:                                         │
│                                                            │
│  ┌─────────────┐    ┌─────────────────────────────────┐   │
│  │ Desired     │    │ Actual State                    │   │
│  │ State       │    │ (what's really running)         │   │
│  │ (your spec) │    │                                 │   │
│  └──────┬──────┘    └────────────────┬────────────────┘   │
│         │                            │                    │
│         └──────────► DIFF ◄──────────┘                    │
│                        │                                  │
│                        ▼                                  │
│               Controllers act to                          │
│               close the gap                               │
│                                                            │
│  Key insight: You declare WHAT, K8s decides HOW           │
└────────────────────────────────────────────────────────────┘
```

---

## 📐 Anatomy of a Kubernetes Object (YAML)

Every Kubernetes object YAML has exactly these 4 top-level fields:

```yaml
# ── 1. apiVersion ─────────────────────────────────────────
# Which API group and version handles this object type
apiVersion: apps/v1

# ── 2. kind ───────────────────────────────────────────────
# What TYPE of object this is
kind: Deployment

# ── 3. metadata ───────────────────────────────────────────
# Identity information about the object
metadata:
  name: my-webapp              # unique name in its namespace
  namespace: production         # which namespace it lives in
  labels:                       # key-value tags for selecting/grouping
    app: webapp
    tier: frontend
    env: production
  annotations:                  # non-identifying metadata (notes, tools)
    deployment.kubernetes.io/revision: "3"
    contact: "team-frontend@company.com"

# ── 4. spec ───────────────────────────────────────────────
# The DESIRED STATE — what you want
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: nginx
          image: nginx:1.21
          ports:
            - containerPort: 80

# ── 5. status (managed by K8s — do NOT set this) ──────────
# The ACTUAL STATE — what currently exists
# Kubernetes fills this in automatically
status:
  availableReplicas: 3
  readyReplicas: 3
  replicas: 3
  updatedReplicas: 3
```

### The 4 Fields Rule
```
apiVersion  → Which API handles this? (required)
kind        → What type of object?    (required)
metadata    → Who is this object?     (required — at minimum: name)
spec        → What do you want?       (required for most objects)
status      → What actually exists?   (managed by K8s — never set manually)
```

---

## 🗺️ The Complete Kubernetes Object Map

```
KUBERNETES OBJECTS
══════════════════════════════════════════════════════════════

WORKLOADS (run your applications)
  ├── Pod                → Smallest deployable unit (1+ containers)
  ├── ReplicaSet         → Ensures N pod replicas always running
  ├── Deployment         → Manages ReplicaSets, rolling updates
  ├── StatefulSet        → Ordered pods with stable IDs & storage
  ├── DaemonSet          → One pod per node (logging, monitoring)
  ├── Job                → Run a task to completion once
  └── CronJob            → Scheduled Jobs (like Linux cron)

NETWORKING
  ├── Service            → Stable endpoint to reach a set of pods
  ├── Ingress            → HTTP routing rules (L7 load balancer)
  ├── NetworkPolicy      → Firewall rules for pod traffic
  └── EndpointSlice      → Tracks pod IPs behind a Service

CONFIGURATION
  ├── ConfigMap          → Non-sensitive config data
  └── Secret             → Sensitive data (passwords, tokens)

STORAGE
  ├── PersistentVolume   → A piece of actual storage (admin-created)
  ├── PersistentVolumeClaim → A request for storage (by a pod)
  └── StorageClass       → Template for dynamic volume provisioning

SECURITY
  ├── ServiceAccount     → Identity for pods (not human users)
  ├── Role               → Permissions in a namespace
  ├── ClusterRole        → Permissions cluster-wide
  ├── RoleBinding        → Assign Role to a user/SA in a namespace
  └── ClusterRoleBinding → Assign ClusterRole to user/SA globally

CLUSTER
  ├── Namespace          → Virtual clusters for isolation
  ├── Node               → A worker machine in the cluster
  ├── LimitRange         → Default limits per namespace
  ├── ResourceQuota      → Max resources per namespace
  └── PodDisruptionBudget → Min available pods during disruptions

SCALING
  └── HorizontalPodAutoscaler → Auto-scale pods based on metrics
```

---

## 🔗 How Core Objects Relate — The Hierarchy

```
                        DEPLOYMENT
                     (manages updates)
                            │
                   creates and manages
                            │
                            ▼
                       REPLICASET
                    (maintains replica count)
                            │
                   creates and manages
                            │
               ┌────────────┼────────────┐
               ▼            ▼            ▼
             POD           POD          POD
           (runs          (runs        (runs
          container)    container)   container)
               │            │            │
               └────────────┼────────────┘
                            │
                     selected by labels
                            │
                            ▼
                         SERVICE
                    (stable network access
                     to the pods above)
                            │
                            ▼
                          INGRESS
                    (HTTP routing, TLS,
                     external access)
```

### Why This Hierarchy Exists

```
You create:  DEPLOYMENT
             ├── Keeps revision history for rollback
             ├── Manages rolling update strategy
             └── Creates → REPLICASET
                           ├── Ensures exact replica count
                           └── Creates → PODS
                                         └── Runs your containers

You DON'T create pods directly in production because:
  ❌ If a pod dies → it's gone forever (no controller to recreate it)
  ❌ No rolling updates
  ❌ No rollback
  ❌ No replica management

Always use Deployment (or StatefulSet/DaemonSet) in production!
```

---

## 🏷️ Labels, Selectors & Annotations — How Objects Connect

### Labels
```yaml
# Labels are key-value pairs used to:
# 1. Identify and organize objects
# 2. Select groups of objects (via selectors)
# 3. Filter objects in kubectl

metadata:
  labels:
    app: webapp           # what app is this?
    tier: frontend        # which tier?
    env: production       # which environment?
    version: "1.2.3"      # which version?
    team: platform        # which team owns this?

# Rules:
# → Key max length: 63 chars
# → Value max length: 63 chars
# → Can use prefix: app.kubernetes.io/name: webapp
```

### Selectors — How Objects "Find" Each Other

```yaml
# ReplicaSet selects its pods via labels:
spec:
  selector:
    matchLabels:
      app: webapp         # find all pods with app=webapp
  template:
    metadata:
      labels:
        app: webapp       # make sure created pods have this label

# Service selects its pods via labels:
spec:
  selector:
    app: webapp           # route traffic to pods with app=webapp

# CRITICAL: selector and pod labels MUST MATCH or nothing works!

# Two types of selectors:
# 1. Equality-based (simple)
   app=webapp
   env!=production

# 2. Set-based (more powerful)
   app in (webapp, api)
   tier notin (database)
   env exists
```

### Annotations
```yaml
# Annotations store non-identifying metadata
# NOT used for selecting objects
# Used for: tools, operators, documentation

metadata:
  annotations:
    # Kubernetes own annotations
    kubectl.kubernetes.io/last-applied-configuration: "..."
    deployment.kubernetes.io/revision: "3"

    # Tool integrations
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"

    # Human documentation
    team: "platform-team"
    contact: "devops@company.com"
    docs: "https://wiki.company.com/webapp"
    last-updated: "2026-01-15"
```

---

## 🌐 Namespaced vs Cluster-Scoped Resources

```
NAMESPACED resources (exist inside a namespace):
  pods, deployments, services, configmaps, secrets,
  replicasets, statefulsets, daemonsets, jobs, cronjobs,
  ingresses, networkpolicies, serviceaccounts,
  roles, rolebindings, persistentvolumeclaims,
  horizontalpodautoscalers, poddisruptionbudgets

CLUSTER-SCOPED resources (exist globally):
  nodes, namespaces, persistentvolumes, storageclasses,
  clusterroles, clusterrolebindings,
  customresourcedefinitions (CRDs)

# Check which resources are namespaced:
kubectl api-resources --namespaced=true
kubectl api-resources --namespaced=false
```

### The 4 Default Namespaces

```
default          → Where resources go if no namespace specified
                   Use for development, NOT production

kube-system      → Kubernetes system components
                   (apiserver, etcd, scheduler, coredns, kube-proxy)
                   Never put your app here!

kube-public      → Readable by everyone including unauthenticated
                   Contains cluster-info configmap

kube-node-lease  → Node heartbeat Lease objects
                   One per node, updated every second
                   Helps control plane detect node failures faster
```

---

## 📦 Object Ownership & Garbage Collection

```
When a ReplicaSet is deleted → its pods get deleted too
When a Deployment is deleted → its ReplicaSets AND pods get deleted

This is because of OWNER REFERENCES:

Pod YAML has:
  ownerReferences:
    - apiVersion: apps/v1
      kind: ReplicaSet
      name: webapp-rs-7d9f8b
      uid: abc123...
      controller: true
      blockOwnerDeletion: true

When ReplicaSet is deleted:
  → Garbage collector finds all objects with ownerRef pointing to it
  → Deletes them automatically (cascading delete)

CASCADE DELETE OPTIONS:
  kubectl delete deployment my-app                  # default: cascades
  kubectl delete deployment my-app --cascade=false  # orphan pods (keep running)
  kubectl delete deployment my-app --cascade=orphan # same
```

---

## 🔑 apiVersion Reference — Which to Use

```
apiVersion: v1               → Core API group
  Resources: Pod, Service, ConfigMap, Secret, Namespace,
             Node, PersistentVolume, PersistentVolumeClaim,
             ServiceAccount, Endpoints, Events, LimitRange,
             ResourceQuota

apiVersion: apps/v1          → Apps group
  Resources: Deployment, ReplicaSet, StatefulSet, DaemonSet

apiVersion: batch/v1         → Batch group
  Resources: Job, CronJob

apiVersion: networking.k8s.io/v1   → Networking
  Resources: Ingress, NetworkPolicy

apiVersion: rbac.authorization.k8s.io/v1   → RBAC
  Resources: Role, ClusterRole, RoleBinding, ClusterRoleBinding

apiVersion: storage.k8s.io/v1     → Storage
  Resources: StorageClass

apiVersion: autoscaling/v2        → Autoscaling
  Resources: HorizontalPodAutoscaler

# Quick lookup:
kubectl api-resources | grep Deployment
# NAME          SHORTNAMES   APIVERSION   NAMESPACED   KIND
# deployments   deploy       apps/v1      true         Deployment
```

---

## 💻 Hands-on — Explore Objects

```bash
# See all available object types
kubectl api-resources

# Get object documentation inline
kubectl explain pod
kubectl explain deployment.spec.strategy
kubectl explain service.spec.type

# See a full object in YAML (learn the structure)
kubectl get deployment my-app -o yaml

# See status vs spec
kubectl get pod my-pod -o jsonpath='{.spec}'    # what you wanted
kubectl get pod my-pod -o jsonpath='{.status}'  # what actually exists

# View owner references
kubectl get pod my-pod -o jsonpath='{.metadata.ownerReferences}'

# Label operations
kubectl get pods --show-labels                    # show all labels
kubectl get pods -l app=webapp                    # filter by label
kubectl get pods -l 'env in (prod, staging)'      # set-based selector
kubectl label pod my-pod version=v2               # add label
kubectl label pod my-pod version-                 # remove label (note the -)

# Annotation operations
kubectl annotate pod my-pod team="platform"       # add annotation
kubectl describe pod my-pod | grep Annotations    # view annotations
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| Creating bare pods in production | Always use Deployment — bare pods don't self-heal |
| Wrong `apiVersion` for a resource | Check `kubectl api-resources` — wrong version = `no matches for kind` error |
| Label selector not matching pod labels | Service won't route to any pods — always verify selector matches pod template labels |
| Editing `status` in YAML | Status is managed by K8s — any changes you make are ignored |
| Deleting a ReplicaSet thinking pods survive | By default, deleting a RS cascades to delete its pods |
| Thinking annotations affect scheduling/selection | Only **labels** are used for selection — annotations are just metadata |

---

## 🎓 CKA Exam Tips

```
✅ Know the 4 top-level fields: apiVersion, kind, metadata, spec
✅ Know which apiVersion to use for common objects:
   Pod/Service/ConfigMap/Secret → v1
   Deployment/DaemonSet/StatefulSet → apps/v1
   Job/CronJob → batch/v1
   Ingress/NetworkPolicy → networking.k8s.io/v1
   Role/ClusterRole → rbac.authorization.k8s.io/v1

✅ Use kubectl explain when you forget a field name:
   kubectl explain deployment.spec.template.spec.containers

✅ Labels vs Annotations:
   Labels  → selecting, filtering, grouping (functional)
   Annotations → storing metadata for tools and humans (informational)

✅ If a Service can't reach pods:
   → Check selector vs pod labels match exactly
   kubectl get endpoints my-service
   → Empty endpoints = selector doesn't match any pods

✅ Always check ownerReferences to understand deletion behavior
✅ kube-system namespace → system pods, never touch in exam unless asked
```

---

## ❓ Interview & Scenario Questions

### Q1: What is a Kubernetes object and what is the difference between spec and status?
**Answer:**
A Kubernetes object is a persistent record of the cluster's desired state. Every object has two key sections: **spec** (the desired state — what YOU declare) and **status** (the actual state — what Kubernetes updates automatically to reflect reality). Controllers constantly compare spec vs status and take action to reconcile any differences. You should never manually set the `status` field — Kubernetes owns it.

---

### Q2: Why should you use a Deployment instead of creating pods directly?
**Answer:**
Creating bare pods means if the pod crashes, it's gone — no controller will recreate it. A Deployment provides: automatic pod recreation via ReplicaSet, rolling updates with zero downtime, rollback to previous versions, scaling, and revision history. In production, you almost never create pods directly — you always use Deployment (stateless apps), StatefulSet (stateful apps), DaemonSet (node-level agents), or Job/CronJob (batch work).

---

### Q3: What are labels and why are they important?
**Answer:**
Labels are key-value pairs attached to Kubernetes objects. They serve as the primary mechanism for objects to select and relate to each other. A Service uses a label selector to find which pods to route traffic to. A ReplicaSet uses a selector to find which pods it "owns." Without correct label matching, services can't reach pods and replica controllers can't manage pods. Labels are also used by kubectl to filter resources. Key distinction: labels are for selection (functional), while annotations are for metadata (informational).

---

### Q4: Scenario — You created a Service but it's not routing traffic to your pods. What's the first thing you check?
**Answer:**
Check if the Service selector matches the pod labels:
```bash
# Check Service selector
kubectl describe service my-svc | grep Selector
# Selector: app=webapp

# Check pod labels
kubectl get pods --show-labels
# NAME        LABELS
# my-pod-1    app=web-app  ← MISMATCH! webapp vs web-app

# Verify by checking Endpoints
kubectl get endpoints my-svc
# NAME     ENDPOINTS   AGE
# my-svc   <none>      5m  ← no endpoints = selector doesn't match
```
Fix: either update the Service selector OR update the pod labels to match.

---

### Q5: What is the difference between a namespaced resource and a cluster-scoped resource? Give examples.
**Answer:**
Namespaced resources exist within a namespace and are isolated to it — examples: Pod, Deployment, Service, ConfigMap, Secret, Role, RoleBinding, PVC. They require `-n namespace` to access across namespaces. Cluster-scoped resources exist globally across all namespaces — examples: Node, Namespace, PersistentVolume, StorageClass, ClusterRole, ClusterRoleBinding. They don't belong to any namespace and are accessible cluster-wide.

---

### Q6: Scenario — You delete a Deployment. What happens to the ReplicaSet and pods?
**Answer:**
By default, Kubernetes uses **cascade deletion**. Deleting a Deployment automatically deletes its ReplicaSets, which then deletes all the pods. This happens through `ownerReferences` — each pod has an ownerReference to its ReplicaSet, and each ReplicaSet has an ownerReference to the Deployment. The garbage collector follows this chain and deletes everything. If you want to keep the pods running while deleting the Deployment, use `kubectl delete deployment my-app --cascade=orphan` — the pods will keep running but are now "orphaned" (no controller managing them).

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│          TOPIC 1.6 — KUBERNETES OBJECTS OVERVIEW             │
├──────────────────────────────────────────────────────────────┤
│  EVERY OBJECT HAS:                                           │
│  apiVersion → which API group/version                        │
│  kind       → what type of object                            │
│  metadata   → name, namespace, labels, annotations           │
│  spec       → desired state (YOU write this)                 │
│  status     → actual state  (K8s writes this)                │
├──────────────────────────────────────────────────────────────┤
│  OBJECT HIERARCHY:                                           │
│  Deployment → ReplicaSet → Pod → Container                   │
├──────────────────────────────────────────────────────────────┤
│  COMMON apiVersions:                                         │
│  v1           → Pod, Service, ConfigMap, Secret, NS, PV, PVC │
│  apps/v1      → Deployment, ReplicaSet, StatefulSet, DS      │
│  batch/v1     → Job, CronJob                                 │
│  networking/v1→ Ingress, NetworkPolicy                       │
│  rbac.../v1   → Role, ClusterRole, RoleBinding               │
├──────────────────────────────────────────────────────────────┤
│  LABELS vs ANNOTATIONS:                                      │
│  Labels      → selection, filtering, grouping (functional)   │
│  Annotations → tools, docs, metadata (informational only)    │
├──────────────────────────────────────────────────────────────┤
│  NAMESPACED vs CLUSTER-SCOPED:                               │
│  Namespaced: Pod, Deploy, Svc, CM, Secret, Role, PVC         │
│  Cluster:    Node, Namespace, PV, StorageClass, ClusterRole  │
├──────────────────────────────────────────────────────────────┤
│  DEFAULT NAMESPACES:                                         │
│  default      → your resources (dev)                         │
│  kube-system  → K8s system components                        │
│  kube-public  → publicly readable data                       │
│  kube-node-lease → node heartbeat objects                    │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `07-namespaces.md`
> *Namespace isolation, resource boundaries, cross-namespace communication*
