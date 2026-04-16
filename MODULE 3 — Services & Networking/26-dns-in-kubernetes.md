# 📌 Topic 3.8 — DNS in Kubernetes

> **Module 3 — Services & Networking**
> **File:** `26-dns-in-kubernetes.md`
> **CKA Weight:** ~20% (Networking) — DNS troubleshooting is a common CKA scenario

---

## 🎯 What You'll Learn
- How DNS works in Kubernetes — CoreDNS architecture
- Service DNS — all name formats from short to FQDN
- Pod DNS — how pods get DNS names
- The `/etc/resolv.conf` inside every pod
- DNS search domains — why short names work
- CoreDNS configuration — the Corefile
- Custom DNS — forwarding, overrides, stub zones
- DNS troubleshooting — common failures and fixes
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Phone Book Analogy 📖
> DNS in Kubernetes is like a **smart company phone book** that:
> - Is **automatically updated** whenever a new department (Service) is created
> - Has **different name formats** for different situations:
>   - Short name: "marketing" (same floor/namespace)
>   - Full name: "marketing.sales.company.com" (anywhere in the company)
>   - Pod direct: "10-244-1-5.marketing.company.com" (specific desk)
> - Is maintained by **CoreDNS** (the company operator)
> - Every employee (pod) has the operator's number pre-configured in their desk phone

---

## 🔬 CoreDNS Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   COREDNS IN KUBERNETES                     │
│                                                             │
│  Runs as: Deployment in kube-system namespace               │
│  Exposed as: Service named "kube-dns" (ClusterIP)           │
│  ClusterIP: 10.96.0.10 (typical, first IP in service CIDR)  │
│  Ports: 53/UDP, 53/TCP (DNS), 9153/TCP (metrics)            │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                 CoreDNS Pod                          │  │
│  │                                                      │  │
│  │  Plugins (processed in order):                       │  │
│  │  → errors     (log errors)                           │  │
│  │  → health     (health endpoint /health)              │  │
│  │  → kubernetes (serve cluster.local zone)             │  │
│  │  → forward    (forward unknown → upstream DNS)       │  │
│  │  → cache      (cache responses)                      │  │
│  │  → loop       (detect forwarding loops)              │  │
│  │  → reload     (auto-reload config)                   │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  kubectl get svc kube-dns -n kube-system                   │
│  # NAME       TYPE        CLUSTER-IP   PORT(S)              │
│  # kube-dns   ClusterIP   10.96.0.10   53/UDP,53/TCP        │
└─────────────────────────────────────────────────────────────┘
```

---

## 📦 Pod DNS Configuration — /etc/resolv.conf

Every pod automatically gets a `/etc/resolv.conf` injected by kubelet:

```bash
kubectl exec my-pod -- cat /etc/resolv.conf

# Content:
nameserver 10.96.0.10            # CoreDNS ClusterIP — all DNS queries go here
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

### Breaking Down Each Field

```
nameserver 10.96.0.10
  → IP of CoreDNS service
  → ALL DNS queries from this pod go here first

search default.svc.cluster.local svc.cluster.local cluster.local
  → Search domains appended to short names
  → Tried in ORDER until one resolves

options ndots:5
  → If name has fewer than 5 dots → try search domains first
  → If name has 5+ dots → try as absolute name first
  → "webapp-svc" has 0 dots < 5 → search domains appended first

HOW SHORT NAMES RESOLVE:
  You query: "webapp-svc"

  Step 1: webapp-svc.default.svc.cluster.local  → FOUND ✅ (stops here)
  Step 2: webapp-svc.svc.cluster.local           → would try if step 1 fails
  Step 3: webapp-svc.cluster.local               → would try if step 2 fails
  Step 4: webapp-svc                             → would try as absolute

  Result: webapp-svc → 10.96.15.100 (ClusterIP)
```

---

## 🌐 Service DNS — All Name Formats

### Format Hierarchy

```
FQDN (Fully Qualified Domain Name):
  <service-name>.<namespace>.svc.<cluster-domain>
  webapp-svc.production.svc.cluster.local

  cluster-domain = cluster.local (default)
  configured in: kubelet --cluster-domain flag

EXAMPLES for Service "api-svc" in namespace "backend":

  From the SAME namespace (backend):
  ┌────────────────────────────────────────────────────────┐
  │  api-svc                    ← shortest, works!         │
  │  api-svc.backend            ← with namespace           │
  │  api-svc.backend.svc        ← with type                │
  │  api-svc.backend.svc.cluster.local  ← full FQDN ✅     │
  └────────────────────────────────────────────────────────┘

  From a DIFFERENT namespace (frontend):
  ┌────────────────────────────────────────────────────────┐
  │  api-svc              ← ❌ FAILS (search appends frontend ns)│
  │  api-svc.backend      ← ✅ works (cross-namespace short) │
  │  api-svc.backend.svc  ← ✅ works                         │
  │  api-svc.backend.svc.cluster.local ← ✅ always works    │
  └────────────────────────────────────────────────────────┘

⚠️  Cross-namespace: always use at least <svc>.<namespace>
```

### Special Built-in Services

```
kubernetes Service (API server):
  kubernetes.default.svc.cluster.local → 10.96.0.1
  (always first IP in service CIDR)
  Pods use this to reach apiserver internally

kube-dns Service (CoreDNS):
  kube-dns.kube-system.svc.cluster.local → 10.96.0.10
  (nameserver used in all pod resolv.conf)
```

---

## 🏠 Pod DNS — Direct Pod Names

Pods also get DNS names, but they're rarely used directly:

```
POD DNS FORMAT:
  <pod-ip-dashes>.<namespace>.pod.<cluster-domain>
  10-244-1-5.default.pod.cluster.local → 10.244.1.5

  Dots in IP replaced with dashes!
  10.244.1.5 → 10-244-1-5

STATEFULSET POD DNS (most useful pod DNS):
  <pod-name>.<headless-service>.<namespace>.svc.<cluster-domain>
  mysql-0.mysql.default.svc.cluster.local → Pod IP

  This works because StatefulSet + Headless Service creates
  individual DNS A records per pod.

HOSTNAME AND SUBDOMAIN (custom pod DNS):
spec:
  hostname: my-pod              # sets pod hostname
  subdomain: my-service         # must match a headless service name!
  # Creates DNS: my-pod.my-service.default.svc.cluster.local

  Without subdomain: hostname just sets the pod's /etc/hostname
  With subdomain:    creates a proper DNS entry (requires headless svc)
```

---

## ⚙️ CoreDNS Configuration — The Corefile

CoreDNS is configured via a ConfigMap in kube-system:

```bash
kubectl get configmap coredns -n kube-system -o yaml
```

```
# Default Corefile content:
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf {           # forward non-cluster queries
       max_concurrent 1000                  # to node's DNS
    }
    cache 30
    loop
    reload
    loadbalance
}
```

### Key Sections Explained

```
kubernetes cluster.local in-addr.arpa ip6.arpa {
  → Handles queries for cluster.local (K8s service/pod DNS)
  → in-addr.arpa = reverse DNS lookups
  → pods insecure = create pod DNS records (without auth)
  → ttl 30 = 30-second TTL for K8s records

forward . /etc/resolv.conf {
  → Forward all non-cluster queries
  → . means "everything" (root zone)
  → /etc/resolv.conf = node's DNS config (e.g., AWS/GCP DNS)
  → So external names (google.com) resolved via node's DNS

cache 30
  → Cache all responses for 30 seconds
  → Reduces load on upstream DNS servers
```

---

## 🔧 Custom DNS Configuration

### Scenario 1: Forward Specific Domain to Custom DNS Server

```yaml
# Edit CoreDNS ConfigMap to add custom stub zone
kubectl edit configmap coredns -n kube-system

# Add stub zone for company.internal:
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
        }
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
    company.internal:53 {           # ← NEW stub zone
        errors
        cache 30
        forward . 10.0.0.100        # forward to internal DNS server
    }
```

### Scenario 2: Override Specific Hostname

```yaml
# Add a custom hosts file entry
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local ...
        hosts {                      # ← custom hosts
            1.2.3.4 my-custom-host.example.com
            fallthrough
        }
        forward . /etc/resolv.conf
        cache 30
    }
```

### Scenario 3: Per-Pod DNS Config (dnsConfig)

```yaml
# Override DNS settings for a specific pod
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  dnsPolicy: None                   # None | ClusterFirst | Default | ClusterFirstWithHostNet

  dnsConfig:
    nameservers:
      - 8.8.8.8                     # Google DNS instead of CoreDNS
      - 8.8.4.4
    searches:
      - my-custom.search.domain
      - default.svc.cluster.local   # still include K8s search domain
    options:
      - name: ndots
        value: "5"
      - name: timeout
        value: "2"
```

### DNS Policies

```
dnsPolicy: ClusterFirst (DEFAULT)
  → Use CoreDNS first (cluster.local resolved internally)
  → Unknown domains forwarded to node's DNS
  → Used by 99% of pods

dnsPolicy: Default
  → Use node's DNS directly (NOT CoreDNS)
  → No cluster.local resolution
  → Pod can't resolve service names!
  → Use for: system pods that need host DNS only

dnsPolicy: None
  → Completely custom — you specify dnsConfig
  → No nameserver unless you set it in dnsConfig

dnsPolicy: ClusterFirstWithHostNet
  → Like ClusterFirst but for pods with hostNetwork: true
  → hostNetwork pods normally use host DNS (Default behavior)
  → This overrides to use CoreDNS
```

---

## 🔍 DNS Troubleshooting

```bash
# ── CHECK COREDNS IS RUNNING ──────────────────────────────
kubectl get pods -n kube-system -l k8s-app=kube-dns
# Should show 2 Running pods (default is 2 replicas)

kubectl get svc kube-dns -n kube-system
# Should show ClusterIP = 10.96.0.10 (or similar)

# ── TEST DNS FROM INSIDE A POD ────────────────────────────
kubectl run dns-test --image=busybox:1.35 --rm -it -- sh
# Inside the shell:
nslookup kubernetes.default          # test Service DNS
nslookup google.com                  # test external DNS
cat /etc/resolv.conf                 # check DNS config
nslookup webapp-svc.production       # cross-namespace
nslookup mysql-0.mysql.default.svc.cluster.local  # StatefulSet pod

# ── SPECIFIC DNS DEBUGGING ────────────────────────────────
# Using dnsutils image (has dig, nslookup, host)
kubectl run dnsutils --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3 \
  --rm -it -- bash

# Inside:
dig webapp-svc.default.svc.cluster.local
dig @10.96.0.10 webapp-svc.default.svc.cluster.local  # query CoreDNS directly
host webapp-svc.default
nslookup webapp-svc

# ── CHECK COREDNS LOGS ────────────────────────────────────
kubectl logs -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system coredns-xxx
# Look for: errors, SERVFAIL responses, loop detection

# ── COREDNS CONFIG ────────────────────────────────────────
kubectl get configmap coredns -n kube-system -o yaml
```

---

## 🚨 Common DNS Issues & Fixes

```
ISSUE: DNS not resolving inside pod
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Symptoms: nslookup fails, connection refused to services

Checks:
  1. CoreDNS pods running?
     kubectl get pods -n kube-system -l k8s-app=kube-dns

  2. Pod's resolv.conf correct?
     kubectl exec my-pod -- cat /etc/resolv.conf
     → Should show nameserver 10.96.0.10

  3. Can pod reach CoreDNS?
     kubectl exec my-pod -- nc -zv 10.96.0.10 53

  4. CoreDNS logs showing errors?
     kubectl logs -n kube-system -l k8s-app=kube-dns

  Fixes:
  → Restart CoreDNS: kubectl rollout restart deploy/coredns -n kube-system
  → Check NetworkPolicy blocking port 53
  → Check dnsPolicy on the pod

ISSUE: Service name not resolving
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Symptoms: nslookup webapp-svc → NXDOMAIN

Checks:
  1. Service exists?
     kubectl get svc webapp-svc

  2. Service in right namespace?
     kubectl get svc -A | grep webapp

  3. Using right name format?
     In different ns → use webapp-svc.production (not just webapp-svc)

  4. Typo in service name?
     Compare: kubectl get svc → exact name

ISSUE: External DNS not resolving
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Symptoms: nslookup google.com fails

Checks:
  1. CoreDNS forward config
     kubectl get configmap coredns -n kube-system -o yaml
     → Should have: forward . /etc/resolv.conf

  2. Node's DNS working?
     ssh node && cat /etc/resolv.conf && nslookup google.com

  3. Egress NetworkPolicy blocking?
     Check if egress to port 53 is allowed

ISSUE: DNS loop detected
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Symptoms: CoreDNS CrashLoopBackOff, logs show "Loop"

Cause: Node's /etc/resolv.conf points to 127.0.0.53 (systemd-resolved)
       CoreDNS forwards to it → it forwards back → loop!

Fix: Edit CoreDNS Corefile
     Replace: forward . /etc/resolv.conf
     With:    forward . 8.8.8.8 8.8.4.4  (or your upstream DNS)
```

---

## 💻 Key DNS Commands Summary

```bash
# Check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl get svc kube-dns -n kube-system
kubectl get cm coredns -n kube-system -o yaml

# Test DNS from inside cluster
kubectl run test --image=busybox:1.35 --rm -it -- nslookup <service>
kubectl run test --image=busybox:1.35 --rm -it -- \
  nslookup <svc>.<namespace>.svc.cluster.local

# Check pod DNS config
kubectl exec <pod> -- cat /etc/resolv.conf

# Edit CoreDNS config
kubectl edit configmap coredns -n kube-system
kubectl rollout restart deployment/coredns -n kube-system

# Check DNS from pod directly
kubectl exec <pod> -- nslookup kubernetes.default
kubectl exec <pod> -- nslookup google.com
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| Using short name across namespaces | Short name only works in same namespace — add `.namespace` for cross-ns |
| dnsPolicy: Default | Pod uses node DNS, NOT CoreDNS — service names won't resolve! |
| Editing coredns ConfigMap wrong format | Incorrect Corefile syntax crashes CoreDNS — validate before applying |
| Forgetting DNS egress in NetworkPolicy | Pods with strict egress can't resolve any names |
| ndots:5 performance impact | Short names trigger 3 DNS searches before absolute — use FQDN in perf-critical code |
| CoreDNS not HA | Default 2 replicas — node failure can cause DNS outage; increase for production |

---

## 🎓 CKA Exam Tips

```
✅ Test DNS immediately when debugging connectivity:
   kubectl exec <pod> -- nslookup <service>
   kubectl exec <pod> -- cat /etc/resolv.conf

✅ DNS name formats to know:
   Same namespace:   <svc>
   Cross-namespace:  <svc>.<ns>
   Full FQDN:        <svc>.<ns>.svc.cluster.local
   Pod (StatefulSet): <pod>.<headless-svc>.<ns>.svc.cluster.local

✅ CoreDNS lives in kube-system:
   kubectl get pods -n kube-system | grep dns
   kubectl get svc kube-dns -n kube-system

✅ CoreDNS config = ConfigMap:
   kubectl get cm coredns -n kube-system -o yaml

✅ If service DNS broken:
   1. Check service exists: kubectl get svc
   2. Check CoreDNS running: kubectl get pods -n kube-system
   3. Test from pod: kubectl exec pod -- nslookup svc-name
   4. Check NetworkPolicy (port 53 blocked?)

✅ cluster.local is the default cluster domain
   Can be changed in kubelet config (--cluster-domain)

✅ Always use busybox:1.35 for DNS testing (has nslookup):
   kubectl run test --image=busybox:1.35 --rm -it -- nslookup <target>
```

---

## ❓ Interview & Scenario Questions

### Q1: How does a pod resolve a Kubernetes Service name?
**Answer:**
Every pod has `/etc/resolv.conf` injected by kubelet, containing the CoreDNS ClusterIP as `nameserver` and search domains (`<namespace>.svc.cluster.local svc.cluster.local cluster.local`). When a pod queries a short name like `webapp-svc`, the OS appends search domains in order until one resolves. For a pod in the `default` namespace, it tries `webapp-svc.default.svc.cluster.local` first. CoreDNS answers with the Service's ClusterIP from its built-in Kubernetes plugin, which monitors Service and Endpoints objects. The pod then connects to that ClusterIP, and kube-proxy's iptables rules handle routing to the actual pods.

---

### Q2: Why does DNS sometimes fail with a `NXDOMAIN` error when accessing a service from a different namespace?
**Answer:**
When accessing from a different namespace (say `frontend` namespace), the short name `webapp-svc` gets search domains appended: `webapp-svc.frontend.svc.cluster.local` — which doesn't exist (the service is in `production` namespace). The fix is to use the cross-namespace format: `webapp-svc.production` or the full FQDN `webapp-svc.production.svc.cluster.local`. This is a very common mistake — always use at least `<service>.<namespace>` for cross-namespace DNS resolution.

---

### Q3: What is CoreDNS and how do you troubleshoot a DNS failure?
**Answer:**
CoreDNS is the DNS server running in the `kube-system` namespace that provides DNS resolution for all Kubernetes Services and pods. Troubleshooting steps:
```bash
# 1. Check CoreDNS is running
kubectl get pods -n kube-system -l k8s-app=kube-dns

# 2. Check pod's DNS config
kubectl exec my-pod -- cat /etc/resolv.conf
# Should show nameserver 10.96.0.10

# 3. Test DNS from inside pod
kubectl run test --image=busybox:1.35 --rm -it \
  -- nslookup kubernetes.default

# 4. Check CoreDNS logs for errors
kubectl logs -n kube-system -l k8s-app=kube-dns

# 5. Check NetworkPolicy isn't blocking port 53
kubectl get netpol -A | grep deny
```

---

### Q4: What is `dnsPolicy: None` and when would you use it?
**Answer:**
`dnsPolicy: None` completely disables automatic DNS configuration for a pod — no CoreDNS, no search domains, no nameservers are injected. You must provide everything manually via `dnsConfig`. Use this when: a pod needs to use completely custom DNS servers (like a specific corporate DNS), or when a pod needs to avoid the overhead of cluster search domains for performance-critical external DNS queries, or when you're running a DNS server pod itself that shouldn't be configured to use itself.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│            TOPIC 3.8 — DNS IN KUBERNETES                     │
├──────────────────────────────────────────────────────────────┤
│  COREDNS:                                                    │
│  Namespace: kube-system                                      │
│  Service:   kube-dns  ClusterIP: 10.96.0.10                  │
│  Config:    ConfigMap "coredns" in kube-system               │
├──────────────────────────────────────────────────────────────┤
│  SERVICE DNS FORMATS:                                        │
│  <svc>                              ← same namespace only    │
│  <svc>.<ns>                         ← cross-namespace ✅     │
│  <svc>.<ns>.svc                     ← with type             │
│  <svc>.<ns>.svc.cluster.local       ← full FQDN ✅           │
├──────────────────────────────────────────────────────────────┤
│  POD DNS (StatefulSet):                                      │
│  <pod>.<headless-svc>.<ns>.svc.cluster.local                 │
├──────────────────────────────────────────────────────────────┤
│  resolv.conf fields:                                         │
│  nameserver: CoreDNS IP (10.96.0.10)                         │
│  search:     <ns>.svc.cluster.local svc.cluster.local ...    │
│  options:    ndots:5                                         │
├──────────────────────────────────────────────────────────────┤
│  DNS POLICIES:                                               │
│  ClusterFirst (default) → CoreDNS first, node DNS fallback  │
│  Default               → node DNS only (no cluster DNS!)    │
│  None                  → fully custom via dnsConfig          │
├──────────────────────────────────────────────────────────────┤
│  TROUBLESHOOTING:                                            │
│  kubectl get pods -n kube-system | grep dns                  │
│  kubectl run test --image=busybox:1.35 --rm -it \            │
│    -- nslookup <svc>.<ns>.svc.cluster.local                  │
│  kubectl exec <pod> -- cat /etc/resolv.conf                  │
│  kubectl logs -n kube-system -l k8s-app=kube-dns             │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `27-cni-plugins.md`
> *Flannel, Calico, Weave, Cilium — comparison, config, and when to use each*
