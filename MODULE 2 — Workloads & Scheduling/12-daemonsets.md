# 📌 Topic 2.4 — DaemonSets

> **Module 2 — Workloads & Scheduling**
> **File:** `12-daemonsets.md`
> **CKA Weight:** ~15% (Workloads) — DaemonSets appear in CKA for logging/networking scenarios

---

## 🎯 What You'll Learn
- What a DaemonSet is and the unique problem it solves
- How DaemonSets differ from Deployments and ReplicaSets
- Full DaemonSet YAML anatomy
- Update strategies for DaemonSets
- How to target specific nodes with DaemonSets
- Real-world use cases (logging, monitoring, networking, security)
- How kube-proxy and CNI plugins use DaemonSets
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Hotel Housekeeper Analogy 🏨
> A DaemonSet is like a **hotel housekeeping policy**:
>
> - The policy says: **"Every floor must have exactly one housekeeper"**
> - When a **new floor is built** (new node added) → a housekeeper is assigned automatically
> - When a **floor is demolished** (node removed) → that housekeeper is removed automatically
> - You can **restrict** which floors need housekeepers (node selectors/affinity)
> - The hotel doesn't ask "how many housekeepers do we need?" (no replica count!)
>   — the count is determined entirely by the number of floors (nodes)
>
> This is DaemonSet: **one pod per node**, automatically, always.

---

## 🔬 What is a DaemonSet?

A DaemonSet ensures that **exactly one copy of a pod runs on every node** (or a subset of nodes) in the cluster. Unlike Deployments where you specify a replica count, a DaemonSet's replica count is always equal to the number of matching nodes.

```
┌─────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER                       │
│                                                             │
│  DaemonSet: log-collector                                   │
│  (one pod per node automatically)                           │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Node 1     │  │   Node 2     │  │   Node 3     │      │
│  │              │  │              │  │              │      │
│  │ [log-collect]│  │ [log-collect]│  │ [log-collect]│      │
│  │ [app-pod-1]  │  │ [app-pod-2]  │  │ [app-pod-3]  │      │
│  │ [app-pod-4]  │  │ [app-pod-5]  │  │ [app-pod-6]  │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                             │
│  Node 4 added to cluster:                                   │
│  ┌──────────────┐                                           │
│  │   Node 4     │  ← DaemonSet automatically creates       │
│  │ [log-collect]│    log-collector pod here! ✅             │
│  └──────────────┘                                           │
│                                                             │
│  Node 2 removed from cluster:                               │
│  → log-collector pod on Node 2 deleted automatically ✅     │
└─────────────────────────────────────────────────────────────┘
```

---

## 🆚 DaemonSet vs Deployment vs ReplicaSet

```
┌─────────────────────────────────────────────────────────────┐
│  REPLICASET / DEPLOYMENT                                    │
│  → You specify: "I want N replicas"                         │
│  → Scheduler picks which nodes to run on                    │
│  → If you have 10 nodes and replicas=3:                     │
│    only 3 nodes get a pod (scheduler picks which 3)         │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  DAEMONSET                                                  │
│  → You specify: "I want 1 pod on EVERY node"                │
│  → No replica count field!                                  │
│  → If you have 10 nodes: 10 pods (always)                   │
│  → Node added: 11 pods automatically                        │
│  → Node removed: 10 pods automatically                      │
│  → DaemonSet controller handles placement (not scheduler*)  │
└─────────────────────────────────────────────────────────────┘

* In modern K8s (1.12+), DaemonSets use the scheduler with special
  tolerations. In older versions, kubelet placed them directly.
```

---

## 📄 DaemonSet YAML — Full Anatomy

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  namespace: kube-system       # system daemons usually in kube-system
  labels:
    app: log-collector

spec:
  # ── NOTE: No "replicas" field! ──────────────────────────
  # Count is automatically = number of matching nodes

  # ── POD SELECTOR ────────────────────────────────────────
  selector:
    matchLabels:
      app: log-collector       # must match template labels

  # ── UPDATE STRATEGY ─────────────────────────────────────
  updateStrategy:
    type: RollingUpdate        # RollingUpdate | OnDelete
    rollingUpdate:
      maxSurge: 0              # for DaemonSet: surge per node
      maxUnavailable: 1        # 1 node at a time during update

  # ── MIN READY SECONDS ───────────────────────────────────
  minReadySeconds: 10

  # ── POD TEMPLATE ────────────────────────────────────────
  template:
    metadata:
      labels:
        app: log-collector     # must match selector
    spec:

      # ── NODE TARGETING ──────────────────────────────────
      nodeSelector:
        kubernetes.io/os: linux    # only run on Linux nodes

      # ── TOLERATIONS ─────────────────────────────────────
      # Important: allows pods to run on control plane nodes
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
        - key: node.kubernetes.io/not-ready
          operator: Exists
          effect: NoExecute
          tolerationSeconds: 10
        - key: node.kubernetes.io/unreachable
          operator: Exists
          effect: NoExecute
          tolerationSeconds: 10

      # ── HOST NETWORK ─────────────────────────────────────
      # Common for networking agents (kube-proxy, CNI)
      hostNetwork: true        # use node's network namespace
      hostPID: false           # share host PID namespace
      hostIPC: false           # share host IPC namespace

      # ── CONTAINERS ──────────────────────────────────────
      containers:
        - name: log-collector
          image: fluentd:v1.14
          resources:
            limits:
              cpu: "200m"
              memory: "200Mi"
            requests:
              cpu: "100m"
              memory: "100Mi"

          # ── HOST VOLUME MOUNTS ──────────────────────────
          # Common: mount host paths to read logs/system info
          volumeMounts:
            - name: varlog
              mountPath: /var/log
              readOnly: true
            - name: varlibdockercontainers
              mountPath: /var/lib/docker/containers
              readOnly: true
            - name: config
              mountPath: /etc/fluentd/

          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName  # inject node name

          securityContext:
            privileged: false
            readOnlyRootFilesystem: true

      # ── HOST PATH VOLUMES ───────────────────────────────
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: varlibdockercontainers
          hostPath:
            path: /var/lib/docker/containers
        - name: config
          configMap:
            name: fluentd-config

      # Service account for RBAC
      serviceAccountName: log-collector-sa

      # Priority class (system daemons get high priority)
      priorityClassName: system-node-critical

      # Termination grace period
      terminationGracePeriodSeconds: 30
```

---

## 🎯 Targeting Specific Nodes

### Method 1: nodeSelector (Simple)

```yaml
# Run only on nodes with specific labels
spec:
  template:
    spec:
      nodeSelector:
        kubernetes.io/os: linux        # only Linux nodes
        node-type: gpu                 # only GPU nodes
        environment: production        # only production nodes

# Label a node:
kubectl label node node1 node-type=gpu
```

### Method 2: nodeAffinity (Advanced)

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/os
                    operator: In
                    values: [linux]
                  - key: node-type
                    operator: NotIn
                    values: [windows, arm]
```

### Method 3: Tolerations (Run on Tainted Nodes)

```yaml
# Control plane nodes have a taint that prevents normal pods
# Add toleration to allow DaemonSet pods on control plane too

spec:
  template:
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
        # Also needed for older clusters:
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule

# Without this toleration:
# DaemonSet only runs on worker nodes (misses control plane)

# With this toleration:
# DaemonSet runs on ALL nodes including control plane
```

---

## 🔄 DaemonSet Update Strategies

### RollingUpdate (Default)

```
Behavior: Updates one node at a time (by default)
  → Marks old pod for deletion on node
  → Creates new pod on same node
  → Waits for new pod to be Ready
  → Moves to next node

  maxUnavailable: 1  → update 1 node at a time
  maxSurge: 0        → no extra pods (can't have 2 daemons on one node!)

# Trigger update by changing pod template:
kubectl set image daemonset/log-collector \
  log-collector=fluentd:v1.15

# Watch update progress:
kubectl rollout status daemonset/log-collector
kubectl rollout history daemonset/log-collector

# Rollback:
kubectl rollout undo daemonset/log-collector
```

### OnDelete

```yaml
updateStrategy:
  type: OnDelete
  # Pod is only updated when you MANUALLY delete it
  # New pod (with new spec) is created to replace deleted one
  # Use for: precise control over when each node updates
  #          maintenance windows per node
  #          canary updates (update only specific nodes manually)

# Update flow with OnDelete:
kubectl set image daemonset/log-collector log-collector=fluentd:v1.15
# Nothing happens yet!

# Manually delete pod on node1 to trigger update there:
kubectl delete pod log-collector-node1-xxx
# A new pod with the new image is created on node1

# Update node2 when ready:
kubectl delete pod log-collector-node2-xxx
```

---

## 🌍 Real-World DaemonSet Use Cases

### 1. Log Collection
```
Every node generates logs → every node needs a log collector

Popular tools: Fluentd, Filebeat, Logstash, Promtail
What they do: Read /var/log/* and container logs
              Ship logs to Elasticsearch, Loki, Splunk, etc.

DaemonSet because:
  → Every node has logs
  → New nodes need automatic log collection
  → Can't miss any node
```

### 2. Monitoring / Metrics
```
Every node has hardware metrics (CPU, memory, disk, network)

Popular tools: Prometheus Node Exporter, Datadog Agent, New Relic
What they do: Expose node-level metrics on port 9100
              Prometheus scrapes each node's metrics

DaemonSet because:
  → Every node has metrics to collect
  → Node Exporter needs host network access
  → Must run on ALL nodes for complete visibility
```

### 3. Container Networking (CNI Plugins)
```
CNI plugins handle pod networking on each node

Popular tools: Calico, Flannel, Weave, Cilium
What they do: Set up network interfaces for pods
              Configure routing tables on host
              Handle pod-to-pod communication

DaemonSet because:
  → Every node needs networking configured
  → Must run before any app pods can have network
  → Needs privileged access to host network namespace

# See your CNI DaemonSet:
kubectl get ds -n kube-system
```

### 4. kube-proxy
```
kube-proxy itself runs as a DaemonSet!

kubectl get ds kube-proxy -n kube-system
# NAME         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
# kube-proxy   3         3         3       3            3

What it does: Programs iptables/IPVS rules for Service routing
              Must run on every node (traffic can arrive at any node)
```

### 5. Security / Compliance Agents
```
Security scanning, intrusion detection, compliance checking

Popular tools: Falco, Aqua Security, Sysdig, CrowdStrike
What they do: Monitor syscalls and file access on each node
              Alert on suspicious activity
              Enforce security policies

DaemonSet because:
  → Security must cover ALL nodes (can't miss any)
  → Needs access to host kernel/syscalls
  → New nodes should auto-get security scanning
```

### 6. Storage Drivers
```
CSI (Container Storage Interface) node plugins

Popular tools: AWS EBS CSI, Azure Disk CSI, Ceph, NFS
What they do: Handle volume mount/unmount operations on each node
              Must be present wherever pods with volumes run

DaemonSet because:
  → Storage operations happen on the node where pod runs
  → Every node might need to mount volumes
```

---

## 💻 DaemonSet Commands Reference

```bash
# ── CREATE ────────────────────────────────────────────────
kubectl apply -f daemonset.yaml
# Note: No imperative create for DaemonSet (unlike Deployment)
# Must use YAML

# ── VIEW ──────────────────────────────────────────────────
kubectl get daemonsets
kubectl get ds                    # shorthand
kubectl get ds -n kube-system     # see system daemonsets

# Output:
# NAME          DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR
# kube-proxy    3         3         3       3            3           kubernetes.io/os=linux
# log-collector 3         3         3       3            3           <none>

# See which nodes have DaemonSet pods
kubectl get pods -l app=log-collector -o wide
# NAME                   READY   STATUS    NODE
# log-collector-node1    1/1     Running   node1
# log-collector-node2    1/1     Running   node2
# log-collector-node3    1/1     Running   node3

# Detailed info
kubectl describe daemonset log-collector

# ── UPDATE ────────────────────────────────────────────────
kubectl set image daemonset/log-collector \
  log-collector=fluentd:v1.15

kubectl rollout status daemonset/log-collector
kubectl rollout history daemonset/log-collector
kubectl rollout undo daemonset/log-collector

# ── DELETE ────────────────────────────────────────────────
kubectl delete daemonset log-collector
# Deletes all DaemonSet pods across all nodes

# ── CHECK DaemonSet on SPECIFIC NODE ─────────────────────
kubectl get pods -l app=log-collector \
  --field-selector spec.nodeName=node1
```

---

## 🔍 DaemonSet Internals — How Pods Are Placed

```
Pre-K8s 1.12:
  DaemonSet controller set pod.spec.nodeName directly
  → Bypassed scheduler entirely
  → Pods couldn't respect resource requests/limits from scheduler
  → Pods couldn't use pod priority

K8s 1.12+ (current):
  DaemonSet controller creates pods with:
  → nodeAffinity targeting a specific node
  → Scheduler schedules the pod
  BUT: Special handling ensures they bypass normal scheduling rules
  → Ignores node taints (or adds tolerations automatically)
  → Not subject to bin-packing (always go to their target node)

This is why DaemonSet pods can run on tainted nodes
(like control-plane nodes) even without explicit tolerations —
the DaemonSet controller adds default tolerations automatically
for some well-known taints.

Default tolerations automatically added by DaemonSet:
  node.kubernetes.io/not-ready:NoExecute
  node.kubernetes.io/unreachable:NoExecute
  node.kubernetes.io/disk-pressure:NoSchedule
  node.kubernetes.io/memory-pressure:NoSchedule
  node.kubernetes.io/pid-pressure:NoSchedule
  node.kubernetes.io/unschedulable:NoSchedule
  node.kubernetes.io/network-unavailable:NoSchedule (if hostNetwork)
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| Adding `replicas` field to DaemonSet | DaemonSets don't have a `replicas` field — count = nodes |
| Expecting DaemonSet on control plane by default | Control plane has `NoSchedule` taint — add toleration explicitly |
| Using Deployment when you need one-per-node | Deployments don't guarantee one per node — RS might put multiple on same node |
| Not setting resource limits | DaemonSet pods on every node can exhaust resources cluster-wide if uncapped |
| Forgetting hostPath volumes need `privileged: true` | Some host paths require privileged security context |
| DaemonSet with OnDelete never updates | With OnDelete, you MUST manually delete pods — they don't auto-update |

---

## 🎓 CKA Exam Tips

```
✅ DaemonSet has NO replicas field — count = number of nodes
   (If you forget this in exam, apiserver will reject it)

✅ DaemonSet YAML generation — no imperative create command!
   Must write YAML from scratch or use:
   kubectl create deployment temp --image=<img> --dry-run=client -o yaml
   Then change: kind: Deployment → kind: DaemonSet
                remove: replicas field
                remove: strategy field (or change to updateStrategy)

✅ Run DaemonSet on control plane node:
   Add toleration for: node-role.kubernetes.io/control-plane

✅ Check DaemonSets in kube-system (system DaemonSets):
   kubectl get ds -n kube-system
   → kube-proxy is always there!

✅ Update DaemonSet image:
   kubectl set image daemonset/<n> <container>=<image>
   kubectl rollout status daemonset/<n>

✅ Difference from Deployment:
   DaemonSet: one per node, no replicas count
   Deployment: N replicas, scheduler picks nodes
```

---

## ❓ Interview & Scenario Questions

### Q1: What is a DaemonSet and how does it differ from a Deployment?
**Answer:**
A DaemonSet ensures exactly one pod runs on every node (or a subset of nodes) in the cluster. Unlike a Deployment where you specify a fixed replica count and the scheduler distributes pods, a DaemonSet's pod count automatically equals the number of matching nodes. When a new node is added, the DaemonSet controller automatically creates a pod on it; when a node is removed, its pod is garbage collected. Deployments are for stateless apps that need N replicas; DaemonSets are for node-level agents that must run everywhere.

---

### Q2: Give 5 real-world examples of when you'd use a DaemonSet.
**Answer:**
1. **Log collectors** (Fluentd, Filebeat) — every node generates container/system logs that need shipping to a centralized store
2. **Node metrics exporters** (Prometheus Node Exporter) — collect hardware metrics (CPU, memory, disk) from every node
3. **CNI plugins** (Calico, Flannel, Cilium) — set up pod networking on each node; every node must have this to run pods
4. **kube-proxy** — programs iptables/IPVS rules for Service routing; must run on every node since traffic can arrive at any node
5. **Security agents** (Falco, Sysdig) — monitor syscalls and host-level activity for intrusion detection across every node

---

### Q3: Scenario — You deploy a DaemonSet for log collection but notice it doesn't run on the control plane node. How do you fix this?
**Answer:**
The control plane node has a `NoSchedule` taint (`node-role.kubernetes.io/control-plane:NoSchedule`) that prevents regular pods from being scheduled there. To run the DaemonSet on the control plane, add a toleration:
```yaml
spec:
  template:
    spec:
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
```
After applying this change, the DaemonSet controller will create a pod on the control plane node. Note: older clusters use `node-role.kubernetes.io/master` instead.

---

### Q4: How do DaemonSet updates work differently from Deployment updates?
**Answer:**
DaemonSets support two update strategies. **RollingUpdate** (default) updates one node at a time — it deletes the old pod on a node and creates a new one with the updated spec, then moves to the next node. `maxUnavailable` controls how many nodes can be updating simultaneously (default: 1). **OnDelete** doesn't automatically update pods — pods are only updated when you manually delete them, and the replacement uses the new spec. OnDelete is useful when you need precise control over when each node updates, such as during maintenance windows. Unlike Deployments, DaemonSets don't keep old ReplicaSets for rollback — but `kubectl rollout undo daemonset/<name>` still works.

---

### Q5: Scenario — You have a 5-node cluster. You apply a DaemonSet but only 3 pods are created. What are the possible reasons?
**Answer:**
Possible reasons why only 3 of 5 nodes got DaemonSet pods:
1. **nodeSelector mismatch** — DaemonSet has a `nodeSelector` and only 3 nodes have matching labels
2. **Node taints** — 2 nodes have taints the DaemonSet doesn't tolerate (e.g., control-plane nodes)
3. **Node conditions** — 2 nodes are in `NotReady`, `MemoryPressure`, or `DiskPressure` state
4. **nodeAffinity rules** — DaemonSet has affinity rules that exclude 2 nodes

Debug:
```bash
kubectl describe daemonset my-ds     # check NodeSelector
kubectl get nodes -o wide            # check node status
kubectl describe nodes | grep Taint  # check node taints
kubectl get pods -l app=my-ds -o wide # see which nodes have pods
```

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│               TOPIC 2.4 — DAEMONSETS                         │
├──────────────────────────────────────────────────────────────┤
│  PURPOSE: One pod per node — automatically                   │
│  apiVersion: apps/v1                                         │
│  Shorthand: ds                                               │
│  NO replicas field! Count = number of matching nodes         │
├──────────────────────────────────────────────────────────────┤
│  REAL-WORLD USES:                                            │
│  Log collectors  → Fluentd, Filebeat                         │
│  Monitoring      → Prometheus Node Exporter                  │
│  Networking      → kube-proxy, Calico, Flannel, Cilium       │
│  Security        → Falco, Sysdig, CrowdStrike                │
│  Storage         → CSI node plugins                          │
├──────────────────────────────────────────────────────────────┤
│  UPDATE STRATEGIES:                                          │
│  RollingUpdate → auto, one node at a time (default)          │
│  OnDelete      → only when you manually delete the pod       │
├──────────────────────────────────────────────────────────────┤
│  RUN ON CONTROL PLANE (add toleration):                      │
│  - key: node-role.kubernetes.io/control-plane                │
│    operator: Exists                                          │
│    effect: NoSchedule                                        │
├──────────────────────────────────────────────────────────────┤
│  KEY COMMANDS:                                               │
│  kubectl get ds [-n kube-system]                             │
│  kubectl describe ds <n>                                  │
│  kubectl set image ds/<n> <c>=<img>                       │
│  kubectl rollout status ds/<n>                            │
│  kubectl rollout undo ds/<n>                              │
├──────────────────────────────────────────────────────────────┤
│  ⚠️  NO imperative create for DaemonSet — YAML only!         │
│  Tip: generate from Deployment YAML + change kind            │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `13-statefulsets.md`
> *Ordered pods, stable network identities, persistent storage per pod*
