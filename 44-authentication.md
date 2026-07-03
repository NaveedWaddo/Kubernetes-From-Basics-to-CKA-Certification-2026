# 📌 Topic 6.2 — Authentication

> **Module 6 — Security**
> **File:** `44-authentication.md`
> **CKA Weight:** 🔥 HIGH (~15%) — Authentication is foundational for all security CKA tasks

---

## 🎯 What You'll Learn
- What authentication means in Kubernetes
- Two identity types: Human Users vs ServiceAccounts
- X.509 certificate authentication — the most common method
- Bearer token authentication
- OIDC authentication — enterprise SSO integration
- kubeconfig deep dive — clusters, users, contexts
- How the API server validates identity
- Creating users with certificates (CKA critical!)
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The Passport Control Analogy ✈️
> The Kubernetes API server is like **international passport control**:
>
> - Every person (human user or application) must **identify themselves**
>   before entering the country (accessing the cluster)
> - Border control (API server) checks your **passport** (credentials)
> - Different passport types are accepted:
>   - **X.509 Certificate** = physical passport (most trusted, long-lived)
>   - **Bearer Token** = temporary visa (short-lived, for services)
>   - **OIDC Token** = international driving license (from trusted 3rd party)
>   - **ServiceAccount Token** = employee badge (for pods inside the building)
>
> Key insight: Kubernetes **has no built-in user database!**
> It trusts external identity providers — certs signed by cluster CA,
> tokens from trusted issuers, or OIDC from your company's IdP.

---

## 🔬 Two Identity Types in Kubernetes

```
┌─────────────────────────────────────────────────────────────┐
│  TYPE 1: Human Users (Normal Users)                         │
│                                                             │
│  → People: developers, admins, CI/CD pipelines             │
│  → NOT stored in Kubernetes (no "User" object!)             │
│  → Managed EXTERNALLY: certificates, OIDC, webhooks        │
│  → K8s trusts the external identity provider               │
│  → kubectl commands run as a user                          │
│  → Subject in RoleBinding: kind: User                      │
│                                                             │
│  Common methods:                                            │
│  X.509 certificates (CN = username, O = group)             │
│  OIDC tokens (from Okta, Google, Azure AD, Keycloak)        │
│  Webhook authentication (external validator)               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  TYPE 2: ServiceAccounts (Pod Identities)                   │
│                                                             │
│  → Applications: pods, controllers, operators              │
│  → ARE stored in Kubernetes (ServiceAccount object exists!) │
│  → Managed INTERNALLY by Kubernetes                         │
│  → Automatically created per namespace (default SA)        │
│  → Token automatically mounted into pods                   │
│  → Subject in RoleBinding: kind: ServiceAccount            │
│                                                             │
│  Details in Topic 6.4!                                      │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔐 Authentication Method 1: X.509 Certificates

### How It Works

```
CERTIFICATE AUTHENTICATION FLOW:

  1. Admin/User generates a private key:
     openssl genrsa -out user.key 2048

  2. User creates Certificate Signing Request (CSR):
     openssl req -new -key user.key \
       -subj "/CN=jane/O=developers" \
       -out user.csr
     # CN = username in Kubernetes
     # O  = group membership in Kubernetes

  3. Admin signs CSR with cluster CA:
     openssl x509 -req -in user.csr \
       -CA /etc/kubernetes/pki/ca.crt \
       -CAkey /etc/kubernetes/pki/ca.key \
       -CAcreateserial \
       -out user.crt -days 365
     # Or use K8s CertificateSigningRequest API!

  4. User gets: user.key + user.crt
     Added to kubeconfig as credentials

  5. kubectl command:
     → kubectl sends request with client cert
     → API server: "Is this cert signed by my CA?"
     → YES → extracts CN (username) and O (groups)
     → "jane" belongs to "developers" group
     → RBAC decides what jane can do

USERNAME AND GROUPS from certificate:
  CN (Common Name)       = Kubernetes username
  O  (Organization)      = Kubernetes group
  
  Example: /CN=jane/O=developers/O=team-alpha
  Username: jane
  Groups:   developers, team-alpha
```

### Creating a User with K8s CertificateSigningRequest API (CKA Method)

```bash
# Step 1: Generate private key
openssl genrsa -out jane.key 2048

# Step 2: Create CSR
openssl req -new -key jane.key \
  -subj "/CN=jane/O=developers" \
  -out jane.csr

# Step 3: Base64-encode the CSR
cat jane.csr | base64 | tr -d '\n'

# Step 4: Create CertificateSigningRequest object
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: $(cat jane.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400   # 24 hours
  usages:
    - client auth             # for kubectl authentication
EOF

# Step 5: Approve the CSR (admin action)
kubectl get csr
# NAME   AGE   SIGNERNAME                            REQUESTOR   CONDITION
# jane   10s   kubernetes.io/kube-apiserver-client   admin       Pending

kubectl certificate approve jane
# jane approved ✅

# Step 6: Extract the signed certificate
kubectl get csr jane \
  -o jsonpath='{.status.certificate}' | base64 -d > jane.crt

# Step 7: Add to kubeconfig
kubectl config set-credentials jane \
  --client-certificate=jane.crt \
  --client-key=jane.key

kubectl config set-context jane-context \
  --cluster=my-cluster \
  --user=jane \
  --namespace=development

kubectl config use-context jane-context

# Verify:
kubectl auth whoami
# Username: jane
# Groups: [developers]
```

---

## 🎫 Authentication Method 2: Bearer Tokens

### ServiceAccount Tokens (Legacy)

```yaml
# Automatically mounted into pods at:
# /var/run/secrets/kubernetes.io/serviceaccount/token

# Pod can use it to call API server:
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/api/v1/namespaces/default/pods

# Static long-lived token (legacy — avoid in production):
apiVersion: v1
kind: Secret
metadata:
  name: my-sa-token
  annotations:
    kubernetes.io/service-account.name: my-sa
type: kubernetes.io/service-account-token
# K8s auto-fills .data.token with the JWT token
```

### Projected ServiceAccount Tokens (Modern — K8s 1.21+)

```yaml
# Short-lived, audience-bound, automatically rotated
spec:
  volumes:
    - name: token
      projected:
        sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 3600    # 1 hour
              audience: my-api-server    # bound to specific audience
  containers:
    - name: app
      volumeMounts:
        - name: token
          mountPath: /var/run/secrets/tokens

# Benefits over legacy tokens:
# ✅ Short-lived (expiry enforced)
# ✅ Audience-bound (can't be reused for other services)
# ✅ Automatically rotated by kubelet
# ✅ More secure (shorter window if stolen)
```

### Static Bootstrap Tokens (Cluster Setup)

```bash
# Used during cluster bootstrap (kubeadm)
# Format: <token-id>.<token-secret>
# Example: abcdef.1234567890abcdef

# Create bootstrap token:
kubeadm token create
# Returns: abcdef.1234567890abcdef

kubeadm token list
# List active bootstrap tokens

# Used in node join command:
kubeadm join <master>:6443 \
  --token abcdef.1234567890abcdef \
  --discovery-token-ca-cert-hash sha256:...
```

---

## 🌐 Authentication Method 3: OIDC

```
OIDC = OpenID Connect
     = OAuth 2.0 + Identity Layer
     = Enterprise SSO integration

How it works:
  ┌─────────┐                ┌──────────────┐         ┌──────────────┐
  │  User   │─── login ────►│  OIDC IdP    │         │  K8s API     │
  │ (Jane)  │               │  (Okta/      │         │  Server      │
  │         │◄── JWT ───────│   Google/    │         │              │
  │         │   ID Token    │   Azure AD/  │         │              │
  │         │               │   Keycloak)  │         │              │
  │         │─── kubectl ─────────────────────────►  │              │
  │         │   (with JWT   │              │         │  Validates:  │
  │         │    in Bearer) │              │         │  - JWT sig   │
  │         │               │              │         │  - expiry    │
  │         │               │              │         │  - issuer    │
  └─────────┘               └──────────────┘         └──────────────┘

API Server OIDC flags (configured in kube-apiserver):
  --oidc-issuer-url=https://accounts.google.com
  --oidc-client-id=my-kubernetes-app
  --oidc-username-claim=email      # which JWT claim = username
  --oidc-groups-claim=groups       # which JWT claim = groups
  --oidc-ca-file=/path/to/oidc-ca.crt

Benefits:
  ✅ Centralized identity (company SSO)
  ✅ MFA enforced at IdP level
  ✅ Short-lived tokens (no long-lived certs to manage)
  ✅ Works with existing company infrastructure
  ✅ Easy user onboarding/offboarding (managed at IdP)
```

---

## 📁 kubeconfig Deep Dive

```yaml
# ~/.kube/config — Full structure
apiVersion: v1
kind: Config
current-context: prod-jane    # active context

# ── CLUSTERS ──────────────────────────────────────────────
clusters:
  - name: prod-cluster
    cluster:
      server: https://prod-api.example.com:6443
      certificate-authority-data: <base64-ca-cert>
      # OR: certificate-authority: /path/to/ca.crt
      # insecure-skip-tls-verify: true  ← NEVER in production!

  - name: dev-cluster
    cluster:
      server: https://dev-api.example.com:6443
      certificate-authority-data: <base64-ca-cert>

# ── USERS ─────────────────────────────────────────────────
users:
  # X.509 certificate user:
  - name: jane-cert
    user:
      client-certificate-data: <base64-cert>   # jane.crt
      client-key-data: <base64-key>             # jane.key

  # Token-based user:
  - name: jane-token
    user:
      token: eyJhbGciOiJSUzI1NiIs...  # bearer token

  # OIDC user (via kubelogin plugin):
  - name: jane-oidc
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1beta1
        command: kubectl
        args:
          - oidc-login
          - get-token
          - --oidc-issuer-url=https://accounts.google.com
          - --oidc-client-id=my-k8s-app

# ── CONTEXTS ──────────────────────────────────────────────
contexts:
  - name: prod-jane
    context:
      cluster: prod-cluster
      user: jane-cert
      namespace: production      # optional default namespace

  - name: dev-jane
    context:
      cluster: dev-cluster
      user: jane-cert
      namespace: development
```

### kubeconfig Management Commands

```bash
# View current config
kubectl config view
kubectl config view --minify    # only current context
kubectl config view --raw       # show secrets (base64)

# Context operations
kubectl config get-contexts
kubectl config current-context
kubectl config use-context prod-jane
kubectl config set-context --current --namespace=production

# Add cluster
kubectl config set-cluster my-cluster \
  --server=https://api.example.com:6443 \
  --certificate-authority=ca.crt

# Add credentials
kubectl config set-credentials jane \
  --client-certificate=jane.crt \
  --client-key=jane.key

# Add context
kubectl config set-context prod-jane \
  --cluster=my-cluster \
  --user=jane \
  --namespace=production

# Remove
kubectl config unset users.jane
kubectl config delete-context old-context
kubectl config delete-cluster old-cluster

# Merge kubeconfigs
KUBECONFIG=~/.kube/config:~/new-cluster.yaml \
  kubectl config view --flatten > ~/.kube/merged-config
```

---

## 🔍 Checking Your Identity

```bash
# Who am I? (K8s 1.28+)
kubectl auth whoami
# ATTRIBUTE   VALUE
# Username    jane
# Groups      [developers system:authenticated]

# Older method:
kubectl get --raw /api/v1 | head  # if this works, you're authenticated

# Decode your token (if using ServiceAccount):
# The token is a JWT — decode the payload:
cat /var/run/secrets/kubernetes.io/serviceaccount/token | \
  cut -d. -f2 | base64 -d | python3 -m json.tool
# Shows: sub, iss, aud, exp, namespace, serviceaccount name

# Test authentication by checking permissions:
kubectl auth can-i get pods
# yes (or no, if not authorized)
```

---

## 🏗️ System Components Authentication

```
Kubernetes system components have special user identities:

Component               → Username
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
kube-scheduler          → system:kube-scheduler
kube-controller-manager → system:kube-controller-manager
kube-proxy              → system:node-proxier
kubelet                 → system:node:<node-name>
CoreDNS                 → system:serviceaccount:kube-system:coredns

Built-in groups:
  system:masters               → cluster-admin level (highest!)
  system:authenticated         → all authenticated users
  system:unauthenticated       → anonymous requests
  system:nodes                 → all kubelet nodes
  system:serviceaccounts       → all service accounts
  system:serviceaccounts:<ns>  → all SAs in namespace

The admin kubeconfig (from kubeadm) uses:
  CN=kubernetes-admin, O=system:masters
  → member of system:masters group → cluster-admin access
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| Kubernetes has a user database | K8s has NO user objects — identity from external sources |
| Certificates never expire | certs have expiry (`-days` flag) — rotate before expiry! |
| `insecure-skip-tls-verify: true` | Vulnerable to MITM — never in production |
| Long-lived static tokens | Use short-lived projected tokens or OIDC instead |
| Sharing kubeconfig with secrets | kubeconfig contains credentials — treat like a password file |
| group from O field works with RBAC | Yes! RoleBinding with `kind: Group` + `name: developers` |

---

## 🎓 CKA Exam Tips

```
✅ Create a user certificate — full workflow:
   1. openssl genrsa -out user.key 2048
   2. openssl req -new -key user.key -subj "/CN=user/O=group" -out user.csr
   3. Create CertificateSigningRequest K8s object
   4. kubectl certificate approve <name>
   5. kubectl get csr <name> -o jsonpath='{.status.certificate}' | base64 -d > user.crt
   6. kubectl config set-credentials ...
   7. kubectl config set-context ...

✅ Check current user:
   kubectl auth whoami   (K8s 1.28+)
   kubectl config current-context

✅ Check CSR status:
   kubectl get csr
   kubectl certificate approve <name>
   kubectl certificate deny <name>

✅ CertificateSigningRequest YAML:
   spec.request: <base64-csr>
   spec.signerName: kubernetes.io/kube-apiserver-client
   spec.usages: [client auth]

✅ Username = CN field of certificate
   Group = O field of certificate
   Multiple groups: /CN=user/O=group1/O=group2
```

---

## ❓ Interview & Scenario Questions

### Q1: Does Kubernetes have a built-in user management system?
**Answer:**
No — Kubernetes has no built-in user database or "User" API object. Human users are managed externally and Kubernetes trusts external identity providers: X.509 certificates signed by the cluster CA, OIDC tokens from providers like Google/Okta/Azure AD, or webhook authentication. The only identity type managed BY Kubernetes is ServiceAccounts (for pods and applications). This design means user management (creating, revoking users) happens outside K8s — typically by revoking certificates or removing users from the OIDC provider.

---

### Q2: Walk through the steps to create a new Kubernetes user with certificate authentication.
**Answer:**
```bash
# 1. Generate private key
openssl genrsa -out alice.key 2048

# 2. Create CSR (CN=username, O=group)
openssl req -new -key alice.key \
  -subj "/CN=alice/O=team-alpha" -out alice.csr

# 3. Create K8s CertificateSigningRequest
kubectl apply -f - <<EOF
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: alice
spec:
  request: $(cat alice.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages: [client auth]
EOF

# 4. Approve
kubectl certificate approve alice

# 5. Extract certificate
kubectl get csr alice \
  -o jsonpath='{.status.certificate}' | base64 -d > alice.crt

# 6. Configure kubectl
kubectl config set-credentials alice \
  --client-certificate=alice.crt --client-key=alice.key
kubectl config set-context alice \
  --cluster=my-cluster --user=alice
```

---

### Q3: What is the difference between X.509 certificate auth and OIDC auth?
**Answer:**
**X.509 certificates** are issued by the cluster's own CA, embedded in kubeconfig, and long-lived (until expiry). They're simple to set up but hard to revoke (must wait for expiry or rebuild entire CA). Username comes from the CN field, groups from the O fields. **OIDC** delegates identity to an external Identity Provider (Okta, Google, Azure AD). Tokens are short-lived JWTs, automatically expired. Revoking a user means removing them from the IdP — much easier. OIDC supports MFA at the IdP level and integrates with corporate SSO. OIDC is preferred for production enterprise clusters; certificates are simpler for development and automation.

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│               TOPIC 6.2 — AUTHENTICATION                     │
├──────────────────────────────────────────────────────────────┤
│  TWO IDENTITY TYPES:                                         │
│  Human Users  → no K8s object, external auth (cert/OIDC)    │
│  ServiceAccounts → K8s objects, tokens, for pods            │
├──────────────────────────────────────────────────────────────┤
│  AUTH METHODS:                                               │
│  X.509 cert → CN=username, O=group, signed by cluster CA    │
│  Bearer token → ServiceAccount JWT tokens                   │
│  OIDC → external IdP (Google/Okta/Azure AD), enterprise SSO │
│  Webhook → external auth service                            │
├──────────────────────────────────────────────────────────────┤
│  CREATE USER (CKA workflow):                                 │
│  1. openssl genrsa -out user.key 2048                        │
│  2. openssl req -new -key user.key \                         │
│       -subj "/CN=<name>/O=<group>" -out user.csr             │
│  3. kubectl apply -f csr-object.yaml                         │
│  4. kubectl certificate approve <name>                       │
│  5. Extract cert → add to kubeconfig                         │
├──────────────────────────────────────────────────────────────┤
│  kubeconfig contains: clusters + users + contexts            │
│  kubectl config use-context <ctx>                            │
│  kubectl auth whoami  (K8s 1.28+)                           │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `45-authorization-rbac.md`
> *Roles, ClusterRoles, RoleBindings, ClusterRoleBindings — CKA Critical!*
