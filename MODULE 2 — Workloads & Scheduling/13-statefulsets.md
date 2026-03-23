# 📌 Topic 2.5 — StatefulSets

> **Module 2 — Workloads & Scheduling**
> **File:** `13-statefulsets.md`
> **CKA Weight:** ~15% (Workloads) — StatefulSets appear in storage + workload CKA tasks

---

## 🎯 What You'll Learn
- What a StatefulSet is and why Deployments fail for stateful apps
- Stable network identity — how pod names and DNS work
- Stable persistent storage — volumeClaimTemplates
- Ordered creation, deletion, and scaling
- StatefulSet update strategies
- Headless Services — the DNS backbone of StatefulSets
- Real-world use cases (databases, Kafka, Zookeeper, Elasticsearch)
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Numbered Apartment Analogy 🏢
> Imagine an apartment building where **identity matters**:
>
> **Regular Deployment apartments (anonymous):**
> - Apartments have random names: apt-x7k2p, apt-9mnd3
> - If one is destroyed, a new one is built with a DIFFERENT random name
> - No one cares which apartment is which — they're all identical
> - Good for: stateless apps (nginx, APIs)
>
> **StatefulSet apartments (named, persistent identity):**
> - Apartments are numbered: apt-0, apt-1, apt-2
> - Each has its OWN storage locker (PVC) that follows it
> - If apt-1 is destroyed, a new apt-1 is built — SAME name, SAME locker
> - apt-0 is always the "primary" (databases elect leaders by number)
> - Good for: databases (MySQL, PostgreSQL, MongoDB, Kafka)
>
> The identity (name + storage) is what makes stateful apps work!

---

## 🔬 Why Deployments Fail for Stateful Apps

```
PROBLEM 1: Unstable Pod Names
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Deployment pod names are random:
  mysql-7d9f8b-xk2pq   (primary)
  mysql-7d9f8b-nm3dp   (replica)

Pod crashes and restarts:
  mysql-7d9f8b-xk2pq → GONE
  mysql-7d9f8b-ab7cd  ← NEW random name

Problem: Other pods that knew the primary was "xk2pq"
         can no longer find it! Replication breaks! ❌

PROBLEM 2: Shared Storage Causes Data Corruption
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Deployment: all pods share same PVC
  [mysql-pod-1] ──► PVC ◄── [mysql-pod-2]
  Both writing to same storage = CORRUPTION ❌

PROBLEM 3: No Guaranteed Startup Order
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Kafka broker-2 needs broker-0 to be ready before it starts
Deployment starts all pods simultaneously — chaos! ❌

SOLUTION: StatefulSet
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Stable pod names:    mysql-0, mysql-1, mysql-2 (always)
✅ Individual PVCs:     each pod gets its own storage
✅ Ordered startup:     0 starts first, then 1, then 2
✅ Ordered shutdown:    2 stops first, then 1, then 0
✅ Stable DNS:          mysql-0.mysql.default.svc.cluster.local
```

---

## 🗺️ StatefulSet Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                      StatefulSet: mysql                      │
│                                                              │
│  Headless Service: mysql (clusterIP: None)                   │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Pod: mysql-0  (PRIMARY / LEADER)                       │ │
│  │  DNS: mysql-0.mysql.default.svc.cluster.local           │ │
│  │  PVC: data-mysql-0  (10Gi) ← DEDICATED, persistent      │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Pod: mysql-1  (REPLICA)                                │ │
│  │  DNS: mysql-1.mysql.default.svc.cluster.local           │ │
│  │  PVC: data-mysql-1  (10Gi) ← DEDICATED, persistent      │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Pod: mysql-2  (REPLICA)                                │ │
│  │  DNS: mysql-2.mysql.default.svc.cluster.local           │ │
│  │  PVC: data-mysql-2  (10Gi) ← DEDICATED, persistent      │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                              │
│  Key guarantees:                                             │
│  ✅ mysql-0 always starts FIRST                              │
│  ✅ Each pod has stable DNS name (even after restart)        │
│  ✅ Each pod has its OWN PVC (not shared)                    │
│  ✅ PVCs survive pod deletion (data persists!)               │
└──────────────────────────────────────────────────────────────┘
```

---

## 📄 StatefulSet YAML — Full Anatomy

```yaml
# Step 1: Headless Service (REQUIRED for stable DNS)
apiVersion: v1
kind: Service
metadata:
  name: mysql                    # THIS name is used in pod DNS!
  namespace: default
  labels:
    app: mysql
spec:
  clusterIP: None                # ← HEADLESS: no virtual IP
  selector:
    app: mysql                   # selects StatefulSet pods
  ports:
    - port: 3306
      targetPort: 3306
      name: mysql

---
# Step 2: StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: default

spec:
  # ── REPLICA COUNT ───────────────────────────────────────
  replicas: 3

  # ── LINK TO HEADLESS SERVICE ────────────────────────────
  serviceName: "mysql"           # must match headless service name
                                  # this enables stable DNS per pod

  # ── POD SELECTOR ────────────────────────────────────────
  selector:
    matchLabels:
      app: mysql

  # ── POD MANAGEMENT POLICY ───────────────────────────────
  podManagementPolicy: OrderedReady   # OrderedReady | Parallel
  # OrderedReady: pods start/stop one at a time (default)
  # Parallel:     pods start/stop simultaneously (for faster ops)

  # ── UPDATE STRATEGY ─────────────────────────────────────
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0               # only update pods with index >= partition
                                  # partition: 2 → only mysql-2 updates (canary!)

  # ── PERSISTENT VOLUME CLAIM RETENTION ──────────────────
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain          # Retain | Delete PVCs when SS deleted
    whenScaled: Retain           # Retain | Delete PVCs when scaled down

  # ── POD TEMPLATE ────────────────────────────────────────
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
              name: mysql
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
          volumeMounts:
            - name: data          # references volumeClaimTemplate name
              mountPath: /var/lib/mysql

          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"

          readinessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - mysqladmin ping -h localhost
            initialDelaySeconds: 30
            periodSeconds: 10

          livenessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - mysqladmin ping -h localhost
            initialDelaySeconds: 60
            periodSeconds: 20

  # ── VOLUME CLAIM TEMPLATES ──────────────────────────────
  # This is the KEY difference from Deployments!
  # Creates a DEDICATED PVC for EACH pod automatically
  volumeClaimTemplates:
    - metadata:
        name: data               # referenced in volumeMounts above
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: "standard"
        resources:
          requests:
            storage: 10Gi
```

---

## 🌐 Stable Network Identity — DNS Deep Dive

```
StatefulSet pod naming format:
  <statefulset-name>-<ordinal>
  mysql-0, mysql-1, mysql-2, ...

Headless Service DNS format:
  <pod-name>.<service-name>.<namespace>.svc.cluster.local

Examples:
  mysql-0.mysql.default.svc.cluster.local  ← resolves to mysql-0's IP
  mysql-1.mysql.default.svc.cluster.local  ← resolves to mysql-1's IP
  mysql-2.mysql.default.svc.cluster.local  ← resolves to mysql-2's IP

  Shorter form (within same namespace):
  mysql-0.mysql
  mysql-1.mysql

Why this matters:
  Pod mysql-0 crashes and restarts → NEW IP (10.244.1.99 → 10.244.1.105)
  BUT DNS name stays the same: mysql-0.mysql.default.svc.cluster.local
  DNS now resolves to new IP automatically ✅

  Other services can ALWAYS reach mysql-0 by DNS name
  regardless of how many times it restarts!

Regular Service DNS (for clients):
  mysql.default.svc.cluster.local ← goes to ANY healthy pod (load balanced)

Use the headless DNS for:
  → Replication: replicas connect to mysql-0.mysql specifically
  → Leader election: apps know mysql-0 is always "first" node
  → Peer discovery: Kafka/Zookeeper find each other by ordinal

Use the regular Service DNS for:
  → Application clients connecting to the database
  → Load balanced reads across all replicas
```

---

## 🗄️ volumeClaimTemplates — Per-Pod Storage

```
DEPLOYMENT (shared storage — WRONG for stateful apps):
  [mysql-0] ──────┐
  [mysql-1] ──────┼──► PVC: mysql-data (single PVC)
  [mysql-2] ──────┘         │
                          [10Gi PV]
  → All pods write to same storage = CORRUPTION ❌

STATEFULSET (dedicated storage — CORRECT):
  [mysql-0] ──► PVC: data-mysql-0 ──► [10Gi PV]
  [mysql-1] ──► PVC: data-mysql-1 ──► [10Gi PV]
  [mysql-2] ──► PVC: data-mysql-2 ──► [10Gi PV]
  → Each pod has its own dedicated storage ✅

PVC naming format:
  <volumeClaimTemplate.name>-<statefulset-name>-<ordinal>
  data-mysql-0
  data-mysql-1
  data-mysql-2

CRITICAL: PVCs are NOT deleted when pods are deleted!
  kubectl delete pod mysql-0
  → Pod mysql-0 is recreated
  → Reconnects to SAME PVC: data-mysql-0
  → Data is preserved! ✅

  kubectl delete statefulset mysql
  → Pods deleted
  → PVCs REMAIN (by default)
  → Data survives StatefulSet deletion ✅ (or ❌ if you wanted cleanup)
  → Delete PVCs manually: kubectl delete pvc data-mysql-0
```

---

## 🔢 Ordered Operations

### Ordered Creation (default: OrderedReady)

```
Creating StatefulSet with replicas: 3

  Step 1: Create mysql-0
          → Wait for mysql-0 to be Running AND Ready
          → mysql-0 passes readiness probe ✅

  Step 2: Create mysql-1
          → Wait for mysql-1 to be Running AND Ready ✅

  Step 3: Create mysql-2
          → Wait for mysql-2 to be Running AND Ready ✅

  All 3 pods running ✅

Why this matters for databases:
  → mysql-0 starts first → initializes as PRIMARY
  → mysql-1 starts after → finds mysql-0 running → joins as REPLICA
  → mysql-2 starts after → joins as REPLICA
  → If started simultaneously, all might try to be PRIMARY!
```

### Ordered Deletion (Reverse Order)

```
Scaling down from 3 to 1 (or deleting StatefulSet):

  Step 1: Delete mysql-2 → wait for it to terminate ✅
  Step 2: Delete mysql-1 → wait for it to terminate ✅
  Step 3: Delete mysql-0 → wait for it to terminate ✅

Why reverse order:
  → mysql-2 is the last replica → safe to remove first
  → mysql-0 is the primary → remove last
  → Prevents data loss if replicas depend on primary being alive
```

### Parallel Policy (for faster operations)

```yaml
podManagementPolicy: Parallel
# All pods created simultaneously
# All pods deleted simultaneously
# Use when: ordering doesn't matter (e.g., stateless-ish workloads
#           using StatefulSet only for stable names)
```

---

## 🔄 StatefulSet Update Strategies

### RollingUpdate (Default)

```
Update goes in REVERSE ordinal order: mysql-2, mysql-1, mysql-0
Why reverse? Primary (mysql-0) updated LAST = less disruption

  Start: mysql-0(v1) mysql-1(v1) mysql-2(v1)

  Step 1: Delete mysql-2, create mysql-2 with v2
          Wait for Ready ✅
  Step 2: Delete mysql-1, create mysql-1 with v2
          Wait for Ready ✅
  Step 3: Delete mysql-0, create mysql-0 with v2
          Wait for Ready ✅

  End: mysql-0(v2) mysql-1(v2) mysql-2(v2)
```

### Partition — Canary Releases for StatefulSets

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 2    # only update pods with ordinal >= 2

# With partition: 2 and replicas: 3:
# → mysql-2 gets updated (ordinal 2 >= 2)
# → mysql-0, mysql-1 stay on OLD version (ordinal 0,1 < 2)

# Use case: Test new version on ONE replica before full rollout
# → Update only mysql-2 (partition: 2)
# → Monitor it for errors
# → Set partition: 1 → mysql-1 also updates
# → Set partition: 0 → all update including mysql-0 (primary)

kubectl patch statefulset mysql \
  -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
```

---

## 🌍 Real-World Use Cases

```
StatefulSet is required for:

┌─────────────────────────────────────────────────────────────┐
│  DATABASES                                                  │
│  MySQL, PostgreSQL → primary-replica setup                  │
│  MongoDB → replica set (each node needs stable identity)    │
│  Cassandra → ring topology (each node has unique position)  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  MESSAGE QUEUES                                             │
│  Kafka → brokers identified by broker.id (0,1,2)            │
│          producers/consumers connect to specific brokers    │
│  RabbitMQ → cluster nodes need stable names                 │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  COORDINATION SERVICES                                      │
│  Zookeeper → leader election by server.N in config          │
│  etcd → (ironically, etcd itself runs as StatefulSet)       │
│  Consul → cluster members need stable addresses             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  SEARCH ENGINES                                             │
│  Elasticsearch → nodes form clusters, need stable DNS       │
│  Solr → ZooKeeper-based clustering uses stable names        │
└─────────────────────────────────────────────────────────────┘
```

---

## 💻 StatefulSet Commands

```bash
# ── CREATE ────────────────────────────────────────────────
kubectl apply -f statefulset.yaml

# ── VIEW ──────────────────────────────────────────────────
kubectl get statefulsets
kubectl get sts                     # shorthand

# Output:
# NAME    READY   AGE
# mysql   3/3     5m

# See pods (note stable names!)
kubectl get pods -l app=mysql
# NAME      READY   STATUS    RESTARTS
# mysql-0   1/1     Running   0
# mysql-1   1/1     Running   0
# mysql-2   1/1     Running   0

# See PVCs created by StatefulSet
kubectl get pvc
# NAME           STATUS   VOLUME    CAPACITY
# data-mysql-0   Bound    pvc-xxx   10Gi
# data-mysql-1   Bound    pvc-yyy   10Gi
# data-mysql-2   Bound    pvc-zzz   10Gi

# Detailed info
kubectl describe sts mysql

# ── SCALE ─────────────────────────────────────────────────
kubectl scale sts mysql --replicas=5
# → Creates mysql-3, mysql-4 in order
# → Creates PVCs: data-mysql-3, data-mysql-4

kubectl scale sts mysql --replicas=1
# → Deletes mysql-4 first, then mysql-3, then mysql-2, then mysql-1
# → PVCs data-mysql-1 through data-mysql-4 REMAIN

# ── UPDATE ────────────────────────────────────────────────
kubectl set image sts/mysql mysql=mysql:8.1
kubectl rollout status sts/mysql
kubectl rollout history sts/mysql
kubectl rollout undo sts/mysql

# ── EXEC INTO SPECIFIC POD ────────────────────────────────
kubectl exec -it mysql-0 -- mysql -u root -p  # connect to primary
kubectl exec -it mysql-1 -- mysql -u root -p  # connect to replica

# ── DELETE ────────────────────────────────────────────────
kubectl delete sts mysql            # deletes pods, PVCs REMAIN
kubectl delete sts mysql --cascade=orphan  # pods keep running

# Delete PVCs manually after (if desired)
kubectl delete pvc data-mysql-0 data-mysql-1 data-mysql-2
```

---

## 🆚 StatefulSet vs Deployment — When to Use Which

```
┌──────────────────────┬────────────────┬──────────────────────┐
│ Feature              │ Deployment     │ StatefulSet          │
├──────────────────────┼────────────────┼──────────────────────┤
│ Pod naming           │ Random suffix  │ Ordered (0,1,2...)   │
│ Pod DNS              │ Via Service    │ Stable per-pod DNS   │
│ Storage              │ Shared PVC     │ Dedicated PVC/pod    │
│ Startup order        │ Random         │ Ordered (0→1→2)      │
│ Shutdown order       │ Random         │ Reverse (2→1→0)      │
│ Use case             │ Stateless apps │ Stateful apps        │
│ Examples             │ nginx, API     │ MySQL, Kafka, etcd   │
│ Scale up             │ Any replica    │ Strictly ordered     │
│ Update order         │ Random         │ Reverse (2→1→0)      │
└──────────────────────┴────────────────┴──────────────────────┘
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| Forgetting Headless Service | StatefulSet needs `serviceName` pointing to a Headless Service for DNS to work |
| Deleting StatefulSet thinking PVCs are deleted | PVCs persist by default — you must delete them manually |
| Using `clusterIP: None` in regular Service | That makes it headless — only do this for StatefulSet DNS |
| Scaling down and expecting PVCs to be deleted | PVCs remain after scale-down — you must delete manually |
| Not setting `podManagementPolicy: Parallel` for stateless-like SS | Slow ordered startup when not needed |
| Running StatefulSet without StorageClass | volumeClaimTemplates need a StorageClass to dynamically provision PVs |

---

## 🎓 CKA Exam Tips

```
✅ StatefulSet ALWAYS needs a Headless Service first
   Create the Service BEFORE the StatefulSet

✅ Headless Service = clusterIP: None

✅ Pod DNS format:
   <pod-name>.<service-name>.<namespace>.svc.cluster.local
   mysql-0.mysql.default.svc.cluster.local

✅ PVCs are named: <template-name>-<sts-name>-<ordinal>
   e.g., data-mysql-0

✅ StatefulSet shorthand: sts
   kubectl get sts
   kubectl describe sts mysql

✅ Update goes in REVERSE order: N-1, N-2, ..., 0

✅ Use partition for canary: only update pods with ordinal >= partition

✅ If asked to check which pod is primary in a DB:
   → It's always <sts-name>-0 by convention
```

---

## ❓ Interview & Scenario Questions

### Q1: What problems does StatefulSet solve that Deployment cannot?
**Answer:**
StatefulSets solve three core problems for stateful applications: (1) **Stable network identity** — pods get predictable, stable DNS names (`mysql-0.mysql`) that persist across restarts, so other pods can always find them; (2) **Dedicated persistent storage** — each pod gets its own PVC via `volumeClaimTemplates`, preventing data corruption from shared storage; (3) **Ordered operations** — pods start in order (0→1→2) and stop in reverse (2→1→0), enabling databases to properly elect primaries before replicas join.

---

### Q2: What is a Headless Service and why is it required for StatefulSets?
**Answer:**
A Headless Service is created with `clusterIP: None`. Unlike regular Services that provide a virtual IP and load balance traffic, a Headless Service creates individual DNS records for each pod. For a StatefulSet, this means each pod gets its own DNS entry: `mysql-0.mysql.default.svc.cluster.local`. This stable, predictable DNS is what enables database replication, peer discovery, and leader election. Without the Headless Service, pods would have random IPs with no stable DNS, breaking stateful application clustering. The `serviceName` field in StatefulSet spec MUST reference this Headless Service.

---

### Q3: What happens to PVCs when you delete a StatefulSet?
**Answer:**
By default, PVCs are NOT deleted when a StatefulSet is deleted. The pods are removed but the PVCs persist, preserving all data. This is intentional — data safety is prioritized over cleanup. To delete PVCs you must either: manually run `kubectl delete pvc <pvc-name>`, or configure `persistentVolumeClaimRetentionPolicy.whenDeleted: Delete` (Kubernetes 1.23+) in the StatefulSet spec to auto-delete PVCs when the StatefulSet is deleted. Similarly, when scaling down, PVCs for removed pods remain by default.

---

### Q4: Explain the `partition` field in StatefulSet rolling updates.
**Answer:**
The `partition` field enables canary-style deployments for StatefulSets. When set, only pods with an ordinal index **greater than or equal to** the partition value are updated. For example, with `replicas: 3` and `partition: 2`, only `mysql-2` is updated — `mysql-0` and `mysql-1` keep the old version. This lets you test the new version on one replica, verify it's stable, then gradually lower the partition value to roll it out further. Setting `partition: 0` allows all pods to update.

---

### Q5: Scenario — You scale a StatefulSet from 5 to 3 replicas. What happens?
**Answer:**
With the default `OrderedReady` policy, pods are deleted in reverse ordinal order:
1. `mysql-4` is deleted first — waits for termination
2. `mysql-3` is deleted next — waits for termination
3. `mysql-0`, `mysql-1`, `mysql-2` continue running

The PVCs `data-mysql-3` and `data-mysql-4` are **NOT deleted** — they remain in the cluster. If you scale back up to 5, the same pods will be created and will reattach to their original PVCs, recovering all their data. To clean up the PVCs you must manually delete them.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│               TOPIC 2.5 — STATEFULSETS                       │
├──────────────────────────────────────────────────────────────┤
│  PURPOSE: Stateful apps needing stable identity + storage    │
│  apiVersion: apps/v1    Shorthand: sts                       │
├──────────────────────────────────────────────────────────────┤
│  3 KEY GUARANTEES:                                           │
│  1. Stable pod names:    <sts>-0, <sts>-1, <sts>-2           │
│  2. Stable DNS:          <pod>.<svc>.<ns>.svc.cluster.local  │
│  3. Dedicated storage:   one PVC per pod (volumeClaimTemplates)│
├──────────────────────────────────────────────────────────────┤
│  REQUIRED HEADLESS SERVICE:                                  │
│  spec.clusterIP: None                                        │
│  StatefulSet.spec.serviceName = headless service name        │
├──────────────────────────────────────────────────────────────┤
│  PVC NAMING: <template>-<sts-name>-<ordinal>                 │
│  data-mysql-0, data-mysql-1, data-mysql-2                    │
│  PVCs survive pod deletion AND StatefulSet deletion!         │
├──────────────────────────────────────────────────────────────┤
│  OPERATIONS ORDER:                                           │
│  Create/Scale up:    0 → 1 → 2 (sequential)                  │
│  Delete/Scale down:  2 → 1 → 0 (reverse)                    │
│  Update:             2 → 1 → 0 (reverse)                    │
├──────────────────────────────────────────────────────────────┤
│  PARTITION (canary):                                         │
│  partition: 2 → only update pods with index >= 2             │
├──────────────────────────────────────────────────────────────┤
│  USE CASES:                                                  │
│  Databases: MySQL, PostgreSQL, MongoDB, Cassandra            │
│  Queues:    Kafka, RabbitMQ                                  │
│  Coord:     Zookeeper, etcd, Consul                          │
│  Search:    Elasticsearch, Solr                              │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `14-jobs-and-cronjobs.md`
> *Batch processing, one-off tasks, scheduled cron jobs — complete guide*
