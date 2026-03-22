# 📌 Topic 1.5 — kubectl Basics

> **Module 1 — Core Concepts & Kubernetes Architecture**
> **File:** `05-kubectl-basics.md`
> **CKA Weight:** 🔥 CRITICAL — Every single CKA task uses kubectl. Speed = score.

---

## 🎯 What You'll Learn
- What kubectl is and how it works internally
- Installation and configuration
- kubeconfig — contexts, clusters, users
- Imperative vs Declarative approaches
- Essential command categories with examples
- Output formats (wide, yaml, json, jsonpath)
- Shortcuts, aliases and speed tips for CKA exam
- Interview & scenario-based questions with answers

---

## 🧠 Core Concept Explained

### The TV Remote Analogy 📺
> `kubectl` is like a **universal TV remote** for your Kubernetes cluster:
>
> - The remote (kubectl) doesn't DO anything itself
> - It sends **signals (API calls)** to the TV (kube-apiserver)
> - The TV interprets the signal and takes action
> - You can control ANY cluster with the same remote — just point it at a different TV (context)
> - Without the remote, you'd have to manually press buttons on the TV (direct API calls)

---

## 🔧 How kubectl Works Internally

```
You type:  kubectl get pods

           │
           ▼
┌─────────────────────────────────────────────────┐
│                    kubectl                      │
│                                                 │
│  1. Reads ~/.kube/config (kubeconfig)           │
│     → Which cluster am I talking to?            │
│     → Which user credentials do I use?          │
│     → Which namespace is default?               │
│                                                 │
│  2. Constructs HTTP request:                    │
│     GET https://apiserver:6443/api/v1/pods      │
│                                                 │
│  3. Sends request with TLS client cert          │
│                                                 │
│  4. Receives JSON response from apiserver       │
│                                                 │
│  5. Formats and displays output to terminal     │
└─────────────────────────────────────────────────┘
           │
           ▼
     kube-apiserver
     (authenticates, authorizes, returns data)
```

---

## 📦 Installation

```bash
# ── Linux ─────────────────────────────────────────────────
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
  https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl

# Verify
kubectl version --client

# ── macOS (Homebrew) ──────────────────────────────────────
brew install kubectl

# ── Windows (Chocolatey) ─────────────────────────────────
choco install kubernetes-cli

# ── Check version ─────────────────────────────────────────
kubectl version --client
# Client Version: v1.32.0
```

---

## 🗂️ kubeconfig — The Configuration File

### What is kubeconfig?
kubeconfig is a YAML file that tells kubectl:
- **WHERE** to connect (cluster API server address)
- **WHO** you are (credentials — certificates or tokens)
- **WHAT** is your current working context (cluster + user + namespace combo)

```
Default location: ~/.kube/config

You can override with:
  export KUBECONFIG=/path/to/custom/config
  kubectl --kubeconfig=/path/to/config get pods
```

### kubeconfig Structure

```yaml
# ~/.kube/config

apiVersion: v1
kind: Config

# ── CLUSTERS ──────────────────────────────────────────────
# Where to connect (server addresses + CA certs)
clusters:
  - name: production-cluster
    cluster:
      server: https://prod-apiserver:6443
      certificate-authority-data: <base64-ca-cert>

  - name: dev-cluster
    cluster:
      server: https://dev-apiserver:6443
      certificate-authority-data: <base64-ca-cert>

# ── USERS ─────────────────────────────────────────────────
# Who you are (credentials)
users:
  - name: admin-user
    user:
      client-certificate-data: <base64-cert>
      client-key-data: <base64-key>

  - name: dev-user
    user:
      token: <bearer-token>

# ── CONTEXTS ──────────────────────────────────────────────
# Named combinations of cluster + user + namespace
contexts:
  - name: prod-admin
    context:
      cluster: production-cluster
      user: admin-user
      namespace: default

  - name: dev-dev
    context:
      cluster: dev-cluster
      user: dev-user
      namespace: development

# ── CURRENT CONTEXT ───────────────────────────────────────
# Which context is active right now
current-context: prod-admin
```

### Context Management Commands

```bash
# View entire kubeconfig
kubectl config view

# View kubeconfig with secrets revealed
kubectl config view --raw

# ── CONTEXTS ──────────────────────────────────────────────
# List all contexts
kubectl config get-contexts

# Output:
# CURRENT   NAME         CLUSTER     AUTHINFO    NAMESPACE
# *         prod-admin   prod        admin-user  default
#           dev-dev      dev         dev-user    development

# Switch context (switch cluster you're talking to)
kubectl config use-context dev-dev

# Get current context
kubectl config current-context

# ── NAMESPACES ────────────────────────────────────────────
# Set default namespace for current context
kubectl config set-context --current --namespace=kube-system

# Now all commands use kube-system by default
kubectl get pods   # ← shows kube-system pods

# ── MERGING CONFIGS ───────────────────────────────────────
# Use multiple kubeconfig files
export KUBECONFIG=~/.kube/config:~/.kube/cluster2-config

# Merge into one file
kubectl config view --flatten > ~/.kube/merged-config
```

---

## 🎭 Imperative vs Declarative

```
IMPERATIVE — "Do this specific action right now"
  → You tell K8s HOW to do something step by step
  → Good for: quick tasks, CKA exam speed, one-time operations
  → Bad for: version control, repeatability

  Examples:
  kubectl run nginx --image=nginx
  kubectl create deployment my-app --image=my-app:v1
  kubectl scale deployment my-app --replicas=5
  kubectl expose deployment my-app --port=80
  kubectl delete pod my-pod

DECLARATIVE — "This is what I want, figure out how"
  → You describe DESIRED STATE in a YAML file
  → K8s figures out what changes are needed
  → Good for: production, GitOps, version control, teams
  → Bad for: quick ad-hoc tasks

  Examples:
  kubectl apply -f deployment.yaml      ← create or update
  kubectl apply -f ./manifests/         ← entire directory
  kubectl apply -f https://url/file.yaml ← from URL
  kubectl delete -f deployment.yaml     ← delete from file

CKA EXAM TIP:
  Use IMPERATIVE for speed, then --dry-run=client -o yaml
  to generate YAML you can customize!

  kubectl create deployment my-app \
    --image=nginx \
    --replicas=3 \
    --dry-run=client \
    -o yaml > deployment.yaml

  # Now edit deployment.yaml and apply it!
```

---

## 🗺️ Essential kubectl Commands — Full Reference

### 🔍 GET — View Resources

```bash
# Basic get
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get nodes

# All namespaces
kubectl get pods --all-namespaces
kubectl get pods -A              # shorthand

# Specific namespace
kubectl get pods -n kube-system

# Show more info (IPs, nodes, etc.)
kubectl get pods -o wide

# Watch live updates
kubectl get pods -w

# Get multiple resource types at once
kubectl get pods,services,deployments

# Get all resources in namespace
kubectl get all
kubectl get all -n my-namespace

# Filter by label
kubectl get pods -l app=nginx
kubectl get pods -l env=prod,tier=backend

# Get a specific resource
kubectl get pod my-pod
kubectl get pod my-pod -o yaml    # full YAML
kubectl get pod my-pod -o json    # full JSON
```

---

### 📋 DESCRIBE — Detailed Info + Events

```bash
# Describe a resource (most useful for debugging!)
kubectl describe pod my-pod
kubectl describe node node1
kubectl describe deployment my-app
kubectl describe service my-svc

# The Events section at the bottom is GOLD for debugging:
# Events:
#   Warning  BackOff    5m    kubelet  Back-off restarting failed container
#   Warning  Failed     5m    kubelet  Error: ImagePullBackOff
```

---

### ➕ CREATE / APPLY — Create Resources

```bash
# Imperative create
kubectl run nginx --image=nginx                        # creates pod
kubectl run nginx --image=nginx --port=80              # pod with port
kubectl run nginx --image=nginx --env="ENV=prod"       # with env var
kubectl run nginx --image=nginx --labels="app=web"     # with label

kubectl create deployment my-app --image=nginx
kubectl create deployment my-app --image=nginx --replicas=3

kubectl create namespace my-ns
kubectl create configmap my-config --from-literal=key=value
kubectl create secret generic my-secret --from-literal=pass=secret123

# Declarative apply
kubectl apply -f pod.yaml
kubectl apply -f ./k8s/               # directory
kubectl apply -f https://raw.githubusercontent.com/.../file.yaml

# Generate YAML without creating (CKA MUST KNOW!)
kubectl run nginx --image=nginx --dry-run=client -o yaml
kubectl create deployment my-app --image=nginx --dry-run=client -o yaml
kubectl create service clusterip my-svc --tcp=80:80 --dry-run=client -o yaml
```

---

### ✏️ EDIT / PATCH — Modify Resources

```bash
# Interactive edit (opens in vim/nano)
kubectl edit deployment my-app
kubectl edit pod my-pod
kubectl edit configmap my-config

# Patch (non-interactive, scriptable)
kubectl patch deployment my-app -p '{"spec":{"replicas":5}}'

# Set image (for rolling updates)
kubectl set image deployment/my-app nginx=nginx:1.21

# Scale
kubectl scale deployment my-app --replicas=5

# Label a resource
kubectl label pod my-pod env=production
kubectl label node node1 disk=ssd

# Annotate a resource
kubectl annotate pod my-pod description="web server"
```

---

### 🗑️ DELETE — Remove Resources

```bash
# Delete by name
kubectl delete pod my-pod
kubectl delete deployment my-app
kubectl delete service my-svc

# Delete from file
kubectl delete -f deployment.yaml

# Delete all pods in namespace
kubectl delete pods --all -n my-namespace

# Delete with label selector
kubectl delete pods -l app=nginx

# Force delete (stuck terminating pods)
kubectl delete pod my-pod --force --grace-period=0

# Delete namespace (deletes ALL resources inside!)
kubectl delete namespace my-ns
```

---

### 🔬 EXEC / LOGS — Debug Inside Containers

```bash
# Get logs
kubectl logs my-pod
kubectl logs my-pod -c container-name   # specific container in pod
kubectl logs my-pod --previous          # logs from previous crashed container
kubectl logs my-pod -f                  # follow/tail live logs
kubectl logs my-pod --tail=50           # last 50 lines
kubectl logs my-pod --since=1h          # logs from last 1 hour

# Execute commands inside container
kubectl exec my-pod -- ls /app
kubectl exec my-pod -- cat /etc/config
kubectl exec -it my-pod -- /bin/bash    # interactive shell
kubectl exec -it my-pod -c sidecar -- /bin/sh  # specific container

# Copy files to/from container
kubectl cp my-pod:/var/log/app.log ./app.log     # from pod
kubectl cp ./config.yaml my-pod:/etc/config.yaml  # to pod

# Port forward (access pod/service locally)
kubectl port-forward pod/my-pod 8080:80
kubectl port-forward svc/my-svc 8080:80
kubectl port-forward deployment/my-app 8080:80
```

---

### 🚦 Node Management Commands

```bash
# Cordon node (prevent new pods being scheduled)
kubectl cordon node1

# Uncordon node (allow scheduling again)
kubectl uncordon node1

# Drain node (evict all pods, then cordon)
kubectl drain node1 --ignore-daemonsets
kubectl drain node1 --ignore-daemonsets --delete-emptydir-data

# Top (resource usage) — requires metrics-server
kubectl top nodes
kubectl top pods
kubectl top pods -n kube-system
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory
```

---

### 📤 OUTPUT FORMATS — Very Important for CKA

```bash
# Default table output
kubectl get pods

# Wide output (more columns)
kubectl get pods -o wide

# YAML output (full manifest)
kubectl get pod my-pod -o yaml

# JSON output
kubectl get pod my-pod -o json

# JSONPath (extract specific fields)
kubectl get pod my-pod -o jsonpath='{.status.podIP}'
kubectl get nodes -o jsonpath='{.items[*].metadata.name}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName

# Sort by field
kubectl get pods --sort-by=.metadata.creationTimestamp
kubectl get pods --sort-by=.status.startTime

# Name only (for scripting)
kubectl get pods -o name
# pod/my-pod
# pod/another-pod
```

---

## ⚡ CKA Speed Tips — Aliases & Shortcuts

### Set Up These Aliases (Do this at START of CKA exam!)

```bash
# Paste this at the beginning of your CKA exam session:

# kubectl alias
alias k=kubectl

# Auto-complete
source <(kubectl completion bash)
complete -F __start_kubectl k

# Dry run shortcut
export do="--dry-run=client -o yaml"

# Force delete shortcut
export now="--force --grace-period=0"

# Namespace shortcut
alias kn='kubectl config set-context --current --namespace'

# Now you can do:
k get pods                          # instead of kubectl get pods
k run nginx --image=nginx $do       # generates YAML
k delete pod my-pod $now            # force delete
kn kube-system                      # switch namespace
```

### Common Shorthand Resource Names

```bash
# Full name → shorthand
pods          → po
deployments   → deploy
services      → svc
namespaces    → ns
nodes         → no
configmaps    → cm
secrets       → (no shorthand)
replicasets   → rs
persistentvolumeclaims → pvc
persistentvolumes      → pv
serviceaccounts        → sa
ingresses     → ing
networkpolicies → netpol

# Examples:
kubectl get po
kubectl get deploy
kubectl get svc
kubectl get ns
kubectl get pvc
kubectl get sa
```

---

## 🔑 Explain — Learn API Fields

```bash
# Learn what fields an object supports
kubectl explain pod
kubectl explain pod.spec
kubectl explain pod.spec.containers
kubectl explain pod.spec.containers.resources
kubectl explain deployment.spec.strategy

# Very useful when writing YAML — you don't need to memorize every field!
```

---

## 🌐 Useful Cluster-Level Commands

```bash
# Cluster info
kubectl cluster-info
kubectl cluster-info dump    # full debug dump

# API resources (what can I create?)
kubectl api-resources
kubectl api-resources --namespaced=true    # namespace-scoped
kubectl api-resources --namespaced=false   # cluster-scoped

# API versions
kubectl api-versions

# Check your permissions
kubectl auth can-i create pods
kubectl auth can-i delete deployments
kubectl auth can-i '*' '*'                 # can I do everything?
kubectl auth can-i create pods --as=dev-user  # as another user

# Events (great for debugging)
kubectl get events
kubectl get events --sort-by='.lastTimestamp'
kubectl get events -n my-namespace
kubectl get events --field-selector reason=BackOff
```

---

## 📝 Real-World Workflow Example

```bash
# Scenario: Deploy a web app, expose it, debug it

# Step 1: Create deployment
kubectl create deployment webapp --image=nginx:1.21 --replicas=3

# Step 2: Expose it as a service
kubectl expose deployment webapp --port=80 --target-port=80 --type=ClusterIP

# Step 3: Check everything
kubectl get pods,deployments,services

# Step 4: Test connectivity
kubectl run test-pod --image=busybox --rm -it -- wget -qO- http://webapp

# Step 5: Something is wrong — debug
kubectl get pods                          # check pod status
kubectl describe pod <pod-name>           # check events
kubectl logs <pod-name>                   # check logs
kubectl exec -it <pod-name> -- /bin/bash  # get inside

# Step 6: Update image
kubectl set image deployment/webapp nginx=nginx:1.22
kubectl rollout status deployment/webapp  # watch rollout
kubectl rollout history deployment/webapp # check versions

# Step 7: Rollback if needed
kubectl rollout undo deployment/webapp

# Step 8: Scale up
kubectl scale deployment webapp --replicas=5

# Step 9: Clean up
kubectl delete deployment webapp
kubectl delete service webapp
```

---

## ⚠️ Common Mistakes & Gotchas

| Mistake | Reality |
|---------|---------|
| `kubectl apply` vs `kubectl create` | `apply` creates OR updates (idempotent). `create` fails if resource exists |
| Forgetting `-n namespace` | By default, kubectl uses the `default` namespace. Always specify! |
| `kubectl get pod` (singular) | Works fine — K8s accepts singular and plural |
| Editing a running pod | Most pod fields are immutable — delete and recreate, or edit the Deployment |
| `kubectl delete` is instant | Pods go through graceful termination (default 30s). Use `--grace-period=0` to skip |
| `kubectl exec` vs `kubectl run` | `exec` goes into existing pod. `run` creates a NEW temporary pod |

---

## 🎓 CKA Exam Tips

```
✅ Set up alias at exam start:  alias k=kubectl
✅ Enable autocomplete:  source <(kubectl completion bash)
✅ Master --dry-run=client -o yaml  → generate YAML quickly
✅ Use kubectl explain  → look up field names without memorizing
✅ kubectl describe  → always check Events section for clues
✅ -o wide  → see pod IPs and which node they're on
✅ --all-namespaces (-A)  → when you don't know which namespace
✅ Time-saving: use imperative commands for simple resources
   then edit YAML for complex ones
✅ kubectl get events --sort-by='.lastTimestamp'
   → great first step when something is broken
✅ Read the question carefully — some tasks specify a namespace!
```

---

## ❓ Interview & Scenario Questions

### Q1: What is kubeconfig and what does it contain?
**Answer:**
kubeconfig is a YAML file (default: `~/.kube/config`) that kubectl uses to connect to clusters. It contains three sections: **clusters** (API server addresses and CA certificates), **users** (credentials — client certificates, tokens, or OIDC), and **contexts** (named combinations of cluster + user + optional namespace). The `current-context` field tells kubectl which context to use by default.

---

### Q2: What is the difference between `kubectl apply` and `kubectl create`?
**Answer:**
`kubectl create` is imperative — it creates a resource and fails with an error if the resource already exists. `kubectl apply` is declarative — it creates the resource if it doesn't exist, or updates it if it does (idempotent). `apply` also stores the last applied configuration as an annotation on the object, enabling it to detect and remove fields that were deleted from the YAML. In production and CI/CD, always use `apply`.

---

### Q3: Scenario — You need to quickly create a pod YAML without typing it from scratch. How?
**Answer:**
Use `--dry-run=client -o yaml` to generate YAML without actually creating the resource:
```bash
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
# Edit pod.yaml to add any custom fields
kubectl apply -f pod.yaml
```
This is a core CKA exam technique — never type YAML from scratch when you can generate it!

---

### Q4: How do you switch between multiple clusters in kubectl?
**Answer:**
Using contexts in kubeconfig:
```bash
# List all contexts
kubectl config get-contexts

# Switch to a different cluster context
kubectl config use-context production-cluster

# Or for a single command only
kubectl --context=staging-cluster get pods

# View current context
kubectl config current-context
```

---

### Q5: What does `kubectl explain` do and why is it useful in the CKA exam?
**Answer:**
`kubectl explain` shows the API schema for any Kubernetes object and its fields — like built-in documentation. It's extremely useful in the CKA exam because you're allowed to use it and it saves time looking up field names.
```bash
kubectl explain pod.spec.containers.resources.limits
# Shows: type, description, required or optional
```

---

## 📚 Quick Summary / Cheatsheet

```
┌──────────────────────────────────────────────────────────────┐
│               TOPIC 1.5 — kubectl BASICS                     │
├──────────────────────────────────────────────────────────────┤
│  CONFIG                                                      │
│  kubeconfig:      ~/.kube/config                             │
│  View contexts:   kubectl config get-contexts                │
│  Switch context:  kubectl config use-context <name>          │
│  Set namespace:   kubectl config set-context --current \     │
│                    --namespace=<ns>                          │
├──────────────────────────────────────────────────────────────┤
│  MUST-KNOW COMMANDS                                          │
│  kubectl get <resource> [-n ns] [-o wide/yaml] [-A]          │
│  kubectl describe <resource> <name>                          │
│  kubectl apply -f <file>                                     │
│  kubectl delete -f <file>                                    │
│  kubectl logs <pod> [--previous] [-f]                        │
│  kubectl exec -it <pod> -- /bin/bash                         │
│  kubectl run <name> --image=<img> --dry-run=client -o yaml   │
│  kubectl explain <resource>.<field>                          │
├──────────────────────────────────────────────────────────────┤
│  CKA EXAM SETUP (do this first!)                             │
│  alias k=kubectl                                             │
│  source <(kubectl completion bash)                           │
│  complete -F __start_kubectl k                               │
│  export do="--dry-run=client -o yaml"                        │
│  export now="--force --grace-period=0"                       │
├──────────────────────────────────────────────────────────────┤
│  SHORTHAND RESOURCES                                         │
│  po=pods  deploy=deployments  svc=services  ns=namespaces    │
│  cm=configmaps  rs=replicasets  pvc=persistentvolumeclaims   │
└──────────────────────────────────────────────────────────────┘
```

---

> **Next Topic →** `06-kubernetes-objects-overview.md`
> *Pods, ReplicaSets, Deployments, Services — the big picture of K8s objects*
