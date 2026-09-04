# Kubernetes & GitOps (ArgoCD) — Revision Notes
**For: DevOps Engineer (~4 yrs experience) | Interview Prep**

Each topic: **Revision Note** (concept) → **Simple Example** (YAML/command) → **Interview Q&A** (how to answer at 4 YOE level).

---

# PART 1: KUBERNETES

## 1. Architecture

**Revision Note**
Control plane = API Server, etcd, Scheduler, Controller Manager. Worker node = kubelet, kube-proxy, container runtime (containerd). API Server is the single entry point — everything talks to it, including kubectl. etcd uses Raft consensus (odd node count); scheduler assigns via predicates+priorities; controller-manager runs reconciliation loops (desired vs actual state).

**Simple Example**
```bash
kubectl get componentstatuses
kubectl get nodes -o wide
```

**Interview Q&A**
> **Q: Walk me through what happens when you run `kubectl apply -f deployment.yaml`.**
> A: kubectl sends the manifest to the API Server, which runs it through admission control (mutating → validating), then persists to etcd. The Deployment controller notices the new/changed object and creates a ReplicaSet, which creates Pods. The Scheduler assigns each Pod to a node based on resource requests, affinity rules, and taints. The kubelet on that node pulls the image via the container runtime (CRI) and starts it, reporting status back to the API Server.

---

## 2. Pods

**Revision Note**
Smallest deployable unit. Usually 1 container, but can have sidecars/init containers sharing network and optionally storage namespace. Pods are ephemeral and immutable by design — never manage them directly in production, use a controller (Deployment/StatefulSet/DaemonSet). Multi-container patterns: sidecar (helper alongside main), init container (run-to-completion before main starts), ambassador/adapter (less common, know the names).

**Simple Example**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
```

**Interview Q&A**
> **Q: Why shouldn't you deploy bare Pods in production?**
> A: A bare Pod has no self-healing — if the node dies or the Pod crashes, nothing recreates it. Controllers like Deployments watch the desired state and recreate Pods automatically, plus give you rolling updates and rollback.

---

## 3. Deployments & ReplicaSets

**Revision Note**
Deployment manages ReplicaSets, which manage Pods — know this chain, it's often asked directly. Gives declarative rolling updates, rollback (`kubectl rollout undo`), and scaling. Strategies: `RollingUpdate` (default, tuned via maxSurge/maxUnavailable) vs `Recreate` (kills all then creates all — for schema-incompatible versions that can't coexist).

**Simple Example**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: myrepo/web-app:1.2.0
        resources:
          requests: { cpu: "100m", memory: "128Mi" }
          limits: { cpu: "500m", memory: "256Mi" }
```

**Interview Q&A**
> **Q: How do you roll back a bad deployment in production?**
> A: `kubectl rollout undo deployment/web-app` — reverts to the previous ReplicaSet. I check history first with `kubectl rollout history deployment/web-app`, and use `--to-revision=<n>` for a specific version. In GitOps setups, I'd instead revert the Git commit and let ArgoCD sync the rollback, so Git stays the single source of truth.
>
> **Q: A rollout reports "successful" but users are seeing errors — why?**
> A: Rollout status only tracks pod creation/Ready state, not real app health beyond the probe. Without a meaningful readiness probe, K8s marks the pod Ready as soon as the container starts, even if it's returning errors. Fix: a readiness probe hitting a real health endpoint, not just a TCP check.

---

## 4. Services & Networking

**Revision Note**
- **ClusterIP** — internal only (default)
- **NodePort** — exposes on each node's IP at a static port; rarely used in prod (port range limits, no health-aware LB routing)
- **LoadBalancer** — provisions a cloud LB (e.g., AWS ALB/NLB)
- Service routes to Pods via label selectors, load-balanced via kube-proxy — **iptables** (default, O(n) rule matching) vs **IPVS** (hash table, O(1), better at scale).
- **CNI**: Calico (NetworkPolicy enforcement, BGP) · Cilium (eBPF, better observability) · AWS VPC CNI (native VPC IPs; ENI limits cause IP exhaustion on small EKS instance types).
- **CoreDNS**: `<svc>.<namespace>.svc.cluster.local`; `ndots:5` causes real resolution overhead — a known perf issue.

**Simple Example**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-svc
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
  type: ClusterIP
```

**Interview Q&A**
> **Q: A Pod can't reach a Service — how do you debug it?**
> A: `kubectl get endpoints web-app-svc` first — if empty, the Service's label selector doesn't match any Pod labels (the #1 cause). Then verify the Pod is Ready (readiness probe passing, since only Ready pods become endpoints). I'd exec into a debug pod, curl the service DNS name, check `kubectl describe svc`, and confirm no NetworkPolicy is blocking traffic.

---

## 5. Ingress & Ingress Controller

**Revision Note**
Ingress routes external HTTP(S) traffic to Services based on host/path rules — but it's just a spec/API object and does **nothing on its own**. It needs an **Ingress Controller** (NGINX, AWS Load Balancer Controller/ALB, Traefik) actually watching Ingress objects and provisioning a real load balancer. On EKS, AWS LB Controller creates an ALB (host/path routing, TLS termination, WAF integration). TLS + cert-manager (Let's Encrypt) is the standard production pattern for automated certs.

**Simple Example**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-svc
            port:
              number: 80
```

**Interview Q&A**
> **Q: What's the difference between a Service of type LoadBalancer and an Ingress?**
> A: A LoadBalancer Service provisions one cloud load balancer per Service — expensive, no path-based routing. Ingress lets one load balancer/controller route to many Services based on host/path rules — cheaper, more flexible for HTTP traffic. For non-HTTP or a single dedicated LB need, LoadBalancer type still makes sense.
>
> **Q: Your ALB Ingress isn't provisioning — what do you check?**
> A: Confirm AWS Load Balancer Controller is installed and running, check its pod logs for reconciliation errors, verify the Ingress has correct annotations/IngressClass, check IAM permissions on the controller's IRSA role, and check subnet tagging (ALB needs properly tagged public/private subnets).

---

## 6. NetworkPolicy

**Revision Note**
Default is allow-all between pods; a policy is additive/whitelist-only once any policy selects a pod (no policy applied = fully open). **Vanilla AWS VPC CNI does not enforce NetworkPolicy** without an add-on (Calico or Cilium) — a real EKS gotcha worth stating plainly. Selectors work on labels (podSelector, namespaceSelector); ingress and egress rules are controlled independently — a common miss is restricting only ingress and leaving egress wide open.

**Simple Example**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
spec:
  podSelector: {}
  ingress:
  - from:
    - podSelector: {}
```

**Interview Q&A**
> **Q: You applied a NetworkPolicy to restrict ingress, but the pod still receives unwanted traffic from another namespace — why?**
> A: Either the CNI doesn't enforce NetworkPolicy at all (e.g., plain AWS VPC CNI without Calico/Cilium), or the `podSelector`/`namespaceSelector` doesn't actually match the intended source, or egress on the source pod's side is separately unrestricted.

---

## 7. ConfigMaps & Secrets

**Revision Note**
ConfigMaps hold non-sensitive config; injected via env vars or mounted files (mounted files auto-update on change if the app watches; env var injection does **not** auto-update). Secrets hold sensitive data but are only **base64-encoded, not encrypted by default** — real security comes from etcd encryption at rest (KMS envelope on EKS) plus RBAC restricting `get`/`list`. In GitOps, plain Secrets shouldn't go into Git — use Sealed Secrets or External Secrets Operator instead.

**Simple Example**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "info"
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQ=
```

<img width="298" height="554" alt="image" src="https://github.com/user-attachments/assets/db2ff65b-9410-4b1c-b545-df2aed2640fc" />   ![Uploading image.png…]()


| Component               | Purpose                                           |
| ----------------------- | ------------------------------------------------- |
| **AWS Secrets Manager** | Stores actual secret; source of truth             |
| **IAM Role**            | Allows access to required AWS secrets             |
| **IRSA**                | Maps K8s ServiceAccount → IAM Role                |
| **ServiceAccount**      | Identity used by ESO Pod                          |
| **ESO**                 | Fetches AWS secret and creates/updates K8s Secret |
| **ClusterSecretStore**  | Defines AWS provider/authentication               |
| **ExternalSecret**      | Defines which AWS secret to retrieve              |
| **Kubernetes Secret**   | ESO-created secret consumed by application        |
| **ArgoCD**              | Deploys manifests from Git                        |

Interview answer

"We store secrets in AWS Secrets Manager and never commit actual values to Git. ESO runs in EKS using a dedicated ServiceAccount. Through IRSA, that ServiceAccount is associated with an IAM role having least-privilege access to Secrets Manager. ClusterSecretStore defines the AWS provider, and ExternalSecret specifies which secret to retrieve. ESO fetches the secret and synchronizes it into a Kubernetes Secret, which the application Pod consumes using secretKeyRef. ArgoCD only deploys the configuration; it never handles the actual secret value."



**Interview Q&A**
> **Q: How do you manage secrets in a GitOps workflow without exposing them in Git?**
> A: I've used Sealed Secrets — encrypt the Secret with the cluster's public key via `kubeseal`, and only the encrypted SealedSecret goes into Git; the in-cluster controller decrypts it back into a normal Secret. Alternatively, External Secrets Operator pulls secrets at runtime from AWS Secrets Manager/Parameter Store, so nothing sensitive ever touches Git — that's what I use on my current project.

---

## 8. Volumes & Persistent Storage

**Revision Note**
PersistentVolume (PV) = actual storage resource; PersistentVolumeClaim (PVC) = a Pod's request for storage; StorageClass enables dynamic provisioning (e.g., AWS EBS via CSI driver). `reclaimPolicy`: Retain (prod DBs, prevents data loss on PVC deletion) vs Delete (ephemeral/dev). Access modes: RWO (EBS, single node only) vs RWX (needs EFS for shared access across nodes).

**Simple Example**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests:
      storage: 5Gi
  storageClassName: gp3
```

**Interview Q&A**
> **Q: A shared PVC needs to be read by pods on different nodes simultaneously — what's the problem and fix?**
> A: EBS volumes are RWO (one node at a time) — pods on different nodes can't share it. Fix: use EFS (RWX-capable) via the EFS CSI driver, or redesign to avoid shared file access (e.g., object storage).

---

## 9. StatefulSets

**Revision Note**
Used when Pods need stable network identity and per-pod storage that persists across restarts (databases, Kafka). Stable pod names (`app-0`, `app-1`...) via a **headless Service** (`clusterIP: None`), giving each pod a predictable DNS name (`app-0.svc.namespace.svc.cluster.local`). Ordered, graceful deploy/scale/delete by default (pod-0 Ready before pod-1 starts) — relax via `podManagementPolicy: Parallel`. `volumeClaimTemplates` gives each pod its **own** PVC that persists across rescheduling — the key differentiator from a Deployment sharing one PVC.

**Simple Example**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db-headless
  replicas: 3
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests: { storage: 10Gi }
```

**Interview Q&A**
> **Q: What's the difference between a Deployment and StatefulSet, storage-wise?**
> A: Deployment Pods are interchangeable and typically don't need persistent identity. StatefulSet Pods get a stable network identity and each gets its own PVC that persists across restarts — needed for databases like PostgreSQL or Kafka where each replica owns its own data.
>
> **Q: A StatefulSet pod (`db-1`) is deleted — what happens to its data?**
> A: The pod is recreated with the same identity (`db-1`) and reattaches to its existing PVC — data persists. Only if the PVC itself is explicitly deleted (not automatic on pod deletion) is data lost, subject to `reclaimPolicy`.

---

## 10. Namespaces & RBAC

**Revision Note**
Namespaces logically isolate resources (dev/staging/prod) — but don't isolate everything (e.g., nodes aren't namespaced). RBAC: Role/RoleBinding (namespace-scoped) vs ClusterRole/ClusterRoleBinding (cluster-wide). A common trick combo: **ClusterRole bound via a RoleBinding** — a reusable, cluster-defined role granted only within one namespace, not cluster-wide. On EKS, **IRSA** (IAM Roles for Service Accounts) lets a pod assume an IAM role via an OIDC-linked ServiceAccount instead of inheriting the node's broad instance role — least-privilege at the pod level.

**Simple Example**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

**Interview Q&A**
> **Q: How do you restrict a CI/CD service account to only deploy in one namespace?**
> A: Create a Role scoped to that namespace with only the verbs it needs, then bind it via a RoleBinding — never a ClusterRoleBinding, which would give cluster-wide access.
>
> **Q: A pod needs to call the AWS S3 API — what's the production-correct way to grant access on EKS, and why not attach the policy to the node role?**
> A: Use IRSA — create an IAM role with the S3 policy, trust it to a ServiceAccount via OIDC, and have the pod use that SA. Attaching to the node role gives every pod on that node S3 access, violating least privilege — IRSA scopes it per-workload.

---

## 11. Pod Security Admission

**Revision Note**
Replaces the deprecated/removed PodSecurityPolicy (PSP); enforced via namespace labels, not a separate policy object. Three levels: **Privileged** (unrestricted), **Baseline** (blocks known privilege escalations), **Restricted** (no root, no privilege escalation, dropped capabilities, seccomp required). Three modes per level — `enforce` (blocks), `audit` (logs), `warn` (client-side warning) — combine to test a stricter policy before enforcing.

**Simple Example**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
```

**Interview Q&A**
> **Q: How do you enforce that no pod in a namespace runs as root, and how do you roll it out safely?**
> A: Label the namespace with Pod Security Admission `enforce: restricted` (or at minimum `baseline` + explicit `runAsNonRoot: true`). To roll out safely, apply `warn`/`audit` mode first to see what would break without blocking anything, fix workloads, then switch to `enforce`.

---

## 12. Probes & Self-Healing

**Revision Note**
- **livenessProbe** — restarts the container if it fails (app is stuck/deadlocked)
- **readinessProbe** — removes the pod from Service endpoints if it fails (temporarily can't serve traffic; pod keeps running)
- **startupProbe** — protects slow-starting apps from being killed by liveness before they're ready; once it succeeds, liveness/readiness take over

**Simple Example**
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 15
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5
```

**Interview Q&A**
> **Q: A Pod keeps restarting (CrashLoopBackOff) but the app "looks fine" in logs — what's happening?**
> A: Usually the liveness probe is misconfigured — too short an `initialDelaySeconds` for a slow-starting app, so Kubernetes kills it before it's ready, and it loops. Check `kubectl describe pod` for probe failure events; fix by increasing delay/timeout or adding a `startupProbe`.

---

## 13. Taints, Tolerations & Affinity

**Revision Note**
**Taints** live on nodes (repel pods); **tolerations** live on pods (permit scheduling despite a taint — they don't attract, only allow). Effects: `NoSchedule`, `PreferNoSchedule`, `NoExecute` (also evicts already-running pods). **nodeSelector** is the simplest constraint — exact label match, no OR logic, no "prefer" option. **nodeAffinity/podAffinity/podAntiAffinity** are more expressive: `requiredDuringScheduling` (hard) vs `preferredDuringScheduling` (soft), with operators (In, NotIn, Exists). `topologySpreadConstraints` handles AZ-level HA — mention explicitly for EKS multi-AZ setups.

**Simple Example**
```yaml
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "gpu"
  effect: "NoSchedule"
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: disktype
          operator: In
          values: ["ssd"]
```

**Interview Q&A**
> **Q: What's the actual difference between nodeSelector and nodeAffinity?**
> A: nodeSelector only does exact key-value matches with no OR logic and no "prefer, don't require" option. nodeAffinity supports operators (In/NotIn/Exists) and both hard (`required`) and soft (`preferred`) constraints — nodeSelector can't express "prefer this but don't block scheduling if unavailable."

---

## 14. Resource Requests, Limits & Autoscaling

**Revision Note**
`requests` drive scheduling decisions (scheduler only looks at requests); `limits` drive OOMKill (memory) or throttling (CPU). **CPU limit throttling can happen even with node headroom** — the CFS (Completely Fair Scheduler) quota is enforced per ~100ms scheduling period, not as a rolling average, so a burst above quota within a period gets throttled regardless of idle node capacity. **QoS classes**: Guaranteed (requests == limits) · Burstable (requests < limits) · BestEffort (neither set) — determines node eviction order under memory pressure (BestEffort evicted first). Autoscaling: **HPA** (replica count, metrics-server, CPU/memory/custom metrics) vs **VPA** (adjusts requests/limits of existing pods, don't run on the same metric as HPA) vs **Cluster Autoscaler** (scales predefined ASGs/node groups by watching unschedulable pods) vs **Karpenter** (provisions right-sized nodes directly, bypassing predefined ASGs).

**Simple Example**
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**Interview Q&A**
> **Q: Why would a Pod get OOMKilled even though the node has free memory?**
> A: OOMKilled happens when the container exceeds its own memory **limit**, not the node's total capacity — the kernel cgroup killer enforces that per-container ceiling regardless of what else is free on the node. Fix: profile actual usage and set a realistic limit, or fix a memory leak.
>
> **Q: CPU usage graphs show 40% utilization but pods are being throttled — explain.**
> A: CFS quota is enforced per scheduling period, not as an average. A pod can burst above quota within a single ~100ms period and get throttled even if average usage looks low over a longer window. Fix: raise the CPU limit, or drop it if isolation isn't critical.
>
> **Q: How do you scale nodes vs pods on EKS, and what triggers each?**
> A: Pod-level: HPA watches metrics and increases replica count. Node-level: Cluster Autoscaler/Karpenter watches for pods stuck Pending due to insufficient capacity and provisions new nodes. They're complementary — HPA creates demand, the node autoscaler supplies capacity.

---

## 15. Priority & Preemption

**Revision Note**
`PriorityClass` assigns an integer priority to pods; higher-priority pending pods can preempt (evict) lower-priority running pods to get scheduled under resource pressure. Only one PriorityClass can have `globalDefault: true`. Real use case: protecting critical infra pods (ingress controllers, CoreDNS) with high priority so they aren't evicted before less-critical app pods.

**Simple Example**
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
```

**Interview Q&A**
> **Q: A critical pod (e.g., CoreDNS) got evicted under memory pressure before less important app pods — how do you prevent recurrence?**
> A: Assign a high-priority PriorityClass to critical infra pods so lower-priority pods are evicted/preempted first, and ensure Guaranteed QoS (requests == limits) for those critical pods so they're not first in line under BestEffort/Burstable eviction either.

---

## 16. Node Operations — Cordon, Drain, PodDisruptionBudget

**Revision Note**
`kubectl cordon <node>` marks a node unschedulable — existing pods keep running, only new pods are blocked. `kubectl drain <node>` cordons AND evicts existing pods gracefully (respecting PodDisruptionBudgets), needing `--ignore-daemonsets` (DaemonSet pods aren't evicted by drain). Standard maintenance flow: cordon → drain → maintain/replace → uncordon. **PodDisruptionBudget (PDB)** caps how many pods of a set can be voluntarily down at once (`minAvailable`/`maxUnavailable`) — drain respects this and stalls if evicting would violate it.

**Simple Example**
```bash
kubectl cordon node-1
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data
kubectl uncordon node-1
```
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: web-app
```

**Interview Q&A**
> **Q: `kubectl drain` hangs and won't complete — why?**
> A: A PodDisruptionBudget on the affected pods doesn't allow further disruption (e.g., `minAvailable` equals current replica count), or a bare pod without a controller is blocking without `--force`, or a DaemonSet pod is blocking without `--ignore-daemonsets`.

---

## 17. ResourceQuota & LimitRange

**Revision Note**
**ResourceQuota** — namespace-level hard cap on total resource consumption (total CPU/memory, object counts like max pods/PVCs/services) — used in multi-tenant clusters so one team can't starve others. **LimitRange** — namespace-level defaults and min/max **per pod/container** (default request/limit if unset, and bounds so no single pod requests absurd amounts). They work together, not instead of each other: ResourceQuota caps the namespace total, LimitRange constrains individual objects within it.

**Simple Example**
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    pods: "20"
```

**Interview Q&A**
> **Q: Two teams share a cluster and one team's workloads are starving the other's — fix?**
> A: Namespace-level ResourceQuota to cap each team's total CPU/memory/object counts, combined with LimitRange to set sane per-pod defaults/bounds so quota isn't exhausted by a few oversized pods.

---

## 18. Rolling Updates & Deployment Strategies

**Revision Note**
Default strategy is `RollingUpdate` (controlled by `maxSurge`/`maxUnavailable`). Blue-Green and Canary aren't native Deployment features — typically implemented via tooling like Argo Rollouts or a service mesh for precise traffic-percentage routing.

**Simple Example**
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

**Interview Q&A**
> **Q: How would you do a zero-downtime deployment for a critical service?**
> A: `maxUnavailable: 0` so old Pods aren't killed until new ones are Ready, combined with a solid readiness probe and a PodDisruptionBudget to protect against simultaneous voluntary disruption during the deploy window. For canary %/automated rollback on metrics, I'd bring in Argo Rollouts instead of a plain Deployment.
>
> **Q: You need 10% canary traffic to a new version before full rollout — native Deployments can't easily do this. Options?**
> A: Either two Deployments behind one Service weighted crudely by replica count, or a service mesh (Istio/Linkerd) for precise percentage-based routing, or a GitOps-native progressive delivery tool like Argo Rollouts for canary/blue-green with automated analysis.

---

## 19. Helm

**Revision Note**
Package manager for Kubernetes — templates raw manifests with Go templating + `values.yaml` for parameterization, packaged as a "chart." Key commands: `install`/`upgrade`/`rollback` — each upgrade is a new revision, so rollback is easy. `helm template` renders without applying (useful for GitOps: render, then let ArgoCD apply). Helm is a templating/packaging tool, **not** a GitOps tool by itself — pairing it with ArgoCD/Flux is the production pattern for actual reconciliation.

**Simple Example**
```bash
helm install web-app ./chart -f values-prod.yaml
helm upgrade web-app ./chart -f values-prod.yaml
helm rollback web-app 3
```

**Interview Q&A**
> **Q: How do you roll back a bad Helm release quickly?**
> A: `helm rollback <release> <revision>` — Helm tracks revision history, so you can revert to the last known-good revision without manually reconstructing manifests.

---

## 20. Service Mesh & Sidecar Containers

**Revision Note**
A **sidecar** is a helper container in the same pod as the main app, sharing network/storage namespace — in a service mesh it's typically an Envoy proxy intercepting all in/out traffic. A **service mesh** (Istio, Linkerd) adds mTLS between services, fine-grained traffic control (canary/weighted routing, retries, circuit breaking), and observability — without app code changes, via sidecar injection. Trade-off: real latency and operational complexity — justify with a concrete need (zero-trust mTLS, fine-grained traffic shifting), not by default. Newer: ambient mesh (sidecar-less, e.g., Istio Ambient) reduces this overhead.

**Interview Q&A**
> **Q: A team wants every pod's outbound traffic logged and mTLS enforced between all services, without changing app code — what do you propose, and what's the cost?**
> A: Service mesh (Istio/Linkerd) with sidecar injection gives mTLS and traffic observability transparently. Cost: added per-hop latency, extra resource overhead per pod, and real operational complexity (mesh upgrades, debugging an extra network layer) — a genuine trade-off, not a free win.

---

## 21. Cluster Upgrades

**Revision Note**
Upgrade the control plane **first**, one minor version at a time (kubelet can be at most one minor version behind the API server — the version skew rule), then upgrade node groups/kubelets. On EKS, control plane upgrade is a managed AWS operation; node groups are upgraded separately (rolling replacement for managed node groups, manual for self-managed). Always check for deprecated/removed API versions before upgrading (`pluto`/`kubent`) — e.g., `extensions/v1beta1` Ingress was removed in 1.22.

**Interview Q&A**
> **Q: Explain the version skew rule you must respect during a cluster upgrade.**
> A: Kubelet version can be at most one minor version behind the API server; upgrade the control plane before node kubelets, one minor version at a time — you can't skip multiple minor versions in a single step.

---

## 22. Troubleshooting Toolkit (very commonly asked)

**Revision Note**
Standard debugging flow: pod status → describe → logs → events → node conditions.

**Simple Example**
```bash
kubectl get pods -n <ns> -o wide
kubectl describe pod <pod> -n <ns>
kubectl logs <pod> -n <ns> --previous
kubectl exec -it <pod> -n <ns> -- sh
kubectl get events -n <ns> --sort-by='.lastTimestamp'
```

**Interview Q&A**
> **Q: A Pod is stuck in `Pending` — what do you check?**
> A: `kubectl describe pod` first — Events usually shows the scheduling failure reason: insufficient CPU/memory on all nodes, unsatisfied affinity/taint, or a PVC that can't bind. I confirm with `kubectl top nodes` and `kubectl get pvc`.
>
> **Q: A Pod is `Running` but the Service sends it no traffic — debug?**
> A: `kubectl get endpoints <svc>` — if empty, the label selector doesn't match the pod's labels (most common cause), or the readiness probe is failing (Running ≠ Ready, only Ready pods become endpoints).

---

# PART 2: GITOPS & ARGOCD

## 23. What is GitOps

**Revision Note**
Git is the single source of truth for application and infrastructure desired state. An agent (ArgoCD/Flux) continuously reconciles the live cluster to match Git — instead of CI pushing directly via `kubectl apply`.

**Interview Q&A**
> **Q: How is GitOps different from a traditional CI/CD pipeline pushing to Kubernetes?**
> A: In push-based CI/CD, the pipeline needs cluster credentials and actively runs `kubectl apply`/`helm upgrade`. In GitOps, the pipeline only builds the image and updates a manifest/values file in Git — the cluster-side agent pulls that change and applies it itself. No cluster credentials leave the cluster, every change is a Git commit (audit trail), and manual drift gets detected and auto-corrected back to Git.

---

## 24. ArgoCD Architecture

**Revision Note**
Three core components: **API Server** (UI/CLI/gRPC, auth), **Repo Server** (clones Git, renders manifests via Helm/Kustomize), **Application Controller** (compares live vs desired state, triggers sync).

**Interview Q&A**
> **Q: Explain ArgoCD's reconciliation loop.**
> A: The Application Controller polls the target cluster and compares live resource state against manifests rendered by the Repo Server from Git. If they differ, the Application shows `OutOfSync`. With `automated` sync enabled, it applies the Git state automatically; otherwise it waits for manual sync. It repeats continuously (default ~3 min, plus on webhook-triggered Git changes).

---

## 25. ArgoCD Application Object

**Revision Note**
The core CRD tying a Git repo path to a destination cluster/namespace.

**Simple Example**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/web-app-manifests.git
    targetRevision: main
    path: k8s/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: prod
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Interview Q&A**
> **Q: What do `prune` and `selfHeal` do, and why are both important?**
> A: `prune: true` deletes resources from the cluster that were removed from Git, avoiding orphaned resources. `selfHeal: true` reverts any manual/out-of-band cluster change back to what's in Git, so nobody drifts silently — even by accident (e.g., a stray `kubectl edit` in prod).

---

## 26. Sync Status & Health Status

**Revision Note**
**Sync Status**: Synced / OutOfSync — does live state match Git? **Health Status**: Healthy / Progressing / Degraded / Missing — is the app actually working? These are independent — an app can be Synced but Degraded (e.g., bad image causing CrashLoopBackOff).

**Interview Q&A**
> **Q: An ArgoCD app shows "Synced" but "Degraded" — what does that tell you?**
> A: Synced means the cluster exactly matches Git — not a deployment/GitOps problem. Degraded means the workload itself is unhealthy despite that — bad image tag, failing probes, crash loop. Go straight to `kubectl describe`/`logs` on the underlying Pods rather than ArgoCD itself.

---

## 27. App of Apps Pattern

**Revision Note**
A parent ArgoCD Application whose "manifests" are actually other Application definitions — used to manage many apps/environments declaratively from one root.

**Interview Q&A**
> **Q: How would you manage 15 microservices across 3 environments with ArgoCD without manually creating 45 Applications?**
> A: App of Apps — one root Application points to a repo/folder containing Application manifests for every service/environment combination (often templated with Helm or generated via ApplicationSet's matrix generator over services × environments). Adding a new service/environment becomes a Git commit, not manual UI work.

---

## 28. ApplicationSet

**Revision Note**
Generates multiple Applications from a single template using generators (List, Git directory, Cluster, Matrix). More scalable than App of Apps for dynamic sets.

**Simple Example**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: services
spec:
  generators:
  - git:
      repoURL: https://github.com/myorg/manifests.git
      revision: main
      directories:
      - path: apps/*
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      source:
        repoURL: https://github.com/myorg/manifests.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
```

**Interview Q&A**
> **Q: When would you use ApplicationSet over App of Apps?**
> A: When the set of apps/environments is dynamic and derived from repo structure or a cluster list — a new folder under `apps/` should automatically become an Application without hand-writing YAML each time. App of Apps suits a small, fairly static list; ApplicationSet scales better with less boilerplate.

---

## 29. Sync Waves & Hooks

**Revision Note**
`sync-wave` annotation controls ordering (lower numbers sync first — e.g., namespace/CRDs before the app). Hooks (`PreSync`, `Sync`, `PostSync`) run jobs at specific points, e.g., a DB migration before app rollout.

**Simple Example**
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
    argocd.argoproj.io/hook: PreSync
```

**Interview Q&A**
> **Q: How do you run a database migration before your app deploys, using ArgoCD?**
> A: Define the migration as a Job with `argocd.argoproj.io/hook: PreSync`. ArgoCD runs it before syncing the rest of the manifests; if the Job fails, sync stops there instead of rolling out against an un-migrated schema.

---

## 30. Rollback in GitOps

**Revision Note**
Two ways: `argocd app rollback <app> <history-id>` reverts the live cluster to a previous ArgoCD-tracked revision, or `git revert` the commit and let ArgoCD auto-sync — the GitOps-pure way, keeping Git as source of truth.

**Interview Q&A**
> **Q: A bad deploy just went out via ArgoCD — walk me through your rollback.**
> A: The GitOps-correct way is `git revert` the bad commit (e.g., a bad image tag) and push — ArgoCD auto-syncs to the known-good state, Git history stays accurate. `argocd app rollback` is faster for an emergency but moves the cluster out of sync with Git HEAD, so I always follow up with a matching Git revert to keep them consistent.

---

## 31. Managing Secrets in GitOps (ArgoCD-specific)

**Revision Note**
Never commit plain Secrets to Git. Patterns: Sealed Secrets (encrypt-then-commit), External Secrets Operator (fetch from AWS Secrets Manager/Vault at runtime), or SOPS (encrypted values file).

**Interview Q&A**
> **Q: If Git is the source of truth, how do secrets fit into a GitOps model without being exposed?**
> A: The Secret's *reference* lives in Git (e.g., an ExternalSecret pointing at an AWS Secrets Manager ARN, or a SealedSecret's encrypted blob) — never the plaintext value. The material stays in a vault and is fetched at sync/runtime, or is encrypted so only the in-cluster controller can decrypt it. I use External Secrets Operator against AWS Secrets Manager for exactly this reason.

---

## 32. Multi-Cluster / Multi-Environment with ArgoCD

**Revision Note**
One ArgoCD instance can manage many clusters (`argocd cluster add`). Environments (dev/staging/prod) are typically separated by directory/branch/Kustomize overlay + separate Applications, often via App-of-Apps or ApplicationSet's cluster generator.

**Interview Q&A**
> **Q: How do you promote a change from staging to production in a GitOps model?**
> A: Promotion happens via Git, not by re-running a pipeline against prod. Common patterns: separate overlays per environment (Kustomize) with a PR bumping the prod overlay's image tag/values once staging is validated, or separate branches with a promotion PR merging staging → main/prod. ArgoCD picks it up automatically for the prod Application — the PR *is* the change record.

---

# PART 3: TOP INTERVIEW QUESTIONS (QUICK-FIRE LIST)

## Kubernetes
1. Explain the control plane components and what each does.
2. What happens end-to-end when you run `kubectl apply -f deployment.yaml`?
3. Deployment vs StatefulSet vs DaemonSet — when do you use each?
4. Difference between `requests` and `limits`? What happens if you exceed each?
5. Liveness vs readiness vs startup probes — a real scenario you've hit.
6. How does a Service find and load-balance to Pods?
7. ClusterIP vs NodePort vs LoadBalancer.
8. Ingress vs Ingress Controller vs LoadBalancer Service.
9. ConfigMaps vs Secrets — are Secrets actually secure by default?
10. Debug a Pod stuck in `Pending`.
11. Debug a Pod in `CrashLoopBackOff`.
12. What causes OOMKilled, and how do you fix it?
13. How does HPA work, and what metrics can it use? HPA vs VPA vs Cluster Autoscaler vs Karpenter.
14. RBAC — Role vs ClusterRole, RoleBinding vs ClusterRoleBinding; the ClusterRole+RoleBinding combo.
15. What is a Namespace used for, and what isn't isolated by it?
16. How do rolling updates work — how do you achieve zero-downtime deployment?
17. PV vs PVC vs StorageClass; RWO vs RWX.
18. Troubleshoot a Pod that can't pull its image.
19. NetworkPolicy default behavior, and why AWS VPC CNI needs an add-on to enforce it.
20. Taints/tolerations vs node affinity vs nodeSelector — what's actually different.
21. Why does CPU throttling happen even with node headroom (CFS quota)?
22. cordon vs drain, and what a PodDisruptionBudget does during a drain.
23. ResourceQuota vs LimitRange.
24. IRSA vs node instance role — why it matters on EKS.
25. Pod Security Admission levels and modes (enforce/audit/warn).
26. Helm rollback vs Deployment rollback — when would you use each.
27. Service mesh + sidecar — what it buys you and what it costs.
28. Cluster upgrade order and the version skew rule.

## GitOps / ArgoCD
29. What is GitOps vs traditional push-based CI/CD?
30. ArgoCD architecture — API Server, Repo Server, Application Controller.
31. Sync Status vs Health Status.
32. What `prune` and `selfHeal` do in a sync policy.
33. How ArgoCD detects and handles configuration drift.
34. App of Apps — why and when.
35. ApplicationSet vs App of Apps.
36. Sync waves and PreSync/PostSync hooks — a real use case.
37. Rolling back a bad deployment in GitOps the "correct" way.
38. Managing secrets in a GitOps repo without exposing plaintext.
39. Promoting a change from staging to production using GitOps.
40. Managing deployments across multiple clusters from one ArgoCD instance.

---

# PART 4: ANSWERS TO ALL 40 QUESTIONS

## Kubernetes

**1. Control plane components**
API Server — single entry point, runs auth + admission control, persists to etcd. etcd — Raft-based key-value store holding all cluster state (odd node count for quorum). Scheduler — assigns Pods to nodes via predicates (filtering) and priorities (scoring). Controller Manager — runs reconciliation loops comparing desired vs actual state (Deployment, Node, ReplicaSet controllers etc.).

**2. `kubectl apply -f deployment.yaml` flow**
kubectl sends the manifest to the API Server → admission control (mutating then validating webhooks) → persisted to etcd. Deployment controller creates a ReplicaSet → ReplicaSet creates Pods. Scheduler assigns each Pod to a node based on requests/affinity/taints. kubelet on that node pulls the image via the container runtime (CRI) and starts it, reporting status back to the API Server.

**3. Deployment vs StatefulSet vs DaemonSet**
Deployment — stateless, interchangeable Pods, for typical app workloads. StatefulSet — stable network identity (`app-0`, `app-1`) + a dedicated PVC per Pod that persists across restarts, ordered deploy/scale — for databases/Kafka. DaemonSet — runs exactly one Pod per matching node, for node-level agents like log shippers, CNI plugins, monitoring exporters.

**4. `requests` vs `limits`**
`requests` is what the Scheduler uses to place a Pod (guaranteed minimum). `limits` is the hard ceiling. Exceeding a memory limit → OOMKilled by the kernel cgroup killer. Exceeding a CPU limit → throttled, not killed. Exceeding requests alone has no direct penalty but affects QoS class and eviction order under pressure.

**5. Liveness vs readiness vs startup probes**
Liveness restarts a stuck/deadlocked container. Readiness pulls a Pod out of Service endpoints when it can't serve traffic yet (Pod keeps running). Startup protects slow-starting apps from being killed by liveness before they're actually up — once it passes, liveness/readiness take over. Real scenario: a Pod loops in CrashLoopBackOff because `initialDelaySeconds` on liveness is too short for a slow-starting app — fix by adding a startupProbe or increasing the delay.

**6. How a Service finds and load-balances to Pods**
The Service's label selector matches Pod labels, and the Endpoints (or EndpointSlice) controller keeps a live list of matching, Ready Pod IPs. kube-proxy programs the node's iptables (default) or IPVS rules to load-balance traffic across those endpoint IPs — no proxying through a central component.

**7. ClusterIP vs NodePort vs LoadBalancer**
ClusterIP — internal-only virtual IP (default). NodePort — exposes a static port on every node's IP, rarely used in prod due to port range limits and no health-aware routing. LoadBalancer — provisions an actual cloud load balancer (e.g., AWS NLB/ALB) pointing at the Service.

**8. Ingress vs Ingress Controller vs LoadBalancer Service**
Ingress is just an API object describing host/path routing rules — it does nothing by itself. An Ingress Controller (NGINX, AWS Load Balancer Controller) watches Ingress objects and provisions/configures the real load balancer. A LoadBalancer Service provisions one dedicated cloud LB per Service with no path-based routing — Ingress lets many Services share one LB with host/path rules, which is both cheaper and more flexible for HTTP.

**9. ConfigMaps vs Secrets — are Secrets secure by default?**
ConfigMaps hold non-sensitive config; Secrets hold sensitive data. No — Secrets are only base64-**encoded**, not encrypted, by default. Real protection comes from enabling etcd encryption at rest (KMS envelope on EKS) and tightly scoping RBAC `get`/`list` on Secrets — base64 alone is trivially reversible.

**10. Debug a Pod stuck in `Pending`**
`kubectl describe pod` first — Events usually shows the exact scheduling failure: insufficient CPU/memory across all nodes, an unsatisfied affinity/taint, or a PVC that can't bind. Cross-check with `kubectl top nodes` and `kubectl get pvc`.

**11. Debug a Pod in `CrashLoopBackOff`**
Check `kubectl logs <pod> --previous` for the actual crash reason, and `kubectl describe pod` for probe failures/OOM events. Common causes: app crashing on bad config/missing env var, a liveness probe firing too early on a slow-starting app, or the container's main process exiting immediately (wrong entrypoint/command).

**12. What causes OOMKilled, and how do you fix it?**
The container exceeded its own memory **limit** — the kernel cgroup killer enforces that per-container ceiling regardless of free node memory. Fix: profile actual memory usage and set a realistic limit, or fix an application memory leak.

**13. HPA vs VPA vs Cluster Autoscaler vs Karpenter**
HPA scales replica **count** based on metrics (CPU/memory/custom, via metrics-server). VPA adjusts the requests/limits of existing Pods (don't run both on the same metric — they can fight). Cluster Autoscaler scales predefined node groups/ASGs by watching for unschedulable Pods. Karpenter provisions right-sized nodes directly, bypassing predefined ASGs, for faster/more efficient scaling.

**14. RBAC — Role vs ClusterRole, RoleBinding vs ClusterRoleBinding**
Role/RoleBinding are namespace-scoped; ClusterRole/ClusterRoleBinding are cluster-wide. A useful combo: a ClusterRole (reusable, defined once) bound via a namespace-scoped RoleBinding — grants those permissions only in that one namespace, not cluster-wide.

**15. What a Namespace isolates — and what it doesn't**
Namespaces logically group namespaced objects (Pods, Services, ConfigMaps, Secrets, Deployments). They do **not** isolate nodes, PersistentVolumes, ClusterRoles, or network traffic by default — cross-namespace traffic is allowed unless a NetworkPolicy restricts it.

**16. Rolling updates & zero-downtime deployment**
Default `RollingUpdate` strategy replaces Pods gradually per `maxSurge`/`maxUnavailable`. For zero downtime: `maxUnavailable: 0` so old Pods aren't killed until new ones are Ready, backed by a real readiness probe (not just a TCP check) and a PodDisruptionBudget to prevent simultaneous voluntary disruption during the rollout.

**17. PV vs PVC vs StorageClass; RWO vs RWX**
PV is the actual provisioned storage resource; PVC is a Pod's request/claim for storage; StorageClass enables dynamic provisioning (e.g., EBS via the CSI driver). RWO (ReadWriteOnce, e.g. EBS) is single-node-only; RWX (ReadWriteMany, needs EFS on AWS) allows simultaneous access from Pods on different nodes.

**18. Troubleshoot a Pod that can't pull its image (`ImagePullBackOff`)**
Check the image name/tag for a typo, verify registry authentication (`imagePullSecrets` configured and valid), confirm network/egress access to the registry from the node, and check for registry rate limiting (common with Docker Hub).

**19. NetworkPolicy default behavior, and the AWS VPC CNI gotcha**
Default is fully open between Pods; a NetworkPolicy becomes additive/whitelist-only once it selects a Pod. Plain AWS VPC CNI does **not enforce** NetworkPolicy at all without an add-on like Calico or Cilium — policies can be applied and look fine but silently do nothing.

**20. Taints/tolerations vs node affinity vs nodeSelector**
Taints live on nodes and repel Pods; tolerations live on Pods and merely permit scheduling despite a taint (they don't attract). nodeSelector is exact-match only, no OR logic, no "prefer" option. nodeAffinity/podAffinity are more expressive — operators (In/NotIn/Exists) and both hard (`required`) and soft (`preferred`) constraints.

**21. Why does CPU throttling happen even with node headroom?**
The CFS (Completely Fair Scheduler) quota is enforced per ~100ms scheduling period, not as a rolling average — a Pod can burst above its quota within a single period and get throttled even though average utilization looks low over a longer window and the node has idle capacity.

**22. cordon vs drain, and PodDisruptionBudget**
`cordon` marks a node unschedulable but leaves existing Pods running. `drain` cordons **and** gracefully evicts existing Pods, respecting any PodDisruptionBudget (`minAvailable`/`maxUnavailable`) — it needs `--ignore-daemonsets` since DaemonSet Pods aren't evicted by drain, and will stall if evicting would violate a PDB.

**23. ResourceQuota vs LimitRange**
ResourceQuota is a namespace-level hard cap on total consumption (total CPU/memory, object counts) — stops one team from starving others in a shared cluster. LimitRange sets defaults and min/max **per Pod/container** within a namespace. They're complementary: ResourceQuota caps the aggregate, LimitRange bounds individual objects.

**24. IRSA vs node instance role**
IRSA (IAM Roles for Service Accounts) lets a specific Pod assume a scoped IAM role via an OIDC-linked ServiceAccount. Attaching a policy to the node's instance role instead gives **every** Pod on that node the same AWS access — a least-privilege violation. IRSA scopes permissions per-workload instead of per-node.

**25. Pod Security Admission levels and modes**
Levels: Privileged (unrestricted), Baseline (blocks known privilege escalations), Restricted (no root, no privilege escalation, dropped capabilities, seccomp required). Modes: `enforce` (blocks), `audit` (logs), `warn` (client-side warning) — apply `warn`/`audit` first to see what would break, then switch to `enforce` once workloads are fixed.

**26. Helm rollback vs Deployment rollback**
`helm rollback` reverts an entire chart **release** — every resource and value the chart manages — to a prior revision in one command. `kubectl rollout undo` only reverts a single Deployment's Pod template via ReplicaSet history. Use Helm rollback when multiple related resources changed together as one release; Deployment rollback for a quick single-resource revert.

**27. Service mesh + sidecar — benefit and cost**
A sidecar (often an Envoy proxy) shares a Pod's network namespace and intercepts its traffic; a service mesh (Istio/Linkerd) uses this to add mTLS between services, fine-grained traffic control (canary, retries, circuit breaking), and observability without app code changes. Cost: real added latency per hop, extra resource overhead per Pod, and operational complexity (mesh upgrades, an extra network layer to debug) — justify with a concrete need, not by default.

**28. Cluster upgrade order and version skew rule**
Upgrade the control plane first, one minor version at a time, then node groups/kubelets — kubelet can be at most one minor version behind the API server, so you can't skip multiple minor versions in a single step. Always check for deprecated/removed API versions before upgrading (`pluto`/`kubent`).

## GitOps / ArgoCD

**29. GitOps vs traditional push-based CI/CD**
Push-based CI/CD needs cluster credentials in the pipeline and runs `kubectl apply`/`helm upgrade` directly. GitOps has the pipeline only update a manifest/values file in Git — an in-cluster agent (ArgoCD) pulls and applies the change itself. Benefits: no cluster credentials leave the cluster, every change is a Git commit (audit trail), and manual drift is detected and can be auto-corrected back to Git.

**30. ArgoCD architecture**
API Server (UI/CLI/gRPC endpoint, auth), Repo Server (clones the Git repo, renders manifests via Helm/Kustomize), Application Controller (continuously compares live cluster state vs the rendered desired state and triggers sync).

**31. Sync Status vs Health Status**
Sync Status (Synced/OutOfSync) answers "does the live cluster match Git?" Health Status (Healthy/Progressing/Degraded/Missing) answers "is the workload actually working?" They're independent — an app can be perfectly Synced but Degraded because of a bad image or failing probes.

**32. What `prune` and `selfHeal` do**
`prune: true` deletes cluster resources that were removed from Git, preventing orphaned resources. `selfHeal: true` automatically reverts any manual/out-of-band change made directly in the cluster back to what's in Git — so nobody can silently drift from source of truth.

**33. How ArgoCD detects and handles drift**
The Application Controller continuously diffs live resource state against the manifests rendered from Git (default ~3 min poll, plus webhook-triggered on Git changes). Any mismatch flips the Application to `OutOfSync`. If `selfHeal` is enabled it auto-corrects immediately; otherwise it waits for a manual sync.

**34. App of Apps — why and when**
A parent Application whose source is itself a folder of other Application manifests. Used to manage many microservices/environments declaratively from one Git root — adding a new service/environment becomes a Git commit instead of manual UI work in ArgoCD.

**35. ApplicationSet vs App of Apps**
ApplicationSet generates many Applications from one template using generators (List, Git directory, Cluster, Matrix) — better when the set is dynamic, e.g. a new folder under `apps/` auto-creates an Application. App of Apps suits a smaller, fairly static list and is simpler to reason about.

**36. Sync waves and PreSync/PostSync hooks**
`sync-wave` annotations control ordering within a sync (lower numbers first — e.g., namespaces/CRDs before the app). Hooks (`PreSync`, `Sync`, `PostSync`) run one-off Jobs at specific points — real use case: running a database migration Job as a `PreSync` hook so it completes before the app rollout, and the sync halts if the migration fails.

**37. Rolling back a bad deployment in GitOps, the "correct" way**
`git revert` the bad commit and push — ArgoCD auto-syncs back to the known-good state and Git history stays accurate. `argocd app rollback` is faster for an emergency but moves the cluster out of sync with Git HEAD, so follow it up with a matching Git revert to keep them consistent.

**38. Managing secrets in a GitOps repo without exposing plaintext**
Only a *reference* to the secret lives in Git — e.g. a SealedSecret's encrypted blob (via `kubeseal`, decrypted in-cluster by its controller) or an ExternalSecret pointing at an AWS Secrets Manager/Vault path, fetched at runtime by External Secrets Operator. Plaintext never touches Git either way.

**39. Promoting a change from staging to production using GitOps**
Promotion happens via Git, not by re-running a pipeline against prod. Typical patterns: separate environment overlays (Kustomize) with a PR bumping the prod overlay's image tag/values once staging is validated, or separate branches with a promotion PR merging staging → main. ArgoCD picks it up automatically for the prod Application — the PR itself is the audit trail.

**40. Managing deployments across multiple clusters from one ArgoCD instance**
One ArgoCD instance can register and manage many target clusters (`argocd cluster add`). Environments/clusters are typically separated by directory/branch/Kustomize overlay plus separate Application objects per cluster, often generated via App of Apps or an ApplicationSet cluster generator rather than managed by hand.

---
*Focus on explaining the "why" behind each answer, not just the definition — that's what separates a 4-year-experience answer from a fresher one.*
