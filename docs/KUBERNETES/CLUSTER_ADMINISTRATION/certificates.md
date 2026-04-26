# Certificates

> The TLS PKI that secures every connection between Kubernetes control-plane components, kubelets, and clients — plus the rotation discipline that keeps it from expiring under your feet.

## Why it exists

Every conversation inside a Kubernetes cluster crosses a network. The API server talks to etcd, the controller manager talks to the API server, kubelets talk to the API server (and back, for `kubectl exec`/`logs`), and human operators talk through `kubectl`. Without TLS and mutual authentication, any of those paths is a takeover vector — etcd alone holds every secret, every ServiceAccount token, and every cluster manifest.

Kubernetes solves this with several **independent CA hierarchies** issued (by default, on kubeadm clusters) at `kubeadm init` and stored in `/etc/kubernetes/pki/`. The CAs sign client and server certificates for each component, and components mutually authenticate on every connection. The hard part is not setting it up — kubeadm does that — it is **noticing that the certs expire one year later** and rotating them before the cluster goes dark.

## Key concepts

- **CA hierarchy**: kubeadm creates **three separate CAs**: `ca` (the cluster CA, signs control-plane and kubelet certs), `front-proxy-ca` (signs the aggregation-layer proxy client cert), and `etcd/ca` (signs etcd peer/server/client certs). Keeping etcd's CA separate limits blast radius if a non-etcd component key leaks.
- **Default validity**: leaf certs issued by kubeadm = **1 year**; CAs = **10 years**. The 1-year window is by design — it forces operators to upgrade or rotate annually.
- **Static-pod certs vs kubeconfig credentials**: control-plane components run as static pods (`/etc/kubernetes/manifests/*.yaml`) and read certs directly from `/etc/kubernetes/pki/`. Components that talk *to* the API server (controller-manager, scheduler, kubelets) authenticate using a **kubeconfig file** that embeds a client cert + key. Both kinds expire. Both are rotated by `kubeadm certs renew`.
- **kubelet client cert vs serving cert**: the kubelet has two cert identities. The **client** cert (in `/var/lib/kubelet/pki/kubelet-client-current.pem`) authenticates *to* the API server. The **serving** cert (`kubelet.crt` or `kubelet-server-current.pem`) is presented *by* the kubelet when the API server connects back (e.g. for `kubectl logs`). Both can auto-rotate via CSR — see "How it works".
- **CSR API**: the `certificates.k8s.io/v1` `CertificateSigningRequest` resource. Kubelets submit CSRs to renew their own certs; an approver controller (or `kubectl certificate approve`) signs them. This is why kubelet certs need no `kubeadm` intervention.
- **`kubeadm certs renew`**: re-signs leaf certs in `/etc/kubernetes/pki/` and the embedded creds in `/etc/kubernetes/{admin,super-admin,controller-manager,scheduler}.conf` using the existing CA. Does **not** touch `kubelet.conf` (kubelet auto-rotates instead).
- **Restart-after-renewal**: not all components reload certs from disk on change. After `kubeadm certs renew`, you must restart the affected static pods (delete the pod files briefly, or `crictl rm` the containers) for them to pick up the new cert. The exception is the kubelet, which does dynamic reload of its own client/serving certs.

## How it works

### Inventory: what kubeadm puts on disk

```text
/etc/kubernetes/pki/
├── ca.{crt,key}                       # cluster CA
├── apiserver.{crt,key}                # apiserver TLS serving cert (signed by ca)
├── apiserver-kubelet-client.{crt,key} # apiserver -> kubelet client auth
├── apiserver-etcd-client.{crt,key}    # apiserver -> etcd client auth
├── front-proxy-ca.{crt,key}           # aggregation-layer CA
├── front-proxy-client.{crt,key}       # aggregator client auth
├── sa.{key,pub}                       # ServiceAccount JWT signing key pair (NOT a cert)
└── etcd/
    ├── ca.{crt,key}                   # etcd CA (separate trust root)
    ├── server.{crt,key}               # etcd serving cert
    ├── peer.{crt,key}                 # etcd-to-etcd peer auth
    └── healthcheck-client.{crt,key}   # etcdctl health probes
```

`sa.key`/`sa.pub` is the signer for ServiceAccount tokens, **not** an X.509 cert — it has no expiry and rotating it invalidates every existing token in the cluster. Treat it as long-lived.

The four kubeconfig files in `/etc/kubernetes/` (`admin.conf`, `super-admin.conf` (1.29+), `controller-manager.conf`, `scheduler.conf`) embed base64-encoded client certs that also expire after 1 year.

### Inspect what you have

```bash
kubeadm certs check-expiration
# CERTIFICATE                EXPIRES                  RESIDUAL TIME   ...
# admin.conf                 Apr 25, 2027 14:02 UTC   364d            ca
# apiserver                  Apr 25, 2027 14:02 UTC   364d            ca
# apiserver-etcd-client      Apr 25, 2027 14:02 UTC   364d            etcd-ca
# ...
# CERTIFICATE AUTHORITY      EXPIRES                  RESIDUAL TIME
# ca                         Apr 23, 2036 14:02 UTC   3650d
```

For a one-off look at a single file:

```bash
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates -subject -issuer
```

### Rotation: the safe sequence

Run on **every control-plane node**, one at a time:

```bash
# 1. Snapshot etcd first — see ../CLUSTER_ADMINISTRATION/etcd-backup-and-restore.md
# 2. Rotate everything kubeadm manages
sudo kubeadm certs renew all

# 3. Restart static-pod control-plane components so they reload from disk.
#    Bouncing the manifest file is the simplest trick:
sudo bash -c 'mv /etc/kubernetes/manifests /tmp/manifests-bak && sleep 20 && mv /tmp/manifests-bak /etc/kubernetes/manifests'

# 4. Refresh your local kubeconfig with the new admin cert
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 5. Verify
kubeadm certs check-expiration
kubectl get nodes
```

The kubelet does **not** need any action — it has been auto-rotating its client cert under `/var/lib/kubelet/pki/` since `--rotate-certificates=true` (default since 1.19).

### Kubelet auto-rotation, in one diagram

```text
kubelet startup
  │
  ▼
read /var/lib/kubelet/pki/kubelet-client-current.pem  (symlink to a dated cert)
  │
  ▼  (when remaining lifetime < 30%)
build CertificateSigningRequest
  │
  ▼
POST  certificates.k8s.io/v1 CSR  ──► kube-controller-manager (csrapproving + csrsigning)
  │                                  │
  ▼                                  ▼
new certificate written, symlink   approved & signed by cluster CA
flipped, in-memory client refreshed
```

Key flags on the kubelet (set via `KubeletConfiguration`, not CLI, in modern setups):

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
rotateCertificates: true            # client cert auto-renewal (default)
serverTLSBootstrap: true            # request a serving cert via CSR (default false)
```

`serverTLSBootstrap: true` is what stops you from being stuck with the self-signed serving cert kubelets generate by default. You will need an approver — RKE2 / kubeadm setups commonly run [`kubelet-csr-approver`](https://github.com/postfinance/kubelet-csr-approver) to auto-approve serving CSRs that match the node's hostname.

### Common ports vs cert pairings

| From | To | Port | Server cert | Client cert |
|---|---|---|---|---|
| `kubectl` / controllers | apiserver | 6443 | `apiserver.crt` (CN/SAN must include the LB) | `admin.conf` / SA token / kubeconfig client cert |
| apiserver | kubelet | 10250 | kubelet serving cert | `apiserver-kubelet-client.crt` |
| apiserver | etcd | 2379 | `etcd/server.crt` | `apiserver-etcd-client.crt` |
| etcd peer | etcd peer | 2380 | `etcd/peer.crt` | `etcd/peer.crt` |
| aggregated apiserver | extension apiserver | varies | extension's serving cert | `front-proxy-client.crt` |

If a connection mysteriously fails after a rename, an LB swap, or a rebuild, **read the SANs** on the relevant serving cert before assuming it is a routing problem.

## Minimal example

### Renew a single component without renewing everything

```bash
sudo kubeadm certs renew apiserver-kubelet-client
sudo kubeadm certs check-expiration | grep apiserver-kubelet-client
# then restart kube-apiserver:
sudo crictl ps --name kube-apiserver -q | xargs sudo crictl stop
```

### Generate a kubeconfig for a new human user (signed by the cluster CA)

```bash
NAME=alice
GROUP=devs
# 1. Generate a key + CSR locally (the user keeps the key)
openssl req -new -nodes -newkey rsa:2048 \
  -keyout $NAME.key -out $NAME.csr \
  -subj "/CN=$NAME/O=$GROUP"

# 2. Submit the CSR through the K8s CSR API so an admin can approve it
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: $NAME-csr
spec:
  request: $(base64 -w0 < $NAME.csr)
  signerName: kubernetes.io/kube-apiserver-client    # signs against the cluster CA
  expirationSeconds: 31536000                        # 1y; subject to apiserver --cluster-signing-duration
  usages: [client auth]
EOF

# 3. Approve and download
kubectl certificate approve $NAME-csr
kubectl get csr $NAME-csr -o jsonpath='{.status.certificate}' | base64 -d > $NAME.crt

# 4. Bind a Role in K8s — see ../SECURITY/authorization-rbac.md
```

This pattern (CSR API + RBAC) is the right way to onboard humans to a small cluster. For larger fleets, use OIDC instead so you do not have to rotate per-user certs by hand.

## Common patterns

- **Annual upgrade as cert rotation**: schedule the K8s minor-version upgrade *just before* certs expire — `kubeadm upgrade apply` auto-renews all kubeadm-managed certs as a side effect, killing two chores with one maintenance window. See `cluster-upgrades.md`.
- **Pre-expiry alerting**: scrape `kube-apiserver` metric `apiserver_client_certificate_expiration_seconds_bucket` (the histogram of *client* cert expirations seen by the apiserver) and alert at ~30 days. For the apiserver's own serving cert, parse `kubeadm certs check-expiration` from a CronJob or use `kube-state-metrics`'s certificate exporter.
- **External CA mode**: at `kubeadm init` time, pass `--feature-gates=...` and pre-place an externally generated CA *cert* (no key) under `/etc/kubernetes/pki/`. kubeadm then expects you to sign all leaf CSRs externally (HSM, internal PKI). Painful — only choose this if compliance forces you to keep CA keys off the cluster.
- **Auto-approving kubelet serving CSRs**: deploy `kubelet-csr-approver` so `serverTLSBootstrap: true` actually produces signed serving certs in production. Without it, CSRs sit in `Pending` and `kubectl logs` silently breaks for new nodes.

## Senior-level gotchas

- **Renewal does not restart pods.** `kubeadm certs renew all` writes new files, but `kube-apiserver`, `controller-manager`, and `scheduler` only read certs at startup. If you skip the restart step, everything *looks* fine until something else triggers a restart weeks later — at which point you have already forgotten the rotation, and a stale-cert mismatch sends you debugging the wrong layer.
- **`kubelet.conf` is on a different rotation track.** It does *not* show up in `kubeadm certs renew` output because kubelet auto-rotates. If a kubelet's client cert *is* expired (e.g. node was offline > 1 year), the auto-rotation cannot bootstrap because it needs a valid client to talk to the API server. Fix: regenerate `kubelet.conf` with `kubeadm kubeconfig user --client-name=system:node:$(hostname) --org=system:nodes` and drop it on the node.
- **etcd has its own CA.** Rotating the cluster CA does not rotate etcd certs and vice versa. If you migrate etcd or add a new etcd member, regenerate `etcd/peer` and `etcd/server` certs explicitly, otherwise the new member is rejected on join.
- **The apiserver SANs are immovable post-init.** `apiserver.crt` includes SANs for the cluster's load-balancer DNS, the service IP `kubernetes.default.svc`, and each control-plane node IP. If you put a new LB in front later, regenerate the cert with the new SANs — otherwise `kubectl` from outside the cluster fails TLS verification with no obvious cause. Edit `/etc/kubernetes/kubeadm-config` ConfigMap → `kubeadm init phase certs apiserver --config <file>`.
- **CA rotation is a *manual project*, not a command.** `kubeadm certs renew` only renews leaves under the existing CA. Rotating the CA itself requires generating a new CA, distributing the bundle (old + new) to every component, restarting everything, then re-issuing leaves under the new CA, then dropping the old CA. Plan a maintenance window. Do not do this on a Friday.
- **`sa.key` rotation invalidates every ServiceAccount token.** Including the ones controllers and operators are using right now. Treat it as never-rotated unless responding to a known compromise. To rotate safely, you publish both old and new public keys via `--service-account-key-file` (repeatable) on the apiserver, restart, then drop the old one — but you also need to restart every workload that holds an old token.
- **Cluster signing duration cap.** Even if you ask for `expirationSeconds: 31536000` in a CSR, the controller-manager caps it at `--cluster-signing-duration` (default 1 year). Bumping this affects *new* certs only — existing leaves keep their original expiry.
- **Managed K8s (EKS/AKS/GKE) hides this entirely.** The control-plane CA, apiserver cert, and etcd certs are the cloud's problem. You are still responsible for: workload-issued certs (cert-manager / Vault), CSI driver client certs, webhook serving certs, and OIDC trust to your IdP. Don't assume "managed" means "no certs to think about."

## Related

- `cluster-upgrades.md` — `kubeadm upgrade` auto-renews control-plane certs; aligning upgrade and rotation cadence.
- `etcd-backup-and-restore.md` — always snapshot etcd before any rotation.
- `kubeadm.md` — initial PKI is laid down at `kubeadm init phase certs`.
- `node-maintenance.md` — node lifecycle interacts with kubelet cert bootstrap.
- `../SECURITY/service-accounts.md` — ServiceAccount JWTs (signed by `sa.key`, not an X.509 cert).
- `../SECURITY/authentication.md` — how the apiserver picks an authenticator chain (X.509 → bearer token → OIDC).
- `../SECURITY/authorization-rbac.md` — the other half of "I have a valid cert" — what it can do.
- `../SECURITY/secrets-management.md` — workload mTLS via cert-manager / Vault, distinct from cluster PKI.
- `../NETWORKING/ingress.md` — TLS termination at the edge uses Secret-stored certs, not cluster PKI.
- `../EXTENSIBILITY/admission-webhooks.md` — webhook serving cert trust and the `caBundle` field.
