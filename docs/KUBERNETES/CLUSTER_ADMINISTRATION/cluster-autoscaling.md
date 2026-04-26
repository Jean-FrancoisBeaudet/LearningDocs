# Cluster Autoscaling

> Automatically growing and shrinking the **node pool** (cluster-level capacity) in response to pending pods and node utilization — distinct from pod-level autoscalers like HPA, VPA, and KEDA, which scale workloads inside a fixed-size cluster.

## Why it exists

Kubernetes' scheduler is excellent at placing pods on nodes, but it cannot *create* nodes. If every node is full and a new pod is scheduled, the pod sits in `Pending` state with `FailedScheduling` events forever — unless something else notices and adds a node. That "something else" is a **cluster autoscaler**.

Two operational shapes drive the need:
1. **Bursty workloads**: CI runners, batch jobs, data pipelines. The cluster has near-zero usage at night and 100× peak at noon. Paying for peak capacity 24/7 is wasteful.
2. **Cost optimization**: even steady workloads pack inefficiently as deployments come and go. Reactively right-sizing the fleet — picking the cheapest instance type that fits the next pod, consolidating idle nodes — saves 30–60% on node bills in practice.

The two dominant tools today are **Cluster Autoscaler** (the original SIG Autoscaling project, node-group-shaped) and **Karpenter** (newer, instance-shaped, generally faster and cheaper). Managed offerings (GKE Autopilot, EKS Auto Mode, EKS Fargate, AKS node autoprovisioning) are increasingly built on top of Karpenter or the same primitives.

## Key concepts

- **Node group / NodePool**: a logical pool of nodes sharing instance type, labels, taints, and zone. Cluster Autoscaler scales node *groups* (one ASG / MIG per group). Karpenter v1 scales by emitting individual instances that match a `NodePool`.
- **Pending pod signal**: both autoscalers react to pods that fail scheduling for *resource* reasons (`FailedScheduling: 0/N nodes are available: insufficient cpu`). They ignore pods that are unschedulable for other reasons (taints, PVC-not-bound, affinity).
- **Bin-packing**: choosing instance sizes that minimize wasted capacity. Cluster Autoscaler picks from pre-defined ASG types ("scale up the m5.4xlarge group"). Karpenter picks the cheapest instance from a *menu* of allowed types per pending pod set ("just-in-time selection").
- **Consolidation / scale-down**: removing or replacing under-utilized nodes. Cluster Autoscaler removes when utilization < `--scale-down-utilization-threshold` for `--scale-down-unneeded-time`. Karpenter consolidates by *replacing* under-utilized nodes with smaller cheaper ones when it can — a stronger signal than "delete if empty."
- **Disruption controls**: `PodDisruptionBudget`, `safe-to-evict` annotations, `do-not-disrupt` annotations, `terminationGracePeriodSeconds`. The autoscaler must respect these or it will evict pods that should not move.
- **Karpenter v1 (GA Aug 2024)**: introduced `NodePool` and `NodeClass` CRDs (replacing `Provisioner` and `AWSNodeTemplate`). Vendor-neutral core lives at `kubernetes-sigs/karpenter`; cloud providers (AWS, Azure, OCI, on-prem via cluster-api) implement their own `NodeClass` shape.
- **Cloud provider integration**: Cluster Autoscaler uses one of ~30 cloud-provider drivers (AWS, GCP, Azure, OpenStack, etc.) compiled in. Karpenter delegates to a per-cloud `NodeClass` controller.

## How it works

### Cluster Autoscaler — control loop

Every `--scan-interval` (default 10s):

```text
┌───────────────────────────────────┐
│ List Pending pods (FailedSchedule)│
└──────────────┬────────────────────┘
               │
               ▼
       ┌─────────────────┐         scale-up
       │ Simulate adding │── if simulated  ─► increase node-group desired
       │ a node from each│   pod fits        capacity by N
       │ node group      │   in some group
       └────────┬────────┘
                │
                ▼
       ┌─────────────────┐
       │ List nodes that │── if utilization
       │ have been under │   below threshold
       │ utilization for │   for unneeded-time
       │ unneeded-time   │
       └────────┬────────┘                   scale-down
                │                             ─────────
                ▼                            cordon, drain,
        For each candidate node:             then ask cloud
          - check PDBs / safe-to-evict       to terminate
          - simulate moving its pods
          - drain if safe
```

Critical flags:

```bash
--scale-down-utilization-threshold=0.5    # default 0.5; node "unneeded" below this
--scale-down-unneeded-time=10m            # how long unneeded before removal
--scan-interval=10s                       # main loop frequency
--max-node-provision-time=15m             # if a requested node never joins, give up
--expander=least-waste                    # which node group to grow when several fit
```

`--expander` strategies: `random`, `most-pods`, `least-waste` (smallest leftover capacity), `price` (cheapest), `priority` (per-group `cluster-autoscaler.kubernetes.io/priority` annotation). `least-waste` and `priority` together cover most production setups.

### Karpenter — JIT provisioning model

Karpenter sidesteps node groups entirely. You define the menu of *allowed* instance shapes via a `NodePool`, and on every pending pod Karpenter computes the cheapest instance that fits:

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: [amd64]
        - key: karpenter.sh/capacity-type
          operator: In
          values: [on-demand, spot]            # mix; Karpenter prefers spot when cheaper
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: [c, m, r]                    # Compute, balanced, memory-optimized
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["5"]                        # only m6 and newer; m5 is deprecated
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      taints:
        - key: workload
          value: batch
          effect: NoSchedule                   # only batch jobs land here
      expireAfter: 720h                        # max node lifetime: 30 days
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 30s
    budgets:
      - nodes: 10%                             # never disrupt more than 10% of capacity at once
  limits:
    cpu: 1000                                  # safety cap on the pool's total CPU
```

The `NodeClass` (cloud-specific) holds the AMI, subnet selectors, security groups, IAM role:

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  amiSelectorTerms:
    - alias: al2023@latest                     # auto-track latest EKS-optimized AMI
  subnetSelectorTerms:
    - tags: { karpenter.sh/discovery: "prod" }
  securityGroupSelectorTerms:
    - tags: { karpenter.sh/discovery: "prod" }
  role: KarpenterNodeRole-prod
```

Provisioning end-to-end takes **~30–60 seconds** in AWS EC2 vs ~3–5 minutes for Cluster Autoscaler scaling an ASG, because Karpenter calls the EC2 API directly instead of asking ASG to scale.

### CA vs Karpenter — when to pick which

| | Cluster Autoscaler | Karpenter |
|---|---|---|
| **Provisioning model** | Scales pre-defined node groups (ASG/MIG) | JIT-provisions any allowed instance type |
| **Speed** | Minutes (ASG round-trip) | Seconds (direct EC2 API) |
| **Bin-packing** | "Pick a node group" | "Pick the cheapest instance for *these* pending pods" |
| **Spot handling** | Per-ASG; spot interruption requires extra controllers | Built-in spot-aware, auto-replaces interrupted nodes |
| **Cloud support** | ~30 providers; mature | AWS, Azure, OCI, GKE, on-prem (cluster-api); newer |
| **Config surface** | Many flags, per-cloud | Two CRDs (`NodePool`, `NodeClass`) |
| **Best for** | Multi-cloud, GCP/Azure where Karpenter is less mature, "we already have ASGs" | AWS-heavy, cost-driven, bursty workloads |

Default for greenfield AWS: Karpenter. Default for established multi-cloud or non-AWS: Cluster Autoscaler. Both can coexist in the same cluster (Karpenter on a tainted pool, CA elsewhere) but it's rarely worth the complexity.

### Managed equivalents

- **GKE Autopilot**: Google fully manages the node layer; you specify pod resources, Google picks instances. Built on the same primitives.
- **EKS Auto Mode** (2024+): EKS-managed Karpenter — no Karpenter install/upgrade for you, but you pay a small premium and lose some configurability.
- **EKS Fargate**: per-pod VMs, no node concept. Great for tiny serverless-ish workloads, expensive at scale, no GPU/local-storage workloads.
- **AKS node autoprovisioning** (preview→GA in recent releases): Azure's Karpenter-based equivalent.

### Interaction with HPA / pod priority / PDBs

The pieces compose: HPA scales the pod replicas, the new pods become Pending, the cluster autoscaler adds nodes. *Order of operations* matters during a scale-down — autoscalers respect `PodDisruptionBudget`s and the eviction API, so a misconfigured PDB (`minAvailable: 100%`) makes nodes look "stuck draining" forever. See `../SCHEDULING/pod-disruption-budgets.md`.

Pods that should never be evicted by a scale-down get the **`cluster-autoscaler.kubernetes.io/safe-to-evict: "false"`** annotation (Cluster Autoscaler) or **`karpenter.sh/do-not-disrupt: "true"`** (Karpenter). Use this for stateful pods that are mid-rebalance or jobs you don't want to lose.

Pod priority interacts in two ways:
- **High-priority pods** can preempt lower-priority pods to schedule, so they trigger scale-up only if there's nothing to preempt.
- The autoscaler's own `--expendable-pods-priority-cutoff` (default `-10`) defines what counts as "expendable" — pods below that priority are not considered when deciding to keep a node.

## Minimal example

Tag the subnets and security groups, install Karpenter via Helm, then apply this — the cluster will provision nodes for the next batch job within a minute.

```yaml
# nodepool-batch.yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: batch
spec:
  template:
    metadata:
      labels:
        workload: batch
    spec:
      requirements:
        - { key: kubernetes.io/arch,           operator: In, values: [amd64] }
        - { key: karpenter.sh/capacity-type,   operator: In, values: [spot] }   # batch tolerates interruption
        - { key: karpenter.k8s.aws/instance-family, operator: In, values: [c7i, c7a, m7i, m7a] }
      taints:
        - { key: workload, value: batch, effect: NoSchedule }
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      expireAfter: 168h           # weekly node refresh — picks up new AMIs without manual upgrade
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 60s
    budgets:
      - nodes: "20%"
  limits:
    cpu: 2000
    memory: 8000Gi
---
# A Job that targets the batch pool.
apiVersion: batch/v1
kind: Job
metadata:
  name: nightly-aggregate
spec:
  ttlSecondsAfterFinished: 86400
  template:
    spec:
      restartPolicy: Never
      tolerations:
        - { key: workload, operator: Equal, value: batch, effect: NoSchedule }
      nodeSelector:
        workload: batch
      containers:
        - name: aggregate
          image: registry.example.com/aggregate:1.4.2
          resources:
            requests: { cpu: "4",   memory: 8Gi }
            limits:   { cpu: "4",   memory: 8Gi }   # Guaranteed QoS
```

What happens: the Job's pod becomes `Pending`. Karpenter sees it, picks the cheapest spot instance from `c7i/c7a/m7i/m7a` that fits 4 CPU + 8 GiB, launches it (~45s), the pod schedules, runs, and Karpenter consolidates the empty node 60s after the Job completes.

## Common patterns

- **Spot pool + on-demand pool, with priority**: tolerate-spot deployments target a spot NodePool with high preferred weight; same workload tolerates an on-demand pool as fallback. Karpenter's `karpenter.sh/capacity-type` mixed in one pool is even simpler — it prefers spot but falls back to on-demand if spot is unavailable.
- **GPU pool with taint**: a `NodePool` with GPU instance families and `nvidia.com/gpu` taint. Only pods that tolerate it land there, preventing accidental general workloads from booking $3/hr GPU nodes.
- **Multi-arch graviton/x86**: requirements include `kubernetes.io/arch In [amd64, arm64]`. Karpenter picks ARM if any pod's image supports it (multi-arch manifest), which is typically 20–30% cheaper.
- **Per-node-pool budgets**: `disruption.budgets` caps how many nodes Karpenter can disrupt at once during consolidation. Pair with `disruption.budgets[].schedule` to forbid disruptions during business hours.
- **Cluster Autoscaler `priority` expander for fallback**: define a "cheap small" group with priority 100 and a "big expensive" group with priority 10. CA grows the small group first, falls back to the big group only when the small one's quota is exhausted.

## Senior-level gotchas

- **Pending pods that don't actually need a node get ignored.** A pod stuck Pending due to an unbound PVC, a missing taint toleration, or affinity that no node can satisfy will never trigger scale-up — it's not a *capacity* problem. Read the pod's `Events`: if you see "didn't match Pod's node affinity" instead of "Insufficient cpu", no autoscaler will help.
- **`safe-to-evict: false` is sticky.** Pods with this annotation block scale-down of *their entire node*, even when 95% of the node is empty. People set it on cron-like jobs and forget. Audit periodically: `kubectl get pods -A -o jsonpath='{range .items[?(@.metadata.annotations.cluster-autoscaler\.kubernetes\.io/safe-to-evict=="false")]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}'`.
- **PDB `minAvailable: 100%` deadlocks scale-down.** A 1-replica Deployment with `minAvailable: 1` is *un-evictable*. The autoscaler retries forever, the node sits there costing money, you find out in the bill. Either use `maxUnavailable: 1` semantics (replicas ≥ 2 and `minAvailable: replicas-1`) or annotate the deployment with `safe-to-evict: true` and accept brief downtime.
- **Karpenter and DaemonSet sizing.** `NodePool.requirements` constrains the *instance*, but each instance also has to fit your DaemonSet pods (CNI, log shipper, monitoring) before it has any room for application pods. A `c7i.large` (2 vCPU) where your DaemonSets request 0.7 vCPU each leaves 0.3 vCPU usable. Either filter to instances with enough headroom (`instance-cpu Gt 4`) or audit DaemonSet requests.
- **`expireAfter` is mandatory in production.** Without it, Karpenter nodes can live for months without picking up the latest AMI — including critical CVE patches. `expireAfter: 720h` (30 days) is a reasonable default; combine with PDBs to make the rolling AMI refresh non-disruptive. This effectively replaces "node OS patching" as a separate workflow.
- **Cluster Autoscaler does not balance across zones automatically.** With a single ASG spanning AZs, AWS may scale all new nodes into one AZ, breaking topology spread. Either run one ASG per AZ (and let CA's `balance-similar-node-groups=true` distribute) or use Karpenter, which is zone-aware natively.
- **Spot interruption ≠ scale-down.** Cluster Autoscaler does *not* react to AWS spot interruption notices. You need a separate `aws-node-termination-handler` (or similar) to drain a node when AWS gives the 2-minute warning. Karpenter handles this natively via the AWS interruption queue.
- **`do-not-disrupt` blocks Karpenter, not the kernel.** The annotation prevents Karpenter-initiated termination, but a spot reclaim, a hardware failure, or a manual `kubectl delete node` will still happen. Workloads that "must never move" need to actually be replicated, not just annotated.
- **Autopilot / Auto Mode hides the controls you'd want during incidents.** When everything's fine, GKE Autopilot or EKS Auto Mode is great. When you need to pin a workload to a specific instance type to debug a noisy-neighbor issue, you find that you can't. Pick managed-mode based on team capacity, not raw cost — once you outgrow it, migration is non-trivial.
- **HPA + Cluster Autoscaler oscillation.** HPA scales pods on CPU, autoscaler adds nodes for the new pods, the new pods bring the average CPU down, HPA scales pods *down*, autoscaler removes the new node, average CPU rises, HPA scales up... Use `HPA.behavior.scaleDown.stabilizationWindowSeconds` (300s+) and a multi-minute `--scale-down-unneeded-time` to break the loop.
- **Init containers count for scale-up sizing, but only briefly.** A pod with a 2 GiB init container and a 200 MiB app container needs a node with 2 GiB free at admission, even though it'll only use 200 MiB after init. Cluster Autoscaler handles this; some older versions did not, leading to over-provisioning. Watch for this if you write big init containers.

## Related

- `cluster-upgrades.md` — Karpenter NodePools naturally implement blue-green worker upgrades by bumping `nodeClassRef` to a new AMI.
- `node-maintenance.md` — `cordon`/`drain` are the actions autoscalers take during scale-down.
- `../SCHEDULING/pod-disruption-budgets.md` — PDBs gate every eviction the autoscaler issues.
- `../SCHEDULING/priority-and-preemption.md` — pod priority changes whether a Pending pod actually triggers scale-up.
- `../SCHEDULING/taints-and-tolerations.md` — separating GPU/spot/batch pools by taint is the canonical autoscaler-friendly pattern.
- `../SCHEDULING/affinity-and-anti-affinity.md` — affinity rules can starve scale-up if no available instance shape can satisfy them.
- `../SCHEDULING/pod-topology-spread-constraints.md` — multi-AZ spread interacts with how the autoscaler distributes new nodes.
- `../WORKLOADS/pods.md` — resource requests are the *only* signal the autoscaler trusts for capacity sizing.
- `../CONFIGURATION/resource-requests-and-limits.md` — under-requested pods cause both noisy-neighbor and autoscaler under-provisioning.
- *Pod-level autoscaling* (HPA, VPA, KEDA) — distinct concern; lives at the workload layer, not the node layer. No section files yet in this scaffold; would belong under `../WORKLOADS/` or a future `AUTOSCALING/` topic folder.
