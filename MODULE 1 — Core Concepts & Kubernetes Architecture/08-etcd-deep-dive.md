# 📌 Topic 1.8 — etcd Deep Dive

> **Module 1 — Core Concepts & Kubernetes Architecture**
> **File:** `08-etcd-deep-dive.md`
> **CKA Weight:** 🔥 CRITICAL — etcd backup & restore is a near-guaranteed CKA exam task

---

## 🎯 What You'll Learn
- What etcd is and why Kubernetes depends on it completely
- etcd internals — Raft consensus, leader election, quorum
- Exactly what Kubernetes stores in etcd
- etcd deployment topologies (stacked vs external)
- **etcd backup — full command with every flag explained**
- **etcd restore — full step-by-step process**
- etcd health checks and monitoring
- Disaster recovery scenarios
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The DNA Vault Analogy 🧬
> etcd is the **DNA vault** of your Kubernetes cluster:
>
> - **DNA** = the complete blueprint of every living thing
> - **etcd** = the complete state of every object in the cluster
>
> Just like destroying a cell's DNA means the cell can't function,
> losing etcd means your cluster loses ALL knowledge of what should exist —
> every pod, deployment, service, secret, RBAC rule — **everything gone**.
>
> That's why:
> - etcd gets its own dedicated hardware in production
> - etcd is backed up regularly and religiously
> - etcd runs in a high-availability cluster of 3 or 5 nodes
> - Restoring etcd = restoring the entire cluster from any point in time

---

## 🔬 What is etcd?

etcd is a **distributed, consistent, highly-available key-value store** written in Go. It was created by CoreOS (now part of Red Hat) and is maintained by the CNCF.

```
Key characteristics:
  ✅ Distributed   → runs across multiple nodes
  ✅ Consistent    → all nodes see the same data (linearizable reads)
  ✅ Highly Available → survives node failures via Raft consensus
  ✅ Key-Value     → stores data as key → value pairs
  ✅ Watch API     → clients can subscribe to changes (kube-apiserver uses this!)
  ✅ Secure        → mutual TLS between all nodes and clients
  ✅ Fast          → designed for small data, high frequency writes
```

---

## ⚙️ Raft Consensus Algorithm — How etcd Stays Consistent

### The Problem etcd Solves
```
If you have 3 etcd nodes, and two clients write different values
at the same time, which value wins? How do all nodes agree?

Without consensus: Split-brain → different nodes have different data ❌
With Raft:         Only ONE leader accepts writes, then replicates ✅
```

### Raft Leader Election
```
NORMAL STATE — Leader is healthy:

  ┌──────────┐      heartbeat      ┌──────────┐
  │  etcd-1  │ ─────────────────► │  etcd-2  │
  │  LEADER  │ ─────────────────► │ FOLLOWER │
  └──────────┘      heartbeat      └──────────┘
       │                                │
       │            heartbeat           │
       ▼                                ▼
  ┌──────────┐                   ┌──────────┐
  │  etcd-3  │                   │  etcd-4  │
  │ FOLLOWER │                   │ FOLLOWER │
  └──────────┘                   └──────────┘

  ALL WRITES go to leader → leader replicates to followers
  READ can come from any node (or leader only for strong consistency)

LEADER FAILS:

  etcd-1 (leader) dies ──────────────────────────────────────────┐
                                                                  │
  Followers notice: no heartbeat for election timeout (150-300ms)│
  Followers start election:                                       │
  → etcd-2 sends RequestVote to etcd-3 and etcd-4               │
  → Majority votes for etcd-2                                    │
  → etcd-2 becomes new LEADER in milliseconds                    │
  → Cluster continues serving requests ✅                        │
```

### Quorum — Why Odd Numbers
```
Quorum = minimum nodes needed to agree for a write to succeed
Formula: floor(N/2) + 1

┌────────┬─────────┬───────────────┬──────────────────────────┐
│ Nodes  │ Quorum  │ Can Tolerate  │ Notes                    │
├────────┼─────────┼───────────────┼──────────────────────────┤
│   1    │    1    │ 0 failures    │ No HA — dev only         │
│   2    │    2    │ 0 failures    │ Worse than 1 (deadlock)  │
│   3    │    2    │ 1 failure     │ Standard HA setup ✅      │
│   4    │    3    │ 1 failure     │ Same as 3, costs more ❌  │
│   5    │    3    │ 2 failures    │ High availability ✅      │
│   7    │    4    │ 3 failures    │ Large critical clusters  │
└────────┴─────────┴───────────────┴──────────────────────────┘

Rule: Always use ODD numbers of etcd nodes!
```

---

## 💾 What Kubernetes Stores in etcd

Every object you create via `kubectl` ultimately gets stored as a key-value pair in etcd:

```
etcd key structure (Kubernetes prefix: /registry/)

/registry/
├── pods/
│   ├── default/
│   │   ├── nginx-pod-abc123     → full Pod JSON/protobuf
│   │   └── webapp-xyz456        → full Pod JSON/protobuf
│   └── kube-system/
│       ├── kube-apiserver-cp    → static pod spec
│       └── etcd-cp              → static pod spec
│
├── deployments/
│   └── default/
│       └── my-webapp            → full Deployment JSON
│
├── replicasets/
├── services/
│   └── default/
│       └── kubernetes           → the default kubernetes service
│
├── secrets/
│   └── default/
│       └── my-secret            → base64 encoded secret data
│
├── configmaps/
├── namespaces/
│   ├── default
│   ├── kube-system
│   └── ...
│
├── nodes/
│   ├── node1                    → node registration info
│   └── node2
│
├── minions/                     → legacy node info
│
├── rbac/
│   ├── roles/
│   ├── rolebindings/
│   ├── clusterroles/
│   └── clusterrolebindings/
│
├── apiregistration.k8s.io/
├── storage.k8s.io/
└── ...everything in your cluster
```

```bash
# You can explore etcd keys directly (from a control plane node):
ETCDCTL_API=3 etcdctl get / --prefix --keys-only \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key | head -30

# Get a specific key (raw protobuf — mostly unreadable but useful to know)
ETCDCTL_API=3 etcdctl get /registry/pods/default/my-pod \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

---

## 🏗️ etcd Deployment Topologies

### Topology 1: Stacked etcd (Default with kubeadm)
```
  ┌────────────────────────────────────────┐
  │          Control Plane Node 1          │
  │  ┌──────────────┐  ┌────────────────┐  │
  │  │ kube-apiserver│  │     etcd       │  │
  │  │  scheduler    │  │   (member 1)   │  │
  │  │  ctrl-manager │  │               │  │
  │  └──────────────┘  └────────────────┘  │
  └────────────────────────────────────────┘

  ┌────────────────────────────────────────┐
  │          Control Plane Node 2          │
  │  ┌──────────────┐  ┌────────────────┐  │
  │  │ kube-apiserver│  │     etcd       │  │
  │  │  scheduler    │  │   (member 2)   │  │
  │  │  ctrl-manager │  │               │  │
  │  └──────────────┘  └────────────────┘  │
  └────────────────────────────────────────┘

  ┌────────────────────────────────────────┐
  │          Control Plane Node 3          │
  │  ┌──────────────┐  ┌────────────────┐  │
  │  │ kube-apiserver│  │     etcd       │  │
  │  │  scheduler    │  │   (member 3)   │  │
  │  │  ctrl-manager │  │               │  │
  │  └──────────────┘  └────────────────┘  │
  └────────────────────────────────────────┘

  Pros: Fewer nodes needed, simpler to set up
  Cons: etcd and control plane share node resources
        If a node fails, both apiserver & etcd lose a member
```

### Topology 2: External etcd (Enterprise Production)
```
  Control Plane Nodes         External etcd Cluster
  ┌──────────────────┐        ┌──────────────────┐
  │  kube-apiserver  │        │   etcd-node-1    │
  │  scheduler       │◄──────►│   (member 1)     │
  │  ctrl-manager    │        └──────────────────┘
  └──────────────────┘
  ┌──────────────────┐        ┌──────────────────┐
  │  kube-apiserver  │        │   etcd-node-2    │
  │  scheduler       │◄──────►│   (member 2)     │
  │  ctrl-manager    │        └──────────────────┘
  └──────────────────┘
  ┌──────────────────┐        ┌──────────────────┐
  │  kube-apiserver  │        │   etcd-node-3    │
  │  scheduler       │◄──────►│   (member 3)     │
  │  ctrl-manager    │        └──────────────────┘
  └──────────────────┘

  Pros: etcd has dedicated resources
        Control plane and etcd failures are independent
  Cons: More nodes to manage (6 instead of 3)
        More complex to set up
```

---

## 💾 etcd BACKUP — Complete Guide (CKA CRITICAL)

### Before You Back Up — Find the Certs

```bash
# Method 1: Check the etcd pod manifest
cat /etc/kubernetes/manifests/etcd.yaml | grep -E "cert|key|ca"

# You'll see lines like:
# --cert-file=/etc/kubernetes/pki/etcd/server.crt
# --key-file=/etc/kubernetes/pki/etcd/server.key
# --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
# --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
# --peer-key-file=/etc/kubernetes/pki/etcd/peer.key

# Method 2: Check apiserver manifest for etcd client certs
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep etcd

# Common cert paths:
# CA:   /etc/kubernetes/pki/etcd/ca.crt
# Cert: /etc/kubernetes/pki/etcd/server.crt   (or peer.crt)
# Key:  /etc/kubernetes/pki/etcd/server.key   (or peer.key)
```

### The Backup Command — Full Breakdown

```bash
# ═══════════════════════════════════════════════════════════
# ETCD BACKUP — MEMORIZE THIS FOR CKA!
# ═══════════════════════════════════════════════════════════

ETCDCTL_API=3 etcdctl snapshot save /opt/backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Breaking down every flag:
# ─────────────────────────────────────────────────────────
# ETCDCTL_API=3
#   → Use etcdctl API version 3 (v2 has different commands)
#   → Always set this — some systems default to v2
#
# etcdctl snapshot save <PATH>
#   → Command to take a snapshot
#   → <PATH> = where to save the snapshot file
#   → Use absolute path! e.g. /opt/backup/etcd-snapshot.db
#
# --endpoints=https://127.0.0.1:2379
#   → Where etcd is listening
#   → Port 2379 = client port (always)
#   → localhost if running from control plane node
#   → Use IP if etcd is on external node
#
# --cacert=/etc/kubernetes/pki/etcd/ca.crt
#   → The Certificate Authority cert to verify etcd server
#
# --cert=/etc/kubernetes/pki/etcd/server.crt
#   → Client certificate (proves we're allowed to connect)
#
# --key=/etc/kubernetes/pki/etcd/server.key
#   → Private key for the client certificate

# ─────────────────────────────────────────────────────────
# Verify the backup was created successfully:
ETCDCTL_API=3 etcdctl snapshot status /opt/backup/etcd-snapshot.db \
  --write-out=table

# Output:
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | fe01cf57 |    10    |    1021    |   2.4 MB   |
# +----------+----------+------------+------------+

# Also verify with ls
ls -lh /opt/backup/etcd-snapshot.db
```

---

## 🔄 etcd RESTORE — Step-by-Step (CKA CRITICAL)

Restoring etcd is a multi-step process. Follow this order exactly.

```
RESTORE OVERVIEW:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Step 1: Stop apiserver (so nothing writes to etcd during restore)
Step 2: Run etcdctl snapshot restore (creates new data dir)
Step 3: Update etcd manifest to use new data directory
Step 4: Restart services
Step 5: Verify cluster is back
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### Step 1: Stop the API Server

```bash
# Move apiserver manifest out of static pod directory
# (kubelet will stop the apiserver pod automatically)
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# Wait for apiserver to stop
watch crictl ps | grep apiserver
# (wait until it disappears from the list)
```

### Step 2: Restore the Snapshot

```bash
# ═══════════════════════════════════════════════════════════
# ETCD RESTORE — MEMORIZE THIS FOR CKA!
# ═══════════════════════════════════════════════════════════

ETCDCTL_API=3 etcdctl snapshot restore /opt/backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored

# Breaking down every flag:
# ─────────────────────────────────────────────────────────
# snapshot restore <PATH>
#   → Path to the snapshot file to restore from
#
# --data-dir=/var/lib/etcd-restored
#   → Where to write the restored etcd data
#   → Use a NEW directory (not the existing /var/lib/etcd)
#   → This avoids corruption if something goes wrong

# Note: For multi-node etcd clusters, additional flags needed:
# --name=<member-name>
# --initial-cluster=<member>=https://<ip>:2380
# --initial-cluster-token=<unique-token>
# --initial-advertise-peer-urls=https://<ip>:2380
# (For single-node / CKA exam, the simple form above is enough)
```

### Step 3: Update etcd Manifest

```bash
# Edit the etcd static pod manifest
vi /etc/kubernetes/manifests/etcd.yaml

# Find this section and update the data-dir:
# BEFORE:
#   - --data-dir=/var/lib/etcd
#   volumeMounts:
#   - mountPath: /var/lib/etcd
#     name: etcd-data
#   volumes:
#   - hostPath:
#       path: /var/lib/etcd   ← change this
#       type: DirectoryOrCreate
#     name: etcd-data

# AFTER:
#   - --data-dir=/var/lib/etcd-restored   ← updated
#   volumeMounts:
#   - mountPath: /var/lib/etcd-restored   ← updated
#     name: etcd-data
#   volumes:
#   - hostPath:
#       path: /var/lib/etcd-restored      ← updated
#       type: DirectoryOrCreate
#     name: etcd-data

# 3 places to update in the YAML:
# 1. --data-dir flag value
# 2. volumeMounts mountPath
# 3. volumes hostPath.path
```

### Step 4: Restart Services

```bash
# Restore apiserver manifest
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# kubelet will restart both etcd and apiserver automatically
# (since they're static pods)

# Wait for components to come back up
watch kubectl get nodes
# (wait for Ready status)

# Or check pods directly
watch crictl ps
```

### Step 5: Verify

```bash
# Cluster should be back and showing pre-backup state
kubectl get nodes
kubectl get pods --all-namespaces
kubectl get deployments

# Check etcd health
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
# Output: https://127.0.0.1:2379 is healthy: successfully committed proposal
```

---

## 🩺 etcd Health & Monitoring

```bash
# ── Health Check ──────────────────────────────────────────
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# ── Member List ───────────────────────────────────────────
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Output (3-node cluster):
# ID       STATUS   NAME    PEER ADDRS               CLIENT ADDRS
# abc123   started  etcd-1  https://10.0.0.1:2380   https://10.0.0.1:2379
# def456   started  etcd-2  https://10.0.0.2:2380   https://10.0.0.2:2379
# ghi789   started  etcd-3  https://10.0.0.3:2380   https://10.0.0.3:2379

# ── Cluster Status ────────────────────────────────────────
ETCDCTL_API=3 etcdctl endpoint status \
  --endpoints=https://127.0.0.1:2379,https://10.0.0.2:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --write-out=table

# ── View etcd pod logs ────────────────────────────────────
kubectl logs etcd-controlplane -n kube-system
kubectl logs etcd-controlplane -n kube-system | tail -20

# ── etcd metrics (Prometheus-compatible) ─────────────────
curl -k https://127.0.0.1:2379/metrics  # if accessible
```

---

## 🔧 etcd Configuration Deep Dive

```bash
# View full etcd configuration (kubeadm cluster)
cat /etc/kubernetes/manifests/etcd.yaml

# Key flags explained:
spec:
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://10.0.0.1:2379   # URL for clients
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt  # server TLS cert
    - --client-cert-auth=true                           # require client certs
    - --data-dir=/var/lib/etcd                          # where data is stored
    - --election-timeout=5000                           # ms before new election
    - --heartbeat-interval=250                          # ms between heartbeats
    - --initial-advertise-peer-urls=https://10.0.0.1:2380 # peer URL
    - --initial-cluster=master=https://10.0.0.1:2380   # initial members
    - --initial-cluster-state=new                       # new or existing
    - --initial-cluster-token=etcd-cluster-1            # unique cluster token
    - --key-file=/etc/kubernetes/pki/etcd/server.key    # server TLS key
    - --listen-client-urls=https://127.0.0.1:2379       # listen address
    - --listen-metrics-urls=http://127.0.0.1:2381       # metrics endpoint
    - --listen-peer-urls=https://10.0.0.1:2380          # peer communication
    - --name=master                                      # this member's name
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
```

---

## 💥 Disaster Recovery Scenarios

### Scenario 1: Single etcd node failure (3-node cluster)
```
Situation: etcd-2 crashes
Impact: None immediately — quorum (2/3) still met
Action:
  1. Fix or replace the node
  2. Add it back as etcd member:
     etcdctl member add etcd-2 --peer-urls=https://10.0.0.2:2380
  3. Start etcd on that node with --initial-cluster-state=existing
```

### Scenario 2: ALL etcd data lost (catastrophic)
```
Situation: /var/lib/etcd deleted or corrupted
Impact: etcd won't start → entire cluster is down
Action:
  1. Stop kube-apiserver (move manifest out)
  2. Restore from latest snapshot:
     etcdctl snapshot restore <backup-file> --data-dir=/var/lib/etcd-new
  3. Update etcd manifest to use new data-dir
  4. Restart everything
  5. Verify cluster state
```

### Scenario 3: etcd member count falls below quorum
```
Situation: 2 out of 3 etcd nodes fail → only 1 remains
Impact: etcd refuses all writes (no quorum) → cluster freezes
        Reads may still work but nothing can be scheduled/updated
Action:
  1. Restore at least 1 more etcd member to re-establish quorum
  2. Or restore from backup on a fresh single-node etcd
Prevention: Use 5-node etcd for critical clusters (can survive 2 failures)
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| Forgetting `ETCDCTL_API=3` | Without it, etcdctl v2 commands are used — different syntax |
| Not providing all 3 cert flags | etcd requires `--cacert`, `--cert`, AND `--key` — any missing = auth error |
| Restoring to same `--data-dir` | Always use a NEW directory — restore to `/var/lib/etcd-restored`, then update manifest |
| Not updating volumeMounts in manifest | etcd manifest has 3 places to update data-dir — miss one and etcd won't start |
| Taking backup without verifying it | Always run `snapshot status` after backup to verify it's valid |
| Backing up from follower with stale data | Always backup from the LEADER node for the freshest data |

---

## 🎓 CKA Exam Tips

```
✅ BACKUP COMMAND — write this on paper before exam:
   ETCDCTL_API=3 etcdctl snapshot save <file>
     --endpoints=https://127.0.0.1:2379
     --cacert=/etc/kubernetes/pki/etcd/ca.crt
     --cert=/etc/kubernetes/pki/etcd/server.crt
     --key=/etc/kubernetes/pki/etcd/server.key

✅ Find cert paths from the etcd manifest:
   grep -i cert /etc/kubernetes/manifests/etcd.yaml

✅ After restore — update 3 places in etcd.yaml:
   1. --data-dir=
   2. volumeMounts mountPath
   3. volumes hostPath.path

✅ Verify backup with snapshot status before submitting:
   ETCDCTL_API=3 etcdctl snapshot status <file> --write-out=table

✅ etcd data directory default: /var/lib/etcd
   After restore use new dir: /var/lib/etcd-restored (or similar)

✅ If you see "context deadline exceeded" in etcdctl:
   → Wrong endpoint, or etcd is down, or wrong certs

✅ exam tip: Copy the etcdctl command from:
   https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/
   (open-book exam — this page is bookmarked on exam day!)
```

---

## ❓ Interview & Scenario Questions

### Q1: What is etcd and why is it critical to Kubernetes?
**Answer:**
etcd is a distributed, consistent, highly-available key-value store that serves as the single source of truth for all Kubernetes cluster state. Every object — pods, deployments, services, secrets, RBAC rules, ConfigMaps, nodes — is stored in etcd as key-value pairs. If etcd is lost without a backup, ALL cluster state is permanently gone. No other Kubernetes component (not even the apiserver) persists data — they all read and write to etcd. This makes etcd the most critical component in the entire cluster for disaster recovery purposes.

---

### Q2: Explain the Raft consensus algorithm in simple terms.
**Answer:**
Raft ensures all etcd nodes agree on data even when some nodes fail:
1. One node is elected **Leader** — it's the only one that accepts writes
2. All writes go to the leader → leader sends copies to all **Followers**
3. A write is only committed when a **majority (quorum)** of nodes acknowledge it
4. Leader sends regular **heartbeats** to followers; if a follower misses heartbeats, it triggers a new **leader election**
5. The new leader is elected by majority vote — so you always need more than half your nodes alive

This is why we need odd numbers: 3 nodes needs 2 for quorum (survives 1 failure), 5 nodes needs 3 (survives 2 failures).

---

### Q3: Walk me through the complete etcd backup process.
**Answer:**
```bash
# 1. Find certificate paths from etcd manifest
grep cert /etc/kubernetes/manifests/etcd.yaml

# 2. Create backup directory
mkdir -p /opt/etcd-backup

# 3. Take the snapshot
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup/etcd-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 4. Verify the backup
ETCDCTL_API=3 etcdctl snapshot status \
  /opt/etcd-backup/etcd-$(date +%Y%m%d).db --write-out=table

# 5. (Production) Copy to off-cluster storage:
# scp or push to S3/GCS/Azure Blob
```

---

### Q4: Scenario — Your cluster is down because etcd data is corrupted. How do you restore?
**Answer:**
```bash
# 1. Stop kube-apiserver to prevent writes during restore
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
# Wait for apiserver to stop: watch crictl ps

# 2. Restore snapshot to new directory
ETCDCTL_API=3 etcdctl snapshot restore /opt/backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored

# 3. Update etcd manifest (3 places!)
vi /etc/kubernetes/manifests/etcd.yaml
# Change --data-dir, volumeMounts.mountPath, volumes.hostPath.path
# All from /var/lib/etcd  →  /var/lib/etcd-restored

# 4. Bring apiserver back
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# 5. Wait for cluster to recover
kubectl get nodes
kubectl get pods -A
```

---

### Q5: What is the difference between stacked etcd and external etcd topology?
**Answer:**
In **stacked etcd**, each control plane node runs both the Kubernetes components (apiserver, scheduler, ctrl-manager) AND an etcd member on the same machine. This is simpler and uses fewer nodes — it's the default kubeadm setup. The risk is that if a control plane node fails, you lose both an apiserver and an etcd member simultaneously.

In **external etcd**, etcd runs on dedicated separate nodes. Control plane nodes only run Kubernetes components. This gives etcd its own resources and means control plane failures and etcd failures are independent of each other. It requires more nodes (6 total for 3+3 HA) but is preferred for high-criticality production clusters.

---

### Q6: What happens to the cluster if etcd goes down completely?
**Answer:**
If all etcd nodes go down:
- **Reads may continue** briefly from apiserver cache, but will become stale
- **All writes stop** — no new pods can be created, scheduled, or updated
- **No self-healing** — crashed pods won't be replaced (controllers can't update etcd)
- **kubectl commands** that need to write will fail; reads may work briefly
- **Existing running pods continue running** on worker nodes — kubelet doesn't need etcd to keep containers running
- The cluster is effectively **frozen** in its last known state until etcd is restored

This is why regular etcd backups and HA etcd clusters (3+ nodes) are absolutely mandatory in production.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│              TOPIC 1.8 — etcd DEEP DIVE                      │
├──────────────────────────────────────────────────────────────┤
│  WHAT IT IS: Distributed key-value store                     │
│  STORES:     ALL Kubernetes cluster state                    │
│  PORT:       2379 (client)  2380 (peer/cluster)              │
│  DATA DIR:   /var/lib/etcd  (default)                        │
│  CERTS:      /etc/kubernetes/pki/etcd/                       │
├──────────────────────────────────────────────────────────────┤
│  BACKUP (memorize!):                                         │
│  ETCDCTL_API=3 etcdctl snapshot save <file>                  │
│    --endpoints=https://127.0.0.1:2379                        │
│    --cacert=/etc/kubernetes/pki/etcd/ca.crt                  │
│    --cert=/etc/kubernetes/pki/etcd/server.crt                │
│    --key=/etc/kubernetes/pki/etcd/server.key                 │
├──────────────────────────────────────────────────────────────┤
│  RESTORE (3 steps):                                          │
│  1. etcdctl snapshot restore <file> --data-dir=/new/dir      │
│  2. Update etcd.yaml: data-dir, mountPath, hostPath (x3)     │
│  3. Restart — kubelet handles static pods automatically       │
├──────────────────────────────────────────────────────────────┤
│  QUORUM:                                                     │
│  3 nodes → quorum=2 → tolerates 1 failure                   │
│  5 nodes → quorum=3 → tolerates 2 failures                  │
│  Always use ODD numbers!                                     │
├──────────────────────────────────────────────────────────────┤
│  HEALTH CHECK:                                               │
│  ETCDCTL_API=3 etcdctl endpoint health ...                   │
│  ETCDCTL_API=3 etcdctl member list ...                       │
└──────────────────────────────────────────────────────────────┘
```

---

> 🎉 **MODULE 1 COMPLETE!**
> You have finished all 8 topics of Core Concepts & Architecture!
>
> **Next Module →** Module 2 — Workloads & Scheduling
> **First topic →** `09-pods-deep-dive.md`
> *Pod lifecycle, multi-container pods, init containers, static pods*
