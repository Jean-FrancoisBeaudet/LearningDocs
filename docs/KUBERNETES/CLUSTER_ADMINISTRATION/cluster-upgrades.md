# Cluster Upgrades

> The disciplined process of moving a Kubernetes cluster from one minor version to the next without breaking workloads, exceeding the supported version-skew window, or losing the API server.

## Why it exists

Kubernetes ships **a new minor release roughly every 4 months**, and each release is supported for ~14 months. Skipping releases is permitted within the version-skew rules, but a cluster left untouched for two release cycles is on borrowed time: CVEs accumulate, deprecated APIs disappear under your feet, and managed-K8s providers force the upgrade on their schedule, not yours.

Upgrades are not just `apt install kubeadm=1.36.0` on every node. The order matters (control plane first, workers last; one CP node at a time), the version skew matters (kubelet ≤ apiserver), the deprecated APIs matter (a removed `policy/v1beta1` PDB will silently fail to apply), and the rollback story matters (there essentially isn't one for control-plane data — etcd schema and CRD conversion can be one-way). Treat every upgrade as a planned operation with a checklist, not a routine package update.

## Key concepts

- **Minor version**: the `Y` in `1.Y.Z`. Kubernetes' contract talks about *minor* version skew. Patch upgrades (`1.36.1 → 1.36.4`) are safe in any order.
- **Version skew policy** (since [KEP-3935](https://github.com/kubernetes/enhancements/issues/3935), v1.28 GA): kubelet and kube-proxy may be **up to 3 minor versions older** than `kube-apiserver`. Other apiservers (aggregated) and `kubectl` may be ±1 minor from apiserver. **Nodes must never be newer than the control plane.** This 3-version window is what enables you to upgrade the control plane through several minors before touching workers — useful for "pull the band-aid" multi-version jumps.
- **Skip-level upgrades**: `kubeadm upgrade` lets you skip *one* minor at a time on the control plane (e.g. 1.34 → 1.36 directly is not a single command, but 1.34 → 1.35 → 1.36 in two `apply` calls back-to-back works without redeploying workloads, because workers can lag).
- **Supported releases (April 2026)**: `1.34`, `1.35`, `1.36`. `1.33` enters maintenance 2026-04-28, EOL 2026-06-28. Run `kubectl version` against your cluster — if `serverVersion` is older than the second-newest supported, you are accruing exposure.
- **Deprecated → removed APIs**: each release graduates some APIs and removes others previously deprecated. `kubectl convert` (now the standalone `kubectl-convert` plugin) and `kubent` (kube-no-trouble) detect manifests using soon-to-be-removed APIs.
- **kubeadm upgrade phases**: `plan` (dry run, prints recommended next version), `apply v1.X.Y` (run on the *first* CP node — actually upgrades the control plane), `node` (run on *every other* CP and worker node — picks up new kubelet config and renews certs).
- **Package pinning**: `kubeadm`, `kubelet`, and `kubectl` are normally pinned in the package manager (`apt-mark hold` / `dnf versionlock`) to prevent an unattended `apt upgrade` from racing past your planned version.

## How it works

### The official upgrade order

```text
1. Pre-upgrade checklist (see below).
2. ETCD SNAPSHOT.  (../CLUSTER_ADMINISTRATION/etcd-backup-and-restore.md)
3. Upgrade kubeadm on first control-plane node.
4. kubeadm upgrade apply v1.X.Y      ← control plane upgrades here
5. Upgrade kubelet + kubectl on first CP node, restart kubelet.
6. For each remaining control-plane node:
     - Upgrade kubeadm
     - kubeadm upgrade node
     - Upgrade kubelet + kubectl, restart kubelet
7. For each worker node, one at a time:
     - kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
     - Upgrade kubeadm + kubelet + kubectl
     - kubeadm upgrade node
     - systemctl restart kubelet
     - kubectl uncordon <node>
8. kubectl get nodes — confirm all show the new version.
9. Smoke test workloads.
```

`kubeadm upgrade apply` is what bumps the API server, controller-manager, scheduler, etcd (if managed by kubeadm), and the *desired* kubelet config. It also auto-renews all kubeadm-managed certs in `/etc/kubernetes/pki/` — see `certificates.md`.

### What `kubeadm upgrade plan` actually shows

```bash
sudo kubeadm upgrade plan
# ...
# Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
# COMPONENT   NODE        CURRENT   TARGET
# kubelet     cp-1        v1.35.4   v1.36.1
# kubelet     worker-1    v1.35.4   v1.36.1
#
# Upgrade to the latest stable version:
# COMPONENT                CURRENT    TARGET
# kube-apiserver           v1.35.4    v1.36.1
# kube-controller-manager  v1.35.4    v1.36.1
# kube-scheduler           v1.35.4    v1.36.1
# kube-proxy               v1.35.4    v1.36.1
# CoreDNS                  v1.11.1    v1.11.3
# etcd                     3.5.15-0   3.5.18-0
```

If `plan` refuses to run because a CRD or addon mismatches, fix that before going further — `apply` will fail in the same place but mid-upgrade is a worse place to be stuck.

### The pre-upgrade checklist

Run **all** of these *before* touching any binaries:

```bash
# 1. Find workloads using deprecated/removed APIs.
#    kubent (kube-no-trouble) reads live cluster + Helm releases.
kubent --target-version 1.36

# 2. Confirm CRDs are at v1 and have a stored conversion strategy if any version was deprecated.
kubectl get crd -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.versions[*].name}{"\n"}{end}'

# 3. Verify all admission webhooks have a still-valid TLS cert and at least one healthy endpoint.
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.webhooks[*].clientConfig.service.name}{"\n"}{end}'

# 4. PDBs in place for everything you care about. A PDB-less StatefulSet is about to take an outage.
kubectl get pdb -A

# 5. Pin a known-good etcd snapshot.
sudo etcdctl snapshot save /backup/etcd-pre-upgrade-$(date +%F).db ...

# 6. Pre-pull the new images on every node to cut the maintenance window.
sudo kubeadm config images pull --kubernetes-version v1.36.1
```

Skipping `kubent` is the most common cause of "upgrade succeeded, half my CI deployments now fail" surprises.

### Worker-node loop, in detail

```bash
NODE=worker-3
TARGET=1.36.1-1.1   # apt package version, includes the kubeadm package revision

kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data --timeout=10m

ssh $NODE 'sudo apt-mark unhold kubeadm && \
           sudo apt-get update && sudo apt-get install -y kubeadm='"$TARGET"' && \
           sudo apt-mark hold kubeadm'
ssh $NODE 'sudo kubeadm upgrade node'
ssh $NODE 'sudo apt-mark unhold kubelet kubectl && \
           sudo apt-get install -y kubelet='"$TARGET"' kubectl='"$TARGET"' && \
           sudo apt-mark hold kubelet kubectl'
ssh $NODE 'sudo systemctl daemon-reload && sudo systemctl restart kubelet'

kubectl uncordon $NODE
kubectl wait --for=condition=Ready node/$NODE --timeout=5m
```

`--ignore-daemonsets` is mandatory — DaemonSets like CNI agents and `kube-proxy` *must* keep running on the node to drain anything else. `--delete-emptydir-data` deletes the contents of any `emptyDir` volume; if a workload depends on emptyDir data surviving a node drain, it was misdesigned anyway.

### Addon and CNI upgrades

`kubeadm upgrade apply` also bumps **CoreDNS** and **kube-proxy** (these ship as kubeadm-managed manifests). It does **not** touch your CNI — Calico / Cilium / Flannel are upgraded independently using their own helm charts or YAML manifests, on whatever cadence their compatibility matrix supports. Always check the CNI's K8s compatibility matrix before bumping the control plane past it.

### Managed K8s upgrade patterns

| Provider | Control-plane upgrade | Worker-pool upgrade |
|---|---|---|
| EKS | `aws eks update-cluster-version` (single-shot, ~30 min) | Replace node group / use Karpenter to drain-and-reprovision |
| AKS | `az aks upgrade --control-plane-only` then `--node-image-only` | Surge upgrade or per-pool `--node-pool` |
| GKE | Auto-upgrade by release channel (rapid/regular/stable) or manual | Surge upgrade or blue-green node pools |

The pattern is identical: **upgrade the control plane, then roll workers blue-green** by creating a new node pool / NodePool at the new version, draining the old. This avoids the in-place package dance entirely and gives you instant rollback (just keep the old node pool around for a day).

## Minimal example

End-to-end, single control-plane kubeadm cluster, 1.35.4 → 1.36.1:

```bash
# --- on the control plane node ---
sudo etcdctl snapshot save /backup/pre-1.36.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

sudo apt-mark unhold kubeadm && sudo apt-get install -y kubeadm=1.36.1-1.1 && sudo apt-mark hold kubeadm

sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.36.1 --yes

sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.36.1-1.1 kubectl=1.36.1-1.1
sudo apt-mark hold kubelet kubectl
sudo systemctl daemon-reload && sudo systemctl restart kubelet

# --- from any kubectl ---
kubectl get nodes        # CP shows v1.36.1, workers still v1.35.4 — fine
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data
# (run the worker package upgrade + 'kubeadm upgrade node' + restart kubelet on worker-1)
kubectl uncordon worker-1
# repeat for every worker
kubectl get nodes        # all v1.36.1
```

## Common patterns

- **One-minor-at-a-time on the control plane**: even though the version-skew policy permits workers to lag 3 minors, kubeadm still requires the *control plane* to step one minor per `apply`. If you are 3 minors behind, plan three back-to-back `apply` calls.
- **Blue-green worker pools (managed K8s)**: provision a new node group at v1.X+1, taint the old pool `NoSchedule`, drain it. Zero in-place upgrades, instant rollback by reverting the taint. Pairs naturally with Karpenter — see `cluster-autoscaling.md`.
- **Surge upgrade on AKS / GKE**: configure `maxSurge: 33%` so upgrades add temporary capacity instead of evicting and waiting. Faster, but costs extra node-hours during the window.
- **Upgrade-as-cert-rotation**: timing the upgrade to your cert renewal window means `kubeadm upgrade apply` rotates everything — see `certificates.md`.
- **Canary cluster**: in larger orgs, run a "canary" cluster one minor ahead of prod with the same workloads. Catches deprecated-API and CNI compatibility issues a week before they bite production.

## Senior-level gotchas

- **`kubectl convert` is a separate binary now.** It moved out of `kubectl` years ago. Install `kubectl-convert` (apt/dnf package) explicitly before you depend on it during a maintenance window.
- **Removed APIs do not auto-migrate stored objects.** If a CRD's `v1beta1` is removed in 1.36 and you stored objects under `v1beta1` years ago, the apiserver after upgrade cannot serve those objects. Set the CRD's `storage: true` to a still-served version *before* upgrading and run `kubectl get <crd> -A -o yaml > rewrite.yaml; kubectl apply -f rewrite.yaml` to force a re-store under the new storage version.
- **CRD conversion webhooks must be online during the upgrade.** If the conversion webhook for a CRD is itself deployed inside the cluster (often the case with operators), and that pod is evicted during a control-plane upgrade, the apiserver may fail to read its own bootstrap CRDs. Drain CRD-conversion-webhook hosts last, or scale the webhook to multiple replicas with anti-affinity.
- **`kubeadm upgrade apply` upgrades etcd's image.** If you run external etcd (not managed by kubeadm), you must upgrade etcd separately and pass `--etcd-upgrade=false`. Forgetting to do so on an external-etcd cluster leaves you with mismatched etcd images on the kubeadm node and your real etcd cluster — confusing and harmless, but it hides the fact that real etcd never actually got patched.
- **kube-proxy is upgraded by `kubeadm upgrade apply`, but only if you didn't replace it.** Many Cilium / Calico setups disable kube-proxy entirely. In that case kubeadm tries to install kube-proxy as part of `apply` and you have to pass `--skip-phases=addon/kube-proxy` or it overwrites your CNI-only proxy assumption.
- **The version-skew policy is a *ceiling*, not a goal.** Workers running 3 minors behind the control plane technically work, but they don't get bug fixes for that long, and the matrix of behaviors gets weird (e.g. a Pod scheduled by a 1.36 scheduler hits a 1.33 kubelet that doesn't understand a new field — silently ignored). Aim to keep nodes within 1 minor in normal operation.
- **DaemonSet rollouts during upgrade can flatline cluster networking.** Bumping Cilium / Calico version *during* a control-plane upgrade is a recipe for "no node is Ready and I cannot tell why." Sequence: control-plane upgrade → wait for stable → CNI upgrade → wait for stable → worker upgrade.
- **"Rollback" mostly means "etcd restore."** kubeadm has no `downgrade` command. Once `apply v1.36.1` writes new manifests and runs migrations, the only way back is restoring the pre-upgrade etcd snapshot and re-installing the old `kubeadm`/`kubelet` packages — which is why step 2 of the checklist is "etcd snapshot." Test that you can actually restore it.
- **Cloud LB health checks fail during apiserver restart.** If the cluster apiserver is behind a single cloud LB, the rolling restart during `kubeadm upgrade apply` can take it out of rotation briefly. Multi-CP nodes mitigate this; a true HA control plane (3+ apiservers behind LB with `--watch-reconnect-timeout` clients) makes the upgrade invisible.
- **Managed K8s "auto-upgrade" can blow your maintenance window.** GKE/AKS can auto-bump the control plane while you sleep. If you have not implemented PDBs or your workloads cannot survive a kubelet restart, disable auto-upgrade and own the cadence.

## Related

- `certificates.md` — `kubeadm upgrade apply` auto-renews all kubeadm-managed certs as a side effect.
- `etcd-backup-and-restore.md` — the *only* real rollback mechanism; snapshot before every upgrade.
- `node-maintenance.md` — `cordon` / `drain` / `uncordon` mechanics used in the worker loop.
- `kubeadm.md` — the bootstrap path that determined what `kubeadm upgrade` will manage on disk.
- `cluster-autoscaling.md` — Karpenter-driven blue-green worker pool upgrades; how Cluster Autoscaler interacts with cordoned nodes.
- `../SCHEDULING/pod-disruption-budgets.md` — drains respect PDBs; misconfigured PDBs are the #1 cause of upgrade-stuck-on-drain.
- `../SCHEDULING/priority-and-preemption.md` — high-priority pods may bypass PDBs during preemption.
- `../EXTENSIBILITY/custom-resource-definitions.md` — CRD versioning, storage version, conversion webhooks.
- `../EXTENSIBILITY/admission-webhooks.md` — webhooks that are scheduled on nodes being upgraded need anti-affinity.
- `../OBSERVABILITY/events.md` — `kubectl get events --sort-by=.lastTimestamp` is your live upgrade dashboard.
