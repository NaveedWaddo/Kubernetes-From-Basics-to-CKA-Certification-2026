# 📌 Topic 1.3 — Control Plane Components (Deep Dive)

> **Module 1 — Core Concepts & Kubernetes Architecture**
> **File:** `03-control-plane-components.md`
> **CKA Weight:** ~10% — Troubleshooting control plane is a frequent CKA task

---

## 🎯 What You'll Learn
- Internal workings of every Control Plane component
- Key configuration flags & file locations
- How components communicate with each other
- Where logs live and how to debug each component
- Static Pods — what they are and why control plane uses them
- High Availability setup for each component
- Interview & scenario-based questions with full answers

---

## 🧠 Core Concept Explained

### The Government Analogy 🏛️
> Think of the Control Plane as a **government cabinet**:
>
> - **kube-apiserver** = The **Parliament / Reception Desk**
>   Everyone must go through here. Nothing bypasses it.
>
> - **etcd** = The **National Archives**
>   Every law, record, decision ever made is stored here permanently.
>
> - **kube-scheduler** = The **HR Department**
>   Decides which worker gets which job based on skills & availability.
>
> - **kube-controller-manager** = The **Ministry of Operations**
>   Multiple departments (ministries) running in parallel making sure
>   everything is working as intended — and fixing it when it's not.
>
> - **cloud-controller-manager** = The **Foreign Affairs Ministry**
>   Handles all communication with outside entities (cloud providers).

---

## 🗺️ Control Plane Internal Communication

```
                        kubectl / external clients
                                   │
                    ┌──────────────▼──────────────┐
                    │        kube-apiserver        │
                    │   Port: 6443 (HTTPS/TLS)    │
                    └──┬──────────┬───────────┬───┘
                       │          │           │
              ┌────────▼──┐  ┌────▼────┐  ┌──▼──────────────────┐
              │   etcd    │  │scheduler│  │ controller-manager   │
              │ Port:2379 │  │(watches │  │ (watches apiserver   │
              │(only API  │  │apiserver│  │  for state changes)  │
              │server     │  │for      │  └──────────────────────┘
              │talks here)│  │unscheduled
              └───────────┘  │pods)    │
                             └─────────┘

  ┌──────────────────────────────────────────────────────────┐
  │                    Worker Nodes                          │
  │  kubelet ──────────► apiserver (reports pod status)      │
  │  apiserver ─────────► kubelet  (sends pod specs)         │
  └──────────────────────────────────────────────────────────┘

KEY RULE: ALL components talk to apiserver — NEVER to each other directly
          ONLY apiserver talks to etcd directly
```

---

## 🔬 COMPONENT 1 — kube-apiserver (Deep Dive)

### What It Does
The API server is the **central nervous system** of Kubernetes. Every single operation — creating a pod, reading a secret, updating a deployment — goes through it.

### Request Processing Pipeline
```
Incoming Request (kubectl apply -f pod.yaml)
         │
         ▼
┌─────────────────────────────────────────────┐
│  1. AUTHENTICATION                          │
│     Who are you?                            │
│     → Check certificates / tokens / OIDC   │
│     → Invalid? → 401 Unauthorized           │
└─────────────────────────┬───────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────┐
│  2. AUTHORIZATION (RBAC)                    │
│     Are you allowed to do this?             │
│     → Check Roles, ClusterRoles, Bindings   │
│     → Denied? → 403 Forbidden               │
└─────────────────────────┬───────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────┐
│  3. ADMISSION CONTROLLERS                   │
│     Should this request be allowed/modified?│
│     → MutatingAdmissionWebhook              │
│       (can modify the request)              │
│     → ValidatingAdmissionWebhook            │
│       (can reject the request)              │
│     → Built-in: LimitRanger, ResourceQuota  │
└─────────────────────────┬───────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────┐
│  4. VALIDATION                              │
│     Is the object schema correct?           │
│     → Check required fields                 │
│     → Check field types and values          │
└─────────────────────────┬───────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────┐
│  5. PERSIST TO etcd                         │
│     Write the object to the cluster store   │
│     → Object is now "desired state"         │
└─────────────────────────┬───────────────────┘
                          │
                          ▼
                   Return Response ✅
```

### Key Configuration Flags
```bash
# View apiserver config (kubeadm clusters)
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Common important flags:
--advertise-address=<control-plane-ip>   # IP to advertise to cluster
--etcd-servers=https://127.0.0.1:2379    # Where etcd is
--service-cluster-ip-range=10.96.0.0/12 # IP range for Services
--authorization-mode=Node,RBAC           # Auth modes enabled
--enable-admission-plugins=...           # Admission controllers
--tls-cert-file=...                      # TLS certificate
--tls-private-key-file=...              # TLS private key
--audit-log-path=...                    # Audit log location
```

### File Locations
```
Config (kubeadm):  /etc/kubernetes/manifests/kube-apiserver.yaml
Certificates:      /etc/kubernetes/pki/apiserver.crt
                   /etc/kubernetes/pki/apiserver.key
Logs:              kubectl logs kube-apiserver-<node> -n kube-system
                   journalctl -u kube-apiserver (if not static pod)
Port:              6443 (HTTPS)
```

---

## 🔬 COMPONENT 2 — etcd (Deep Dive)

### What It Does
etcd is a **distributed, consistent, highly-available key-value store**. It is the ONLY place where Kubernetes stores its state. Every object you create — pods, deployments, secrets, configmaps, RBAC rules — everything lives in etcd.

### Raft Consensus Algorithm (Simplified)
```
3-Node etcd Cluster:

  ┌─────────┐    ┌─────────┐    ┌─────────┐
  │ etcd-1  │    │ etcd-2  │    │ etcd-3  │
  │ LEADER  │◄──►│FOLLOWER │◄──►│FOLLOWER │
  └─────────┘    └─────────┘    └─────────┘

  Write flow:
  1. Client sends write to LEADER
  2. LEADER sends to FOLLOWERS
  3. Majority (quorum) must acknowledge
  4. LEADER commits and responds to client

  If LEADER fails:
  → Followers elect a new leader automatically
  → Cluster remains available if quorum exists

  Quorum table:
  ┌───────┬─────────┬──────────────────┐
  │ Nodes │ Quorum  │ Fault Tolerance  │
  ├───────┼─────────┼──────────────────┤
  │   1   │    1    │  0 (no HA)       │
  │   3   │    2    │  1 node failure  │
  │   5   │    3    │  2 node failures │
  │   7   │    4    │  3 node failures │
  └───────┴─────────┴──────────────────┘
```

### What's Stored in etcd
```
/registry/
├── pods/
│   ├── default/
│   │   └── my-pod  ← full pod JSON
├── deployments/
├── services/
├── secrets/
├── configmaps/
├── nodes/
├── namespaces/
├── rbac/
│   ├── roles/
│   ├── rolebindings/
│   └── clusterroles/
└── ...everything in your cluster
```

### Critical etcd Commands (CKA EXAM MUST KNOW)
```bash
# ── BACKUP etcd ──────────────────────────────────────────
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verify the backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot.db \
  --write-out=table

# ── RESTORE etcd ─────────────────────────────────────────
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored \
  --name=master \
  --initial-cluster=master=https://127.0.0.1:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380

# After restore, update etcd manifest to use new data-dir:
# Edit /etc/kubernetes/manifests/etcd.yaml
# Change: --data-dir=/var/lib/etcd
# To:     --data-dir=/var/lib/etcd-restored

# ── CHECK etcd health ────────────────────────────────────
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# ── LIST etcd members ────────────────────────────────────
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### File Locations
```
Config (kubeadm):  /etc/kubernetes/manifests/etcd.yaml
Data directory:    /var/lib/etcd/
Certificates:      /etc/kubernetes/pki/etcd/
  ├── ca.crt           ← etcd CA cert
  ├── server.crt       ← etcd server cert
  ├── server.key       ← etcd server key
  └── peer.crt         ← peer communication cert
Ports:             2379 (client), 2380 (peer/cluster)
```

---

## 🔬 COMPONENT 3 — kube-scheduler (Deep Dive)

### What It Does
The scheduler **watches for unscheduled pods** (pods with `nodeName: ""`) and assigns the best node to each pod.

### Two-Phase Scheduling Process
```
Phase 1: FILTERING
──────────────────
Find all nodes that CAN run this pod

  Checks performed:
  ├── NodeUnschedulable     → Is node cordoned?
  ├── PodFitsResources      → Is there enough CPU/Memory?
  ├── PodFitsHostPorts      → Are required ports free?
  ├── MatchNodeSelector     → Does node match selector labels?
  ├── NoDiskConflict        → Are required volumes available?
  ├── NoVolumeZoneConflict  → Is PV in the same zone?
  ├── TolerationsTaints     → Does pod tolerate node taints?
  └── CheckNodeMemoryPressure → Is node under pressure?

  Result: List of "feasible" nodes

Phase 2: SCORING
────────────────
Rank the feasible nodes (0-100 score each)

  Score plugins:
  ├── LeastAllocated        → Prefer nodes with more free resources
  ├── BalancedAllocation    → Prefer nodes with balanced CPU/memory
  ├── ImageLocality         → Prefer nodes that already have the image
  ├── NodeAffinity          → Prefer nodes matching affinity rules
  └── TaintToleration       → Score based on toleration preferences

  Result: Highest scored node wins

Final: BINDING
──────────────
Scheduler writes chosen node to pod.spec.nodeName via apiserver
kubelet on that node then picks it up and starts the container
```

### Scheduling Flow Diagram
```
New Pod Created (nodeName = "")
         │
         ▼
   kube-scheduler
   watches API server
         │
         ▼
   Filter Phase
   ┌──────────────────────────────────────┐
   │ All Nodes: node1, node2, node3, node4│
   │                                      │
   │ node1 → NOT enough memory ❌         │
   │ node2 → Taint not tolerated ❌       │
   │ node3 → Feasible ✅                  │
   │ node4 → Feasible ✅                  │
   └──────────────────────────────────────┘
         │
         ▼
   Score Phase
   ┌──────────────────────────────────────┐
   │ node3 → score: 72                    │
   │ node4 → score: 85  ← WINNER         │
   └──────────────────────────────────────┘
         │
         ▼
   Bind: pod.spec.nodeName = "node4"
   (written to etcd via apiserver)
         │
         ▼
   kubelet on node4 starts the pod ✅
```

### Key Configuration Flags
```bash
# View scheduler config
cat /etc/kubernetes/manifests/kube-scheduler.yaml

# Important flags:
--config=/etc/kubernetes/scheduler-config.yaml  # Custom config
--leader-elect=true                              # HA leader election
--bind-address=127.0.0.1                        # Listen address
```

### File Locations
```
Config:  /etc/kubernetes/manifests/kube-scheduler.yaml
Logs:    kubectl logs kube-scheduler-<node> -n kube-system
```

---

## 🔬 COMPONENT 4 — kube-controller-manager (Deep Dive)

### What It Does
Runs many **controllers** in a single binary. Each controller runs a **control loop** — constantly watching the current state and reconciling it toward the desired state.

### The Control Loop (Reconciliation)
```
                    ┌─────────────────────────────┐
                    │       CONTROL LOOP           │
                    │                              │
    ┌──────────┐    │  1. OBSERVE                  │
    │          │    │     Watch apiserver for       │
    │ Desired  │◄───┤     current state             │
    │  State   │    │                              │
    │ (etcd)   │    │  2. DIFF                     │
    └────┬─────┘    │     Compare desired vs actual│
         │          │                              │
         │          │  3. ACT                      │
         ▼          │     Take action to reconcile │
    ┌──────────┐    │     (create/delete/update)   │
    │  Actual  │────┤                              │
    │  State   │    │  4. REPEAT                   │
    │(running) │    │     Loop forever             │
    └──────────┘    └─────────────────────────────┘

Example: ReplicaSet wants 3 pods, only 2 are running
→ Controller OBSERVES: 2 running pods
→ Controller DIFFS:    3 desired - 2 actual = 1 missing
→ Controller ACTS:     Creates 1 new pod via apiserver
→ Controller REPEATS:  Checks again... now 3 running ✅
```

### Key Controllers and What They Do
```
┌─────────────────────────────────────────────────────────┐
│              kube-controller-manager                    │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │ Node Controller                                   │  │
│  │  • Monitors node health every 5s                  │  │
│  │  • Marks node NotReady after 40s no heartbeat     │  │
│  │  • Evicts pods from NotReady node after 5min      │  │
│  └───────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────┐  │
│  │ ReplicaSet Controller                             │  │
│  │  • Ensures correct replica count always           │  │
│  │  • Creates/deletes pods to match desired count    │  │
│  └───────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────┐  │
│  │ Deployment Controller                             │  │
│  │  • Manages ReplicaSets for Deployments            │  │
│  │  • Handles rolling update strategy                │  │
│  │  • Manages revision history for rollbacks         │  │
│  └───────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────┐  │
│  │ Service Account Controller                        │  │
│  │  • Creates "default" SA in every new namespace    │  │
│  └───────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────┐  │
│  │ Endpoints Controller                              │  │
│  │  • Populates Endpoints object for Services        │  │
│  │  • Links Services to healthy Pod IPs              │  │
│  └───────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────┐  │
│  │ Job Controller                                    │  │
│  │  • Tracks Job completion                          │  │
│  │  • Cleans up completed Job pods                   │  │
│  └───────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────┐  │
│  │ Namespace Controller                              │  │
│  │  • Deletes all resources when namespace deleted   │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### Key Configuration Flags
```bash
# View controller-manager config
cat /etc/kubernetes/manifests/kube-controller-manager.yaml

# Important flags:
--cluster-cidr=10.244.0.0/16       # Pod network CIDR
--node-monitor-period=5s            # How often to check nodes
--node-monitor-grace-period=40s     # Before marking NotReady
--pod-eviction-timeout=5m0s         # Before evicting pods
--leader-elect=true                 # HA leader election
--controllers=*,bootstrapsigner,... # Which controllers to run
```

### File Locations
```
Config:  /etc/kubernetes/manifests/kube-controller-manager.yaml
Logs:    kubectl logs kube-controller-manager-<node> -n kube-system
```

---

## 🔬 COMPONENT 5 — cloud-controller-manager (Deep Dive)

### What It Does
Separates cloud-specific logic from the core Kubernetes code. Lets cloud providers implement their own integration without changing Kubernetes itself.

```
WITHOUT cloud-controller-manager (old way):
┌──────────────────────────────────────┐
│       kube-controller-manager        │
│  + AWS-specific code                 │
│  + GCP-specific code                 │
│  + Azure-specific code               │
│  → Messy, hard to maintain           │
└──────────────────────────────────────┘

WITH cloud-controller-manager (current):
┌─────────────────────┐  ┌──────────────────────────┐
│kube-controller-mgr  │  │ cloud-controller-manager  │
│ (generic K8s logic) │  │ (cloud-specific logic)    │
└─────────────────────┘  │  ├── Node Controller       │
                          │  ├── Route Controller      │
                          │  └── Service Controller    │
                          └──────────────────────────┘
                                     │
                          ┌──────────▼──────────┐
                          │   Cloud Provider API │
                          │  (AWS / GCP / Azure) │
                          └─────────────────────┘
```

### What Each Sub-Controller Does
```
Node Controller:
  → Syncs node info from cloud API (instance IDs, regions, zones)
  → Deletes node objects when cloud VM is terminated

Route Controller:
  → Sets up routes in cloud network for pod CIDR blocks
  → Enables cross-node pod communication in cloud VPC

Service Controller:
  → Watches for Services of type LoadBalancer
  → Calls cloud API to create/update/delete load balancers
  → Assigns external IP back to the Service object
```

---

## 🏛️ Static Pods — How Control Plane Runs

> This is critical for CKA! Control plane components run as **Static Pods**.

### What is a Static Pod?
```
Normal Pod:
  kubectl apply → apiserver → etcd → scheduler → kubelet → container

Static Pod:
  kubelet reads YAML file from disk → starts container directly
  (No apiserver, no scheduler, no etcd involvement)
  kubelet creates a "mirror pod" in apiserver so you can see it

Key difference: Static pods are managed by kubelet directly on a node
```

### Static Pod Location
```bash
# Static pod manifests directory
ls /etc/kubernetes/manifests/

# You will see:
# etcd.yaml
# kube-apiserver.yaml
# kube-controller-manager.yaml
# kube-scheduler.yaml

# To modify a control plane component:
# Just edit the YAML file here — kubelet will auto-restart it

# Example: Change apiserver flag
vi /etc/kubernetes/manifests/kube-apiserver.yaml
# Save → kubelet detects change → restarts kube-apiserver pod

# To view static pod config location in kubelet:
cat /var/lib/kubelet/config.yaml | grep staticPodPath
# staticPodPath: /etc/kubernetes/manifests
```

### Why Static Pods for Control Plane?
```
Problem: How does apiserver start if apiserver isn't running yet?
         (Chicken-and-egg problem!)

Solution: Static Pods!
  → kubelet starts running BEFORE apiserver
  → kubelet reads /etc/kubernetes/manifests/
  → kubelet starts etcd, apiserver, scheduler, ctrl-mgr
  → Now the cluster is up and apiserver can serve requests
  → kubelet creates "mirror pods" so you can see them via kubectl
```

---

## 🔍 Viewing Control Plane Components

```bash
# See all control plane pods
kubectl get pods -n kube-system

# Sample output:
# NAME                                   READY   STATUS    RESTARTS
# etcd-controlplane                      1/1     Running   0
# kube-apiserver-controlplane            1/1     Running   0
# kube-controller-manager-controlplane   1/1     Running   0
# kube-scheduler-controlplane            1/1     Running   0
# coredns-xxx-xxx                        1/1     Running   0
# kube-proxy-xxx                         1/1     Running   0

# Check logs of a control plane component
kubectl logs kube-apiserver-controlplane -n kube-system
kubectl logs kube-controller-manager-controlplane -n kube-system
kubectl logs kube-scheduler-controlplane -n kube-system
kubectl logs etcd-controlplane -n kube-system

# Describe a control plane pod (see config flags)
kubectl describe pod kube-apiserver-controlplane -n kube-system

# Check component health (older API, may be deprecated)
kubectl get componentstatuses
# NAME                 STATUS    MESSAGE
# scheduler            Healthy   ok
# controller-manager   Healthy   ok
# etcd-0               Healthy   {"health":"true"}
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| Editing running pods to fix control plane | Static pods are controlled by files in `/etc/kubernetes/manifests/` — edit the file, not the pod |
| `kubectl delete pod kube-apiserver-xxx` | Static pods auto-restart — kubelet will recreate it immediately |
| Forgetting `ETCDCTL_API=3` | Without it, etcdctl uses API v2 which has different commands |
| Not providing certs for etcdctl | etcd requires TLS auth — always pass `--cacert --cert --key` |
| Assuming scheduler runs containers | Scheduler ONLY assigns — kubelet does the actual running |
| Not taking etcd backups in production | Single biggest cause of unrecoverable cluster disasters |

---

## 🎓 CKA Exam Tips

```
✅ Know the manifest file path: /etc/kubernetes/manifests/
   → This is where you fix broken control plane components

✅ If kube-apiserver is broken:
   → You can't use kubectl!
   → Go directly to: /etc/kubernetes/manifests/kube-apiserver.yaml
   → Fix the YAML → kubelet restarts it automatically

✅ etcd backup command — memorize every flag:
   ETCDCTL_API=3 etcdctl snapshot save <path>
     --endpoints=https://127.0.0.1:2379
     --cacert=<ca.crt>
     --cert=<server.crt>
     --key=<server.key>

✅ Find cert paths from the apiserver manifest:
   grep etcd /etc/kubernetes/manifests/kube-apiserver.yaml
   → The cert paths are listed as flags there

✅ Controller manager down = no self-healing
   → Crashed pods won't be replaced
   → Test: delete a pod → new one should appear → if not, check ctrl-mgr

✅ Scheduler down = pods stay Pending
   → kubectl get pods → STATUS: Pending
   → kubectl describe pod → "no nodes available to schedule pods"
```

---

## ❓ Interview & Scenario Questions

### Q1: What is a Static Pod and why does the control plane use them?
**Answer:**
A Static Pod is a pod managed directly by the kubelet on a specific node, without going through the API server. kubelet watches a directory (`/etc/kubernetes/manifests/`) and automatically creates/restarts pods based on YAML files found there. The control plane uses static pods to solve the bootstrap problem — you need kubelet to start the API server before the API server is available to schedule pods. kubelet creates "mirror pods" so you can see them via `kubectl get pods -n kube-system`, but they can only be modified by changing the manifest files on disk.

---

### Q2: Scenario — You run `kubectl get pods` and it hangs forever. What do you check?
**Answer:**
`kubectl` communicates with `kube-apiserver`. If it hangs:
1. Check if apiserver is running: `ssh` to control plane node
2. Check static pod: `ls /etc/kubernetes/manifests/kube-apiserver.yaml`
3. Check if apiserver pod is running: `crictl ps | grep apiserver`
4. Check kubelet status: `systemctl status kubelet`
5. Check apiserver logs: `crictl logs <apiserver-container-id>`
6. Check for YAML syntax errors in the manifest file
7. Check if etcd is healthy (apiserver can't start if etcd is down)

---

### Q3: What happens in the control plane when you scale a Deployment from 3 to 5 replicas?
**Answer:**
```
1. kubectl scale deployment my-app --replicas=5
2. kubectl sends PATCH request to kube-apiserver
3. apiserver validates, authorizes, updates Deployment in etcd (replicas: 5)
4. Deployment Controller (in kube-controller-manager) watches apiserver
   → Sees Deployment replicas changed from 3 to 5
   → Updates the ReplicaSet desired count to 5
5. ReplicaSet Controller sees desired=5, actual=3
   → Creates 2 new Pod objects in etcd (no node assigned yet)
6. kube-scheduler sees 2 unscheduled pods
   → Filters and scores available nodes
   → Assigns nodes to both new pods via apiserver
7. kubelet on assigned nodes picks up the new pods
   → Pulls images, starts containers
8. kubelet reports pod status = Running to apiserver
9. Endpoints Controller updates Service endpoints to include new pods
```

---

### Q4: What is the difference between kube-controller-manager and cloud-controller-manager?
**Answer:**
`kube-controller-manager` runs generic Kubernetes controllers that work across any environment (on-prem, cloud, local) — Node, ReplicaSet, Deployment, Job, Endpoints controllers etc. `cloud-controller-manager` runs cloud-specific controllers that integrate K8s with a specific cloud provider's APIs — it creates actual load balancers on AWS/GCP/Azure when you create a LoadBalancer service, syncs node information from cloud VMs, and sets up cloud network routes. The separation allows Kubernetes core to remain cloud-agnostic while each provider ships their own `cloud-controller-manager`.

---

### Q5: Scenario — Pods are not being scheduled and stay in Pending state. apiserver is healthy. What do you check?
**Answer:**
If apiserver is healthy but pods stay Pending, the scheduler is the likely culprit:
```bash
# Check scheduler pod
kubectl get pods -n kube-system | grep scheduler

# If CrashLoopBackOff or not running:
kubectl logs kube-scheduler-controlplane -n kube-system

# Check the static pod manifest for issues
cat /etc/kubernetes/manifests/kube-scheduler.yaml

# Check if manually scheduled (nodeName set) pods work:
# → If they start, scheduler is definitely the problem

# Also check: resource pressure on all nodes
kubectl describe nodes | grep -A5 "Conditions:"
# → MemoryPressure or DiskPressure can prevent scheduling
```

---

### Q6: How does the Node Controller handle a failed node?
**Answer:**
The Node Controller monitors all nodes and follows this timeline:
- Every **5 seconds** — checks node health via kubelet heartbeats
- After **40 seconds** of no heartbeat — marks node as `NotReady`
- After **5 minutes** (pod-eviction-timeout) of `NotReady` — starts evicting pods from that node
- Evicted pods are recreated on healthy nodes by ReplicaSet Controller (for managed pods)
- Note: Static pods and DaemonSet pods are NOT evicted — they're tied to the node

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│         TOPIC 1.3 — CONTROL PLANE COMPONENTS                │
├──────────────────────────────────────────────────────────────┤
│ STATIC POD PATH:  /etc/kubernetes/manifests/                 │
│ SYSTEM NAMESPACE: kube-system                                │
├─────────────────────────┬────────────────────────────────────┤
│ Component               │ Key Detail                         │
├─────────────────────────┼────────────────────────────────────┤
│ kube-apiserver          │ Port 6443 | auth→authz→admission   │
│ etcd                    │ Port 2379 | backup with etcdctl    │
│ kube-scheduler          │ Filter → Score → Bind              │
│ kube-controller-manager │ Reconcile desired vs actual state  │
│ cloud-controller-manager│ Cloud LBs, nodes, routes           │
├─────────────────────────┴────────────────────────────────────┤
│ ETCD BACKUP (memorize!):                                     │
│  ETCDCTL_API=3 etcdctl snapshot save <file>                  │
│    --endpoints=https://127.0.0.1:2379                        │
│    --cacert=/etc/kubernetes/pki/etcd/ca.crt                  │
│    --cert=/etc/kubernetes/pki/etcd/server.crt                │
│    --key=/etc/kubernetes/pki/etcd/server.key                 │
├──────────────────────────────────────────────────────────────┤
│ DEBUG COMMANDS:                                              │
│  kubectl get pods -n kube-system                             │
│  kubectl logs <component-pod> -n kube-system                 │
│  cat /etc/kubernetes/manifests/<component>.yaml              │
│  systemctl status kubelet                                    │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `04-worker-node-components.md`
> *kubelet, kube-proxy, Container Runtime — deep dive with troubleshooting*
