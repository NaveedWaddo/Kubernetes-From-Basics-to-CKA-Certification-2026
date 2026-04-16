# 📌 Topic 3.2 — Services & ClusterIP

> **Module 3 — Services & Networking**
> **File:** `20-services-clusterip.md`
> **CKA Weight:** 🔥 HIGH (~20%) — Services are in almost every CKA networking task

---

## 🎯 What You'll Learn
- What a Service is and the problem it solves
- ClusterIP — the default service type, fully explained
- How kube-proxy programs iptables for Services
- How CoreDNS resolves Service names
- Endpoints and EndpointSlices — the glue between Service and pods
- Service selectors — how Services find pods
- Headless Services — ClusterIP: None
- Full YAML anatomy with every field explained
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Reception Desk Analogy 🏢
> Think of a Kubernetes Service like a **company reception desk**:
>
> - The **reception desk** (Service) has a **stable phone number** (ClusterIP)
> - Callers (other pods) always dial the SAME number regardless of who's working
> - Behind the desk are **multiple receptionists** (pods) who actually answer
> - If a receptionist quits (pod crashes), a new one is hired — same desk, same number
> - The reception desk **distributes calls** among available receptionists (load balancing)
> - Callers don't need to know which receptionist they'll get — they just dial the number
>
> Without a Service (reception desk):
> - You'd need to know each receptionist's personal mobile (pod IP)
> - If they change phones (pod restart = new IP), you'd lose contact
> - Load balancing would be manual

---

## 🔬 Why Services Exist

```
THE PROBLEM:

  Pod IPs are ephemeral — change every restart:
  ┌────────────────────────────────────────────┐
  │  Before restart:  webapp pod → 10.244.1.5  │
  │  After restart:   webapp pod → 10.244.1.9  │
  │  (different IP!)                            │
  └────────────────────────────────────────────┘

  With 3 replicas, which IP do you connect to?
  frontend ──? 10.244.1.5 (might be dead)
           ──? 10.244.2.3 (still alive)
           ──? 10.244.3.7 (still alive)

THE SOLUTION: Service (ClusterIP)

  Service "webapp-svc" → ClusterIP: 10.96.15.100 (STABLE, never changes)

  frontend ──► 10.96.15.100 (Service)
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
     10.244.1.5  10.244.2.3  10.244.3.7
     (Pod A)     (Pod B)     (Pod C)

  Service provides:
  ✅ Stable IP (never changes while Service exists)
  ✅ Stable DNS name (webapp-svc.default.svc.cluster.local)
  ✅ Load balancing across all healthy pods
  ✅ Automatic pod discovery via label selectors
  ✅ Only routes to READY pods (respects readiness probes)
```

---

## 📄 Service YAML — Full ClusterIP Anatomy

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-svc                    # DNS name of the service
  namespace: default
  labels:
    app: webapp

spec:
  # ── SERVICE TYPE ────────────────────────────────────────
  type: ClusterIP                     # ClusterIP | NodePort | LoadBalancer | ExternalName
                                       # ClusterIP is the DEFAULT if not specified

  # ── SELECTOR ────────────────────────────────────────────
  selector:
    app: webapp                       # route traffic to pods with THIS label
    tier: frontend                    # ALL selector labels must match (AND logic)

  # ── PORTS ───────────────────────────────────────────────
  ports:
    - name: http                      # optional name (required if multiple ports)
      protocol: TCP                   # TCP | UDP | SCTP
      port: 80                        # port on the SERVICE (what callers use)
      targetPort: 8080                # port on the POD (what containers listen on)
      # port != targetPort is common!
      # callers → Service:80 → Pod:8080

    - name: metrics
      protocol: TCP
      port: 9090
      targetPort: prometheus-port     # can reference named port in pod spec!

  # ── CLUSTER IP ──────────────────────────────────────────
  clusterIP: ""                       # leave empty → auto-assigned
  # clusterIP: 10.96.15.100          # OR specify exact IP (rare, for migration)
  # clusterIP: None                  # → Headless Service (no virtual IP)

  # ── SESSION AFFINITY ────────────────────────────────────
  sessionAffinity: None               # None | ClientIP
  # ClientIP → same client always goes to same pod (sticky sessions)
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800           # 3 hours sticky session

  # ── IP FAMILIES ─────────────────────────────────────────
  ipFamilies: [IPv4]                  # IPv4 | IPv6 | dual-stack
  ipFamilyPolicy: SingleStack         # SingleStack | PreferDualStack | RequireDualStack
```

---

## ⚙️ How ClusterIP Works — The Full Flow

### Step 1: Service Creation

```
kubectl apply -f service.yaml
         │
         ▼
apiserver validates and stores Service in etcd
  → Service: webapp-svc
  → ClusterIP: 10.96.15.100 (auto-assigned from service CIDR)
  → selector: app=webapp
```

### Step 2: Endpoints Controller Creates Endpoints

```
Endpoints Controller (in kube-controller-manager) watches:
  → Services
  → Pods matching Service selector

  Finds pods with label app=webapp:
  → pod-1: 10.244.1.5:8080 (Ready ✅)
  → pod-2: 10.244.2.3:8080 (Ready ✅)
  → pod-3: 10.244.3.7:8080 (NOT Ready ❌ — excluded!)

  Creates Endpoints object:
  apiVersion: v1
  kind: Endpoints
  metadata:
    name: webapp-svc      ← same name as Service!
  subsets:
    - addresses:
        - ip: 10.244.1.5
        - ip: 10.244.2.3
      ports:
        - port: 8080

  kubectl get endpoints webapp-svc
  # NAME        ENDPOINTS                         AGE
  # webapp-svc  10.244.1.5:8080,10.244.2.3:8080   5m
```

### Step 3: kube-proxy Programs iptables

```
kube-proxy watches apiserver for Service + Endpoints changes
  → Sees: webapp-svc ClusterIP=10.96.15.100 → pods [10.244.1.5, 10.244.2.3]

  Programs iptables rules on EVERY node:

  Chain KUBE-SERVICES:
    dst=10.96.15.100, port=80 → jump to KUBE-SVC-webapp

  Chain KUBE-SVC-webapp (load balancing):
    → 50% probability: jump to KUBE-SEP-pod1 (10.244.1.5:8080)
    → 50% probability: jump to KUBE-SEP-pod2 (10.244.2.3:8080)

  Chain KUBE-SEP-pod1:
    DNAT: rewrite dst from 10.96.15.100:80 → 10.244.1.5:8080

  Chain KUBE-SEP-pod2:
    DNAT: rewrite dst from 10.96.15.100:80 → 10.244.2.3:8080
```

### Step 4: DNS Registration

```
CoreDNS automatically creates DNS record:
  webapp-svc.default.svc.cluster.local → 10.96.15.100

  Any pod in cluster can resolve:
  nslookup webapp-svc.default.svc.cluster.local
  → 10.96.15.100

  From same namespace:
  nslookup webapp-svc
  → 10.96.15.100 (same result, short name works!)
```

### Step 5: Traffic Flow

```
Frontend pod (10.244.1.8) wants to reach webapp:

  curl http://webapp-svc:80

  1. DNS lookup: webapp-svc → 10.96.15.100
  2. Packet: src=10.244.1.8, dst=10.96.15.100:80
  3. Packet hits iptables on the node
  4. Rule matches KUBE-SVC-webapp
  5. Random selection: pick pod2 (50% chance)
  6. DNAT: dst rewritten from 10.96.15.100:80 → 10.244.2.3:8080
  7. Packet routed to node2 (where pod2 lives)
  8. pod2 receives: src=10.244.1.8, dst=10.244.2.3:8080
  9. pod2 responds: src=10.244.2.3, dst=10.244.1.8
  10. iptables connection tracking: reverse DNAT on return
  11. Frontend receives response from 10.96.15.100 (virtual IP)

  The frontend only ever sees 10.96.15.100 — invisible load balancing! ✅
```

---

## 🌐 CoreDNS — Service DNS Deep Dive

```
EVERY pod has /etc/resolv.conf:
  nameserver 10.96.0.10        ← CoreDNS ClusterIP
  search default.svc.cluster.local svc.cluster.local cluster.local
  options ndots:5

The "search" domains enable short names:
  You type:    webapp-svc
  DNS tries:   webapp-svc.default.svc.cluster.local  ← FOUND ✅

Full DNS name format:
  <service-name>.<namespace>.svc.<cluster-domain>
  webapp-svc.default.svc.cluster.local

  Cross-namespace:
  webapp-svc.production.svc.cluster.local

  Shorter forms (within same namespace):
  webapp-svc                        → works (via search domains)
  webapp-svc.default                → works
  webapp-svc.default.svc            → works
  webapp-svc.default.svc.cluster.local → full FQDN, always works

kubectl get svc -n kube-system | grep dns
# NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
# kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP

kubectl get pods -n kube-system | grep coredns
# coredns-xxx   1/1   Running
```

---

## 🔗 Endpoints & EndpointSlices

```
ENDPOINTS (original, all in one object):
  → Lists ALL pod IPs for a Service in one object
  → Problem at scale: 1 object with 1000 pod IPs
  → Every kube-proxy watches this object → massive updates

ENDPOINTSLICES (modern, K8s 1.21+ default):
  → Splits endpoints into slices of 100 pods each
  → 1000 pods → 10 EndpointSlices
  → Only changed slices are updated → much more efficient
  → Each slice has targetRef pointing to Pod objects

kubectl get endpoints webapp-svc
kubectl get endpointslices -l kubernetes.io/service-name=webapp-svc

# Endpoints show:
# NAME        ENDPOINTS                         AGE
# webapp-svc  10.244.1.5:8080,10.244.2.3:8080   5m

# If endpoints are empty (<none>):
# → Service selector doesn't match any pods
# → No pods with matching labels are Ready
```

---

## 🔧 Headless Services (ClusterIP: None)

```yaml
# Headless Service — no virtual IP, direct pod DNS
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  clusterIP: None         # ← HEADLESS!
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
```

```
REGULAR ClusterIP:
  mysql.default.svc.cluster.local → 10.96.15.100 (virtual IP)
  ALL traffic load-balanced via kube-proxy

HEADLESS (clusterIP: None):
  mysql.default.svc.cluster.local → returns ALL pod IPs directly
  (DNS returns: 10.244.1.5, 10.244.2.3, 10.244.3.7)
  Client can choose which pod to connect to

  Per-pod DNS (for StatefulSets):
  mysql-0.mysql.default.svc.cluster.local → 10.244.1.5 only
  mysql-1.mysql.default.svc.cluster.local → 10.244.2.3 only
  mysql-2.mysql.default.svc.cluster.local → 10.244.3.7 only

Use headless for:
  → StatefulSets (per-pod DNS identity)
  → Client-side load balancing (gRPC, databases)
  → Service discovery without load balancer
```

---

## 💻 Service Commands Reference

```bash
# ── CREATE SERVICE ────────────────────────────────────────
# Expose existing deployment (imperative)
kubectl expose deployment webapp \
  --name=webapp-svc \
  --port=80 \
  --target-port=8080 \
  --type=ClusterIP

# Generate YAML
kubectl expose deployment webapp \
  --port=80 --target-port=8080 \
  --dry-run=client -o yaml > service.yaml

# Create from YAML
kubectl apply -f service.yaml

# ── VIEW SERVICES ─────────────────────────────────────────
kubectl get services
kubectl get svc                     # shorthand

# Output:
# NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   10d
# webapp-svc   ClusterIP   10.96.15.100    <none>        80/TCP    5m

kubectl describe svc webapp-svc
# Shows: Selector, Endpoints, Ports

# ── CHECK ENDPOINTS ───────────────────────────────────────
kubectl get endpoints webapp-svc
kubectl get ep webapp-svc           # shorthand
kubectl get endpointslices

# If endpoints show <none> → selector/label mismatch!
# Fix: verify pod labels match service selector
kubectl get pods --show-labels
kubectl get svc webapp-svc -o jsonpath='{.spec.selector}'

# ── TEST CONNECTIVITY ─────────────────────────────────────
# From inside the cluster (via a test pod)
kubectl run test --image=busybox --rm -it -- \
  wget -qO- http://webapp-svc:80

kubectl run test --image=busybox --rm -it -- \
  nslookup webapp-svc

# Port-forward to test from local machine
kubectl port-forward svc/webapp-svc 8080:80
# Then: curl http://localhost:8080

# ── DELETE SERVICE ────────────────────────────────────────
kubectl delete svc webapp-svc
kubectl delete -f service.yaml
```

---

## 🔍 Service CIDR vs Pod CIDR

```
Pod CIDR:     10.244.0.0/16  → Pod IPs (assigned per node subnet)
Service CIDR: 10.96.0.0/12   → Service ClusterIPs (virtual, no real interface)

These MUST NOT overlap!

Find Service CIDR:
kubectl cluster-info dump | grep service-cluster-ip-range
# OR
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep service-cluster-ip-range

Default kubernetes Service:
kubectl get svc kubernetes
# NAME         TYPE        CLUSTER-IP   PORT(S)
# kubernetes   ClusterIP   10.96.0.1    443/TCP
# ↑ This is the apiserver's Service (ALWAYS at .1 of service CIDR)
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| Service selector doesn't match pod labels | Endpoints will be empty → Service routes to nowhere |
| `port` vs `targetPort` confusion | `port` = Service port (callers use), `targetPort` = container port |
| ClusterIP is reachable from outside cluster | ClusterIP is ONLY reachable from within the cluster |
| Endpoints show pod but Service doesn't work | Check if pod is READY — unready pods are excluded from endpoints |
| Deleting Service deletes pods | Services and pods are independent — deleting Service doesn't delete pods |
| SessionAffinity for stateless apps | SessionAffinity breaks load balancing — only use for truly stateful HTTP |

---

## 🎓 CKA Exam Tips

```
✅ Expose deployment quickly:
   kubectl expose deployment <n> --port=<p> --target-port=<tp>

✅ Service type default is ClusterIP (can omit type field)

✅ If Service not working → check Endpoints first:
   kubectl get endpoints <svc-name>
   If <none> → label mismatch between selector and pods

✅ Test Service from inside cluster:
   kubectl run test --image=busybox --rm -it -- \
     wget -qO- http://<service-name>:<port>

✅ DNS name format:
   <svc>.<namespace>.svc.cluster.local
   Short form works in same namespace: just <svc-name>

✅ Port-forward for local testing:
   kubectl port-forward svc/<name> <local-port>:<svc-port>

✅ Headless service = clusterIP: None
   Used for StatefulSets and direct pod DNS

✅ Check CoreDNS is running:
   kubectl get pods -n kube-system | grep coredns
```

---

## ❓ Interview & Scenario Questions

### Q1: What is a Kubernetes Service and why is it needed?
**Answer:**
A Service is a stable network endpoint that provides reliable access to a dynamic set of pods. It's needed because pod IPs are ephemeral — they change every time a pod restarts. A Service provides a permanent virtual IP (ClusterIP) and DNS name that clients can use regardless of how many times the underlying pods restart or scale. The Service also provides load balancing across all healthy pods matching its label selector, and automatically updates routing when pods are added or removed.

---

### Q2: Explain how iptables rules work with ClusterIP Services.
**Answer:**
kube-proxy watches the apiserver for Service and Endpoints changes and programs iptables rules on every node. When traffic is destined for a Service's ClusterIP, iptables intercepts it in the `KUBE-SERVICES` chain and redirects to the appropriate `KUBE-SVC-*` chain. This chain uses probabilistic rules (using `--probability` in `statistic` module) to select one of the backing pods with equal probability. The selected pod's `KUBE-SEP-*` chain applies DNAT — rewriting the destination from the Service ClusterIP to the actual pod IP. Connection tracking ensures return packets get reverse-NATed automatically.

---

### Q3: Scenario — You created a Service but `kubectl get endpoints` shows `<none>`. What's wrong?
**Answer:**
Empty endpoints mean the Service selector doesn't match any Ready pods. Debug steps:
```bash
# Check Service selector
kubectl describe svc my-svc | grep Selector
# Selector: app=webapp

# Check pod labels
kubectl get pods --show-labels
# NAME      LABELS
# my-pod    app=web-app  ← MISMATCH! webapp vs web-app

# Fix options:
# Option 1: Fix the pod label
kubectl label pod my-pod app=webapp --overwrite

# Option 2: Fix the Service selector
kubectl edit svc my-svc  # change selector to match actual pod labels

# Also check pod readiness
kubectl get pods  # pod must be Running AND Ready (1/1)
```

---

### Q4: What is the difference between `port` and `targetPort` in a Service?
**Answer:**
`port` is the port exposed on the **Service** itself — what clients use to reach the Service. `targetPort` is the port on the **pod/container** where the actual application listens. They can be different: for example, a Service with `port: 80` and `targetPort: 8080` means clients call the Service on port 80, but the Service forwards to the container's port 8080. `targetPort` can also reference a named port from the pod spec (e.g., `targetPort: http`), which is useful when pod port numbers might vary across versions.

---

### Q5: What is a Headless Service and when would you use it?
**Answer:**
A Headless Service is created with `clusterIP: None` — it has no virtual IP and kube-proxy doesn't create iptables rules for it. Instead, DNS returns the individual pod IPs directly. Use cases: (1) **StatefulSets** — each pod gets a stable DNS name like `pod-0.service.namespace.svc.cluster.local`, enabling direct pod-to-pod communication by name; (2) **Client-side load balancing** — gRPC clients or database drivers that want to connect to specific instances; (3) **Service discovery** — when the application needs all pod IPs to manage connections itself.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│            TOPIC 3.2 — SERVICES & CLUSTERIP                  │
├──────────────────────────────────────────────────────────────┤
│  PURPOSE: Stable IP + DNS + load balancing for pods          │
│  DEFAULT type: ClusterIP (internal only)                     │
│  Shorthand: svc                                              │
├──────────────────────────────────────────────────────────────┤
│  HOW IT WORKS:                                               │
│  1. Service created → ClusterIP auto-assigned                │
│  2. Endpoints Controller → finds Ready pods by selector      │
│  3. kube-proxy → programs iptables on every node             │
│  4. CoreDNS → creates DNS record for Service name            │
│  5. Traffic → iptables DNAT → pod IP                         │
├──────────────────────────────────────────────────────────────┤
│  KEY FIELDS:                                                 │
│  selector:    which pods to route to (must match pod labels) │
│  port:        Service port (what clients use)                │
│  targetPort:  Pod port (what containers listen on)           │
│  clusterIP:   virtual IP (auto-assigned or None for headless)│
├──────────────────────────────────────────────────────────────┤
│  DNS FORMAT:                                                 │
│  <svc>.<ns>.svc.cluster.local → ClusterIP                   │
│  Short: <svc> (same ns), <svc>.<ns> (cross-ns)              │
├──────────────────────────────────────────────────────────────┤
│  KEY COMMANDS:                                               │
│  kubectl expose deploy <n> --port=<p> --target-port=<tp>     │
│  kubectl get svc / kubectl get ep                            │
│  kubectl port-forward svc/<n> <local>:<svc-port>            │
│  kubectl run test --image=busybox --rm -it -- wget <svc>     │
├──────────────────────────────────────────────────────────────┤
│  DEBUGGING:                                                  │
│  endpoints <none> → label mismatch or pods not Ready         │
│  ClusterIP unreachable → kube-proxy down or iptables issue   │
│  DNS not resolving → CoreDNS pods down                       │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `21-services-nodeport.md`
> *NodePort — exposing services externally via node ports, port ranges, use cases*
