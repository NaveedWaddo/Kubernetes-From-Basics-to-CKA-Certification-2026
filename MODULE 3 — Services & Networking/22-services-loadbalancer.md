# 📌 Topic 3.4 — Services: LoadBalancer

> **Module 3 — Services & Networking**
> **File:** `22-services-loadbalancer.md`
> **CKA Weight:** ~20% (Networking) — LoadBalancer appears in cloud + external access tasks

---

## 🎯 What You'll Learn
- What LoadBalancer Service type is and how it extends NodePort
- How cloud-controller-manager provisions real cloud load balancers
- Full LoadBalancer YAML anatomy with all annotations
- External IP assignment and how to check it
- MetalLB — LoadBalancer for on-premise/bare metal clusters
- Internal Load Balancers — keeping LB private
- Health checks and session affinity
- LoadBalancer vs NodePort vs Ingress comparison
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Corporate Reception Building Analogy 🏢
> Think of service types as levels of a building's reception:
>
> - **ClusterIP** = Internal intercom (only employees inside can use it)
> - **NodePort** = Side entrance on a back street (visitors must know the obscure address)
> - **LoadBalancer** = Main entrance on the high street with a **full concierge service**:
>   - Visitors call the main number (external IP) → always reaches the front desk
>   - A professional doorman (cloud LB) greets them and routes them inside
>   - Health checks ensure only healthy desks (pods) get visitors
>   - Scales automatically — no single point of failure
>
> LoadBalancer is the **production-grade** way to expose services externally in cloud environments.

---

## 🔬 What is LoadBalancer Service Type?

LoadBalancer **extends NodePort** by provisioning an actual cloud load balancer in front of the cluster. It gives you a single, stable external IP (or hostname) that routes to your service.

```
┌─────────────────────────────────────────────────────────────┐
│                    EXTERNAL CLIENT                          │
│               curl http://54.200.100.50:80                  │
└──────────────────────────┬──────────────────────────────────┘
                           │ hits cloud Load Balancer IP
                           ▼
┌──────────────────────────────────────────────────────────────┐
│              CLOUD LOAD BALANCER (AWS ELB / GCP LB / AKS)   │
│              External IP: 54.200.100.50                      │
│              Health checks nodes automatically               │
└────────┬─────────────────┬──────────────────┬───────────────┘
         │                 │                  │
         ▼                 ▼                  ▼
┌────────────┐    ┌────────────┐    ┌────────────┐
│  Node 1    │    │  Node 2    │    │  Node 3    │
│ :30080     │    │ :30080     │    │ :30080     │
│ (NodePort) │    │ (NodePort) │    │ (NodePort) │
└────────────┘    └────────────┘    └────────────┘
         │                 │                  │
         └─────────────────┼──────────────────┘
                           │ ClusterIP + iptables
                           ▼
              ┌────────────────────────┐
              │  Pods (load balanced)  │
              │  10.244.1.5            │
              │  10.244.2.3            │
              │  10.244.3.7            │
              └────────────────────────┘

Layers stacked:
  LoadBalancer (cloud) → NodePort (all nodes) → ClusterIP → Pods
```

---

## 📄 LoadBalancer YAML — Full Anatomy

```yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp-lb
  namespace: production
  labels:
    app: webapp

  # ── CLOUD PROVIDER ANNOTATIONS ──────────────────────────
  # These control cloud-specific LB behavior
  annotations:
    # ── AWS ELB annotations ─────────────────────────────
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    # "clb" = Classic LB (legacy)
    # "nlb" = Network LB (L4, faster, static IP)
    # "alb" = Application LB (L7, use Ingress instead)

    service.beta.kubernetes.io/aws-load-balancer-internal: "true"
    # Creates internal LB (VPC only, not public internet)

    service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:..."
    # Attach SSL certificate for HTTPS

    service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"

    # ── GCP annotations ─────────────────────────────────
    # cloud.google.com/load-balancer-type: "Internal"
    # networking.gke.io/load-balancer-type: "Internal"

    # ── Azure annotations ────────────────────────────────
    # service.beta.kubernetes.io/azure-load-balancer-internal: "true"

spec:
  # ── SERVICE TYPE ────────────────────────────────────────
  type: LoadBalancer

  # ── SELECTOR ────────────────────────────────────────────
  selector:
    app: webapp

  # ── PORTS ───────────────────────────────────────────────
  ports:
    - name: http
      protocol: TCP
      port: 80              # external port on LB and ClusterIP
      targetPort: 8080      # pod port
      nodePort: 30080       # optional: specify NodePort (auto if omitted)

    - name: https
      protocol: TCP
      port: 443
      targetPort: 8443

  # ── EXTERNAL TRAFFIC POLICY ─────────────────────────────
  externalTrafficPolicy: Cluster   # Cluster | Local

  # ── LOAD BALANCER SOURCE RANGES ─────────────────────────
  # Restrict which external IPs can access this LB
  loadBalancerSourceRanges:
    - "203.0.113.0/24"    # allow only this CIDR
    - "198.51.100.5/32"   # and this specific IP

  # ── LOAD BALANCER IP ────────────────────────────────────
  # Request a specific external IP (if cloud supports it)
  loadBalancerIP: "54.200.100.50"   # cloud must have this IP reserved

  # ── HEALTH CHECK NODE PORT ──────────────────────────────
  # Used with externalTrafficPolicy: Local
  healthCheckNodePort: 32100        # cloud LB checks this port per node
```

---

## ⚙️ How LoadBalancer is Provisioned

```
PROVISIONING FLOW:

  1. Developer runs: kubectl apply -f service.yaml
                     (type: LoadBalancer)

  2. apiserver stores Service in etcd

  3. cloud-controller-manager detects new LoadBalancer Service
     → Calls cloud provider API (AWS/GCP/Azure)
     → Creates actual cloud load balancer resource
     → Configures: health checks, target groups, listeners
     → Targets: all cluster nodes on the nodePort

  4. Cloud LB is ready (takes 30s - 2min)
     → cloud-controller-manager writes external IP to Service:
       status.loadBalancer.ingress[0].ip = "54.200.100.50"
       (or hostname for DNS-based LBs like AWS ALB)

  5. Service shows EXTERNAL-IP:
     kubectl get svc webapp-lb
     # NAME       TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)
     # webapp-lb  LoadBalancer   10.96.15.100    54.200.100.50   80:30080/TCP

  6. External traffic flows:
     client → 54.200.100.50:80 → cloud LB → node:30080 → pod:8080

PENDING state:
  kubectl get svc webapp-lb
  # EXTERNAL-IP: <pending>
  # → cloud-controller-manager still provisioning
  # → or running on-prem without cloud provider (need MetalLB)
```

---

## ☁️ Cloud-Specific Load Balancers

### AWS

```
Classic Load Balancer (CLB):
  → Legacy L4/L7 LB
  → Default if no annotation specified
  → HTTP/HTTPS + TCP

Network Load Balancer (NLB):
  → L4 only (TCP/UDP)
  → Ultra-low latency
  → Static IP addresses
  → Best for high-traffic, low-latency apps
  Annotation: service.beta.kubernetes.io/aws-load-balancer-type: "nlb"

Application Load Balancer (ALB):
  → L7 (HTTP/HTTPS)
  → Path/host-based routing
  → NOT used with Service type=LoadBalancer
  → Used with Ingress + AWS Load Balancer Controller

kubectl get svc -n production
# NAME      TYPE           EXTERNAL-IP
# webapp    LoadBalancer   abc123.us-east-1.elb.amazonaws.com  ← DNS hostname
```

### GCP

```
External Load Balancer (default):
  → L4 TCP/UDP
  → External IP from Google's pool
  → Global or regional

Internal Load Balancer:
  → Only accessible within VPC
  Annotation: cloud.google.com/load-balancer-type: "Internal"

kubectl get svc
# EXTERNAL-IP: 35.200.100.50   ← static IP
```

### Azure

```
External Load Balancer (default):
  → L4 TCP/UDP
  → Public IP from Azure

Internal Load Balancer:
  → Private IP within VNET
  Annotation: service.beta.kubernetes.io/azure-load-balancer-internal: "true"
```

---

## 🏠 MetalLB — LoadBalancer for Bare Metal

On-premise or local clusters (minikube, kind) don't have a cloud provider to create real LBs. MetalLB provides LoadBalancer functionality for bare metal.

```
PROBLEM without MetalLB:
  type: LoadBalancer → EXTERNAL-IP: <pending> forever
  No cloud controller to provision actual LB

SOLUTION: MetalLB
  → Deploys as pods in the cluster
  → Watches for LoadBalancer Services
  → Assigns IPs from a configured IP pool
  → Announces those IPs via ARP (L2 mode) or BGP

INSTALLATION:
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/\
  v0.14.0/config/manifests/metallb-native.yaml

CONFIGURATION:
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.240-192.168.1.250   # IPs to assign to LB services

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system

RESULT:
  kubectl get svc webapp-lb
  # EXTERNAL-IP: 192.168.1.240  ← assigned from pool!
  # curl http://192.168.1.240:80  → reaches pods ✅
```

---

## 🔒 Internal Load Balancer

Sometimes you want a LoadBalancer that's NOT accessible from the internet — only from within the cloud VPC/network.

```yaml
# AWS Internal LB
metadata:
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-internal: "true"

# GCP Internal LB
metadata:
  annotations:
    cloud.google.com/load-balancer-type: "Internal"

# Azure Internal LB
metadata:
  annotations:
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"

# Use case: database services, internal microservices
# Only accessible from within your VPC/cloud network
# Not exposed to public internet
```

---

## 💻 LoadBalancer Commands Reference

```bash
# ── CREATE ────────────────────────────────────────────────
kubectl expose deployment webapp \
  --name=webapp-lb \
  --port=80 \
  --target-port=8080 \
  --type=LoadBalancer

kubectl apply -f loadbalancer-service.yaml

# ── VIEW ──────────────────────────────────────────────────
kubectl get svc webapp-lb
# NAME       TYPE           CLUSTER-IP      EXTERNAL-IP       PORT(S)
# webapp-lb  LoadBalancer   10.96.15.100    54.200.100.50     80:30080/TCP
#                                           ↑ wait for this to appear

# Watch until EXTERNAL-IP is assigned
kubectl get svc webapp-lb -w
# (wait for <pending> → actual IP)

# Get external IP programmatically
kubectl get svc webapp-lb \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
# 54.200.100.50

# For AWS (DNS name, not IP):
kubectl get svc webapp-lb \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
# abc123.us-east-1.elb.amazonaws.com

# ── TEST CONNECTIVITY ─────────────────────────────────────
# From outside cluster:
curl http://54.200.100.50:80

# From inside cluster (ClusterIP still works):
kubectl run test --image=busybox --rm -it -- \
  wget -qO- http://webapp-lb:80

# ── DESCRIBE ──────────────────────────────────────────────
kubectl describe svc webapp-lb
# Shows: LoadBalancer Ingress, Endpoints, Events

# ── UPDATE SOURCE RANGES ──────────────────────────────────
kubectl patch svc webapp-lb \
  -p '{"spec":{"loadBalancerSourceRanges":["203.0.113.0/24"]}}'

# ── DELETE ────────────────────────────────────────────────
kubectl delete svc webapp-lb
# ⚠️ Also deletes the cloud LB — stops billing!
```

---

## 🆚 LoadBalancer vs NodePort vs Ingress

```
┌──────────────┬────────────────────────────────────────────┐
│ Type         │ When to Use                                │
├──────────────┼────────────────────────────────────────────┤
│ NodePort     │ Dev/test, on-prem without MetalLB,          │
│              │ Quick external access needed                │
│              │ Weird port (30000-32767) is acceptable      │
├──────────────┼────────────────────────────────────────────┤
│ LoadBalancer │ Production cloud, need standard ports       │
│              │ (80/443), need stable external IP           │
│              │ L4 (TCP/UDP) traffic                        │
│              │ One LB per service (expensive at scale)     │
├──────────────┼────────────────────────────────────────────┤
│ Ingress      │ Multiple services behind one LB             │
│              │ (cost-efficient!)                           │
│              │ Need path/host routing (L7)                 │
│              │ Need SSL termination at edge                │
│              │ HTTP/HTTPS traffic only                     │
└──────────────┴────────────────────────────────────────────┘

Cost comparison:
  10 services as LoadBalancer: 10 cloud LBs = $$$
  10 services behind 1 Ingress: 1 cloud LB = $
  → Always prefer Ingress for HTTP services at scale!
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| EXTERNAL-IP stays `<pending>` forever | No cloud provider configured — use MetalLB for bare metal |
| Deleting Service doesn't delete cloud LB | ❌ Wait — deleting the Service DOES delete the cloud LB! (watch costs) |
| One LB per service (expensive) | Use Ingress to share one LB across many services |
| loadBalancerIP on unsupported cloud | Some clouds don't support specifying static LB IP — use reserved IPs |
| Forgetting annotation for internal LB | Without annotation, default is public-facing internet LB |
| NodePort range conflict | LoadBalancer uses NodePort internally — same 30000-32767 restriction |

---

## 🎓 CKA Exam Tips

```
✅ LoadBalancer = NodePort + cloud LB provisioned automatically
   Layers: Cloud LB → NodePort → ClusterIP → Pods

✅ Check external IP:
   kubectl get svc <n>  → EXTERNAL-IP column
   kubectl get svc <n> -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

✅ <pending> external IP = cloud controller not running
   (Normal in minikube/kind — use NodePort or MetalLB instead)

✅ Expose quickly:
   kubectl expose deployment <n> --type=LoadBalancer --port=80

✅ Internal LB (cloud-specific annotation varies by provider):
   AWS:   service.beta.kubernetes.io/aws-load-balancer-internal: "true"
   GCP:   cloud.google.com/load-balancer-type: "Internal"
   Azure: service.beta.kubernetes.io/azure-load-balancer-internal: "true"

✅ For CKA exam tasks on local/kind cluster:
   Use NodePort or port-forward instead
   (No cloud provider = no LoadBalancer IP)

✅ healthCheckNodePort + externalTrafficPolicy: Local
   → Cloud LB uses this port to determine if node has local pods
```

---

## ❓ Interview & Scenario Questions

### Q1: How does a LoadBalancer Service differ from NodePort?
**Answer:**
LoadBalancer extends NodePort by also provisioning an actual cloud load balancer. With NodePort, you get external access through `<node-IP>:<nodePort>` but must manage external load balancing yourself. With LoadBalancer, the cloud-controller-manager automatically creates a cloud load balancer (AWS ELB, GCP LB, Azure LB) that gets a single stable external IP or hostname. This cloud LB health-checks nodes and routes traffic to the NodePort on healthy nodes. LoadBalancer gives you a proper production-grade entry point with a standard port (80/443) instead of the awkward 30000-32767 range.

---

### Q2: Why does `EXTERNAL-IP` stay `<pending>` for a LoadBalancer Service?
**Answer:**
`<pending>` means no external IP has been assigned yet. This happens when: (1) the cluster is running without a cloud provider (minikube, kind, bare-metal) — there's no cloud-controller-manager to provision a real LB; (2) the cloud-controller-manager is still provisioning the LB (takes 30s-2min on most clouds); (3) there's a misconfiguration in the cloud provider credentials. For bare-metal clusters, install **MetalLB** which provides LoadBalancer functionality by assigning IPs from a configured pool and announcing them via ARP or BGP.

---

### Q3: Scenario — You have 20 microservices all needing external HTTPS access. Would you create 20 LoadBalancer Services?
**Answer:**
No — that would create 20 cloud load balancers costing significant money. The better approach is **Ingress**: one Ingress controller (backed by one LoadBalancer Service) can route to all 20 services based on host or path rules:
```yaml
# One Ingress handles all 20 services:
rules:
  - host: api.example.com
    http:
      paths:
        - path: /users → users-service
        - path: /orders → orders-service
        - path: /payments → payments-service
```
This way you pay for one cloud LB, get SSL termination at the edge, and have centralized routing. Use LoadBalancer directly only for non-HTTP traffic (TCP/UDP) that Ingress can't handle.

---

### Q4: What is the cloud-controller-manager's role in LoadBalancer Services?
**Answer:**
The cloud-controller-manager runs cloud-specific controllers that interface with the cloud provider's API. Its Service Controller watches for LoadBalancer Services and calls the cloud API to create/update/delete load balancers. When a LoadBalancer Service is created, it provisions the actual cloud LB, configures health checks pointing to the NodePort on each cluster node, and writes the assigned external IP back to the Service's `status.loadBalancer.ingress` field. When the Service is deleted, it deletes the cloud LB. Without the cloud-controller-manager, LoadBalancer Services would just stay in `<pending>` state forever.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│           TOPIC 3.4 — SERVICES: LOADBALANCER                 │
├──────────────────────────────────────────────────────────────┤
│  PURPOSE: Production external access via cloud LB            │
│  Extends: NodePort (uses it internally)                      │
│  Layers: Cloud LB → NodePort → ClusterIP → Pods             │
├──────────────────────────────────────────────────────────────┤
│  PROVISIONING:                                               │
│  cloud-controller-manager → calls cloud API → creates LB    │
│  LB external IP → written to svc.status.loadBalancer.ingress │
├──────────────────────────────────────────────────────────────┤
│  KEY COMMANDS:                                               │
│  kubectl expose deploy <n> --type=LoadBalancer --port=80     │
│  kubectl get svc <n>  → EXTERNAL-IP column                   │
│  kubectl get svc <n> -o jsonpath=\                           │
│    '{.status.loadBalancer.ingress[0].ip}'                    │
├──────────────────────────────────────────────────────────────┤
│  <pending> EXTERNAL-IP → no cloud provider                   │
│  Fix: use MetalLB for bare-metal clusters                    │
├──────────────────────────────────────────────────────────────┤
│  INTERNAL LB ANNOTATIONS:                                    │
│  AWS:   aws-load-balancer-internal: "true"                   │
│  GCP:   load-balancer-type: "Internal"                       │
│  Azure: azure-load-balancer-internal: "true"                 │
├──────────────────────────────────────────────────────────────┤
│  COST TIP:                                                   │
│  1 LB per service = expensive at scale                       │
│  Use Ingress = 1 LB for ALL HTTP services = much cheaper     │
│  LoadBalancer directly: only for non-HTTP (TCP/UDP)          │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `23-services-headless-externalname.md`
> *Headless Services for StatefulSets, ExternalName for external DNS bridging*
