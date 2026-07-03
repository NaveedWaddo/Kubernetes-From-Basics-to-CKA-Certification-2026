# 📌 Topic 6.1 — Kubernetes Security Overview

> **Module 6 — Security**
> **File:** `43-kubernetes-security-overview.md`
> **CKA Weight:** 🔥 HIGH (~15%) — Security is one of the highest-weighted CKA domains

---

## 🎯 What You'll Learn
- The 4C Security Model — Cloud, Cluster, Container, Code
- Kubernetes security attack surface — what needs protecting
- Defense in depth — layered security strategy
- The principle of least privilege in Kubernetes
- Security domains overview — Authentication, Authorization, Admission, Network, Runtime
- Common Kubernetes attack vectors
- Security checklist for production clusters
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Castle Defense Analogy 🏰
> Kubernetes security is like defending a **medieval castle**:
>
> - **Moat (Cloud Security)** — first line of defense, keep attackers away from the walls
> - **Castle Walls (Cluster Security)** — protect the cluster itself (API server, etcd, nodes)
> - **Room Locks (Container Security)** — isolate rooms inside the castle
> - **Safes (Code Security)** — protect valuables (secrets) even if someone gets inside
>
> Security is LAYERED — if one layer fails, the next catches it.
> This is **Defense in Depth** — the core security philosophy.
>
> A flaw in one layer doesn't mean total compromise.
> Multiple independent layers = attacker must breach ALL of them.

---

## 🔬 The 4C Security Model

```
THE 4Cs — Each layer protects the next:

┌─────────────────────────────────────────────────────────────┐
│                    CLOUD                                    │
│  (Infrastructure security — your responsibility or IaaS)    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                  CLUSTER                            │   │
│  │  (Kubernetes control plane + node security)         │   │
│  │                                                     │   │
│  │  ┌───────────────────────────────────────────────┐  │   │
│  │  │               CONTAINER                       │  │   │
│  │  │  (Container runtime + image security)         │  │   │
│  │  │                                               │  │   │
│  │  │  ┌─────────────────────────────────────────┐  │  │   │
│  │  │  │              CODE                        │  │  │   │
│  │  │  │  (Application security + secrets mgmt)  │  │  │   │
│  │  │  └─────────────────────────────────────────┘  │  │   │
│  │  └───────────────────────────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘

If CLOUD is compromised:  attacker has raw compute/network
If CLUSTER is compromised: attacker can control all workloads
If CONTAINER is compromised: attacker can affect the running pod
If CODE is compromised: attacker exploits the application itself
```

---

## 🌩️ Layer 1 — Cloud Security

```
What it covers:
  → Securing the underlying infrastructure
  → Network security (VPC, firewalls, security groups)
  → IAM for cloud services
  → Physical data center security (managed by cloud provider)
  → Encryption of disks, network traffic

Kubernetes-specific concerns:
  ┌─────────────────────────────────────────────────────────┐
  │  API Server Access:                                     │
  │  → Is port 6443 exposed to the internet? (DON'T!)       │
  │  → Use VPN, bastion host, or private endpoint           │
  │  → Security Groups: only allow trusted IPs to 6443      │
  │                                                         │
  │  etcd Access:                                           │
  │  → etcd ports (2379/2380) should NEVER be public        │
  │  → Only API server should reach etcd                    │
  │  → Use encryption at rest for etcd volume               │
  │                                                         │
  │  Node Access:                                           │
  │  → Disable SSH or use SSM/IAP instead of SSH keys       │
  │  → Node security groups: only allow necessary traffic   │
  │  → Use managed node groups (EKS, GKE) when possible     │
  │                                                         │
  │  Cloud IAM:                                             │
  │  → Least privilege IAM roles for nodes (IRSA on EKS)    │
  │  → Workload Identity on GKE                             │
  └─────────────────────────────────────────────────────────┘
```

---

## 🏛️ Layer 2 — Cluster Security

```
What it covers:
  → Kubernetes API server security
  → etcd security
  → Node security (kubelet, runtime)
  → Network policies
  → RBAC (who can do what in the cluster)
  → Admission controllers (what can run)

Key components:
  ┌─────────────────────────────────────────────────────────┐
  │  Authentication (Who are you?):                         │
  │  → TLS client certificates                             │
  │  → Bearer tokens (ServiceAccount tokens)               │
  │  → OIDC (OpenID Connect) — enterprise SSO              │
  │  → Webhook authentication                              │
  │                                                         │
  │  Authorization (What can you do?):                      │
  │  → RBAC (Role-Based Access Control) — main mechanism   │
  │  → ABAC (Attribute-Based) — legacy                     │
  │  → Node authorization — for kubelets                   │
  │  → Webhook authorization                               │
  │                                                         │
  │  Admission Control (Is this allowed?):                  │
  │  → PodSecurity Admission                               │
  │  → ResourceQuota, LimitRange                           │
  │  → Custom webhooks (OPA/Gatekeeper)                    │
  │                                                         │
  │  Network Security:                                      │
  │  → NetworkPolicies (micro-segmentation)                 │
  │  → TLS everywhere (all cluster communication)           │
  │  → mTLS between pods (service mesh)                    │
  └─────────────────────────────────────────────────────────┘
```

---

## 📦 Layer 3 — Container Security

```
What it covers:
  → Container image security (what's IN the image)
  → Container runtime security (how it runs)
  → Pod security contexts (Linux capabilities, users)
  → Resource isolation

Key areas:
  ┌─────────────────────────────────────────────────────────┐
  │  Image Security:                                        │
  │  → Use trusted base images (official, minimal)         │
  │  → Scan images for CVEs (Trivy, Snyk, Clair)           │
  │  → Never run as root in containers                      │
  │  → Use distroless/scratch images for minimal attack     │
  │    surface                                              │
  │  → Sign images (Cosign, Notary)                         │
  │  → Use private registries (not public Docker Hub)       │
  │                                                         │
  │  Runtime Security:                                      │
  │  → securityContext: runAsNonRoot: true                  │
  │  → securityContext: readOnlyRootFilesystem: true        │
  │  → Drop ALL Linux capabilities, add only needed         │
  │  → No privileged containers                             │
  │  → Limit hostPath mounts (never /, /etc, /var)          │
  │  → Pod Security Admission (Restricted profile)          │
  │                                                         │
  │  Runtime Monitoring:                                    │
  │  → Falco — detects anomalous syscalls                   │
  │  → Sysdig — container-aware monitoring                  │
  │  → Datadog/Aqua Security                                │
  └─────────────────────────────────────────────────────────┘
```

---

## 💻 Layer 4 — Code Security

```
What it covers:
  → Application-level security
  → Secure coding practices
  → Secrets management within the application
  → Dependency scanning

Key areas:
  ┌─────────────────────────────────────────────────────────┐
  │  Secrets Management:                                    │
  │  → Never hardcode secrets in code                       │
  │  → Never store secrets in environment variables         │
  │    (visible in ps, logs, kubectl describe)              │
  │  → Use Kubernetes Secrets + volume mounts               │
  │  → Better: External secrets (Vault, AWS SM)             │
  │                                                         │
  │  Supply Chain Security:                                 │
  │  → Scan dependencies for CVEs (OWASP, Snyk)            │
  │  → Pin dependency versions                              │
  │  → Verify checksums/signatures                          │
  │  → SBOM (Software Bill of Materials)                    │
  │                                                         │
  │  Application Hardening:                                 │
  │  → Input validation, output encoding                    │
  │  → Avoid SSRF (Server-Side Request Forgery)             │
  │    — attacker uses app to query metadata service        │
  │  → Principle of least privilege in app code             │
  │  → Rate limiting, authentication in app                 │
  └─────────────────────────────────────────────────────────┘
```

---

## 🔑 Kubernetes Security Domains — Overview

```
┌─────────────────────────────────────────────────────────────┐
│            KUBERNETES SECURITY DOMAINS                      │
├─────────────────────────────────────────────────────────────┤
│  1. AUTHENTICATION (Who are you?)                           │
│     → Certificates, tokens, OIDC, ServiceAccounts          │
│     → Topics: 6.2, 6.4                                     │
│                                                             │
│  2. AUTHORIZATION (What can you do?)                        │
│     → RBAC: Roles, ClusterRoles, Bindings                  │
│     → Topics: 6.3                                          │
│                                                             │
│  3. ADMISSION (Is this resource allowed?)                   │
│     → PodSecurity Admission, ResourceQuota, Webhooks       │
│     → Topics: 6.6, 6.10                                    │
│                                                             │
│  4. NETWORK SECURITY (Who can talk to whom?)                │
│     → NetworkPolicies, TLS, mTLS                           │
│     → Topics: 3.7, 6.7                                     │
│                                                             │
│  5. WORKLOAD SECURITY (How securely does it run?)           │
│     → SecurityContext, capabilities, users                  │
│     → Topics: 6.5                                          │
│                                                             │
│  6. DATA SECURITY (Is sensitive data protected?)            │
│     → Secrets encryption at rest, image security           │
│     → Topics: 6.8, 6.9                                     │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚔️ Common Kubernetes Attack Vectors

```
ATTACK 1: Exposed API Server
  Attacker finds port 6443 open on internet
  → Brute-force or exploit auth bypass
  → Full cluster control
  Prevention: Private endpoint, VPN, firewall rules

ATTACK 2: Compromised ServiceAccount Token
  App has overly-permissive SA token
  Attacker exploits app → steals token from /var/run/secrets/
  → Uses token to query K8s API → exfiltrate secrets, deploy pods
  Prevention: Least-privilege SA, disable automounting,
              use short-lived projected tokens

ATTACK 3: Container Escape
  App has vulnerability, attacker gets RCE inside container
  Container runs as root with hostPID or hostNetwork
  → Escapes to node → accesses all pods on node
  Prevention: Non-root containers, drop capabilities,
              readOnlyRootFilesystem, no privileged mode

ATTACK 4: Supply Chain Attack
  Attacker compromises a dependency or base image
  → Malicious code in your running containers
  Prevention: Image scanning, signed images, SBOM,
              pin dependency versions

ATTACK 5: Secrets Exfiltration from etcd
  Attacker gains etcd access (unprotected port 2379)
  → Reads all Kubernetes secrets (base64-decoded)
  Prevention: etcd encryption at rest, firewall port 2379

ATTACK 6: Privilege Escalation via RBAC
  Pod has wildcard RBAC (* verbs on * resources)
  Attacker exploits → creates privileged pod → node access
  Prevention: Least privilege RBAC, audit bindings regularly

ATTACK 7: Kubernetes Metadata Service SSRF
  App vulnerable to SSRF → attacker queries cloud metadata
  → Gets node IAM credentials → accesses cloud resources
  Prevention: Restrict metadata access, use IRSA/Workload Identity
```

---

## ✅ Kubernetes Security Hardening Checklist

```
CLUSTER-LEVEL:
  [ ] API server not exposed to internet
  [ ] etcd ports (2379/2380) firewalled
  [ ] RBAC enabled (--authorization-mode=RBAC)
  [ ] Node authorization enabled
  [ ] Admission controllers configured
  [ ] Secrets encryption at rest enabled
  [ ] Audit logging enabled
  [ ] Network policies applied to all namespaces
  [ ] Regular certificate rotation
  [ ] kubeconfig files secured (not in public repos!)

NODE-LEVEL:
  [ ] Nodes not directly SSH-accessible from internet
  [ ] kubelet authentication enabled (--anonymous-auth=false)
  [ ] Node OS regularly patched
  [ ] CIS Benchmark applied to nodes

WORKLOAD-LEVEL:
  [ ] No privileged containers
  [ ] runAsNonRoot: true on all containers
  [ ] readOnlyRootFilesystem: true where possible
  [ ] Resource requests and limits set
  [ ] Pod Security Admission enforced (Restricted profile)
  [ ] No hostPath mounts to sensitive directories
  [ ] No hostNetwork, hostPID, hostIPC

SUPPLY CHAIN:
  [ ] All images scanned for CVEs
  [ ] Image signing enabled
  [ ] No :latest tags in production
  [ ] Private registry used (not public Docker Hub)
  [ ] Base images from trusted sources only

ACCESS CONTROL:
  [ ] Least privilege RBAC for all users
  [ ] ServiceAccount tokens not automounted by default
  [ ] Human users via OIDC (not certificates)
  [ ] Regular RBAC audit
  [ ] No cluster-admin bindings except for admins
```

---

## 💻 Security Audit Commands

```bash
# ── CHECK RBAC PERMISSIONS ────────────────────────────────
# What can the current user do?
kubectl auth can-i --list

# Can a specific user do something?
kubectl auth can-i get secrets --as=developer-user -n production

# Check what a ServiceAccount can do
kubectl auth can-i list pods --as=system:serviceaccount:default:my-sa

# ── FIND OVERPERMISSIVE BINDINGS ──────────────────────────
# ClusterRoleBindings to cluster-admin
kubectl get clusterrolebindings \
  -o jsonpath='{range .items[?(@.roleRef.name=="cluster-admin")]}{.metadata.name}{"\t"}{range .subjects[*]}{.kind}{"\t"}{.name}{"\n"}{end}{end}'

# ── CHECK PRIVILEGED PODS ─────────────────────────────────
kubectl get pods -A -o json | jq \
  '.items[] | select(.spec.containers[].securityContext.privileged==true) |
   .metadata.namespace + "/" + .metadata.name'

# ── CHECK PODS RUNNING AS ROOT ────────────────────────────
kubectl get pods -A -o json | jq \
  '.items[] | select(
    .spec.securityContext.runAsNonRoot!=true and
    .spec.containers[].securityContext.runAsNonRoot!=true
  ) | .metadata.namespace + "/" + .metadata.name'

# ── CHECK API SERVER FLAGS ────────────────────────────────
kubectl describe pod kube-apiserver-controlplane -n kube-system \
  | grep -E "anonymous|authorization|admission"

# ── CHECK NETWORK POLICIES ────────────────────────────────
# Namespaces with no NetworkPolicy (exposed!):
kubectl get namespaces -o jsonpath='{.items[*].metadata.name}' | \
  tr ' ' '\n' | while read ns; do
    count=$(kubectl get netpol -n $ns 2>/dev/null | wc -l)
    [ $count -eq 1 ] && echo "NO POLICY: $ns"
  done
```

---

## ⚠️ Common Security Mistakes

| Mistake | Risk | Fix |
|---------|------|-----|
| Running containers as root | Container escape → node compromise | `runAsNonRoot: true` |
| Wildcard RBAC (`*` verbs) | Privilege escalation | Least-privilege specific verbs |
| API server on public internet | Direct cluster takeover | Private endpoint + VPN |
| No encryption at rest for etcd | Secrets readable from etcd | EncryptionConfiguration |
| Auto-mounting SA tokens | Token theft via compromised app | `automountServiceAccountToken: false` |
| No NetworkPolicies | Lateral movement between pods | Apply default-deny + allow rules |
| Secrets in environment variables | Visible in pod describe/logs | Use volume mounts instead |
| Using :latest image tag | Unpredictable, unscanned images | Pin versions, scan images |

---

## 🎓 CKA Exam Tips

```
✅ CKA Security Domain (~15% weight) covers:
   → RBAC (most tested!)
   → ServiceAccounts
   → SecurityContext
   → NetworkPolicy
   → Secrets management

✅ The 4Cs are conceptual — know them for MCQ/design questions:
   Cloud → Cluster → Container → Code

✅ Most common security tasks in CKA:
   1. Create Role/ClusterRole + binding for a user
   2. Fix overpermissive RBAC
   3. Set SecurityContext on a pod
   4. Create/use ServiceAccount
   5. Fix NetworkPolicy issues

✅ Security commands to know:
   kubectl auth can-i <verb> <resource>
   kubectl auth can-i --list
   kubectl auth can-i <verb> <resource> --as=<user>

✅ Always think "least privilege":
   → Give only the access that's needed
   → Use namespaced Roles over ClusterRoles
   → Specific verbs over wildcard (*)
```

---

## ❓ Interview & Scenario Questions

### Q1: Explain the 4C security model for Kubernetes.
**Answer:**
The 4C model represents layered security in a cloud-native environment: (1) **Cloud** — securing the underlying infrastructure: cloud IAM, VPCs, firewalls, API server access, etcd protection. (2) **Cluster** — Kubernetes-level security: RBAC, authentication, admission controllers, network policies, TLS. (3) **Container** — securing what runs: non-root containers, dropped capabilities, read-only filesystems, image scanning, pod security contexts. (4) **Code** — application security: no hardcoded secrets, dependency scanning, avoiding SSRF, input validation. Each layer defends against breaches in the outer layers — defense in depth.

---

### Q2: What is the principle of least privilege in Kubernetes and how is it applied?
**Answer:**
Least privilege means granting only the minimum permissions needed to perform a task — nothing more. In Kubernetes: (1) **RBAC** — create Roles with specific verbs (get, list) on specific resources, not wildcards; use namespaced Roles instead of ClusterRoles; (2) **ServiceAccounts** — disable automounting for pods that don't need API access; create dedicated SAs with minimal permissions; (3) **SecurityContext** — drop all Linux capabilities, add only specific ones needed; run as non-root; (4) **Network** — default-deny NetworkPolicies, allow only required communication; (5) **Cloud IAM** — nodes get minimal IAM roles; use IRSA/Workload Identity for pod-level cloud access.

---

### Q3: What are the most critical Kubernetes security misconfigurations to look for?
**Answer:**
The most critical: (1) **API server exposed to internet** — allows direct attacks on the cluster control plane; (2) **Overpermissive RBAC** — `cluster-admin` bindings or wildcard permissions give attackers full control if they compromise a pod; (3) **Privileged containers** — `privileged: true` is essentially root on the node; (4) **No etcd encryption** — secrets readable in plaintext from etcd; (5) **Automounted ServiceAccount tokens** — every pod gets a token that can be stolen and used to query the API; (6) **No NetworkPolicies** — compromised pod can reach any other pod or service in cluster; (7) **No image scanning** — running images with known CVEs.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│        TOPIC 6.1 — KUBERNETES SECURITY OVERVIEW              │
├──────────────────────────────────────────────────────────────┤
│  THE 4C MODEL:                                               │
│  Cloud     → infrastructure, network, IAM                    │
│  Cluster   → RBAC, auth, admission, NetworkPolicy, TLS       │
│  Container → securityContext, image scanning, capabilities   │
│  Code      → secrets mgmt, dependency scanning, SSRF         │
├──────────────────────────────────────────────────────────────┤
│  SECURITY DOMAINS:                                           │
│  Authentication → who are you? (certs, tokens, OIDC)        │
│  Authorization  → what can you do? (RBAC)                    │
│  Admission      → is resource allowed? (PSA, webhooks)       │
│  Network        → who talks to whom? (NetworkPolicy, TLS)    │
│  Workload       → how securely does it run? (secCtx)         │
│  Data           → is sensitive data protected? (encryption)  │
├──────────────────────────────────────────────────────────────┤
│  PRINCIPLE: LEAST PRIVILEGE everywhere                       │
│  STRATEGY:  DEFENSE IN DEPTH (multiple layers)               │
├──────────────────────────────────────────────────────────────┤
│  TOP MISCONFIGURATIONS:                                      │
│  Public API server, wildcard RBAC, privileged containers,    │
│  no etcd encryption, automounted SA tokens, no NetworkPolicy │
├──────────────────────────────────────────────────────────────┤
│  KEY COMMANDS:                                               │
│  kubectl auth can-i <verb> <resource>                        │
│  kubectl auth can-i --list                                   │
│  kubectl auth can-i get pods --as=<user>                     │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `44-authentication.md`
> *Users vs ServiceAccounts, kubeconfig, X.509 certificates, tokens, OIDC*
