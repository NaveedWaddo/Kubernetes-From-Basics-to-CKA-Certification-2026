# 📌 Topic 1.7 — Namespaces

> **Module 1 — Core Concepts & Kubernetes Architecture**
> **File:** `07-namespaces.md`
> **CKA Weight:** ~10% — Namespace operations appear in many CKA tasks

---

## 🎯 What You'll Learn
- What namespaces are and why they exist
- The 4 default namespaces and their purpose
- Creating, switching, and deleting namespaces
- Resource isolation and cross-namespace communication
- ResourceQuota and LimitRange per namespace
- Namespace-scoped vs cluster-scoped resources
- Real-world namespace strategies (by team, env, app)
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Office Building Analogy 🏢
> Think of a Kubernetes cluster as a **large office building**:
>
> - The **building** = the Kubernetes cluster
> - Each **floor/office suite** = a Namespace
> - **Tenants on each floor** = pods and resources in that namespace
>
> Each office suite has:
> - Its own **rooms** (pods) with their own names
> - Its own **budget** (ResourceQuota)
> - Its own **rules** (RBAC policies)
> - **Locked doors** between suites (network isolation via NetworkPolicy)
>
> Two companies can both have an employee named "John" (pod named "nginx")
> — they don't conflict because they're on different floors (namespaces).
> But the building's infrastructure (elevators, power) = cluster-scoped
> resources that serve everyone.

---

## 🗺️ What Namespaces Provide

```
┌─────────────────────────────────────────────────────────────┐
│                  KUBERNETES CLUSTER                         │
│                                                             │
│  ┌───────────────────┐   ┌───────────────────┐             │
│  │   namespace:      │   │   namespace:      │             │
│  │   team-frontend   │   │   team-backend    │             │
│  │                   │   │                   │             │
│  │  pod: nginx       │   │  pod: nginx  ← same name, no   │
│  │  pod: redis       │   │  pod: postgres    conflict!     │
│  │  svc: webapp      │   │  svc: api                       │
│  │  cm: app-config   │   │  cm: app-config  ← same name!  │
│  │                   │   │                   │             │
│  │  quota: 4 CPUs    │   │  quota: 8 CPUs    │             │
│  │  quota: 8Gi RAM   │   │  quota: 16Gi RAM  │             │
│  └───────────────────┘   └───────────────────┘             │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │  CLUSTER-SCOPED (no namespace — shared by all)        │ │
│  │  Nodes, PersistentVolumes, StorageClasses,            │ │
│  │  ClusterRoles, Namespaces themselves                  │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘

Namespaces provide:
  ✅ Name uniqueness scope (same name OK in different namespaces)
  ✅ Resource isolation (quotas, limits per namespace)
  ✅ Access control (RBAC per namespace)
  ✅ Network isolation (via NetworkPolicies)
  ✅ Organizational grouping (by team, env, app)
```

---

## 🏛️ The 4 Default Namespaces

```
┌──────────────────────────────────────────────────────────────┐
│  1. default                                                  │
│     → Where resources go when no namespace is specified      │
│     → Fine for learning/dev, avoid in production             │
│     → kubectl get pods  (shows default namespace)            │
├──────────────────────────────────────────────────────────────┤
│  2. kube-system                                              │
│     → All Kubernetes system components live here             │
│     → kube-apiserver, etcd, scheduler, controller-manager    │
│     → kube-proxy, CoreDNS, metrics-server                    │
│     → ⚠️  Never deploy your apps here                        │
│     → kubectl get pods -n kube-system                        │
├──────────────────────────────────────────────────────────────┤
│  3. kube-public                                              │
│     → Readable by ALL users (even unauthenticated)           │
│     → Contains ConfigMap: cluster-info                       │
│     → Rarely used directly                                   │
│     → kubectl get cm cluster-info -n kube-public             │
├──────────────────────────────────────────────────────────────┤
│  4. kube-node-lease                                          │
│     → Contains Lease objects — one per node                  │
│     → kubelet updates its Lease every second                 │
│     → Control plane uses this to detect node failures fast   │
│     → kubectl get leases -n kube-node-lease                  │
└──────────────────────────────────────────────────────────────┘
```

---

## 💻 Namespace Operations

### Create a Namespace

```bash
# Imperative
kubectl create namespace team-frontend
kubectl create namespace team-backend
kubectl create namespace monitoring

# Declarative (YAML)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: team-frontend
  labels:
    team: frontend
    env: production
EOF
```

### List and Describe

```bash
# List all namespaces
kubectl get namespaces
kubectl get ns           # shorthand

# Output:
# NAME              STATUS   AGE
# default           Active   10d
# kube-node-lease   Active   10d
# kube-public       Active   10d
# kube-system       Active   10d
# team-frontend     Active   2h
# team-backend      Active   2h

# Describe a namespace
kubectl describe namespace team-frontend

# See resource quota usage for a namespace
kubectl describe namespace team-frontend | grep -A10 "Resource Quotas"
```

### Working with Resources in Namespaces

```bash
# Create a pod in a specific namespace
kubectl run nginx --image=nginx -n team-frontend

# Get resources in a specific namespace
kubectl get pods -n team-frontend
kubectl get all -n team-frontend

# Get resources across ALL namespaces
kubectl get pods --all-namespaces
kubectl get pods -A               # shorthand

# Set default namespace for your session
kubectl config set-context --current --namespace=team-frontend
# Now all commands use team-frontend by default
kubectl get pods   # ← shows team-frontend pods

# Reset to default
kubectl config set-context --current --namespace=default
```

### Delete a Namespace

```bash
# Delete namespace (DELETES ALL RESOURCES INSIDE!)
kubectl delete namespace team-frontend

# ⚠️  WARNING: This deletes everything in the namespace:
#    pods, services, configmaps, secrets, deployments, etc.
#    There is NO undo — be very careful in production!

# Check deletion progress (large namespaces take time)
kubectl get namespace team-frontend
# STATUS: Terminating  ← deletion in progress
```

---

## 🌐 Cross-Namespace Communication

### DNS Format for Cross-Namespace Access

```
Within same namespace:
  → Access service by short name
  → http://my-service
  → http://my-service:8080

Cross-namespace:
  → Must use Fully Qualified Domain Name (FQDN)
  → Format: <service-name>.<namespace>.svc.cluster.local
  → http://my-service.team-backend.svc.cluster.local
  → http://my-service.team-backend.svc.cluster.local:8080
  → Shorthand: http://my-service.team-backend  (also works)

Examples:
  → frontend pod in "team-frontend" namespace
     wants to call backend API in "team-backend" namespace:
     http://api-service.team-backend.svc.cluster.local:3000

  → Any pod accessing CoreDNS (in kube-system):
     CoreDNS ClusterIP: 10.96.0.10
     Or by name: kube-dns.kube-system.svc.cluster.local
```

```
DNS STRUCTURE:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  my-service . team-backend . svc . cluster.local
  ──────────   ────────────   ───   ─────────────
  service      namespace     type   cluster domain
  name                      (svc)

  Valid ways to call within cluster:
  ✅ my-service                         (same namespace only)
  ✅ my-service.team-backend            (cross-namespace shorthand)
  ✅ my-service.team-backend.svc        (with type)
  ✅ my-service.team-backend.svc.cluster.local  (full FQDN)
```

---

## 🔒 Resource Isolation with ResourceQuota

ResourceQuota limits the total resources a namespace can use.

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-frontend-quota
  namespace: team-frontend
spec:
  hard:
    # Compute resources
    requests.cpu: "4"          # max 4 CPU cores requested
    requests.memory: 8Gi       # max 8Gi memory requested
    limits.cpu: "8"            # max 8 CPU limit
    limits.memory: 16Gi        # max 16Gi memory limit

    # Object count limits
    pods: "20"                 # max 20 pods
    services: "10"             # max 10 services
    configmaps: "20"           # max 20 configmaps
    secrets: "20"              # max 20 secrets
    persistentvolumeclaims: "5" # max 5 PVCs
```

```bash
# Apply quota
kubectl apply -f resource-quota.yaml

# View quota usage
kubectl get resourcequota -n team-frontend
kubectl describe resourcequota team-frontend-quota -n team-frontend

# Output:
# Name:                   team-frontend-quota
# Namespace:              team-frontend
# Resource                Used   Hard
# --------                ----   ----
# limits.cpu              2      8
# limits.memory           4Gi    16Gi
# pods                    5      20
# requests.cpu            1      4
# requests.memory         2Gi    8Gi
```

### What Happens When Quota is Exceeded

```bash
# Try to create a pod when quota is full:
kubectl run another-pod --image=nginx -n team-frontend

# Error:
# Error from server (Forbidden): pods "another-pod" is forbidden:
# exceeded quota: team-frontend-quota,
# requested: pods=1, used: pods=20, limited: pods=20
```

---

## 📏 Default Limits with LimitRange

LimitRange sets default CPU/memory requests and limits for containers that don't specify their own.

```yaml
# limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: team-frontend-limits
  namespace: team-frontend
spec:
  limits:
    - type: Container
      default:             # default LIMIT if not specified
        cpu: 500m
        memory: 256Mi
      defaultRequest:      # default REQUEST if not specified
        cpu: 100m
        memory: 128Mi
      max:                 # maximum allowed
        cpu: "2"
        memory: 1Gi
      min:                 # minimum required
        cpu: 50m
        memory: 64Mi

    - type: Pod
      max:
        cpu: "4"
        memory: 2Gi

    - type: PersistentVolumeClaim
      max:
        storage: 10Gi
      min:
        storage: 1Gi
```

---

## 🏗️ Real-World Namespace Strategies

### Strategy 1: By Environment
```
production/
  ├── namespace: prod-frontend
  ├── namespace: prod-backend
  └── namespace: prod-database

staging/
  ├── namespace: staging-frontend
  ├── namespace: staging-backend
  └── namespace: staging-database

development/
  └── namespace: dev (shared for all devs)
```

### Strategy 2: By Team
```
namespace: team-platform     (platform/infra team)
namespace: team-payments     (payments team)
namespace: team-identity     (auth/identity team)
namespace: team-catalog      (product catalog team)
namespace: monitoring        (monitoring stack)
namespace: logging           (logging stack)
```

### Strategy 3: By Application (Microservices)
```
namespace: order-service
namespace: payment-service
namespace: user-service
namespace: notification-service
namespace: api-gateway
```

### Best Practices
```
✅ Use namespaces in production — never put everything in "default"
✅ Apply ResourceQuota to EVERY namespace (prevent noisy neighbor problem)
✅ Apply LimitRange so containers always have resource boundaries
✅ Use RBAC per namespace — teams should only access their own namespace
✅ Use NetworkPolicy to control cross-namespace traffic
✅ Label your namespaces for easy filtering and policy targeting
✅ Create namespaces via YAML (GitOps) not just imperatively
```

---

## 🔐 Namespace RBAC (Access Control)

```yaml
# Give a user access to ONLY one namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer-role
  namespace: team-frontend    # ← scoped to this namespace only
rules:
  - apiGroups: ["", "apps"]
    resources: ["pods", "deployments", "services"]
    verbs: ["get", "list", "create", "update", "delete"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: team-frontend
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer-role
  apiGroup: rbac.authorization.k8s.io

# Result: alice can only manage pods/deployments/services
#         in team-frontend namespace — nothing else!
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| Leaving everything in `default` namespace | No isolation, no quotas, hard to manage at scale |
| `kubectl delete namespace` without backup | Deletes ALL resources — no undo! |
| Forgetting FQDN for cross-namespace calls | Short names only work within the same namespace |
| Thinking namespaces provide network isolation by default | They DON'T — you need NetworkPolicy for that |
| Namespace in `Terminating` forever | Usually a resource with finalizers blocking it — check with `kubectl get all -n <ns>` |
| ResourceQuota without LimitRange | If pods don't have resource requests, quota checks fail |

### Fix a Stuck Terminating Namespace

```bash
# Namespace stuck in Terminating?
kubectl get namespace my-ns
# STATUS: Terminating  ← stuck!

# Check what's blocking it
kubectl get all -n my-ns
kubectl api-resources --verbs=list --namespaced -o name | \
  xargs -n1 kubectl get -n my-ns 2>/dev/null

# Force remove by patching finalizers
kubectl get namespace my-ns -o json > /tmp/ns.json
# Edit /tmp/ns.json → remove the "kubernetes" entry from finalizers array
kubectl replace --raw "/api/v1/namespaces/my-ns/finalize" -f /tmp/ns.json
```

---

## 🎓 CKA Exam Tips

```
✅ Always check which namespace a task specifies!
   Many CKA failures come from working in the wrong namespace.

✅ Quick namespace switch:
   kubectl config set-context --current --namespace=<ns>

✅ See everything across all namespaces:
   kubectl get pods -A
   kubectl get all -A

✅ Namespace in task = always add -n <namespace> to every command
   OR set default context namespace first

✅ Cross-namespace DNS format — memorize:
   <svc>.<namespace>.svc.cluster.local

✅ ResourceQuota blocks pod creation if namespace is full
   → Check quota: kubectl describe ns <namespace>

✅ kube-system namespace = control plane pods
   → Often asked to check: kubectl get pods -n kube-system

✅ Create namespace YAML from scratch:
   kubectl create ns my-ns --dry-run=client -o yaml
```

---

## ❓ Interview & Scenario Questions

### Q1: What is a Kubernetes namespace and what problems does it solve?
**Answer:**
A namespace is a virtual partition inside a Kubernetes cluster that provides scope for names and isolation for resources. It solves several problems: name collision (two teams can have a deployment named "nginx" in different namespaces), resource quotas (limit CPU/memory per team), access control (RBAC policies scoped to a namespace), and organizational grouping (separate resources by team, environment, or application). Namespaces do NOT provide network isolation by default — that requires NetworkPolicy.

---

### Q2: How do you access a service in a different namespace?
**Answer:**
Use the Fully Qualified Domain Name (FQDN):
`<service-name>.<namespace>.svc.cluster.local`
For example, to access service `api-svc` in namespace `backend` from any pod:
```
http://api-svc.backend.svc.cluster.local:8080
```
The shorter form `api-svc.backend` also works within the cluster. The short form `api-svc` only works when both the caller and service are in the same namespace.

---

### Q3: Scenario — A developer says "my pod can't reach the database service." Both are in different namespaces. What do you check?
**Answer:**
Step-by-step:
1. Verify the database service exists: `kubectl get svc -n db-namespace`
2. Check the pod is using the FQDN: `db-service.db-namespace.svc.cluster.local`
3. Check Endpoints: `kubectl get endpoints db-service -n db-namespace` (should show pod IPs)
4. Check NetworkPolicy — is there a policy blocking cross-namespace traffic?
5. Test DNS resolution from the pod: `kubectl exec -it <pod> -- nslookup db-service.db-namespace`
6. Test connectivity: `kubectl exec -it <pod> -- curl http://db-service.db-namespace:5432`

---

### Q4: What happens to all resources when you delete a namespace?
**Answer:**
All resources within the namespace are deleted — pods, deployments, services, configmaps, secrets, PVCs, etc. This is cascade deletion triggered by the namespace controller. The deletion is permanent with no undo. PersistentVolumes are cluster-scoped and are NOT deleted, but the PVCs are deleted, and depending on the reclaim policy (`Retain` or `Delete`), the PV's data may or may not be preserved. Always back up important data before deleting a namespace.

---

### Q5: What is ResourceQuota and why should every namespace have one?
**Answer:**
ResourceQuota is an object that sets hard limits on the total resources (CPU, memory, pod count, service count, etc.) a namespace can consume. Every namespace should have one to prevent the "noisy neighbor" problem — where one team's workload consumes all cluster resources and starves other teams. Without quotas, a misconfigured deployment could spin up hundreds of pods and exhaust cluster capacity. ResourceQuota combined with LimitRange ensures both namespace-level and container-level resource boundaries.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│               TOPIC 1.7 — NAMESPACES                         │
├──────────────────────────────────────────────────────────────┤
│  DEFAULT NAMESPACES:                                         │
│  default         → your resources (dev/testing)              │
│  kube-system     → K8s components (never touch!)             │
│  kube-public     → public cluster info                       │
│  kube-node-lease → node heartbeat objects                    │
├──────────────────────────────────────────────────────────────┤
│  KEY COMMANDS:                                               │
│  kubectl get ns                                              │
│  kubectl create ns <name>                                    │
│  kubectl get pods -n <name>                                  │
│  kubectl get all -A                  (all namespaces)        │
│  kubectl config set-context --current --namespace=<n>        │
│  kubectl delete ns <name>            (⚠️ deletes everything) │
├──────────────────────────────────────────────────────────────┤
│  CROSS-NAMESPACE DNS:                                        │
│  <svc>.<namespace>.svc.cluster.local                         │
├──────────────────────────────────────────────────────────────┤
│  ISOLATION TOOLS:                                            │
│  ResourceQuota → limits total resources per namespace        │
│  LimitRange    → sets default/min/max per container          │
│  RBAC Role     → access control scoped to namespace          │
│  NetworkPolicy → network isolation between namespaces        │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `08-etcd-deep-dive.md`
> *etcd internals, backup & restore, disaster recovery — CKA critical topic*
