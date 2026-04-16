# 📌 Topic 3.1 — Kubernetes Networking Model

> **Module 3 — Services & Networking**
> **File:** `19-kubernetes-networking-model.md`
> **CKA Weight:** 🔥 HIGH (~20%) — Networking is the highest weighted CKA domain

---

## 🎯 What You'll Learn
- The 4 fundamental networking problems Kubernetes solves
- The flat network model — the core K8s networking philosophy
- How pod-to-pod communication works (same node + cross-node)
- What CNI is and why it exists
- How container-to-container communication works inside a pod
- Pod networking internals — veth pairs, bridges, routes
- The complete packet journey from pod A to pod B
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The City Road Network Analogy 🏙️
> Kubernetes networking is like a **city road system**:
>
> - Every **house** (pod) has a **unique street address** (pod IP)
> - Houses can talk directly to each other without going through a central hub
> - The **road network** (CNI plugin) ensures every house is reachable
> - **Street signs** (DNS/Services) help you find addresses by name
> - **Traffic lights** (Network Policies) control who can talk to whom
>
> The KEY rule of Kubernetes networking:
> **Every pod can reach every other pod directly by IP — no NAT, no proxies**
> Just like any house can send mail to any other house directly.

---

## 🔢 The 4 Networking Problems Kubernetes Solves

```
┌─────────────────────────────────────────────────────────────┐
│  Problem 1: Container-to-Container (WITHIN a pod)           │
│                                                             │
│  Solution: Shared network namespace (localhost)             │
│  Containers in same pod share the same IP                   │
│  Communicate via 127.0.0.1:<port>                           │
│  Like processes on the same machine                         │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Problem 2: Pod-to-Pod (WITHIN a node)                      │
│                                                             │
│  Solution: Virtual bridge (cbr0/docker0) on the node        │
│  All pods on same node connect to a Linux bridge            │
│  Direct L2 communication — no NAT                           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Problem 3: Pod-to-Pod (ACROSS nodes)                       │
│                                                             │
│  Solution: CNI plugin (Calico/Flannel/Weave/Cilium)         │
│  Routes packets between node subnets                        │
│  Handles encapsulation (VXLAN) or direct routing (BGP)      │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  Problem 4: Pod-to-Service (stable endpoint)                │
│                                                             │
│  Solution: kube-proxy + DNS (CoreDNS)                       │
│  Services provide stable virtual IP                         │
│  kube-proxy routes Service IP → Pod IP via iptables         │
│  CoreDNS resolves service names to Service IPs              │
└─────────────────────────────────────────────────────────────┘
```

---

## 📐 The Kubernetes Networking Requirements (The Contract)

The Kubernetes networking model has exactly **3 rules** that every CNI plugin must satisfy:

```
RULE 1: Pod-to-Pod (same node) — NO NAT
  All pods on a node must be able to communicate with each other
  directly using their pod IP addresses.
  No Network Address Translation (NAT) required.

RULE 2: Pod-to-Pod (cross-node) — NO NAT
  All pods must be able to communicate with all other pods
  on any node in the cluster, directly using pod IP addresses.
  No NAT. Pod A sees Pod B's REAL IP (not a translated one).

RULE 3: Node-to-Pod — NO NAT
  Agents on a node (like kubelet, system daemons) must be able
  to communicate with all pods on that node.

WHY NO NAT?
  → With NAT, the sender's IP changes → complex debugging
  → Applications would see "wrong" source IPs
  → Makes security policies unreliable
  → Makes pod identity verification impossible
  → Direct IP routing = simpler, more predictable, more secure
```

---

## 🗺️ The Flat Network Model

```
TRADITIONAL NETWORK (with NAT):

  ┌─────────────────────┐      ┌─────────────────────┐
  │  Server 1           │      │  Server 2           │
  │  Private: 10.0.1.5  │      │  Private: 10.0.2.5  │
  │  Public:  54.1.2.3  │      │  Public:  54.4.5.6  │
  └──────────┬──────────┘      └──────────┬──────────┘
             │  NAT                        │  NAT
             └──────────► Internet ◄───────┘
  10.0.1.5 talks to 10.0.2.5 via public IPs + NAT
  Server 2 sees source IP = 54.1.2.3 (not the real 10.0.1.5)

KUBERNETES FLAT NETWORK (no NAT):

  Node 1 (10.0.0.1)          Node 2 (10.0.0.2)
  ┌──────────────────┐        ┌──────────────────┐
  │ Pod A            │        │ Pod C            │
  │ IP: 10.244.1.2   │───────►│ IP: 10.244.2.3   │
  │                  │        │                  │
  │ Pod B            │        │ Pod D            │
  │ IP: 10.244.1.3   │        │ IP: 10.244.2.4   │
  └──────────────────┘        └──────────────────┘

  Pod A (10.244.1.2) → Pod C (10.244.2.3)
  Pod C sees source IP = 10.244.1.2 (Pod A's REAL IP!)
  No NAT! Direct routing via CNI plugin.

  Each node gets a SUBNET of pod IPs:
  Node 1: 10.244.1.0/24  (pods get IPs from this range)
  Node 2: 10.244.2.0/24  (pods get IPs from this range)
  Node 3: 10.244.3.0/24
```

---

## 🔧 Pod Networking Internals — How It Actually Works

### Container-to-Container (Within Pod)

```
┌─────────────────────────────────────────────────────────────┐
│  POD (10.244.1.5)                                           │
│                                                             │
│  ┌─────────────┐    ┌─────────────┐    ┌──────────────┐    │
│  │   pause     │    │   nginx     │    │  log-shipper │    │
│  │ (infra)     │    │ (port 80)   │    │ (port 9090)  │    │
│  └──────┬──────┘    └──────┬──────┘    └──────┬───────┘    │
│         │                  │                   │            │
│         └──────────────────┴───────────────────┘            │
│                      eth0 (10.244.1.5)                      │
│                   Shared Network Namespace                  │
│                                                             │
│  nginx → log-shipper:  curl http://localhost:9090           │
│  (same IP, same eth0, same localhost — like same machine)   │
└─────────────────────────────────────────────────────────────┘
```

### Pod-to-Pod (Same Node)

```
Node 1
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Pod A (10.244.1.2)          Pod B (10.244.1.3)             │
│  ┌──────────────┐            ┌──────────────┐               │
│  │    eth0      │            │    eth0      │               │
│  └──────┬───────┘            └──────┬───────┘               │
│         │                           │                       │
│    veth0│                      veth1│                       │
│    (virtual ethernet pair)         (virtual ethernet pair)  │
│         │                           │                       │
│    ┌────┴───────────────────────────┴────┐                  │
│    │          cbr0 (Linux Bridge)        │                  │
│    │          10.244.1.1                 │                  │
│    └─────────────────────────────────────┘                  │
│                         │                                   │
│                    eth0 (node)                              │
│                    10.0.0.1                                 │
└─────────────────────────────────────────────────────────────┘

Packet flow: Pod A → Pod B (same node)
  1. Pod A sends packet: src=10.244.1.2, dst=10.244.1.3
  2. Packet leaves Pod A's eth0 via veth pair
  3. Arrives at cbr0 bridge (like a virtual switch)
  4. Bridge sees dst=10.244.1.3 is on veth1
  5. Packet delivered to Pod B's eth0
  6. Pod B receives packet — sees src=10.244.1.2 (no NAT!) ✅
```

### Pod-to-Pod (Cross-Node)

```
Node 1 (10.0.0.1)                    Node 2 (10.0.0.2)
┌──────────────────────┐              ┌──────────────────────┐
│ Pod A (10.244.1.2)   │              │ Pod C (10.244.2.3)   │
│ ┌────┐               │              │ ┌────┐               │
│ │eth0│               │              │ │eth0│               │
│ └──┬─┘               │              │ └──┬─┘               │
│    │veth              │              │    │veth              │
│ ┌──┴──────────────┐  │              │ ┌──┴──────────────┐  │
│ │  cbr0 bridge    │  │              │ │  cbr0 bridge    │  │
│ │  10.244.1.1     │  │              │ │  10.244.2.1     │  │
│ └──────┬──────────┘  │              │ └──────┬──────────┘  │
│        │             │              │        │             │
│     ┌──┴──┐          │              │     ┌──┴──┐          │
│     │eth0 │          │              │     │eth0 │          │
│     │node │          │              │     │node │          │
└─────┴──┬──┴──────────┘              └─────┴──┬──┴──────────┘
         │                                      │
         └──────────── Physical/Overlay ─────────┘
                        Network
                     (CNI handles this!)

Packet flow: Pod A → Pod C (cross-node)
  1. Pod A sends: src=10.244.1.2, dst=10.244.2.3
  2. Pod A's route table: 10.244.2.0/24 → via node2 (10.0.0.2)
  3. Packet goes up to Node 1's eth0
  4. CNI plugin intercepts:
     - Flannel: VXLAN encapsulation → add outer header
       outer: src=10.0.0.1, dst=10.0.0.2 (node IPs)
       inner: src=10.244.1.2, dst=10.244.2.3 (pod IPs)
     - Calico: BGP routing → no encapsulation needed
       just route 10.244.2.0/24 via 10.0.0.2
  5. Packet arrives at Node 2
  6. CNI decapsulates (if VXLAN) → delivers to cbr0
  7. cbr0 routes to Pod C's veth pair
  8. Pod C receives: src=10.244.1.2, dst=10.244.2.3 (NO NAT!) ✅
```

---

## 🔌 CNI — Container Network Interface

### What is CNI?

```
CNI = Container Network Interface
  → A standard specification (not a product)
  → Defines how networking plugins work
  → kubelet calls CNI plugin when pod is created/deleted

  kubelet creates pod → needs network
       │
       ▼
  Calls CNI plugin binary: /opt/cni/bin/calico
  Passes: container ID, network namespace path, config
       │
       ▼
  CNI plugin:
  1. Creates veth pair for pod
  2. Assigns IP from pod CIDR range
  3. Sets up routes
  4. Returns IP info to kubelet

  kubelet → tells container runtime → starts container with network
```

### CNI Plugin Locations

```bash
# CNI binary plugins
ls /opt/cni/bin/
# bridge, calico, flannel, host-local, loopback, portmap, ...

# CNI configuration
ls /etc/cni/net.d/
# 10-calico.conflist
# or 10-flannel.conflist
# First file alphabetically is used

# View CNI config
cat /etc/cni/net.d/10-calico.conflist
```

### Popular CNI Plugins Comparison

```
┌──────────────┬──────────────────────────────────────────────┐
│ CNI Plugin   │ Characteristics                              │
├──────────────┼──────────────────────────────────────────────┤
│ Flannel      │ Simple, VXLAN overlay                        │
│              │ No NetworkPolicy support (need Calico+Flannel)│
│              │ Good for: dev/test, simple clusters           │
├──────────────┼──────────────────────────────────────────────┤
│ Calico       │ BGP routing (no encapsulation overhead)       │
│              │ Full NetworkPolicy support                    │
│              │ High performance, scalable                    │
│              │ Good for: production, multi-cloud             │
├──────────────┼──────────────────────────────────────────────┤
│ Weave        │ Mesh network, encrypted by default            │
│              │ Works across multi-cloud without BGP          │
│              │ Good for: multi-cloud, encrypted comms        │
├──────────────┼──────────────────────────────────────────────┤
│ Cilium       │ eBPF-based (kernel-level, very fast)          │
│              │ Replaces kube-proxy entirely                  │
│              │ Advanced NetworkPolicy (L7 aware)             │
│              │ Good for: high-performance, microservices     │
└──────────────┴──────────────────────────────────────────────┘
```

---

## 📦 IP Address Management (IPAM)

```
How pods get their IP addresses:

Cluster CIDR: 10.244.0.0/16  (pod network range)
  → Defined in kube-controller-manager: --cluster-cidr

Node CIDR size: /24 per node (default)
  → Defined in: --node-cidr-mask-size=24

Allocation:
  Node 1 gets: 10.244.0.0/24  (256 pod IPs)
  Node 2 gets: 10.244.1.0/24  (256 pod IPs)
  Node 3 gets: 10.244.2.0/24  (256 pod IPs)
  ...

  When pod is created on Node 1:
  CNI plugin's IPAM assigns next available IP from 10.244.0.0/24
  e.g., 10.244.0.5

Pod IP lifecycle:
  Pod created → IP assigned from node's subnet
  Pod deleted → IP returned to pool
  Pod rescheduled → gets a NEW IP (even if same node!)
  → This is why you need Services (stable IPs)!
```

---

## 🔍 Key Networking Commands

```bash
# ── POD NETWORKING ────────────────────────────────────────
# See pod IPs
kubectl get pods -o wide
# NAME        READY  STATUS   IP           NODE
# webapp-1    1/1    Running  10.244.1.5   node1
# webapp-2    1/1    Running  10.244.2.7   node2

# Get pod IP directly
kubectl get pod my-pod -o jsonpath='{.status.podIP}'

# ── NODE NETWORKING ───────────────────────────────────────
# See node IPs and pod CIDR per node
kubectl get nodes -o wide
kubectl get nodes -o jsonpath=\
  '{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'

# ── INSIDE POD NETWORKING ─────────────────────────────────
# Check pod's network config
kubectl exec my-pod -- ip addr show eth0
kubectl exec my-pod -- ip route show
kubectl exec my-pod -- cat /etc/resolv.conf

# Test connectivity
kubectl exec pod-a -- ping 10.244.2.7           # ping pod-b by IP
kubectl exec pod-a -- curl http://10.244.2.7:80 # HTTP to pod-b
kubectl exec pod-a -- nslookup my-service        # DNS lookup

# ── CNI INFO ──────────────────────────────────────────────
ls /opt/cni/bin/                    # CNI binaries
ls /etc/cni/net.d/                  # CNI configs
cat /etc/cni/net.d/*.conf*          # active CNI config

# ── NODE-LEVEL NETWORKING ─────────────────────────────────
# SSH into node, then:
ip route show         # routing table (shows pod CIDR routes)
ip link show          # network interfaces (veth pairs, cbr0, etc.)
bridge link show      # bridge connections
iptables -t nat -L -n # NAT rules (kube-proxy output)
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| Pod IPs are stable | Pod IPs change on every restart — always use Services |
| All CNI plugins support NetworkPolicy | Flannel alone doesn't — need Calico or Cilium |
| Pods on same node use host network | Pods have their OWN network namespace — isolated from host |
| Container ports need opening in firewall | K8s pods are directly routable (no firewall between pods) |
| Pod CIDR and Service CIDR can overlap | They must NOT overlap with each other or node network |
| CNI plugin is optional | Without CNI, pods get no network — kubectl exec will fail |

---

## 🎓 CKA Exam Tips

```
✅ Know the 3 rules of K8s networking:
   1. Pod-to-pod (same node): direct, no NAT
   2. Pod-to-pod (cross-node): direct, no NAT
   3. Node-to-pod: direct, no NAT

✅ Pod IP is ephemeral — changes on restart
   → Never hardcode pod IPs — use Service names

✅ CNI config location:
   /etc/cni/net.d/       ← config files
   /opt/cni/bin/         ← binary plugins

✅ If pods can't communicate:
   1. Check CNI is installed: ls /etc/cni/net.d/
   2. Check CNI pods running: kubectl get pods -n kube-system
   3. Test with: kubectl exec pod-a -- ping <pod-b-IP>

✅ Pod CIDR: assigned per node from cluster CIDR
   kubectl get nodes -o jsonpath='{.items[*].spec.podCIDR}'

✅ Container-to-container in same pod:
   Use localhost:<port> — they share network namespace
```

---

## ❓ Interview & Scenario Questions

### Q1: Explain the Kubernetes flat networking model.
**Answer:**
The Kubernetes flat network model requires that every pod in the cluster can communicate directly with every other pod using its pod IP address, without any NAT (Network Address Translation). Each pod gets a unique IP from its node's subnet, and the CNI plugin configures routing so packets travel directly between pods. This means Pod A sees Pod B's real IP as the source address, making security policies, logging, and debugging accurate and reliable. The model has three rules: pod-to-pod on same node (no NAT), pod-to-pod across nodes (no NAT), and node-to-pod (no NAT).

---

### Q2: What is CNI and why does Kubernetes need it?
**Answer:**
CNI (Container Network Interface) is a standard specification that defines how networking plugins integrate with container runtimes. Kubernetes needs CNI because it deliberately doesn't include networking implementation — it only defines the requirements (flat network, no NAT). CNI plugins (Calico, Flannel, Cilium, Weave) implement the actual networking: assigning pod IPs from a CIDR range, creating virtual ethernet pairs, setting up Linux bridge or routing, and enabling cross-node pod communication. When kubelet creates a pod, it calls the CNI plugin binary to set up networking before starting the container.

---

### Q3: How does a packet travel from Pod A on Node 1 to Pod B on Node 2?
**Answer:**
1. Pod A sends packet with src=10.244.1.2, dst=10.244.2.3
2. Packet exits Pod A's eth0 via a veth pair to Node 1's bridge (cbr0)
3. Node 1's routing table says: "10.244.2.0/24 is via Node 2 (10.0.0.2)"
4. Packet is handed to the CNI plugin — Flannel wraps it in VXLAN (UDP encapsulation with outer src=Node1-IP, dst=Node2-IP), Calico uses BGP to route directly without encapsulation
5. Packet traverses the physical/cloud network between nodes
6. Node 2 receives packet, CNI decapsulates if needed
7. Node 2's cbr0 bridge delivers to Pod B's veth pair
8. Pod B receives the packet — source IP is still 10.244.1.2 (no NAT!)

---

### Q4: Why are pod IPs considered ephemeral and what is the solution?
**Answer:**
Pod IPs are assigned when a pod is created and released when it's deleted. Every time a pod restarts (crash, rolling update, node eviction), it gets a completely different IP address. This makes direct pod IP communication unreliable — hardcoding a pod IP would break on the first pod restart. The solution is **Services** — a Service provides a stable virtual IP (ClusterIP) that never changes, regardless of how many times pods behind it restart. kube-proxy and CoreDNS work together so applications can reach pods via the stable Service DNS name/IP rather than individual pod IPs.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│       TOPIC 3.1 — KUBERNETES NETWORKING MODEL                │
├──────────────────────────────────────────────────────────────┤
│  3 RULES (no NAT ever):                                      │
│  1. Pod ↔ Pod (same node)   → direct via bridge              │
│  2. Pod ↔ Pod (cross-node)  → via CNI plugin routing         │
│  3. Node ↔ Pod              → direct                         │
├──────────────────────────────────────────────────────────────┤
│  4 NETWORKING PROBLEMS:                                      │
│  Container↔Container → shared network NS (localhost)         │
│  Pod↔Pod same node   → Linux bridge (cbr0)                   │
│  Pod↔Pod cross-node  → CNI plugin (overlay/BGP)              │
│  Pod↔Service         → kube-proxy (iptables) + CoreDNS       │
├──────────────────────────────────────────────────────────────┤
│  CNI:                                                        │
│  Config:   /etc/cni/net.d/                                   │
│  Binaries: /opt/cni/bin/                                     │
│  Flannel: simple VXLAN overlay                               │
│  Calico:  BGP routing, NetworkPolicy                         │
│  Cilium:  eBPF, replaces kube-proxy                          │
├──────────────────────────────────────────────────────────────┤
│  IP ADDRESSING:                                              │
│  Cluster CIDR: e.g. 10.244.0.0/16 (all pod IPs)             │
│  Per-node:     e.g. 10.244.1.0/24 (256 IPs per node)        │
│  Pod IPs are EPHEMERAL → use Services for stability          │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `20-services-clusterip.md`
> *ClusterIP services, kube-proxy, iptables/IPVS, DNS resolution — deep dive*
