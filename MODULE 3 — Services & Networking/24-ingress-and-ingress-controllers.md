# 📌 Topic 3.6 — Ingress & Ingress Controllers

> **Module 3 — Services & Networking**
> **File:** `24-ingress-and-ingress-controllers.md`
> **CKA Weight:** 🔥 HIGH (~20%) — Ingress is a guaranteed CKA networking topic

---

## 🎯 What You'll Learn
- What Ingress is and the problem it solves over LoadBalancer
- Ingress vs Service — the key distinction
- Ingress Controllers — what they are and popular options
- Full Ingress YAML anatomy — rules, paths, hosts, TLS
- Path-based routing — multiple services behind one entry point
- Host-based routing — virtual hosting
- TLS termination — HTTPS with Kubernetes secrets
- Default backend — 404 handling
- Ingress Class — managing multiple controllers
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Smart Receptionist Analogy 🏨
> **Without Ingress** (using LoadBalancer per service):
> Every department has its OWN front door on a different street.
> Visitors must know 10 different addresses for 10 departments.
> Each door costs money to maintain (cloud LB fees).
>
> **With Ingress:**
> ONE main entrance handles ALL visitors.
> A **smart receptionist** (Ingress Controller) reads the visitor's
> destination badge and directs them to the right department:
> - "I'm here for `/api`" → routes to API service
> - "I'm here for `/web`" → routes to Web service
> - "I'm visiting `blog.company.com`" → routes to Blog service
> - Shows HTTPS certificate at the door (TLS termination)
> - All departments reachable through ONE door (one cloud LB!)

---

## 🔬 Why Ingress Exists

```
WITHOUT INGRESS (one LoadBalancer per service):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Internet → 54.1.1.1:80  → frontend-svc  (LoadBalancer #1) $$$
  Internet → 54.1.1.2:80  → api-svc       (LoadBalancer #2) $$$
  Internet → 54.1.1.3:443 → auth-svc      (LoadBalancer #3) $$$
  Internet → 54.1.1.4:80  → blog-svc      (LoadBalancer #4) $$$

  Problems:
  ❌ 4 cloud LBs = high cost
  ❌ 4 different IPs — hard to manage
  ❌ No path routing (/api vs /web)
  ❌ No host routing (app.com vs blog.com)
  ❌ SSL termination at every service

WITH INGRESS (one LoadBalancer for everything):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  Internet → 54.1.1.1:443 (ONE LoadBalancer, ONE IP)
                  │
           [Ingress Controller]
                  │
     ┌────────────┼───────────────┐
     ▼            ▼               ▼
  /api →       /web →         blog.app.com →
  api-svc    frontend-svc      blog-svc
  (ClusterIP)  (ClusterIP)   (ClusterIP)

  Benefits:
  ✅ ONE cloud LB = low cost
  ✅ ONE IP for all services
  ✅ Path-based routing (/api, /web, /blog)
  ✅ Host-based routing (app.com, blog.app.com)
  ✅ SSL termination in ONE place (HTTPS for all)
  ✅ Rate limiting, auth, rewrites (controller features)
```

---

## 🏗️ Ingress Architecture — Two Parts

```
IMPORTANT: Ingress alone does NOTHING.
           It needs an Ingress CONTROLLER to function!

┌─────────────────────────────────────────────────────────────┐
│  INGRESS RESOURCE (your YAML — the rules)                   │
│  → Defines routing rules: host, path → service              │
│  → Defines TLS certificates                                 │
│  → Just a config object in etcd — no functionality alone    │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ read by
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  INGRESS CONTROLLER (deployed separately)                   │
│  → Watches Ingress resources in cluster                     │
│  → Implements the routing rules                             │
│  → Actually proxies the HTTP traffic                        │
│  → Examples: NGINX, Traefik, HAProxy, Istio, AWS ALB        │
│  → Usually deployed as a Deployment + LoadBalancer Service  │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ routes to
                          ▼
┌─────────────────────────────────────────────────────────────┐
│  BACKEND SERVICES (ClusterIP)                               │
│  → api-service, web-service, auth-service                   │
└─────────────────────────────────────────────────────────────┘

Analogy:
  Ingress resource = the rulebook (menu)
  Ingress controller = the chef who reads the menu and cooks
```

---

## 📄 Ingress YAML — Full Anatomy

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: production
  labels:
    app: webapp

  # ── CONTROLLER ANNOTATIONS ─────────────────────────────
  # These control NGINX Ingress Controller behavior
  annotations:
    # Which controller handles this Ingress
    kubernetes.io/ingress.class: "nginx"        # legacy method
    # (or use spec.ingressClassName below — preferred)

    # NGINX specific configurations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/rate-limit: "100"       # req/min
    nginx.ingress.kubernetes.io/auth-url: "http://auth-svc/validate"

spec:
  # ── INGRESS CLASS ────────────────────────────────────────
  ingressClassName: nginx          # preferred method (K8s 1.18+)

  # ── TLS CONFIGURATION ────────────────────────────────────
  tls:
    - hosts:
        - app.example.com
        - api.example.com
      secretName: tls-secret       # K8s Secret with cert+key

    - hosts:
        - blog.example.com
      secretName: blog-tls-secret

  # ── ROUTING RULES ────────────────────────────────────────
  rules:
    # ── HOST-BASED ROUTING ─────────────────────────────────
    - host: app.example.com        # matches this hostname
      http:
        paths:
          # Path-based routing within the host
          - path: /api             # matches /api and /api/*
            pathType: Prefix       # Prefix | Exact | ImplementationSpecific
            backend:
              service:
                name: api-service
                port:
                  number: 80

          - path: /               # matches everything else
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80

    - host: api.example.com        # separate host
      http:
        paths:
          - path: /v1
            pathType: Prefix
            backend:
              service:
                name: api-v1-service
                port:
                  number: 8080
          - path: /v2
            pathType: Prefix
            backend:
              service:
                name: api-v2-service
                port:
                  number: 8080

    # ── WILDCARD HOST ─────────────────────────────────────
    - host: "*.example.com"        # matches any subdomain
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: catch-all-service
                port:
                  number: 80

    # ── NO HOST (catches all requests without host match) ──
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: default-backend
                port:
                  number: 80

  # ── DEFAULT BACKEND ──────────────────────────────────────
  # Handles requests that don't match any rule (404/503 page)
  defaultBackend:
    service:
      name: default-404-service
      port:
        number: 80
```

---

## 🛤️ Path Types Explained

```
pathType: Prefix
  → Matches URL prefix (with trailing slash normalization)
  → path: /api matches: /api, /api/, /api/users, /api/v1/test
  → path: /   matches: everything

pathType: Exact
  → Matches EXACTLY (case-sensitive, no trailing slash)
  → path: /api matches: /api ONLY
  → /api/ → NO MATCH, /api/users → NO MATCH

pathType: ImplementationSpecific
  → Behavior defined by the Ingress Controller
  → NGINX: supports regex when nginx.ingress.kubernetes.io/use-regex: "true"
  → path: /api/.* matches regex pattern

PATH PRIORITY (multiple paths on same host):
  Exact matches take priority over Prefix
  Longer Prefix paths have priority over shorter ones
  /api/v1 wins over /api wins over /

EXAMPLE:
  Request: GET /api/v1/users

  path: /api/v1   (Prefix) → MATCH ✅ (longer = higher priority)
  path: /api      (Prefix) → would also match but loses to above
  path: /         (Prefix) → would also match but lowest priority
```

---

## 🔒 TLS Configuration — HTTPS

```yaml
# Step 1: Create TLS secret
# Option A: From files
kubectl create secret tls tls-secret \
  --cert=path/to/tls.crt \
  --key=path/to/tls.key \
  -n production

# Option B: From YAML
apiVersion: v1
kind: Secret
metadata:
  name: tls-secret
  namespace: production
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-certificate>
  tls.key: <base64-encoded-private-key>
```

```
TLS TERMINATION FLOW:

  Browser → HTTPS (443) → Ingress Controller
                              ↓
                    Decrypts TLS (uses secret cert)
                              ↓
                    HTTP (80) → Backend Service → Pod

  The pod receives PLAIN HTTP — SSL terminated at Ingress!

  Benefits:
  → Centralized cert management (one place to update certs)
  → Pods don't need SSL code
  → Can use cert-manager for auto-renewal (Let's Encrypt)

HTTP → HTTPS redirect (NGINX):
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
```

---

## ⚙️ Popular Ingress Controllers

```
┌──────────────────┬────────────────────────────────────────────┐
│ Controller       │ Characteristics                            │
├──────────────────┼────────────────────────────────────────────┤
│ NGINX Ingress    │ Most popular, widely supported             │
│ (k8s community)  │ Annotation-rich, feature complete          │
│                  │ Good docs and community                     │
│                  │ Install: helm/manifests                     │
├──────────────────┼────────────────────────────────────────────┤
│ Traefik          │ Auto-discovery, dynamic config              │
│                  │ Built-in dashboard                          │
│                  │ Good for microservices                      │
│                  │ Popular in Docker Swarm too                 │
├──────────────────┼────────────────────────────────────────────┤
│ AWS ALB          │ Uses AWS Application Load Balancer          │
│ (AWS LBC)        │ Native AWS integration                      │
│                  │ Supports WAF, Cognito auth                  │
│                  │ Each Ingress → new ALB (can be costly)      │
├──────────────────┼────────────────────────────────────────────┤
│ GCP GCLB         │ Uses Google Cloud Load Balancer             │
│ (GKE Ingress)    │ Global load balancing, CDN integration      │
│                  │ Default in GKE clusters                     │
├──────────────────┼────────────────────────────────────────────┤
│ HAProxy          │ High performance, fine-grained control      │
│                  │ Used in enterprise environments              │
├──────────────────┼────────────────────────────────────────────┤
│ Istio Gateway    │ Service mesh gateway                        │
│                  │ mTLS, advanced traffic management           │
│                  │ Used when Istio service mesh is deployed    │
└──────────────────┴────────────────────────────────────────────┘
```

---

## 🏷️ IngressClass — Multiple Controllers

```yaml
# IngressClass allows multiple controllers in same cluster
# (e.g., NGINX for internal, AWS ALB for external)

apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"  # default for all Ingresses
spec:
  controller: k8s.io/ingress-nginx

---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: alb
spec:
  controller: ingress.k8s.aws/alb
  parameters:
    apiGroup: elbv2.k8s.aws
    kind: IngressClassParams
    name: alb-params

# Ingress specifies which class to use:
spec:
  ingressClassName: nginx   # use NGINX controller
  # or
  ingressClassName: alb     # use AWS ALB controller
```

---

## 🛠️ Installing NGINX Ingress Controller

```bash
# ── Using Helm (recommended) ──────────────────────────────
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# ── Using kubectl (official manifests) ───────────────────
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/\
  controller-v1.10.0/deploy/static/provider/cloud/deploy.yaml

# ── Verify installation ───────────────────────────────────
kubectl get pods -n ingress-nginx
# NAME                                        READY   STATUS
# ingress-nginx-controller-xxx                1/1     Running

kubectl get svc -n ingress-nginx
# NAME                       TYPE           CLUSTER-IP    EXTERNAL-IP
# ingress-nginx-controller   LoadBalancer   10.96.50.10   54.200.1.50
#                                                         ↑ This is the entry point!

# ── minikube ──────────────────────────────────────────────
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

---

## 💻 Ingress Commands Reference

```bash
# ── CREATE ────────────────────────────────────────────────
kubectl apply -f ingress.yaml

# Create imperatively (basic)
kubectl create ingress my-ingress \
  --rule="app.example.com/api=api-svc:80" \
  --rule="app.example.com/=frontend-svc:80"

# With TLS
kubectl create ingress my-ingress \
  --rule="app.example.com/*=webapp:80,tls=tls-secret"

# ── VIEW ──────────────────────────────────────────────────
kubectl get ingress
kubectl get ing                   # shorthand

# Output:
# NAME             CLASS   HOSTS             ADDRESS        PORTS   AGE
# webapp-ingress   nginx   app.example.com   54.200.1.50    80,443  5m

kubectl describe ingress webapp-ingress
# Shows: Rules, Backend, TLS, Events

# Get ingress address
kubectl get ing webapp-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# ── TEST ──────────────────────────────────────────────────
# Test path routing (add Host header for host-based routing)
curl http://54.200.1.50/api -H "Host: app.example.com"
curl https://app.example.com/api  # with DNS configured

# Test from inside cluster
kubectl run test --image=busybox --rm -it -- \
  wget -qO- --header="Host: app.example.com" \
  http://ingress-nginx-controller.ingress-nginx/api

# ── TLS SECRET ────────────────────────────────────────────
kubectl create secret tls my-tls \
  --cert=cert.pem \
  --key=key.pem

# ── INGRESS CLASS ─────────────────────────────────────────
kubectl get ingressclass
kubectl describe ingressclass nginx
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| Creating Ingress without controller | Nothing happens — Ingress needs a controller to work |
| Backend service not ClusterIP | Ingress routes to ClusterIP Services — not pods directly |
| Wrong `ingressClassName` | Ingress ignored by controller if class doesn't match |
| Forgetting namespace for TLS secret | TLS secret must be in the SAME namespace as the Ingress |
| Path `/api` not matching `/api/users` | Use `pathType: Prefix` for prefix matching (not `Exact`) |
| Multiple Ingresses with same host/path | Behavior undefined — avoid conflicts |
| Annotations vary by controller | NGINX annotations don't work with Traefik or AWS ALB |

---

## 🎓 CKA Exam Tips

```
✅ Ingress apiVersion: networking.k8s.io/v1
   (not networking.k8s.io/v1beta1 — that's deprecated!)

✅ Quick Ingress creation:
   kubectl create ingress <n> \
     --rule="<host>/<path>=<svc>:<port>" \
     --class=nginx

✅ Always check Ingress Controller is running:
   kubectl get pods -n ingress-nginx

✅ Backend services must exist and be ready:
   kubectl get svc  →  services referenced in Ingress must exist

✅ TLS secret must be in SAME namespace as Ingress

✅ PathType matters:
   Prefix  → /api matches /api, /api/users, /api/v1
   Exact   → /api matches ONLY /api

✅ Test Ingress routing with Host header:
   curl -H "Host: app.example.com" http://<ingress-ip>/api

✅ Get Ingress address:
   kubectl get ing <n> -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

✅ ingressClassName vs annotation:
   Modern: spec.ingressClassName: nginx
   Legacy: annotations: kubernetes.io/ingress.class: nginx
   Both work — use spec.ingressClassName for new Ingresses
```

---

## ❓ Interview & Scenario Questions

### Q1: What is the difference between an Ingress and a Service?
**Answer:**
A Service provides internal cluster networking and basic external access (via NodePort/LoadBalancer) at Layer 4 (TCP/UDP). An Ingress works at Layer 7 (HTTP/HTTPS) and provides content-based routing — it can route requests based on hostname (`api.example.com` vs `app.example.com`) and URL path (`/api` vs `/web`). One Ingress can route to many different Services based on these rules, allowing multiple services to share a single external IP/LoadBalancer. Ingress also handles TLS termination centrally. An Ingress REQUIRES a backing Service (ClusterIP) as its target.

---

### Q2: What is an Ingress Controller and why is it needed?
**Answer:**
An Ingress Controller is a component (usually deployed as a Deployment) that reads Ingress resource definitions and implements them. The Ingress resource is just a configuration object stored in etcd — it has no functionality on its own. The controller (NGINX, Traefik, AWS ALB, etc.) watches for Ingress resources and configures an actual reverse proxy/load balancer to enforce the routing rules. Without a controller, Ingress resources are just ignored. Different controllers offer different features and annotations, so choosing the right controller matters for your use case.

---

### Q3: Scenario — You have 3 services (frontend, api, auth) and want to expose them all via HTTPS on a single IP. Design the Ingress.
**Answer:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [app.example.com]
      secretName: app-tls-secret
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /auth
            pathType: Prefix
            backend:
              service:
                name: auth-svc
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
```

---

### Q4: What is the `pathType` field and what are the differences?
**Answer:**
`pathType` controls how the path is matched: **Prefix** matches the path and all sub-paths (e.g., `/api` matches `/api`, `/api/`, `/api/users`, `/api/v1/test`). **Exact** matches the path precisely, case-sensitively, without trailing slash normalization (`/api` only matches `/api` — not `/api/` or `/api/users`). **ImplementationSpecific** delegates matching behavior to the controller (NGINX supports regex with this). In practice, `Prefix` is used most commonly. When multiple paths match, Exact takes priority over Prefix, and longer prefixes take priority over shorter ones.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│         TOPIC 3.6 — INGRESS & INGRESS CONTROLLERS            │
├──────────────────────────────────────────────────────────────┤
│  PURPOSE: L7 HTTP routing, TLS termination, one LB for many  │
│  apiVersion: networking.k8s.io/v1  Shorthand: ing            │
│  REQUIRES: Ingress Controller deployed separately            │
├──────────────────────────────────────────────────────────────┤
│  TWO PARTS:                                                  │
│  Ingress resource → routing rules (YAML in etcd)             │
│  Ingress controller → implements rules (NGINX/Traefik/etc)   │
├──────────────────────────────────────────────────────────────┤
│  ROUTING TYPES:                                              │
│  Path-based:  /api → api-svc, /web → frontend-svc           │
│  Host-based:  api.example.com → api-svc                      │
│  Combined:    host + path together                           │
├──────────────────────────────────────────────────────────────┤
│  PATH TYPES:                                                 │
│  Prefix → /api matches /api, /api/users (most used)          │
│  Exact  → /api matches ONLY /api                             │
├──────────────────────────────────────────────────────────────┤
│  TLS:                                                        │
│  Create secret: kubectl create secret tls <n> \              │
│    --cert=cert.pem --key=key.pem                             │
│  Reference in spec.tls[].secretName                         │
├──────────────────────────────────────────────────────────────┤
│  KEY COMMANDS:                                               │
│  kubectl get ing / kubectl describe ing <n>                  │
│  kubectl create ingress <n> \                                │
│    --rule="host/path=svc:port"                               │
│  minikube addons enable ingress                               │
├──────────────────────────────────────────────────────────────┤
│  CKA MUST KNOW:                                              │
│  → Check controller: kubectl get pods -n ingress-nginx       │
│  → TLS secret: same namespace as Ingress                     │
│  → ingressClassName: nginx (in spec, not annotation)         │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `25-network-policies.md`
> *Ingress/Egress rules, pod selectors, namespace isolation — CKA critical!*
