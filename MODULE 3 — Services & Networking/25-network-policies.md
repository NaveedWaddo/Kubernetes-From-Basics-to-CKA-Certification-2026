# 📌 Topic 3.7 — Network Policies

> **Module 3 — Services & Networking**
> **File:** `25-network-policies.md`
> **CKA Weight:** 🔥 CRITICAL (~20%) — Network Policies are a guaranteed CKA task

---

## 🎯 What You'll Learn
- What Network Policies are and why they matter
- Default behavior — all pods can talk to all pods
- Ingress rules — control who can SEND traffic to a pod
- Egress rules — control where a pod can SEND traffic
- Pod selectors, namespace selectors, IP blocks
- Default deny all — locking down a namespace
- Combining rules — AND vs OR logic
- Real-world policy patterns
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Office Firewall Analogy 🔥
> By default, Kubernetes networking is like an **open office** —
> every desk (pod) can talk to every other desk freely.
>
> **Network Policies** are like installing **security doors and badges**:
> - **Ingress policy** = "Who is ALLOWED to enter my office?"
>   → Only badge holders from the `frontend` team can enter
> - **Egress policy** = "Where am I ALLOWED to go from my office?"
>   → I can only go to the `database` room, nowhere else
>
> Critical rule: If you install ANY door (select a pod with NetworkPolicy),
> that pod gets a **default deny** — ALL traffic blocked UNLESS explicitly allowed.
>
> If no NetworkPolicy selects a pod → the pod is wide open (default allow all).

---

## 🔬 Default Behavior — No NetworkPolicy

```
WITHOUT any NetworkPolicy:
  All pods can communicate with all other pods — no restrictions

  Pod A → Pod B ✅ (any namespace)
  Pod A → Pod C ✅ (any namespace)
  Pod A → External IP ✅
  External → Pod A ✅

This is the Kubernetes default — open networking.
NetworkPolicy is OPT-IN — you must explicitly apply restrictions.

⚠️  IMPORTANT: NetworkPolicy requires a CNI plugin that supports it!
  ✅ Calico, Cilium, Weave, Antrea
  ❌ Flannel (alone) does NOT support NetworkPolicy
     (use Flannel + Calico or switch to Calico/Cilium)
```

---

## 📐 NetworkPolicy Structure

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: my-policy
  namespace: default        # policies are namespaced!

spec:
  # ── WHICH PODS THIS POLICY APPLIES TO ──────────────────
  podSelector:
    matchLabels:
      app: webapp           # applies to pods with app=webapp
                            # empty podSelector {} = ALL pods in namespace

  # ── WHAT TYPES OF TRAFFIC ARE CONTROLLED ───────────────
  policyTypes:
    - Ingress               # control incoming traffic
    - Egress                # control outgoing traffic
    # If Ingress listed → all ingress blocked unless in ingress rules
    # If Egress listed  → all egress blocked unless in egress rules

  # ── INGRESS RULES (who can SEND TO this pod) ───────────
  ingress:
    - from:                 # allow from...
        - podSelector:
            matchLabels:
              app: frontend  # pods with app=frontend
        - namespaceSelector:
            matchLabels:
              env: production # pods in production namespace
      ports:
        - protocol: TCP
          port: 8080         # only on this port

  # ── EGRESS RULES (where this pod can SEND TO) ──────────
  egress:
    - to:                   # allow to...
        - podSelector:
            matchLabels:
              app: database  # pods with app=database
      ports:
        - protocol: TCP
          port: 5432         # only PostgreSQL port
```

---

## 🔄 How NetworkPolicy Works

```
┌─────────────────────────────────────────────────────────────┐
│  WHEN a NetworkPolicy selects a pod:                        │
│                                                             │
│  Default behavior changes from ALLOW ALL to DENY ALL        │
│  for the policy types listed (Ingress/Egress)               │
│                                                             │
│  Then: each rule in ingress/egress ADDS allowed traffic     │
│                                                             │
│  Multiple NetworkPolicies selecting same pod:               │
│  → Rules are UNIONED (combined with OR)                     │
│  → If ANY policy allows traffic, it's allowed               │
│  → More policies = more permissions (never more restriction)│
└─────────────────────────────────────────────────────────────┘

EXAMPLE:
  Pod "webapp" has app=webapp label
  Policy 1 selects webapp → allows ingress from frontend
  Policy 2 selects webapp → allows ingress from monitoring

  Result: webapp allows ingress from BOTH frontend AND monitoring
  (NOT just monitoring because Policy 2 was applied later)
```

---

## 🛡️ PATTERN 1 — Default Deny All (Most Restrictive)

```yaml
# Apply this first → locks down ALL pods in namespace
# Then add specific allow policies on top

# Deny ALL ingress to all pods in namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}           # {} = selects ALL pods
  policyTypes:
    - Ingress
  # No ingress rules = deny all ingress

---
# Deny ALL egress from all pods in namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  # No egress rules = deny all egress

---
# Deny BOTH ingress and egress (most restrictive)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

---

## 🟢 PATTERN 2 — Allow All (Permissive, rarely used)

```yaml
# Allow ALL ingress to all pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
  namespace: development
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - {}                    # empty rule = allow all ingress

---
# Allow ALL egress from all pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-egress
  namespace: development
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - {}                    # empty rule = allow all egress
```

---

## 🔒 PATTERN 3 — Three-Tier Application (Real World)

```
Architecture:
  Internet → frontend → backend → database
  
Rules:
  frontend: allow ingress from anywhere on 80
            allow egress to backend on 8080
            allow egress to DNS (port 53)

  backend:  allow ingress from frontend on 8080
            allow egress to database on 5432
            allow egress to DNS (port 53)

  database: allow ingress from backend on 5432
            deny all egress
```

```yaml
# ── FRONTEND POLICY ──────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - ports:                   # allow from anywhere on port 80
        - port: 80
          protocol: TCP
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - port: 8080
    - ports:                   # allow DNS resolution
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP

---
# ── BACKEND POLICY ──────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: frontend
      ports:
        - port: 8080
  egress:
    - to:
        - podSelector:
            matchLabels:
              tier: database
      ports:
        - port: 5432
    - ports:                   # DNS
        - port: 53
          protocol: UDP

---
# ── DATABASE POLICY ─────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: backend
      ports:
        - port: 5432
  # No egress rules = deny all egress from database
```

---

## 🌐 Selector Types — Who Can Talk to Whom

### podSelector — Select by Pod Labels

```yaml
ingress:
  - from:
      - podSelector:
          matchLabels:
            app: frontend    # only pods with app=frontend in SAME namespace
```

### namespaceSelector — Select by Namespace Labels

```yaml
# First, label your namespaces:
kubectl label namespace monitoring env=monitoring

ingress:
  - from:
      - namespaceSelector:
          matchLabels:
            env: monitoring  # allow from any pod in monitoring namespace
```

### Combined — podSelector AND namespaceSelector (AND logic)

```yaml
# This is a SINGLE "from" entry with BOTH selectors
# = pods with app=prometheus IN namespaces with env=monitoring

ingress:
  - from:
      - namespaceSelector:     # ← SAME "-" list item
          matchLabels:         # Both in the same
            env: monitoring    # "-" block = AND
        podSelector:           # ← NO additional "-"
          matchLabels:
            app: prometheus
```

### Multiple Entries — OR logic

```yaml
# These are SEPARATE "from" entries
# = (pods with app=frontend) OR (pods in monitoring namespace)

ingress:
  - from:
      - podSelector:           # ← first "-" entry
          matchLabels:
            app: frontend
      - namespaceSelector:     # ← second "-" entry (OR)
          matchLabels:
            env: monitoring
```

### ⚠️ AND vs OR — The Critical Distinction

```
COMMON MISTAKE: Confusing AND vs OR

# This means: pods with app=frontend  OR  pods in monitoring namespace
  from:
    - podSelector:
        matchLabels:
          app: frontend
    - namespaceSelector:        # ← NEW "-" = OR
        matchLabels:
          env: monitoring

# This means: pods with app=frontend  AND  in monitoring namespace
  from:
    - podSelector:              # ← SAME "-" entry
        matchLabels:
          app: frontend
      namespaceSelector:        # ← NO new "-" = AND
        matchLabels:
          env: monitoring

THE RULE:
  Same "-" list item (no new dash) = AND
  New "-" list item (new dash)     = OR
```

### ipBlock — Select by IP Range

```yaml
ingress:
  - from:
      - ipBlock:
          cidr: 203.0.113.0/24    # allow from this IP range
          except:
            - 203.0.113.5/32      # but NOT this specific IP

egress:
  - to:
      - ipBlock:
          cidr: 0.0.0.0/0         # allow all external IPs
          except:
            - 10.0.0.0/8          # but not internal RFC1918
            - 172.16.0.0/12
            - 192.168.0.0/16
```

---

## 🏗️ Cross-Namespace Policies

```yaml
# Allow backend in "production" namespace to receive
# traffic from frontend in "staging" namespace

# Step 1: Label the staging namespace
kubectl label namespace staging env=staging

# Step 2: Create NetworkPolicy in production namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-staging-frontend
  namespace: production           # policy lives in production
spec:
  podSelector:
    matchLabels:
      tier: backend               # targets backend pods in production
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              env: staging        # allow from staging namespace
          podSelector:
            matchLabels:
              tier: frontend      # specifically frontend pods
      ports:
        - port: 8080
```

---

## 📋 DNS Egress — Always Allow!

```yaml
# ⚠️  CRITICAL: If you deny all egress, pods CANNOT resolve DNS!
# CoreDNS runs in kube-system namespace on port 53
# You must explicitly allow DNS egress in strict policies

egress:
  # Allow DNS to kube-system (CoreDNS)
  - to:
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: kube-system
    ports:
      - port: 53
        protocol: UDP
      - port: 53
        protocol: TCP

  # OR allow DNS to anywhere (simpler but less restrictive):
  - ports:
      - port: 53
        protocol: UDP
      - port: 53
        protocol: TCP
```

---

## 💻 NetworkPolicy Commands

```bash
# ── CREATE ────────────────────────────────────────────────
kubectl apply -f network-policy.yaml

# ── VIEW ──────────────────────────────────────────────────
kubectl get networkpolicies
kubectl get netpol                    # shorthand
kubectl get netpol -n production

# Output:
# NAME              POD-SELECTOR   AGE
# frontend-policy   tier=frontend  5m
# backend-policy    tier=backend   5m
# database-policy   tier=database  5m

kubectl describe netpol frontend-policy

# ── TEST CONNECTIVITY (verify policy works) ───────────────
# Test that frontend CAN reach backend:
kubectl exec -n production frontend-pod -- \
  curl -v http://backend-svc:8080

# Test that frontend CANNOT reach database directly:
kubectl exec -n production frontend-pod -- \
  curl -v http://database-svc:5432
# Should timeout/be refused

# Test DNS still works:
kubectl exec -n production backend-pod -- \
  nslookup database-svc

# Quick connectivity test pod:
kubectl run test-pod --image=busybox --rm -it \
  -n production -- /bin/sh
# Then: wget -qO- http://service-name:port

# ── DEBUG POLICY ISSUES ───────────────────────────────────
# Check which policies affect a pod
kubectl get netpol -n production -o yaml | \
  grep -A5 "podSelector"

# Check pod labels
kubectl get pod my-pod --show-labels

# Check namespace labels
kubectl get namespace --show-labels
```

---

## 🔍 Real-World Policy Patterns

```yaml
# PATTERN: Allow monitoring namespace to scrape all pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: production
spec:
  podSelector: {}               # all pods
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - port: 9090            # Prometheus metrics port
          protocol: TCP

---
# PATTERN: Isolate namespaces (deny cross-namespace by default)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-cross-namespace
  namespace: team-a
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}       # only allow from SAME namespace (empty = all pods)
                                # namespaceSelector not specified = same namespace

---
# PATTERN: Allow only specific port from internet
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: public-web
  namespace: production
spec:
  podSelector:
    matchLabels:
      role: public-api
  policyTypes:
    - Ingress
  ingress:
    - ports:
        - port: 443
          protocol: TCP
      # No "from" = allow from ANY source (including internet)
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| NetworkPolicy with unsupported CNI | Flannel alone ignores NetworkPolicy — rules have no effect |
| Forgetting DNS egress | Pods can't resolve names after strict egress policy |
| AND vs OR in `from` list | New `-` entry = OR, same entry = AND — easy to confuse |
| Policy in wrong namespace | NetworkPolicy is namespaced — must be in same ns as target pods |
| podSelector: {} means no pods | `{}` means ALL pods — `matchLabels: {}` also matches all |
| Expecting policies to apply to Services | NetworkPolicy applies to pods — Service traffic follows pod rules |

---

## 🎓 CKA Exam Tips

```
✅ NetworkPolicy apiVersion: networking.k8s.io/v1

✅ Default deny pattern (memorize!):
   spec:
     podSelector: {}
     policyTypes: [Ingress]
   # No rules = deny all ingress

✅ Allow all pattern:
   ingress:
     - {}   # empty rule = allow everything

✅ Always allow DNS (port 53 UDP+TCP) when restricting egress!

✅ AND vs OR in selectors:
   Same "-" = AND (podSelector + namespaceSelector together)
   New "-"  = OR  (separate entries in from/to list)

✅ Label namespaces for namespace selectors:
   kubectl label namespace <ns> key=value

✅ Test policy with quick pod:
   kubectl run test --image=busybox --rm -it \
     -n <ns> -- wget -qO- http://<target>:<port>

✅ Check pod labels match policy selector:
   kubectl get pod <p> --show-labels

✅ shorthand: netpol
   kubectl get netpol -n <namespace>
```

---

## ❓ Interview & Scenario Questions

### Q1: What is a NetworkPolicy and what is the default behavior without one?
**Answer:**
A NetworkPolicy is a Kubernetes resource that controls how pods can communicate with each other and with external endpoints. Without any NetworkPolicy, all pods can communicate freely with all other pods across all namespaces — the default is open networking. NetworkPolicy is opt-in: as soon as a pod is selected by at least one NetworkPolicy, that pod gets a default-deny for the policy types listed (Ingress/Egress), and only explicitly allowed traffic is permitted. Pods not selected by any policy remain fully open.

---

### Q2: What is the difference between Ingress and Egress in NetworkPolicy?
**Answer:**
**Ingress** rules control traffic COMING INTO the selected pods — they define who is allowed to send traffic to those pods. **Egress** rules control traffic GOING OUT FROM the selected pods — they define where those pods are allowed to send traffic. For example, a database pod might have: Ingress allowing only backend pods on port 5432, and Egress denying everything (databases shouldn't initiate outbound connections). Both types can be combined in the same policy, and if a `policyTypes` field lists a type, ALL traffic of that type is denied unless explicitly allowed.

---

### Q3: Scenario — You need to allow pods in the `monitoring` namespace to scrape metrics (port 9090) from all pods in the `production` namespace. Write the NetworkPolicy.
**Answer:**
```yaml
# First label the monitoring namespace:
# kubectl label namespace monitoring purpose=monitoring

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-scrape
  namespace: production         # policy in production
spec:
  podSelector: {}               # applies to ALL pods in production
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              purpose: monitoring  # from monitoring namespace
      ports:
        - port: 9090
          protocol: TCP
```

---

### Q4: Explain the AND vs OR logic in NetworkPolicy `from` selectors.
**Answer:**
In the `from` list, **separate `-` entries** are evaluated with **OR** logic — traffic is allowed if it matches ANY of the entries. **Multiple selectors within the same `-` entry** are evaluated with **AND** logic — traffic must match ALL selectors in that entry.

```yaml
# OR: frontend pods OR monitoring namespace pods
from:
  - podSelector:          # entry 1
      matchLabels: {app: frontend}
  - namespaceSelector:    # entry 2 (OR)
      matchLabels: {env: monitoring}

# AND: frontend pods IN monitoring namespace
from:
  - podSelector:          # single entry
      matchLabels: {app: frontend}
    namespaceSelector:    # (AND, same entry — no dash)
      matchLabels: {env: monitoring}
```
This is one of the most commonly confused aspects of NetworkPolicy — the indentation and dash placement critically changes the behavior.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│              TOPIC 3.7 — NETWORK POLICIES                    │
├──────────────────────────────────────────────────────────────┤
│  DEFAULT: All pods open (no restriction)                     │
│  NetworkPolicy = OPT-IN restrictions                         │
│  Requires CNI that supports it: Calico, Cilium, Weave        │
├──────────────────────────────────────────────────────────────┤
│  KEY FIELDS:                                                 │
│  podSelector → which pods this policy applies to             │
│  policyTypes → [Ingress] | [Egress] | [Ingress, Egress]     │
│  ingress.from → who can send TO selected pods                │
│  egress.to   → where selected pods can send TO               │
├──────────────────────────────────────────────────────────────┤
│  SELECTORS:                                                  │
│  podSelector       → select by pod labels (same namespace)   │
│  namespaceSelector → select by namespace labels              │
│  ipBlock           → select by IP CIDR range                 │
│  Combined AND: same "-" entry (no new dash)                  │
│  Combined OR:  new "-" entry (separate dash)                 │
├──────────────────────────────────────────────────────────────┤
│  COMMON PATTERNS:                                            │
│  Default deny all:  podSelector:{}, policyTypes:[I/E], no rules│
│  Allow all:         ingress/egress: [{}]                     │
│  Allow DNS:         egress port 53 UDP+TCP always!           │
├──────────────────────────────────────────────────────────────┤
│  KEY COMMANDS:                                               │
│  kubectl get netpol [-n <ns>]                                │
│  kubectl describe netpol <n>                                 │
│  kubectl label namespace <ns> key=value  (for ns selector)   │
│  kubectl run test --image=busybox --rm -it -- wget <target>  │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `26-dns-in-kubernetes.md`
> *CoreDNS, FQDN format, service DNS, pod DNS, custom DNS config*
