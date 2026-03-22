# ☸️ Kubernetes — From Basics to CKA Certification 2026
### Complete Learning Syllabus | Full Stack · DevOps · Architect

---

> 📌 **How This Course Works**
> Each topic below gets its **own dedicated `.md` file** with:
> - 📖 In-depth explanation
> - 💡 Real-world examples
> - 🗺️ Diagrams & visual flows (ASCII + Mermaid)
> - 🔁 Scenario-based questions
> - 🎯 CKA exam tips & traps
> - 🛠️ Hands-on labs / commands
> - ❓ Interview Q&A section

---

## 📚 COURSE MODULES OVERVIEW

| # | Module | Topics Count | CKA Weight |
|---|--------|-------------|------------|
| 1 | Core Concepts & Architecture | 8 | ~10% |
| 2 | Workloads & Scheduling | 10 | ~15% |
| 3 | Services & Networking | 9 | ~20% |
| 4 | Storage | 7 | ~10% |
| 5 | Configuration & Secrets | 8 | ~10% |
| 6 | Security | 10 | ~15% |
| 7 | Cluster Maintenance | 7 | ~10% |
| 8 | Observability & Troubleshooting | 8 | ~30% |
| 9 | Helm & Package Management | 5 | Bonus |
| 10 | Advanced & Architect Level | 10 | Bonus |

> **Total: 82 topic files** — each one structured with diagrams, examples, scenarios & interview Q&A

---

## 🟢 MODULE 1 — Core Concepts & Kubernetes Architecture

> **Goal:** Understand what Kubernetes is, why it exists, and how every component fits together.

| # | Topic File | Description |
|---|-----------|-------------|
| 1.1 | `01-what-is-kubernetes.md` | What is Kubernetes? History, why containers, why orchestration |
| 1.2 | `02-kubernetes-architecture.md` | Control Plane vs Worker Nodes, full architecture diagram |
| 1.3 | `03-control-plane-components.md` | API Server, etcd, Scheduler, Controller Manager, Cloud Controller |
| 1.4 | `04-worker-node-components.md` | kubelet, kube-proxy, Container Runtime (containerd / CRI-O) |
| 1.5 | `05-kubectl-basics.md` | kubectl install, config, context, basic commands cheatsheet |
| 1.6 | `06-kubernetes-objects-overview.md` | Pods, ReplicaSets, Deployments, Services — the big picture |
| 1.7 | `07-namespaces.md` | Namespace concept, defaults, resource isolation, cross-ns communication |
| 1.8 | `08-etcd-deep-dive.md` | What etcd stores, backup & restore **(CKA Critical)** |

---

## 🟡 MODULE 2 — Workloads & Scheduling

> **Goal:** Master how applications are deployed, scaled, and scheduled across cluster nodes.

| # | Topic File | Description |
|---|-----------|-------------|
| 2.1 | `09-pods-deep-dive.md` | Pod lifecycle, multi-container pods, init containers, static pods |
| 2.2 | `10-replicasets.md` | ReplicaSet purpose, selectors, scaling, difference from RC |
| 2.3 | `11-deployments.md` | Rolling updates, rollback, strategies, revision history |
| 2.4 | `12-daemonsets.md` | DaemonSet use cases — logging agents, monitoring, networking |
| 2.5 | `13-statefulsets.md` | Ordered pods, stable network IDs, persistent storage per pod |
| 2.6 | `14-jobs-and-cronjobs.md` | Batch processing, one-off jobs, scheduled cron jobs |
| 2.7 | `15-resource-requests-limits.md` | CPU/Memory requests vs limits, QoS classes, OOMKilled |
| 2.8 | `16-node-selectors-affinity.md` | nodeSelector, nodeAffinity, podAffinity, podAntiAffinity |
| 2.9 | `17-taints-and-tolerations.md` | Tainting nodes, toleration rules, NoSchedule / NoExecute |
| 2.10 | `18-scheduler-deep-dive.md` | How scheduler works, manual scheduling, custom schedulers, priority classes |

---

## 🔵 MODULE 3 — Services & Networking

> **Goal:** Understand how pods communicate inside and outside the cluster.

| # | Topic File | Description |
|---|-----------|-------------|
| 3.1 | `19-kubernetes-networking-model.md` | Flat network model, CNI explained, pod-to-pod communication |
| 3.2 | `20-services-clusterip.md` | ClusterIP service, kube-proxy, iptables/IPVS, DNS resolution |
| 3.3 | `21-services-nodeport.md` | NodePort mechanics, port ranges, use cases |
| 3.4 | `22-services-loadbalancer.md` | LoadBalancer service, cloud integration, external IPs |
| 3.5 | `23-services-headless-externalname.md` | Headless services for StatefulSets, ExternalName for external DNS |
| 3.6 | `24-ingress-and-ingress-controllers.md` | Ingress resources, NGINX/Traefik controllers, TLS, path routing |
| 3.7 | `25-network-policies.md` | Ingress/Egress rules, pod selectors, namespace isolation **(CKA Critical)** |
| 3.8 | `26-dns-in-kubernetes.md` | CoreDNS, FQDN format, service DNS, pod DNS, custom DNS |
| 3.9 | `27-cni-plugins.md` | Flannel, Calico, Weave, Cilium — comparison and config |

---

## 🟠 MODULE 4 — Storage

> **Goal:** Learn how to persist data and manage storage across pods and nodes.

| # | Topic File | Description |
|---|-----------|-------------|
| 4.1 | `28-volumes-overview.md` | emptyDir, hostPath, configMap volumes, secret volumes |
| 4.2 | `29-persistent-volumes.md` | PV concept, access modes, reclaim policies, static provisioning |
| 4.3 | `30-persistent-volume-claims.md` | PVC concept, binding lifecycle, storage classes |
| 4.4 | `31-storage-classes.md` | Dynamic provisioning, provisioner, parameters, default StorageClass |
| 4.5 | `32-statefulset-storage.md` | volumeClaimTemplates, stable storage per pod |
| 4.6 | `33-csi-drivers.md` | Container Storage Interface, popular drivers (EBS, NFS, Ceph) |
| 4.7 | `34-volume-snapshots.md` | VolumeSnapshot, VolumeSnapshotClass, restore from snapshot |

---

## 🟣 MODULE 5 — Configuration & Secrets

> **Goal:** Decouple configuration from application code, manage sensitive data securely.

| # | Topic File | Description |
|---|-----------|-------------|
| 5.1 | `35-environment-variables.md` | Env vars in pods, valueFrom, dynamic injection |
| 5.2 | `36-configmaps.md` | Creating ConfigMaps, mounting as files vs env vars, hot reload |
| 5.3 | `37-secrets.md` | Secret types, base64 encoding, mounting secrets, best practices |
| 5.4 | `38-resource-quotas.md` | Namespace-level quotas, compute/object quotas |
| 5.5 | `39-limit-ranges.md` | Default limits, min/max per container/pod/PVC |
| 5.6 | `40-labels-annotations-selectors.md` | Label strategies, annotations use cases, selector types |
| 5.7 | `41-downward-api.md` | Exposing pod metadata to containers (name, namespace, labels) |
| 5.8 | `42-pod-disruption-budgets.md` | PDB concept, minAvailable, maxUnavailable, drain protection |

---

## 🔴 MODULE 6 — Security

> **Goal:** Secure the cluster, workloads, and communication. *(High CKA weight!)*

| # | Topic File | Description |
|---|-----------|-------------|
| 6.1 | `43-kubernetes-security-overview.md` | 4C security model: Cloud, Cluster, Container, Code |
| 6.2 | `44-authentication.md` | Users vs ServiceAccounts, kubeconfig, certificates, tokens |
| 6.3 | `45-authorization-rbac.md` | Roles, ClusterRoles, RoleBindings, ClusterRoleBindings **(CKA Critical)** |
| 6.4 | `46-service-accounts.md` | Default SA, custom SA, automounting tokens, OIDC |
| 6.5 | `47-security-contexts.md` | runAsUser, runAsNonRoot, fsGroup, privileged, capabilities |
| 6.6 | `48-pod-security-admission.md` | PSA Privileged/Baseline/Restricted, replacing PodSecurityPolicy |
| 6.7 | `49-network-security-tls.md` | TLS certificates in K8s, cert rotation, kube-apiserver certs |
| 6.8 | `50-secrets-encryption-at-rest.md` | EncryptionConfiguration, encrypting secrets in etcd |
| 6.9 | `51-image-security.md` | Private registries, imagePullSecrets, image scanning |
| 6.10 | `52-admission-controllers.md` | MutatingWebhook, ValidatingWebhook, OPA/Gatekeeper |

---

## ⚫ MODULE 7 — Cluster Maintenance

> **Goal:** Manage, upgrade, and maintain a production Kubernetes cluster.

| # | Topic File | Description |
|---|-----------|-------------|
| 7.1 | `53-cluster-upgrade.md` | kubeadm upgrade, control plane → node upgrade order **(CKA Critical)** |
| 7.2 | `54-node-drain-cordon-uncordon.md` | Safely evicting pods, maintenance windows |
| 7.3 | `55-etcd-backup-restore.md` | etcdctl snapshot save/restore, disaster recovery **(CKA Critical)** |
| 7.4 | `56-os-upgrades.md` | Node OS upgrades without downtime |
| 7.5 | `57-kubeadm-cluster-setup.md` | Bootstrap cluster from scratch with kubeadm |
| 7.6 | `58-high-availability-cluster.md` | Multi-master setup, stacked vs external etcd topology |
| 7.7 | `59-certificate-management.md` | kubeadm certs, renewal, manual cert creation with openssl |

---

## 🟤 MODULE 8 — Observability & Troubleshooting ⚠️

> **Goal:** Debug cluster issues, monitor health, solve CKA troubleshooting scenarios.
> *(This module carries **30% of the CKA exam** — practice heavily!)*

| # | Topic File | Description |
|---|-----------|-------------|
| 8.1 | `60-logging-architecture.md` | Container logs, node logs, cluster-level logging patterns |
| 8.2 | `61-kubectl-logs-exec-describe.md` | Deep dive: logs, exec, describe, events, debug |
| 8.3 | `62-metrics-server.md` | Installing metrics-server, kubectl top node/pod |
| 8.4 | `63-probes-healthchecks.md` | Liveness, Readiness, Startup probes — types and gotchas |
| 8.5 | `64-troubleshoot-application-failure.md` | CrashLoopBackOff, ImagePullBackOff, Pending — step-by-step |
| 8.6 | `65-troubleshoot-control-plane.md` | Broken API server, scheduler, controller-manager diagnosis |
| 8.7 | `66-troubleshoot-worker-nodes.md` | NotReady nodes, kubelet failures, certificate issues |
| 8.8 | `67-troubleshoot-networking.md` | Service not reachable, DNS failures, CNI issues, kube-proxy |

---

## 💎 MODULE 9 — Helm & Package Management

> **Goal:** Use Helm to manage complex application deployments *(Real-world critical)*

| # | Topic File | Description |
|---|-----------|-------------|
| 9.1 | `68-helm-introduction.md` | What is Helm, charts, releases, repositories |
| 9.2 | `69-helm-commands.md` | install, upgrade, rollback, uninstall, list, status |
| 9.3 | `70-helm-chart-structure.md` | Chart.yaml, values.yaml, templates, helpers |
| 9.4 | `71-helm-create-custom-chart.md` | Building a chart from scratch, Go templates |
| 9.5 | `72-helm-advanced.md` | Hooks, conditionals, dependencies, OCI registries |

---

## 🚀 MODULE 10 — Advanced & Architect Level

> **Goal:** Go beyond CKA — production, GitOps, service mesh, multi-cluster architecture.

| # | Topic File | Description |
|---|-----------|-------------|
| 10.1 | `73-kubernetes-operators.md` | CRDs, custom controllers, Operator pattern, Operator SDK |
| 10.2 | `74-autoscaling-hpa-vpa-keda.md` | HPA, VPA, KEDA — event-driven autoscaling |
| 10.3 | `75-cluster-autoscaler.md` | Node auto-provisioning, scale-up/down logic |
| 10.4 | `76-service-mesh-istio.md` | Sidecar proxy, traffic management, mTLS, observability |
| 10.5 | `77-gitops-argocd-fluxcd.md` | GitOps principles, ArgoCD setup, sync policies |
| 10.6 | `78-multitenancy-patterns.md` | Namespace-per-team, vcluster, Hierarchical Namespaces |
| 10.7 | `79-multi-cluster-federation.md` | Cluster federation, Submariner, Cluster API |
| 10.8 | `80-kubernetes-on-cloud.md` | EKS, GKE, AKS — managed vs self-managed tradeoffs |
| 10.9 | `81-ci-cd-with-kubernetes.md` | Jenkins, GitHub Actions, Tekton pipelines on K8s |
| 10.10 | `82-kubernetes-best-practices.md` | Production checklist: security, HA, cost, performance |

---

## 🎓 CKA EXAM REFERENCE (2026)

### Exam Format
| Detail | Info |
|--------|------|
| Duration | 2 Hours |
| Format | Performance-based (hands-on terminal) |
| Passing Score | 66% |
| Questions | ~17 performance tasks |
| Environment | Remote desktop + kubectl access |
| Open Book | ✅ Yes — kubernetes.io/docs allowed |

### Domain Weightage
```
+---------------------------------------------+
|  Troubleshooting              ████████  30%  |
|  Services & Networking        ██████    20%  |
|  Workloads & Scheduling       █████     15%  |
|  Security (RBAC + certs)      █████     15%  |
|  Storage                      ███       10%  |
|  Cluster Architecture         ███       10%  |
+---------------------------------------------+
```

### 🔥 Must-Know Commands for CKA
```bash
# ── Cluster Info ──────────────────────────────────────────
kubectl cluster-info
kubectl get nodes -o wide
kubectl get all --all-namespaces

# ── Quick Pod Debugging ────────────────────────────────────
kubectl describe pod <pod-name>
kubectl logs <pod-name> --previous
kubectl exec -it <pod-name> -- /bin/sh
kubectl get events --sort-by='.lastTimestamp'

# ── etcd Backup (CKA Critical) ─────────────────────────────
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# ── etcd Restore ───────────────────────────────────────────
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd.db \
  --data-dir=/var/lib/etcd-restored

# ── Cluster Upgrade ────────────────────────────────────────
kubeadm upgrade plan
kubeadm upgrade apply v1.32.0
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>

# ── RBAC Quick Create ─────────────────────────────────────
kubectl create role pod-reader --verb=get,list,watch --resource=pods
kubectl create rolebinding dev-binding --role=pod-reader --user=dev
kubectl auth can-i list pods --as=dev

# ── NetworkPolicy ─────────────────────────────────────────
kubectl get networkpolicies --all-namespaces

# ── Certificate Check ─────────────────────────────────────
kubeadm certs check-expiration
```

---

## 📁 Recommended GitHub Folder Structure

```
kubernetes-cka-2026/
│
├── README.md                               ← This syllabus
│
├── module-01-core-concepts/
│   ├── 01-what-is-kubernetes.md
│   ├── 02-kubernetes-architecture.md
│   ├── 03-control-plane-components.md
│   ├── 04-worker-node-components.md
│   ├── 05-kubectl-basics.md
│   ├── 06-kubernetes-objects-overview.md
│   ├── 07-namespaces.md
│   └── 08-etcd-deep-dive.md
│
├── module-02-workloads-scheduling/
├── module-03-services-networking/
├── module-04-storage/
├── module-05-configuration/
├── module-06-security/
├── module-07-cluster-maintenance/
├── module-08-troubleshooting/
├── module-09-helm/
├── module-10-advanced/
│
├── labs/
│   ├── lab-01-setup-local-cluster.md
│   ├── lab-02-deploy-app-end-to-end.md
│   └── lab-cka-mock-scenarios.md
│
└── cheatsheets/
    ├── kubectl-cheatsheet.md
    └── cka-exam-day-tips.md
```

---

## 🗓️ Suggested 10-Week Learning Timeline

| Week | Modules | Focus Area |
|------|---------|------------|
| Week 1–2 | Module 1 + 2 | Architecture, Pods, Deployments, Scheduling |
| Week 3–4 | Module 3 + 4 | Networking, Services, Ingress, Storage |
| Week 5–6 | Module 5 + 6 | Config, Secrets, RBAC, Security |
| Week 7 | Module 7 | Cluster Upgrade, etcd Backup/Restore |
| Week 8 | Module 8 | Troubleshooting (practice daily!) |
| Week 9 | Module 9 + 10 | Helm, GitOps, Advanced Architecture |
| Week 10 | Full Revision | Mock exams → killer.sh → Schedule exam |

---

## 🛠️ Tools & Setup You'll Need

| Tool | Purpose |
|------|---------|
| `minikube` or `kind` | Local cluster for daily practice |
| `kubectl` | Primary Kubernetes CLI |
| `kubeadm` | Cluster bootstrap & upgrades |
| `etcdctl` | etcd backup/restore operations |
| `helm` | Package manager for K8s |
| `k9s` | Terminal UI — great for exploration |
| `VSCode + YAML plugin` | Writing & validating manifests |
| `killer.sh` | Official CKA exam simulator |
| `Play with K8s` | Free browser-based cluster |

---

## 📌 Every Topic File Template

```
## 📌 Topic N.N — [Topic Title]

### 🎯 What You'll Learn
### 🧠 Core Concept Explained (with analogy)
### 🗺️ Architecture / Flow Diagram (ASCII or Mermaid)
### 💻 Hands-on Commands & YAML Examples
### 🔁 Real-World Scenario
### ⚠️ Common Mistakes & Gotchas
### 🎓 CKA Exam Tips
### ❓ Interview & Scenario Questions (with answers)
### 📚 Quick Summary / Cheatsheet
```

---

> ✅ **Total: 82 topic files across 10 modules**
> 🚀 **Next Step:** Say `"start topic 1.1"` → we generate `01-what-is-kubernetes.md`
> 📁 **Push each file to GitHub** as we go — your personal CKA knowledge base!
