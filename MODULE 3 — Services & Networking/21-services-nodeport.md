# 📌 Topic 3.3 — Services: NodePort

> **Module 3 — Services & Networking**
> **File:** `21-services-nodeport.md`
> **CKA Weight:** 🔥 HIGH (~20%) — NodePort appears in CKA external access tasks

---

## 🎯 What You'll Learn
- What NodePort is and the problem it solves over ClusterIP
- How NodePort traffic flows from outside the cluster to pods
- Port ranges, port allocation and restrictions
- Full NodePort YAML anatomy
- How NodePort builds on top of ClusterIP
- NodePort vs LoadBalancer — when to use each
- Real-world use cases and limitations
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Hotel Room Extension Analogy 📞
> ClusterIP is like an **internal office extension** — only works inside the building.
>
> NodePort is like adding an **external phone line** to that extension:
> - Every hotel switchboard (node) gets the SAME external number (nodePort)
> - A caller from OUTSIDE can dial: `<any-hotel-switchboard>:<external-number>`
> - The switchboard forwards the call internally to the right extension (ClusterIP)
> - Which receptionist (pod) answers depends on load balancing
>
> Key insight: NodePort **extends** ClusterIP — it doesn't replace it.
> ClusterIP still exists and works internally.
> NodePort just opens an additional door from the outside.

---

## 🔬 What is NodePort?

NodePort exposes a Service on a **static port on every node's IP address**. Traffic arriving at `<NodeIP>:<NodePort>` is forwarded to the Service's ClusterIP, then load balanced to one of the backing pods.

```
┌─────────────────────────────────────────────────────────────┐
│                    EXTERNAL CLIENT                          │
│                 curl http://54.1.2.3:30080                  │
└─────────────────────────┬───────────────────────────────────┘
                          │ hits ANY node on nodePort
                          ▼
┌──────────────────────────────────────────────────────────────┐
│  KUBERNETES CLUSTER                                          │
│                                                             │
│  Node 1 (54.1.2.3)    Node 2 (54.1.2.4)   Node 3 (54.1.2.5)│
│  Port 30080 OPEN  ←── Port 30080 OPEN ←── Port 30080 OPEN   │
│       │                    │                    │            │
│       └────────────────────┴────────────────────┘            │
│                            │                                 │
│                    ClusterIP: 10.96.15.100                   │
│                    (internal virtual IP)                     │
│                            │                                 │
│              ┌─────────────┼─────────────┐                   │
│              ▼             ▼             ▼                   │
│          [Pod A]       [Pod B]       [Pod C]                  │
│       10.244.1.5    10.244.2.3    10.244.3.7                  │
└──────────────────────────────────────────────────────────────┘

Key: ALL 3 nodes listen on port 30080
     Even if the pod is on Node 3, hitting Node 1:30080 works!
     kube-proxy routes the traffic across nodes as needed
```

---

## 📄 NodePort YAML — Full Anatomy

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-nodeport
  namespace: default
  labels:
    app: webapp

spec:
  # ── SERVICE TYPE ────────────────────────────────────────
  type: NodePort              # ← this is what makes it NodePort

  # ── SELECTOR ────────────────────────────────────────────
  selector:
    app: webapp               # route to pods with this label

  # ── PORTS ───────────────────────────────────────────────
  ports:
    - name: http
      protocol: TCP
      port: 80                # ClusterIP port (internal access)
      targetPort: 8080        # pod/container port
      nodePort: 30080         # external port on every node
                              # Range: 30000-32767
                              # If omitted → auto-assigned in range

  # ── CLUSTER IP ──────────────────────────────────────────
  # Auto-assigned — NodePort always gets a ClusterIP too!
  # clusterIP: 10.96.15.100  (example)

  # ── EXTERNAL TRAFFIC POLICY ─────────────────────────────
  externalTrafficPolicy: Cluster  # Cluster (default) | Local

  # ── OPTIONAL: Specific node IPs to use ──────────────────
  # externalIPs:
  #   - 54.1.2.3
```

---

## 🔢 Port Ranges & Allocation

```
NodePort valid range: 30000 - 32767
(configured in kube-apiserver: --service-node-port-range)

Port assignment:
  Specified:   nodePort: 30080  → uses exactly 30080 if available
  Unspecified: nodePort omitted → auto-assigns a free port in range

Check auto-assigned port:
  kubectl get svc webapp-nodeport
  # NAME               TYPE       CLUSTER-IP     PORT(S)
  # webapp-nodeport    NodePort   10.96.15.100   80:30080/TCP
  #                                              ↑  ↑
  #                                         ClusterIP:NodePort

  Format: <port>:<nodePort>/protocol
  80:30080/TCP means:
  → ClusterIP port = 80 (internal)
  → NodePort = 30080 (external, on every node)

Change the default range (kube-apiserver flag):
  --service-node-port-range=20000-25000
  (requires apiserver restart)
```

---

## 🔄 NodePort Traffic Flow — Step by Step

```
SCENARIO: External client → Node 2:30080 → Pod A on Node 1

Step 1: Client sends request
  src: 203.0.113.5 (client IP)
  dst: 54.1.2.4:30080 (Node 2's IP, NodePort)

Step 2: Node 2's iptables intercepts
  KUBE-NODEPORTS chain:
  dst-port=30080 → jump to KUBE-SVC-webapp

Step 3: Load balancing selection
  KUBE-SVC-webapp randomly picks Pod A (10.244.1.5:8080)
  (Pod A is on Node 1, not Node 2!)

Step 4: SNAT + DNAT applied
  DNAT: dst rewritten → 10.244.1.5:8080
  SNAT: src rewritten → Node 2's internal IP (10.0.0.2)
        (so return traffic comes back to Node 2)

Step 5: Packet forwarded to Node 1
  Node 1 receives: src=10.0.0.2, dst=10.244.1.5:8080

Step 6: Pod A processes request
  Pod A sees source IP = 10.0.0.2 (Node 2's IP, not client's!)
  ← This is the SNAT side effect (source IP hidden!)

Step 7: Response returns
  Pod A → 10.0.0.2 (Node 2) → Node 2 reverse-NATs → client

⚠️  Pod sees node IP, not original client IP!
    Fix: externalTrafficPolicy: Local (see below)
```

---

## 🎯 externalTrafficPolicy — Local vs Cluster

```
externalTrafficPolicy: Cluster (DEFAULT):
  → Traffic can go to pods on ANY node
  → SNAT applied → pod sees node IP (not client IP)
  → Extra hop if pod is on different node
  → Equal load balancing across all pods

  Pros:  Simple, works always, even load distribution
  Cons:  Extra network hop possible, client IP hidden from pods

externalTrafficPolicy: Local:
  → Traffic ONLY goes to pods on THE NODE being hit
  → NO SNAT → pod sees REAL client IP ✅
  → If no local pods: connection dropped (returns error)!
  → Uneven load balancing (depends on pod placement)

  Pros:  Real client IP preserved, no extra hop
  Cons:  Pods must exist on every node (use DaemonSet)
         Load imbalance if uneven pod distribution
         If no local pod: traffic dropped!

VISUAL:

  Cluster (default):
  Client → Node1:30080 → iptables → picks Pod on Node3 → extra hop

  Local:
  Client → Node1:30080 → iptables → only pods on Node1
           (if no pods on Node1: CONNECTION REFUSED!)
```

```yaml
# Use Local for: getting real client IPs, lower latency
# Combine with DaemonSet so every node always has a pod
spec:
  type: NodePort
  externalTrafficPolicy: Local
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
```

---

## 🌍 Real-World NodePort Use Cases

```
✅ Development / Testing:
   → Quick external access without cloud load balancer
   → Share app with team: http://<node-ip>:30080
   → No cost (no cloud LB fees)

✅ On-Premise / Bare Metal:
   → Cloud LB not available
   → Put external load balancer (HAProxy, nginx) in front
   → LB → NodePort on multiple nodes

✅ CI/CD Pipelines:
   → Temporary exposure for integration tests
   → GitHub Actions runner → NodePort → test pods

✅ Learning / Lab Environments:
   → minikube: minikube service webapp-svc --url
   → kind: port-forward or NodePort

✅ Simple Home Lab:
   → Single node cluster → NodePort exposes service

❌ NOT recommended for production:
   → Port range 30000-32767 is unusual (users expect 80/443)
   → No SSL termination
   → No path-based routing
   → Need LoadBalancer or Ingress instead
```

---

## 🆚 NodePort vs ClusterIP vs LoadBalancer

```
┌──────────────────┬──────────────┬───────────────┬────────────────┐
│ Feature          │ ClusterIP    │ NodePort      │ LoadBalancer   │
├──────────────────┼──────────────┼───────────────┼────────────────┤
│ Accessible from  │ Inside only  │ Outside + In  │ Outside + In   │
│ External access  │ ❌           │ ✅ (via node) │ ✅ (via LB IP) │
│ Port             │ Any          │ 30000-32767   │ 80, 443, any   │
│ Cloud needed     │ No           │ No            │ Yes            │
│ Cost             │ Free         │ Free          │ $ (cloud LB)   │
│ HA               │ Built-in     │ Manual (need  │ Built-in       │
│                  │              │  external LB) │                │
│ Use case         │ Internal     │ Dev/test/bare │ Production     │
│                  │ microservices│ metal         │ cloud          │
└──────────────────┴──────────────┴───────────────┴────────────────┘
```

---

## 💻 NodePort Commands Reference

```bash
# ── CREATE NodePort Service ───────────────────────────────
# Imperative with specific nodePort
kubectl expose deployment webapp \
  --name=webapp-np \
  --port=80 \
  --target-port=8080 \
  --type=NodePort
# Note: can't specify nodePort imperatively → auto-assigned

# From YAML (specify nodePort)
kubectl apply -f nodeport-service.yaml

# ── VIEW ──────────────────────────────────────────────────
kubectl get svc webapp-np
# NAME        TYPE       CLUSTER-IP      PORT(S)        AGE
# webapp-np   NodePort   10.96.15.100    80:31245/TCP   5m
#                                           ↑ auto-assigned

# Get just the NodePort number
kubectl get svc webapp-np \
  -o jsonpath='{.spec.ports[0].nodePort}'
# 31245

# Get node IPs to access service
kubectl get nodes -o wide
# NAME    STATUS   INTERNAL-IP   EXTERNAL-IP
# node1   Ready    10.0.0.1      54.1.2.3
# node2   Ready    10.0.0.2      54.1.2.4

# Access the service
curl http://54.1.2.3:31245         # via Node 1
curl http://54.1.2.4:31245         # via Node 2 (same result)

# ── minikube special ──────────────────────────────────────
minikube service webapp-np          # opens browser
minikube service webapp-np --url    # prints URL
# http://127.0.0.1:31245

# ── CHANGE nodePort ───────────────────────────────────────
kubectl edit svc webapp-np
# Change spec.ports[0].nodePort value

kubectl patch svc webapp-np \
  -p '{"spec":{"ports":[{"port":80,"nodePort":30090}]}}'
```

---

## 🔧 How NodePort Builds on ClusterIP

```
NodePort service internally has TWO access points:

  1. ClusterIP access (internal):
     webapp-np.default.svc.cluster.local → 10.96.15.100:80
     (Works from any pod inside cluster)

  2. NodePort access (external):
     <any-node-IP>:30080
     (Works from anywhere that can reach a node)

iptables rules created by kube-proxy:

  KUBE-SERVICES chain:
    dst=10.96.15.100, port=80 → KUBE-SVC-webapp  ← ClusterIP rule

  KUBE-NODEPORTS chain:
    dst-port=30080 → KUBE-SVC-webapp              ← NodePort rule

  KUBE-SVC-webapp:
    33% → KUBE-SEP-pod1 (DNAT to 10.244.1.5:8080)
    33% → KUBE-SEP-pod2 (DNAT to 10.244.2.3:8080)
    33% → KUBE-SEP-pod3 (DNAT to 10.244.3.7:8080)

Both ClusterIP and NodePort traffic ends up at the same chain!
NodePort is just an additional entry point.
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| Specifying nodePort outside 30000-32767 | apiserver rejects — "provided port is not in the valid range" |
| Expecting client IP with default policy | `externalTrafficPolicy: Cluster` SNATs traffic — use `Local` for real client IPs |
| Hitting a node with no pods (Local policy) | Connection dropped — pods must exist on that node |
| Using NodePort in production | No SSL, weird ports, no path routing — use Ingress + LoadBalancer instead |
| Forgetting NodePort also creates ClusterIP | NodePort services have both a ClusterIP AND a NodePort |
| Using same nodePort for different services | Conflict — each nodePort must be unique across all NodePort services |

---

## 🎓 CKA Exam Tips

```
✅ NodePort auto-assigned if not specified:
   Check: kubectl get svc <n> -o jsonpath='{.spec.ports[0].nodePort}'

✅ Valid nodePort range: 30000-32767
   (Common trap: specifying 8080 or 80 → rejected!)

✅ Get node IP for external access:
   kubectl get nodes -o wide → INTERNAL-IP or EXTERNAL-IP

✅ Convert ClusterIP to NodePort:
   kubectl edit svc <name> → change type: ClusterIP → NodePort

✅ minikube: access NodePort service:
   minikube service <svc-name> --url

✅ Format in kubectl get svc output:
   80:30080/TCP → ClusterPort:NodePort/Protocol

✅ NodePort creates ClusterIP too:
   Both 10.96.x.x:80 AND <node-ip>:30080 work

✅ If asked for "external access" without cloud:
   NodePort is the answer for on-prem/dev
   LoadBalancer is the answer for cloud
```

---

## ❓ Interview & Scenario Questions

### Q1: What is NodePort and how does it differ from ClusterIP?
**Answer:**
NodePort extends ClusterIP by opening a specific port (in the range 30000-32767) on every node in the cluster. External traffic hitting `<any-node-IP>:<nodePort>` is forwarded to the Service's ClusterIP, then load balanced to the backing pods. ClusterIP is internal-only (only pods inside the cluster can reach it), while NodePort is accessible from outside the cluster through any node's IP. A NodePort service always gets a ClusterIP too — it builds on top of ClusterIP, adding external access.

---

### Q2: What is `externalTrafficPolicy` and when would you use `Local`?
**Answer:**
`externalTrafficPolicy` controls how external traffic is routed to pods. With **Cluster** (default), traffic can be forwarded to pods on any node, which requires SNAT — the pod sees the node's IP as source, not the real client IP. With **Local**, traffic is only forwarded to pods on the same node that received the traffic, no SNAT is applied, so the pod sees the real client IP. Use `Local` when you need the original client IP for logging, security, or geo-routing. Caveat: with `Local`, if a node has no local pods, traffic to that node is dropped — typically combine with DaemonSets to ensure pods exist on every node.

---

### Q3: Scenario — You deployed an app and want to test it from outside the cluster without setting up a cloud load balancer. What do you do?
**Answer:**
Create a NodePort Service:
```bash
kubectl expose deployment my-app \
  --name=my-app-np \
  --port=80 \
  --target-port=8080 \
  --type=NodePort

# Find the assigned NodePort
kubectl get svc my-app-np
# PORT(S): 80:31234/TCP → NodePort is 31234

# Find a node's external IP
kubectl get nodes -o wide
# EXTERNAL-IP: 54.1.2.3

# Access from outside
curl http://54.1.2.3:31234
```

---

### Q4: Why is NodePort not recommended for production use?
**Answer:**
NodePort has several production limitations: (1) **Non-standard ports** — users expect port 80/443, not 30000-32767; (2) **No SSL termination** — no built-in TLS; (3) **No path-based routing** — can't route `/api` to one service and `/web` to another; (4) **Manual HA** — you need an external load balancer pointing to multiple node IPs; (5) **Security** — opening high ports on all nodes increases attack surface; (6) **Port conflicts** — each NodePort must be unique across all services. Production workloads should use **LoadBalancer** (cloud) or **Ingress** (HTTP routing + TLS) instead.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│             TOPIC 3.3 — SERVICES: NODEPORT                   │
├──────────────────────────────────────────────────────────────┤
│  PURPOSE: External access via node IPs                       │
│  Builds ON TOP of ClusterIP (still has ClusterIP too!)       │
│  Port range: 30000-32767                                     │
├──────────────────────────────────────────────────────────────┤
│  ACCESS:                                                     │
│  Internal: <ClusterIP>:<port>   (same as ClusterIP)          │
│  External: <NodeIP>:<nodePort>  (any node works!)            │
├──────────────────────────────────────────────────────────────┤
│  PORT FIELDS:                                                │
│  port:       ClusterIP port (internal callers)               │
│  targetPort: Pod port (container)                            │
│  nodePort:   External port on every node (30000-32767)       │
├──────────────────────────────────────────────────────────────┤
│  externalTrafficPolicy:                                      │
│  Cluster (default): any node→any pod, SNAT hides client IP  │
│  Local:             same node only, preserves client IP      │
├──────────────────────────────────────────────────────────────┤
│  KEY COMMANDS:                                               │
│  kubectl expose deploy <n> --type=NodePort --port=<p>        │
│  kubectl get svc <n> -o jsonpath='{.spec.ports[0].nodePort}' │
│  kubectl get nodes -o wide  (find node IPs)                  │
│  minikube service <svc> --url                                │
├──────────────────────────────────────────────────────────────┤
│  USE CASES:                                                  │
│  ✅ Dev/test, on-prem, bare metal, CI/CD                     │
│  ❌ Production (use LoadBalancer + Ingress instead)           │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `22-services-loadbalancer.md`
> *LoadBalancer service, cloud integration, external IPs, cloud provider controllers*
