# 📌 Topic 1.2 — Kubernetes Architecture

> **Module 1 — Core Concepts & Kubernetes Architecture**
> **File:** `02-kubernetes-architecture.md`
> **CKA Weight:** ~10% — Understanding architecture is essential for ALL other topics

---

## 🎯 What You'll Learn
- The two main parts of a Kubernetes cluster
- Control Plane components and their roles
- Worker Node components and their roles
- How all components communicate with each other
- Full architecture flow diagrams
- What happens step-by-step when you run `kubectl apply`
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Simple Analogy 🏭
> Think of a Kubernetes cluster like a **large factory**:
>
> - **Control Plane** = The **Management Office** (CEO, HR, Planner, Database)
>   - Makes all decisions
>   - Keeps records of everything
>   - Assigns work to workers
>   - Monitors progress
>
> - **Worker Nodes** = The **Factory Floor** (actual workers doing the job)
>   - Receive instructions from management
>   - Run the actual work (containers)
>   - Report status back to management
>
> You (the developer) talk to **management (Control Plane)** via `kubectl`.
> Management then figures out **which worker** should do what.

---

## 🗺️ Full Kubernetes Architecture Diagram

```
                        YOU (Developer)
                             │
                    kubectl apply -f app.yaml
                             │
                             ▼
╔═══════════════════════════════════════════════════════╗
║                   CONTROL PLANE                       ║
║                  (Master Node)                        ║
║                                                       ║
║  ┌─────────────────┐    ┌──────────────────────────┐  ║
║  │   kube-apiserver │◄──►│          etcd            │  ║
║  │  (Front Door)    │    │  (Cluster Database)      │  ║
║  └────────┬────────┘    └──────────────────────────┘  ║
║           │                                            ║
║  ┌────────▼────────┐    ┌──────────────────────────┐  ║
║  │ kube-scheduler  │    │  kube-controller-manager  │  ║
║  │ (Pod Assigner)  │    │  (State Reconciler)       │  ║
║  └─────────────────┘    └──────────────────────────┘  ║
║                                                       ║
║  ┌─────────────────────────────────────────────────┐  ║
║  │         cloud-controller-manager (optional)     │  ║
║  │         (Cloud Provider Integration)            │  ║
║  └─────────────────────────────────────────────────┘  ║
╚═══════════════════════════════════════════════════════╝
                   │              │              │
          ┌────────┘              │              └────────┐
          ▼                       ▼                       ▼
╔═══════════════╗       ╔═══════════════╗       ╔═══════════════╗
║  WORKER NODE 1║       ║  WORKER NODE 2║       ║  WORKER NODE 3║
║               ║       ║               ║       ║               ║
║  ┌──────────┐ ║       ║  ┌──────────┐ ║       ║  ┌──────────┐ ║
║  │  kubelet │ ║       ║  │  kubelet │ ║       ║  │  kubelet │ ║
║  └──────────┘ ║       ║  └──────────┘ ║       ║  └──────────┘ ║
║  ┌──────────┐ ║       ║  ┌──────────┐ ║       ║  ┌──────────┐ ║
║  │kube-proxy│ ║       ║  │kube-proxy│ ║       ║  │kube-proxy│ ║
║  └──────────┘ ║       ║  └──────────┘ ║       ║  └──────────┘ ║
║  ┌──────────┐ ║       ║  ┌──────────┐ ║       ║  ┌──────────┐ ║
║  │Container │ ║       ║  │Container │ ║       ║  │Container │ ║
║  │ Runtime  │ ║       ║  │ Runtime  │ ║       ║  │ Runtime  │ ║
║  └──────────┘ ║       ║  └──────────┘ ║       ║  └──────────┘ ║
║               ║       ║               ║       ║               ║
║  [Pod] [Pod]  ║       ║  [Pod] [Pod]  ║       ║  [Pod] [Pod]  ║
╚═══════════════╝       ╚═══════════════╝       ╚═══════════════╝
```

---

## 🏛️ PART 1 — Control Plane Components

> The brain of the cluster. Makes all decisions. Typically runs on dedicated master node(s).

---

### 1️⃣ kube-apiserver — The Front Door

```
┌────────────────────────────────────────────────────┐
│                  kube-apiserver                    │
│                                                    │
│  • Only entry point into the cluster               │
│  • Exposes the Kubernetes REST API                 │
│  • Authenticates & Authorizes all requests         │
│  • Validates and processes API objects             │
│  • Writes/reads state to/from etcd                 │
│                                                    │
│  kubectl ──► apiserver ──► etcd                    │
│  kubelet  ──► apiserver                            │
│  scheduler ──► apiserver                           │
└────────────────────────────────────────────────────┘
```

**Key facts:**
- All communication in K8s goes THROUGH the API server
- It's stateless — actual state is stored in etcd
- Can be scaled horizontally for HA (multiple replicas)
- Uses TLS for all communication
- Default port: **6443**

---

### 2️⃣ etcd — The Cluster Database

```
┌────────────────────────────────────────────────────┐
│                      etcd                          │
│                                                    │
│  • Distributed key-value store                     │
│  • Stores ALL cluster state:                       │
│    - Node info                                     │
│    - Pod definitions                               │
│    - Secrets & ConfigMaps                          │
│    - RBAC policies                                 │
│    - Service definitions                           │
│    - Everything in your YAML files                 │
│                                                    │
│  • Uses Raft consensus algorithm                   │
│  • Needs odd number of nodes for quorum (1, 3, 5)  │
│  • Default port: 2379 (client), 2380 (peer)        │
└────────────────────────────────────────────────────┘
```

**Key facts:**
- The ONLY persistent storage in the entire cluster
- If etcd dies and you have no backup → cluster state is GONE
- **Backup etcd regularly** — this is a CKA exam task!
- Only the API server talks directly to etcd
- Uses the `etcdctl` CLI tool

---

### 3️⃣ kube-scheduler — The Pod Assigner

```
┌────────────────────────────────────────────────────┐
│                  kube-scheduler                    │
│                                                    │
│  Watches for: Pods with no assigned node           │
│  Job: Picks the best node for each pod             │
│                                                    │
│  Decision factors:                                 │
│  ├── Resource requirements (CPU/Memory)            │
│  ├── Node capacity (how much is free?)             │
│  ├── Affinity / Anti-affinity rules                │
│  ├── Taints and Tolerations                        │
│  ├── Data locality                                 │
│  └── Custom scheduling policies                   │
│                                                    │
│  Process:                                          │
│  1. Filter  → which nodes CAN run this pod?        │
│  2. Score   → which node is BEST?                  │
│  3. Assign  → write node name to pod via apiserver │
└────────────────────────────────────────────────────┘
```

**Key facts:**
- Scheduler only DECIDES — it doesn't actually start containers
- After scheduling, kubelet on the target node starts the pod
- You can bypass scheduler by setting `nodeName` directly
- Custom schedulers are possible for special use cases

---

### 4️⃣ kube-controller-manager — The State Reconciler

```
┌────────────────────────────────────────────────────┐
│             kube-controller-manager                │
│                                                    │
│  Runs multiple controllers in ONE binary:          │
│                                                    │
│  ┌─────────────────────────────────────────────┐  │
│  │ Node Controller                              │  │
│  │  → Monitors nodes, marks NotReady if down   │  │
│  ├─────────────────────────────────────────────┤  │
│  │ ReplicaSet Controller                        │  │
│  │  → Ensures correct # of pod replicas        │  │
│  ├─────────────────────────────────────────────┤  │
│  │ Deployment Controller                        │  │
│  │  → Manages rolling updates & rollbacks      │  │
│  ├─────────────────────────────────────────────┤  │
│  │ Service Account Controller                   │  │
│  │  → Creates default service accounts         │  │
│  ├─────────────────────────────────────────────┤  │
│  │ Job Controller                               │  │
│  │  → Manages Job/CronJob execution            │  │
│  └─────────────────────────────────────────────┘  │
│                                                    │
│  Core loop: Watch state → Compare desired vs actual│
│             → Take action to reconcile             │
└────────────────────────────────────────────────────┘
```

**Key facts:**
- Each controller watches the API server for changes
- Core principle: **desired state vs actual state**
- If a pod dies, ReplicaSet controller creates a new one
- Runs as a single process but contains many logical controllers

---

### 5️⃣ cloud-controller-manager — Cloud Integration

```
┌────────────────────────────────────────────────────┐
│           cloud-controller-manager                 │
│                                                    │
│  Only present in cloud deployments (EKS, GKE, AKS)│
│                                                    │
│  Integrates K8s with cloud provider APIs:          │
│  ├── Node Controller → syncs cloud VM info         │
│  ├── Route Controller → sets up cloud network      │
│  └── Service Controller → creates cloud LBs        │
│                                                    │
│  Example: You create a Service type=LoadBalancer   │
│  → cloud-controller-manager calls AWS API          │
│  → Creates an actual ELB/ALB on AWS                │
│  → Assigns external IP back to Service             │
└────────────────────────────────────────────────────┘
```

---

## ⚙️ PART 2 — Worker Node Components

> The muscle of the cluster. Actually runs your application containers.

---

### 1️⃣ kubelet — The Node Agent

```
┌────────────────────────────────────────────────────┐
│                     kubelet                        │
│                                                    │
│  • Runs on EVERY worker node (and master too)      │
│  • The only K8s component that runs containers     │
│  • Talks to the API server constantly              │
│                                                    │
│  Responsibilities:                                 │
│  ├── Registers the node with the cluster           │
│  ├── Watches for pods assigned to its node         │
│  ├── Tells container runtime to start/stop pods    │
│  ├── Reports pod status back to API server         │
│  ├── Runs liveness/readiness probes                │
│  └── Manages container logs                        │
│                                                    │
│  Does NOT manage containers not created by K8s     │
└────────────────────────────────────────────────────┘
```

**Key facts:**
- kubelet is the only K8s component NOT deployed as a pod
- It's a system service managed by systemd
- If kubelet dies → node goes NotReady
- Communicates with container runtime via CRI (Container Runtime Interface)

---

### 2️⃣ kube-proxy — The Network Rules Manager

```
┌────────────────────────────────────────────────────┐
│                   kube-proxy                       │
│                                                    │
│  • Runs on every node as a DaemonSet               │
│  • Maintains network rules (iptables / IPVS)       │
│  • Enables Service → Pod routing                   │
│                                                    │
│  When you create a Service:                        │
│  kube-proxy writes iptables rules so that:         │
│  traffic → Service IP → gets forwarded → Pod IP    │
│                                                    │
│  Modes:                                            │
│  ├── iptables (default, most common)               │
│  ├── IPVS (better performance at scale)            │
│  └── userspace (legacy, rarely used)               │
└────────────────────────────────────────────────────┘
```

---

### 3️⃣ Container Runtime — The Container Engine

```
┌────────────────────────────────────────────────────┐
│               Container Runtime                    │
│                                                    │
│  The software that actually runs containers        │
│                                                    │
│  Supported runtimes (via CRI):                     │
│  ├── containerd  ← Most common (default in K8s)    │
│  ├── CRI-O       ← Lightweight, OpenShift uses     │
│  └── Docker*     ← Was popular, deprecated in K8s  │
│       * Docker uses containerd internally anyway   │
│                                                    │
│  CRI = Container Runtime Interface                 │
│  → Standard API between kubelet & runtime          │
│  → Allows swapping runtimes without changing K8s   │
└────────────────────────────────────────────────────┘
```

---

## 🔄 How It All Works Together — Step by Step

### Scenario: You run `kubectl apply -f deployment.yaml`

```
Step 1: kubectl
  └─► Reads your YAML file
  └─► Sends HTTP POST to kube-apiserver

Step 2: kube-apiserver
  └─► Authenticates your request (who are you?)
  └─► Authorizes your request (are you allowed?)
  └─► Validates the manifest (is the YAML correct?)
  └─► Stores the Deployment object in etcd
  └─► Returns 200 OK to kubectl

Step 3: kube-controller-manager
  └─► Deployment Controller sees new Deployment in etcd
  └─► Creates a ReplicaSet object
  └─► ReplicaSet Controller sees ReplicaSet
  └─► Creates 3 Pod objects (if replicas: 3)
  └─► Pods are created but have no node assigned yet

Step 4: kube-scheduler
  └─► Sees 3 Pods with nodeName: "" (unscheduled)
  └─► Filters nodes: which nodes have enough CPU/RAM?
  └─► Scores nodes: which is most suitable?
  └─► Assigns nodeName to each Pod via apiserver
  └─► Writes assignment back to etcd

Step 5: kubelet (on chosen Worker Node)
  └─► Watches API server for pods assigned to its node
  └─► Sees the new Pod assigned to it
  └─► Calls Container Runtime (containerd) via CRI
  └─► Container Runtime pulls the image from registry
  └─► Container Runtime starts the container
  └─► kubelet reports Pod status = Running to apiserver

Step 6: kube-proxy (on all nodes)
  └─► If a Service exists for this Deployment
  └─► Updates iptables rules to route traffic to new pods
  └─► Traffic to Service ClusterIP now reaches new pods
```

```
FULL FLOW DIAGRAM:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

kubectl apply
     │
     ▼
kube-apiserver ──► validates ──► stores in etcd
     │
     ├──────────────────────────────────┐
     ▼                                  ▼
kube-controller-manager          kube-scheduler
  (creates ReplicaSet/Pods)       (assigns nodes to pods)
     │                                  │
     └──────────────┬───────────────────┘
                    ▼
              kube-apiserver
              (updates pod with nodeName)
                    │
                    ▼
             kubelet on Node
             (starts container via CRI)
                    │
                    ▼
          Container Runtime
          (pulls image, runs container)
                    │
                    ▼
            Pod is RUNNING ✅

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 🔍 Control Plane vs Worker Node — Side by Side

```
┌─────────────────────────────────────────────────────────┐
│              CONTROL PLANE (Master)                     │
├──────────────────────┬──────────────────────────────────┤
│ Component            │ Responsibility                    │
├──────────────────────┼──────────────────────────────────┤
│ kube-apiserver       │ REST API gateway, auth, validation│
│ etcd                 │ Key-value store, cluster database │
│ kube-scheduler       │ Assigns pods to nodes             │
│ kube-ctrl-manager    │ Reconciles desired vs actual state│
│ cloud-ctrl-manager   │ Cloud provider integration        │
└──────────────────────┴──────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                  WORKER NODE                            │
├──────────────────────┬──────────────────────────────────┤
│ Component            │ Responsibility                    │
├──────────────────────┼──────────────────────────────────┤
│ kubelet              │ Node agent, starts/stops pods     │
│ kube-proxy           │ Network rules, service routing    │
│ Container Runtime    │ Actually runs containers          │
└──────────────────────┴──────────────────────────────────┘
```

---

## 🏗️ High Availability Architecture

> In production, the Control Plane is replicated for fault tolerance.

```
                    LOAD BALANCER
                    (HAProxy / AWS ELB)
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
   ┌────────────┐  ┌────────────┐  ┌────────────┐
   │  Master 1  │  │  Master 2  │  │  Master 3  │
   │ API Server │  │ API Server │  │ API Server │
   │ Scheduler  │  │ Scheduler  │  │ Scheduler  │
   │ Ctrl-Mgr   │  │ Ctrl-Mgr   │  │ Ctrl-Mgr   │
   └────────────┘  └────────────┘  └────────────┘
          │               │               │
          └───────────────┼───────────────┘
                          ▼
                 ┌─────────────────┐
                 │   etcd Cluster  │
                 │  (3 or 5 nodes) │
                 │  Raft consensus │
                 └─────────────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
      Worker 1        Worker 2        Worker 3
```

**Why odd numbers for etcd?**
```
Quorum = (N/2) + 1 nodes must agree for writes

1 node  → quorum=1 → no fault tolerance
3 nodes → quorum=2 → tolerates 1 failure  ✅
5 nodes → quorum=3 → tolerates 2 failures ✅
4 nodes → quorum=3 → tolerates 1 failure  ❌ (same as 3, wasteful)
```

---

## 💻 Useful Commands for Architecture Exploration

```bash
# See all nodes (master + workers)
kubectl get nodes

# See system pods (control plane components)
kubectl get pods -n kube-system

# Typical output:
# NAME                               READY   STATUS
# coredns-xxx                        1/1     Running   ← DNS
# etcd-controlplane                  1/1     Running   ← etcd
# kube-apiserver-controlplane        1/1     Running   ← API Server
# kube-controller-manager-xxx        1/1     Running   ← Ctrl Mgr
# kube-proxy-xxx                     1/1     Running   ← kube-proxy
# kube-scheduler-controlplane        1/1     Running   ← Scheduler

# Describe a node to see its components
kubectl describe node <node-name>

# Check component health (older clusters)
kubectl get componentstatuses

# Check which node a pod is running on
kubectl get pods -o wide

# Check kubelet status on a node
systemctl status kubelet

# Check kubelet logs
journalctl -u kubelet -f
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| "Master node runs my apps" | By default, pods are NOT scheduled on control plane nodes |
| "Docker is the container runtime" | K8s deprecated Docker support; containerd is the default |
| "etcd is just a database" | etcd is THE source of truth — losing it = losing cluster state |
| "Scheduler starts containers" | Scheduler only ASSIGNS nodes — kubelet actually starts them |
| "kube-proxy routes all traffic" | kube-proxy manages iptables rules; CNI plugin handles pod networking |
| "One master is fine" | Single master = single point of failure; use 3+ for production |

---

## 🎓 CKA Exam Tips

```
✅ Know which components run on Control Plane vs Worker Node
✅ Know what each component does — exam gives broken clusters to fix
✅ Control plane components run as Static Pods in /etc/kubernetes/manifests/
   → To fix broken API server: edit the manifest file directly
✅ kubelet is a systemd service — not a pod!
   → Check: systemctl status kubelet
   → Logs: journalctl -u kubelet
✅ etcd backup is a guaranteed CKA task — know the etcdctl command cold
✅ If a pod is stuck "Pending" → usually a scheduler issue
✅ If a pod is "Terminating" forever → usually a kubelet/node issue
```

---

## ❓ Interview & Scenario Questions

### Q1: Name all Control Plane components and their roles.
**Answer:**
- **kube-apiserver** — REST API gateway; single entry point; handles auth/authz/validation; reads/writes to etcd
- **etcd** — Distributed key-value store; only persistent storage in the cluster; stores all cluster state
- **kube-scheduler** — Watches for unscheduled pods; picks the best node using filtering + scoring
- **kube-controller-manager** — Runs all controllers (Node, ReplicaSet, Deployment, Job, etc.); reconciles desired vs actual state
- **cloud-controller-manager** — (Cloud only) Integrates with cloud provider APIs for LBs, nodes, routes

---

### Q2: What happens if the kube-scheduler goes down?
**Answer:**
Existing pods continue running unaffected. However, NEW pods that need to be scheduled will remain in **Pending** state with no node assignment. No new deployments, scaling operations, or pod restarts will work correctly until the scheduler is restored.

---

### Q3: Scenario — A pod is stuck in `Pending` state. What could be wrong?
**Answer:**
Most common reasons:
1. **Scheduler is down** — no component to assign the pod to a node
2. **Insufficient resources** — no node has enough CPU/memory
3. **Node affinity/taints** — pod has rules that no node satisfies
4. **PVC not bound** — pod needs persistent storage that isn't available
5. **Image pull limits** — some cases where image can't be resolved

Debugging steps:
```bash
kubectl describe pod <pod-name>
# Look at "Events" section at the bottom
# It will say: "0/3 nodes available: insufficient memory" etc.
```

---

### Q4: What is the difference between kubelet and kube-proxy?
**Answer:**
```
kubelet    → Manages POD lifecycle on a node
             Starts/stops containers
             Reports pod health to API server
             Runs as a systemd service

kube-proxy → Manages NETWORK RULES on a node
             Maintains iptables/IPVS rules
             Enables Service → Pod routing
             Runs as a DaemonSet pod
```

---

### Q5: Why does etcd need an odd number of nodes?
**Answer:**
etcd uses the **Raft consensus algorithm** which requires a quorum — a majority of nodes must agree before committing a write. The formula is `(N/2) + 1`. With an odd number: 3 nodes requires 2 for quorum (tolerates 1 failure), 5 nodes requires 3 for quorum (tolerates 2 failures). Using an even number like 4 gives the same fault tolerance as 3 but costs more — so odd numbers are always used.

---

### Q6: Scenario — You have a 3-node cluster. The control plane node crashes. What happens to running pods?
**Answer:**
Running pods **continue to run** on worker nodes — they are not immediately affected. However:
- No new pods can be scheduled (scheduler is down)
- No self-healing (controller manager is down, so crashed pods won't be replaced)
- No kubectl commands work (API server is down)
- Existing application traffic still flows (kube-proxy rules are still active)

This is why **production clusters use 3+ control plane nodes** for high availability.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────┐
│           TOPIC 1.2 — KUBERNETES ARCHITECTURE            │
├──────────────────────────────────────────────────────────┤
│ CONTROL PLANE COMPONENTS:                                │
│  kube-apiserver      → REST gateway, auth, validation    │
│  etcd                → Cluster database (key-value)      │
│  kube-scheduler      → Assigns pods to nodes             │
│  kube-ctrl-manager   → Reconciles desired vs actual      │
│  cloud-ctrl-manager  → Cloud provider integration        │
├──────────────────────────────────────────────────────────┤
│ WORKER NODE COMPONENTS:                                  │
│  kubelet             → Node agent, manages pod lifecycle │
│  kube-proxy          → Network rules (iptables/IPVS)     │
│  container runtime   → Actually runs containers          │
├──────────────────────────────────────────────────────────┤
│ KEY FLOW:                                                │
│  kubectl → apiserver → etcd                              │
│  controller-manager → creates pods                       │
│  scheduler → assigns pods to nodes                       │
│  kubelet → starts containers via CRI                     │
├──────────────────────────────────────────────────────────┤
│ CRITICAL FACTS:                                          │
│  • Only apiserver talks to etcd directly                 │
│  • kubelet is a systemd service, NOT a pod               │
│  • etcd needs odd numbers (1, 3, 5) for quorum           │
│  • Control plane pods live in kube-system namespace      │
│  • Static pod manifests: /etc/kubernetes/manifests/      │
└──────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `03-control-plane-components.md`
> *Deep dive into each Control Plane component — internals, config flags, troubleshooting*
