# Kubernetes — Interview Refresher

> Single-document refresher for a broad technical interview where Kubernetes is one of several technologies that might come up — not a K8s-specific deep dive. Targets "experienced practitioner" depth: enough that an interviewer can tell you've actually run K8s in production. Reflects K8s 1.33+ semantics (current stable: 1.36, April 2026). For deeper notes per topic, see the [Further study](#6-further-study) index at the end.

---

## 1. Core concepts and architecture

Kubernetes is a **declarative reconciliation system**. You write the desired state into the API server; controllers continuously compare actual state against desired state and make changes. Internalize that mental model first — almost every "weird" behavior makes sense once you accept that nothing is request/response.

### Control plane

| Component | Role | What an interviewer wants you to know |
|---|---|---|
| `kube-apiserver` | The only thing that talks to etcd. RESTful, stateless, horizontally scalable. | All other components talk *only* to the apiserver — never to each other, never to etcd directly. Watch streams (long-lived HTTP) are how controllers stay current. |
| `etcd` | Cluster source of truth. Distributed key-value store using Raft consensus. | Lose etcd → lose the cluster brain (existing workloads keep running, but you can't change anything). Always backed up. Three or five voting members is the standard. |
| `kube-scheduler` | Decides which node a Pod runs on. | Two phases: filtering (predicates — does the node *fit*?) then scoring (priorities — which fits *best*?). It doesn't actually bind the Pod until it patches `spec.nodeName`. |
| `kube-controller-manager` | Hosts built-in reconcile loops (Deployment, ReplicaSet, Node, Job, ServiceAccount, ...). | Each controller watches one or more resource kinds and drives actual → desired. The work-queue + reconcile pattern is the same one you'd implement in a custom operator. |
| `cloud-controller-manager` | Bridges to the cloud provider for nodes, routes, and load balancers. | Lets the core control plane stay cloud-agnostic. On managed K8s (EKS, AKS, GKE) this is the bit you don't see. |

### Node components

- **kubelet** — the agent on each node. Watches the apiserver for Pods bound to its node, talks to the container runtime via the Container Runtime Interface (CRI) gRPC API, reports status back. The only component that has to know about the local container runtime.
- **kube-proxy** — programs Service IP routing (iptables, IPVS, or — increasingly — replaced by an eBPF dataplane like Cilium). Watches Services and EndpointSlices.
- **container runtime** — `containerd` or CRI-O are standard now (Docker shim is gone since 1.24).

### The reconciliation loop, in one paragraph

Every K8s controller runs the same loop: **list-and-watch** the resources it owns, compute the diff between observed state and desired state, and emit imperative API calls to close that gap. The loop is *level-triggered, not edge-triggered* — losing an event isn't catastrophic because the next reconcile sees the same diff. This is why K8s tolerates restarts, network blips, and missed events: it eventually converges. It's also why most "why isn't my change taking effect" questions resolve to "your controller hasn't reconciled yet" or "another controller is fighting yours."

---

## 2. Workload and networking primitives

### Workloads

- **Pod** — the unit of scheduling and the unit of one IP. Containers in a Pod share network and IPC namespaces, mount the same volumes, and live and die together. *You almost never create bare Pods in production* — wrap them in something that owns their lifecycle.

- **ReplicaSet** — keeps N copies of a Pod template running. You almost never write these by hand either; Deployments own them.

- **Deployment** — the default for stateless workloads. Owns ReplicaSets to enable **rolling updates**: it creates a new RS at the target revision and scales it up while scaling the old one down, controlled by `maxSurge` (extra Pods allowed during rollout) and `maxUnavailable` (how many can be unhealthy at once). Keeps a revision history (default 10) for `kubectl rollout undo`.

- **StatefulSet** — for workloads that need **stable identity** (deterministic Pod names like `kafka-0`, `kafka-1`), **ordered rollout** (Pod 0 must be Ready before Pod 1 starts), and **stable, per-Pod persistent storage** via `volumeClaimTemplates`. Pair with a **headless Service** (`clusterIP: None`) so each Pod is reachable at a stable DNS name. Use for databases, brokers, leader-elected systems.

- **DaemonSet** — one Pod per node (or per node matching a selector). Use for node-level agents: log shippers, CNI plugins, node exporters, CSI node plugins.

- **Job / CronJob** — run-to-completion workloads. `parallelism` (concurrent Pods), `completions` (total successful Pods needed), `backoffLimit` (retries before failing), `activeDeadlineSeconds` (kill switch). CronJob is a Job factory; `concurrencyPolicy: Forbid` is almost always what you want.

- **Init containers vs native sidecars** — init containers run sequentially and complete before app containers start (good for migrations, secret pre-fetches, schema setup). **Native sidecar containers** are init containers with `restartPolicy: Always`; they start before app containers, *stay running for the Pod's lifetime*, and don't block Pod termination. Native sidecars went **beta in 1.29 and GA in 1.33** — they fix the long-standing "service-mesh proxy exits before the main container finishes draining" problem. Before 1.29, people hacked this with shared volumes and `preStop` sleep tricks.

### Networking

The K8s networking model has three rules: **every Pod gets its own IP**, **Pods can talk to each other without NAT**, and **agents on a node can talk to all Pods on that node**. The model is intentionally boring — the complexity is offloaded to the CNI plugin (Calico, Cilium, Flannel).

- **Service** — a stable virtual IP and DNS name for a set of Pods. Selectors match Pod labels and produce **EndpointSlices** (the modern, sharded replacement for `Endpoints` — relevant if you've ever debugged DNS round-robin behavior).
  - `ClusterIP` (default): in-cluster only.
  - `NodePort`: opens a high port on every node — generally a dev/test convenience.
  - `LoadBalancer`: provisions a cloud LB via the cloud-controller-manager.
  - `ExternalName`: a CNAME, no proxying. Useful for cross-cluster references.
  - **Headless** (`clusterIP: None`): no virtual IP — DNS returns Pod IPs directly. Required for StatefulSets and for client-side load balancing (gRPC, etc.).

- **Ingress vs Gateway API** — Ingress is the legacy HTTP(S) entry point: pick a controller (NGINX, Traefik) and write `Ingress` resources with hosts and paths. **Gateway API** is the strategic replacement — `Gateway`/`GatewayClass`/`HTTPRoute` GA since Gateway API v1.0; `GRPCRoute` GA in v1.1; `TLSRoute` GA in v1.5 (Feb 2026). Use Gateway API for new clusters: clean separation between platform team (`Gateway`) and app teams (`*Route`), proper L4/L7 split, and richer routing semantics.

- **NetworkPolicy** — Pod-level firewall rules expressed as label selectors. **Default-allow** unless a policy selects the Pod, then **deny by default** for the matching direction. The standard production pattern: a namespace-wide `default-deny` policy plus explicit allow policies per workload. NetworkPolicy enforcement requires a CNI that implements it (Calico, Cilium yes; Flannel no).

### Configuration

- **ConfigMap** — non-secret key/value data, mounted as files or env vars. `immutable: true` is a large perf win on big clusters.
- **Secret** — same shape, stored separately, *base64-encoded, not encrypted at rest by default*. Never call a Secret "encrypted" in an interview — the data is obfuscated, not protected. For real protection: enable etcd encryption-at-rest **and** fetch from a real KMS (External Secrets Operator + AWS Secrets Manager / Vault, or Sealed Secrets for GitOps-friendly storage).

---

## 3. Operational topics that come up in interviews

### Scheduling

- `nodeSelector` — simple label match. Coarse but readable.
- **Node affinity** — same idea with operators (`In`, `NotIn`, `Exists`) and *required vs preferred* (`requiredDuringSchedulingIgnoredDuringExecution` is a hard constraint; `preferred` is a weighted hint).
- **Pod affinity / anti-affinity** — schedule near (or away from) Pods matching a selector. Anti-affinity is how you spread replicas across zones.
- **Taints and tolerations** — the *node* says "don't run here unless you tolerate me"; the Pod opts in. Use for dedicated node pools (GPU, ARM, Spot).
- **Topology spread constraints** — the modern primitive for "spread my replicas evenly across zones." Replaces awkward pod-anti-affinity hacks.
- **PriorityClass and preemption** — high-priority Pods can evict lower-priority ones to make room. Use sparingly; preemption interacts badly with PDBs.
- **PodDisruptionBudgets (PDB)** — bounds *voluntary* disruption (drains, evictions). Set `minAvailable` or `maxUnavailable`. PDBs do *not* protect against involuntary disruption (node crash, OOM).

### Resource management

- **Requests** = scheduler input; reserves capacity on a node.
- **Limits** = enforced by the kubelet at runtime; CPU is throttled, **memory is OOM-killed**.
- **QoS class** is derived: `Guaranteed` (requests == limits for all containers), `Burstable` (some requests set), `BestEffort` (none set). Eviction order under node pressure: BestEffort → Burstable → Guaranteed.
- **Footgun**: setting only limits (no requests) makes the scheduler think the Pod needs nothing, so it bin-packs it onto a busy node where it then gets throttled. Always set requests; consider whether limits are needed at all (CPU limits especially are debated — many shops disable CPU limits on latency-sensitive workloads).
- For **JVM / Go**: container limits don't automatically translate to runtime memory caps. Set `-XX:MaxRAMPercentage` (Java) or `GOMEMLIMIT` (Go ≥ 1.19) or the runtime keeps allocating until OOMKilled.

### Health checks

Three probes, three different jobs:

- **Liveness** — "is this process wedged?" Failure → kubelet restarts the container. Get it wrong on a slow-start app and you'll create restart loops.
- **Readiness** — "should this Pod receive traffic?" Failure → kubelet removes the Pod IP from the Service's EndpointSlice. *No restart, no eviction.*
- **Startup** — "has the app finished booting?" Liveness/readiness are paused until the startup probe succeeds. Use this for slow boots (large JVMs, DB warmup) instead of cranking liveness `initialDelaySeconds`.

The classic interview confusion: "readiness ≠ healthy" and "liveness on a slow-start app causes restart loops." Be ready to articulate both.

### Rollouts

`kubectl rollout status deploy/foo` — watch a rollout. `kubectl rollout undo deploy/foo` — go back one revision. Rollouts are blocked while a Pod is failing readiness. PDBs don't apply to Deployment rollouts directly, but they *do* apply when a node is drained mid-rollout — that interaction is where most "the rollout is stuck" tickets come from.

### Scaling

- **HPA (Horizontal Pod Autoscaler)** — scales replicas based on metrics. v2 supports CPU/memory plus custom and external metrics (queue depth, request rate). Tune `behavior.scaleDown.stabilizationWindowSeconds` to avoid flapping.
- **VPA (Vertical Pod Autoscaler)** — adjusts requests/limits. `Auto` mode evicts Pods to apply changes (disruptive); most teams use `recommendation` mode and apply manually.
- **KEDA** — event-driven autoscaling. Scales on queue lag, Kafka topic offset, etc., and can scale to zero. The right choice for spiky/bursty workloads.
- **Cluster Autoscaler vs Karpenter** — Cluster Autoscaler scales pre-defined node groups. Karpenter (originally AWS, now broadly adopted) provisions nodes *just-in-time* based on pending Pod shapes — better bin-packing, faster scale-out, more flexible instance selection.

### RBAC

- **Role / ClusterRole** — verb-on-resource permissions. `Role` is namespace-scoped; `ClusterRole` is cluster-scoped *or* used with a `RoleBinding` to grant cluster-wide-defined permissions inside one namespace.
- **RoleBinding / ClusterRoleBinding** — binds a Role/ClusterRole to a subject (User, Group, or ServiceAccount).
- **ServiceAccount** — the workload's identity. Tokens used to be long-lived JWTs in a Secret; now they are **bound, projected** tokens with audience claims and short TTLs, rotated automatically by the kubelet.
- **Pod Security Standards** (replacement for the deprecated PodSecurityPolicy, stable since 1.25): three profiles — `privileged`, `baseline`, `restricted` — applied to namespaces with labels. Three modes: `enforce`, `audit`, `warn`. Most teams enforce `baseline` cluster-wide and `restricted` for app namespaces.

### Observability

- **Logs** — write structured JSON to stdout/stderr; let the cluster collect (Fluent Bit / Vector → Loki / Elasticsearch / cloud logging). Don't write log files inside containers.
- **Metrics** — `metrics-server` is the lightweight sampler that backs `kubectl top` and basic HPA. **Prometheus** (kube-prometheus-stack) is the real metrics platform; expose `/metrics` and use `ServiceMonitor` / `PodMonitor` to scrape.
- **Events** — `kubectl get events --sort-by=.lastTimestamp` is the first thing to check when something is wrong. Most "why is my Pod Pending" answers are sitting in events.
- **Tracing** — OpenTelemetry collector is the standard; export to Jaeger / Tempo / a vendor.

### Common failure modes (recognize on sight)

| Symptom | Top causes | First check |
|---|---|---|
| `CrashLoopBackOff` | App crashes on start, missing config, bad command. | `kubectl logs --previous` |
| `ImagePullBackOff` | Wrong image name/tag, private registry creds missing, network. | `kubectl describe pod` (events) |
| `OOMKilled` | Limit too low, memory leak, JVM/Go runtime not aware of cgroup. | `kubectl describe pod` → last termination state; check `GOMEMLIMIT` / `MaxRAMPercentage` |
| `Pending` | No node fits requests, taints not tolerated, PVC unbound, image pull secrets missing in namespace. | `kubectl describe pod` (scheduler events) |
| Node `NotReady` | kubelet down, runtime down, network partition, disk pressure. | `kubectl describe node`, then SSH and check kubelet/containerd. |
| DNS resolution flaky | CoreDNS overloaded, `ndots:5` causing search-domain explosion, NetworkPolicy blocking 53. | `kubectl exec ... -- nslookup` from inside the Pod, check CoreDNS metrics. |

The diagnostic sequence to internalize: **`get` → `describe` → `logs --previous` → `events` → `debug`**. `kubectl debug` (ephemeral containers, GA since 1.25) lets you attach a shell to a running Pod's network namespace without modifying it — invaluable for distroless images.

---

## 4. Trade-offs and things learned the hard way

The kind of practical commentary that signals real production experience.

- **Liveness probes cost more outages than they prevent.** Start with a readiness probe only. Add liveness *only* if you have evidence of wedge-able state, and make it lazy (long `failureThreshold`, conservative `initialDelaySeconds`, prefer a startup probe for boot time).

- **Default-deny NetworkPolicy is the right hill to die on, but it will break your team for a week.** Roll it out namespace by namespace, with an audit period using a CNI that supports policy logging (Calico, Cilium).

- **Don't put your CI/CD in the same cluster as the workloads it deploys.** When the cluster is unhealthy, you lose the ability to fix it.

- **Helm rollback fails when CRDs drift.** Helm doesn't manage CRDs the same way as other resources. For anything operator-shaped, prefer Kustomize + GitOps, or use Helm with explicit CRD lifecycle management.

- **Every CRD is a long-term maintenance commitment.** Operators are not free — you own conversion webhooks across cluster upgrades, finalizer cleanup, and reconciliation correctness. Reach for well-maintained ones (cert-manager, External Secrets, Argo CD) before writing your own.

- **Argo CD's "app-of-apps" pattern is a trap if your team doesn't already do GitOps discipline.** Without strong PR review and environment promotion, a typo fans out across every cluster in seconds.

- **CPU limits are usually wrong by default.** They cause throttling on bursty workloads (Node, Java GC, Go goroutine spikes). Many production clusters set CPU requests but no CPU limits. Always set memory limits, though — they're how you contain leaks.

- **PDBs and Cluster Autoscaler don't always cooperate.** A `minAvailable: 100%` PDB blocks scale-down indefinitely. Always set PDBs to allow at least one replica down; otherwise drains stall.

- **etcd is the bottleneck you'll hit before CPU.** Watch `apiserver_request_duration_seconds` and etcd commit latency. Large CRDs, big ConfigMaps, and chatty controllers all hit etcd hard.

- **`kubectl apply -f` in production is not GitOps.** It's "I have credentials and I clicked enter." Move to Argo CD or Flux *before* an incident caused by config drift makes the case for you.

---

## 5. Likely interview questions

Concise, strong answers — say enough to demonstrate depth, then stop.

**Q: Walk me through what happens when I `kubectl apply -f deploy.yaml`.**
A: kubectl POSTs to the apiserver, which authenticates the request, runs it through admission controllers (mutating then validating — Pod Security admission, mutating/validating webhooks like Kyverno), and persists the Deployment object in etcd. The Deployment controller in kube-controller-manager sees the new object via its watch and creates a ReplicaSet matching the desired template. The ReplicaSet controller sees that and creates Pods. The scheduler sees Pods with no `nodeName`, picks nodes, and patches `spec.nodeName`. The kubelet on each chosen node sees a Pod bound to it, pulls the image via CRI, starts containers, and reports status back. None of these components talk to each other directly — they all coordinate through the apiserver via watches.

**Q: Difference between liveness and readiness probes?**
A: Liveness asks "is the process wedged?" — failure restarts the container. Readiness asks "should this Pod receive traffic?" — failure removes the Pod IP from the Service's EndpointSlice but doesn't restart anything. The classic mistake is treating liveness as a general health check; if the app has slow startup, liveness will restart it before it's ready, creating a CrashLoop. Use a startup probe to gate slow boots, readiness for traffic gating, and liveness only when you have specific evidence the process can wedge.

**Q: How does a Service load-balance? Why is it bad for long-lived gRPC?**
A: A `ClusterIP` Service is a virtual IP programmed by kube-proxy (or the CNI's eBPF dataplane). The kernel rewrites packets so each new connection lands on one Pod IP from the EndpointSlice. The catch: gRPC multiplexes many requests over a single long-lived HTTP/2 connection, so once the connection is established every request goes to the same Pod regardless of how many replicas exist. Solutions: a headless Service plus client-side load balancing (gRPC's built-in resolver), or a real L7 proxy (Envoy / a service mesh) in front.

**Q: StatefulSet vs Deployment — when do you use which?**
A: Deployment for stateless workloads where Pods are interchangeable. StatefulSet when Pods need stable identity (predictable name and DNS), ordered startup/shutdown, or stable per-Pod storage. Classic StatefulSet use cases: databases (Postgres, MySQL), brokers (Kafka, RabbitMQ), distributed systems with leader election. The trade-off is operational complexity — StatefulSet rollouts are slower (one Pod at a time), and storage tied to a specific Pod is a recovery concern.

**Q: How do you safely roll a node out of a cluster?**
A: `kubectl cordon <node>` to mark it unschedulable, then `kubectl drain <node> --ignore-daemonsets --delete-emptydir-data`. Drain respects PDBs, evicts Pods one at a time, waits for them to come up elsewhere. Once the node is empty, do maintenance, then `kubectl uncordon`. The drain blocks if a PDB doesn't allow disruption — that's the system working correctly. If you need to force it, your PDB config is wrong.

**Q: What is a PodDisruptionBudget, and how does it interact with cluster autoscaling?**
A: A PDB bounds *voluntary* disruption — drains and evictions, not crashes. You set `minAvailable: 2` or `maxUnavailable: 25%`, and the eviction API refuses to take a Pod down if doing so would violate the budget. With Cluster Autoscaler / Karpenter, an over-strict PDB (say `minAvailable: 100%`) will block scale-down indefinitely — the autoscaler can't evict the workload, so the node stays. Always set PDBs that allow at least one replica disruption.

**Q: Walk me through debugging a Pod stuck in Pending.**
A: `kubectl describe pod` first — the events almost always tell you why. Usual suspects: (1) no node has enough free CPU/memory matching the Pod's *requests* — fix by reducing requests or adding nodes; (2) nodeSelector / taint mismatch — check labels and tolerations; (3) PVC unbound — `kubectl get pvc`, check StorageClass and `volumeBindingMode`; (4) image pull secret missing in this namespace; (5) scheduler healthy but no node matches affinity / topology constraints. If `describe` shows no scheduling events at all, suspect the scheduler itself.

**Q: RBAC: Role vs ClusterRole — when does the cluster form matter?**
A: Use `Role` for namespace-scoped permissions (read pods in `team-a`). Use `ClusterRole` when (a) the resource is cluster-scoped (Nodes, PersistentVolumes, CRDs, ClusterRoles), (b) you want to grant the same permissions across many namespaces (define once, bind per-namespace with `RoleBinding`), or (c) the permission is genuinely cluster-wide. The `ClusterRoleBinding` form grants the ClusterRole *cluster-wide*, which is usually overkill — prefer a `RoleBinding` referencing the ClusterRole inside specific namespaces.

**Q: Ingress vs Service of type LoadBalancer — when do you need both?**
A: A LoadBalancer Service gives you one cloud LB per Service — expensive, no path/host routing, no sensible TLS termination. Ingress (or Gateway API now) sits *behind* a single LoadBalancer Service for the ingress controller and fans out to many backend Services based on host/path rules. The pattern in production: cloud LB → ingress controller (NGINX, Traefik, Envoy via Gateway API) → ClusterIP Services → Pods. Use a raw LoadBalancer Service only for non-HTTP traffic (TCP databases, UDP) or where you specifically want a dedicated LB per app.

**Q: What does OOMKilled mean, and how do you investigate?**
A: The kubelet's cgroup enforcement killed the container because it exceeded its memory limit. `kubectl describe pod` shows `Last State: Terminated, Reason: OOMKilled`. Investigation order: (1) is the limit just too low? Compare to actual usage in metrics. (2) Memory leak? Take a heap dump (Java) or pprof profile (Go) before the next OOMKill. (3) Is the runtime aware of the cgroup limit? Java needs `-XX:MaxRAMPercentage` (or `-XX:+UseContainerSupport` on JDK 8u191+); Go needs `GOMEMLIMIT` set. Workloads that look fine on a laptop but die in K8s are almost always (3).

**Q: Difference between `kubectl apply` and `kubectl create`?**
A: `create` is imperative and fails if the object exists. `apply` is declarative and uses *server-side apply* (or three-way merge in older clients) to compute the diff between your manifest, the live object, and the last-applied annotation. `apply` is what GitOps tools wrap. For "I'm running a one-off command" use `create`; for "this manifest defines the truth" use `apply`.

**Q: How do you handle secrets properly in production?**
A: Don't store them in plain ConfigMaps or check them into Git. Three real options: (1) **External Secrets Operator** + a real KMS (AWS Secrets Manager, GCP Secret Manager, Vault) — pulls into K8s Secrets at runtime; (2) **Sealed Secrets** — encrypts the Secret with a controller-held key so the encrypted form is safe to commit; (3) **Vault Agent / CSI driver** — direct Vault integration, secrets never touch etcd. Independently, enable etcd encryption-at-rest so even Secrets stored directly are at least encrypted at the storage layer.

---

## 6. Further study

Companion notes in this folder, in a sensible reading order:

1. **[CORE_CONCEPTS/](./CORE_CONCEPTS/)** — control plane and node components in detail.
2. **[WORKLOADS/](./WORKLOADS/)** — Pod, Deployment, StatefulSet, DaemonSet, Job/CronJob, init/sidecar containers, lifecycle.
3. **[CONFIGURATION/](./CONFIGURATION/)** — ConfigMaps, Secrets, env vars, resource requests/limits, Downward API.
4. **[NETWORKING/](./NETWORKING/)** — Services, Ingress, Gateway API, NetworkPolicy, CoreDNS, CNI.
5. **[STORAGE/](./STORAGE/)** — volumes, PV/PVC, StorageClass, CSI, snapshots.
6. **[OBSERVABILITY/](./OBSERVABILITY/)** — probes, logging, metrics, events, tracing.
7. **[SCHEDULING/](./SCHEDULING/)** — nodeSelector, affinity, taints, topology spread, priority, PDBs.
8. **[SECURITY/](./SECURITY/)** — authn, RBAC, ServiceAccounts, SecurityContext, Pod Security Standards, secrets management.
9. **[CLUSTER_ADMINISTRATION/](./CLUSTER_ADMINISTRATION/)** — kubeadm, upgrades, etcd backup, certificates, autoscaling.
10. **[TROUBLESHOOTING/](./TROUBLESHOOTING/)** — `kubectl debug`, Pod failures, networking issues, resource exhaustion, NotReady nodes.
11. **[HELM/](./HELM/)** — charts, templating, releases, repositories.
12. **[PACKAGING_AND_GITOPS/](./PACKAGING_AND_GITOPS/)** — Kustomize, Argo CD, Flux, CI/CD patterns.
13. **[EXTENSIBILITY/](./EXTENSIBILITY/)** — CRDs, operators, admission webhooks, finalizers, owner references.
