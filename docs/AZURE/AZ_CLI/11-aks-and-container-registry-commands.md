# Azure CLI — AKS, ACR, and Container Apps

> **`az aks` for Kubernetes, `az acr` for images, `az containerapp` for serverless containers.**
> **Related:** [AZ-104 / 09 Containers](../AZ-104/09-containers.md) · [AZURE/CONTAINER](../CONTAINER/) · [08 Compute commands (ACI)](08-compute-commands.md#azure-container-instances-az-container)
> **Sources:** [`az aks`](https://learn.microsoft.com/en-us/cli/azure/aks) · [`az acr`](https://learn.microsoft.com/en-us/cli/azure/acr) · [`az containerapp`](https://learn.microsoft.com/en-us/cli/azure/containerapp)

---

## Table of contents

1. [ACR — Azure Container Registry](#acr--azure-container-registry)
2. [ACR tasks & imports](#acr-tasks--imports)
3. [AKS — create & lifecycle](#aks--create--lifecycle)
4. [Credentials & kubectl](#credentials--kubectl)
5. [Node pools](#node-pools)
6. [Upgrade & maintenance](#upgrade--maintenance)
7. [AKS networking & identity](#aks-networking--identity)
8. [`az aks command invoke` (private clusters)](#az-aks-command-invoke-private-clusters)
9. [Azure Container Apps](#azure-container-apps)

---

## ACR — Azure Container Registry

### Create & authenticate

```bash
# Create (Basic/Standard/Premium — Premium = replication, firewall, private endpoints)
az acr create -g rg-dev -n mycr --sku Standard --admin-enabled false --location eastus

az acr list -o table
az acr show -g rg-dev -n mycr

# Login for local docker / buildx
az acr login -n mycr                       # uses az CLI token; pushes into docker creds store

# Without docker CLI — just prints the token
az acr login -n mycr --expose-token
```

- Prefer **Entra ID / AKS kubelet identity** over the admin account — `--admin-enabled false` is the production default. [`az acr login`](https://learn.microsoft.com/en-us/cli/azure/acr#az-acr-login) uses your CLI identity and requires [`AcrPush`](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-roles) / `AcrPull`.
- Premium tier unlocks [geo-replication](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-geo-replication), [private endpoints](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-private-link), customer-managed keys.

### Images & repositories

```bash
az acr repository list -n mycr -o table
az acr repository show-tags -n mycr --repository myapp -o table

az acr repository show-manifests -n mycr --repository myapp \
  --query "[].{digest:digest, tags:tags, created:createdTime}" -o table

az acr repository delete -n mycr --image myapp:old-tag --yes
az acr repository untag -n mycr --image myapp:latest

# Retention (Premium)
az acr config retention update -n mycr --status enabled --days 30 --type UntaggedManifests
```

### Replication

```bash
az acr replication create -r mycr -l westeurope -n westeurope
az acr replication list -r mycr -o table
```

- Each replica is billed separately — plan regions accordingly.

### Firewall & private endpoints

```bash
az acr update -n mycr --public-network-enabled false      # private-only (Premium)
az acr network-rule add -n mycr --ip-address 203.0.113.0/24
az acr network-rule list -n mycr
```

## ACR tasks & imports

Tasks = build/patch container images in the registry (no local Docker needed).

```bash
# Build from source
az acr build -r mycr -t myapp:v1 ./src       # uses local Dockerfile

# Build on trigger (GitHub push)
az acr task create \
  -r mycr -n build-myapp \
  -c https://github.com/org/repo.git -f Dockerfile \
  --commit-trigger-enabled true --git-access-token <pat> \
  --image myapp:{{.Run.ID}} --image myapp:latest

az acr task list -r mycr -o table
az acr task run -r mycr -n build-myapp
az acr task delete -r mycr -n build-myapp --yes

# Import image from another registry (no local pull/push)
az acr import -n mycr --source docker.io/library/nginx:latest --image nginx:latest
```

## AKS — create & lifecycle

```bash
# Modern baseline — managed identity, Azure CNI Overlay, Cilium, Entra integration, Azure RBAC
az aks create \
  -g rg-dev -n aks-prod \
  --location eastus \
  --kubernetes-version 1.30.5 \
  --node-count 3 --node-vm-size Standard_D4s_v5 \
  --enable-managed-identity \
  --network-plugin azure --network-plugin-mode overlay \
  --network-dataplane cilium \
  --service-cidr 10.100.0.0/16 --pod-cidr 10.200.0.0/16 \
  --enable-aad --aad-admin-group-object-ids <groupObjId> \
  --enable-azure-rbac \
  --enable-oidc-issuer --enable-workload-identity \
  --attach-acr mycr \
  --zones 1 2 3 \
  --load-balancer-sku standard

# List / inspect
az aks list -o table
az aks show -g rg-dev -n aks-prod
az aks show -g rg-dev -n aks-prod --query "{rg:resourceGroup, ver:kubernetesVersion, fqdn:fqdn, oidc:oidcIssuerProfile.issuerUrl}"

# Stop / start (save $ without deleting)
az aks stop  -g rg-dev -n aks-prod
az aks start -g rg-dev -n aks-prod

# Delete
az aks delete -g rg-dev -n aks-prod --yes --no-wait
```

- `--attach-acr mycr` grants the AKS kubelet identity `AcrPull` on the registry — one command replaces separate role-assignment wiring.
- `--network-plugin-mode overlay` (Azure CNI Overlay) gives you large pod CIDRs without exhausting vnet IPs.

## Credentials & kubectl

```bash
# Fetch kubeconfig (merged into ~/.kube/config)
az aks get-credentials -g rg-dev -n aks-prod

# Admin creds (bypass Entra — requires Cluster Admin role)
az aks get-credentials -g rg-dev -n aks-prod --admin

# Separate file
az aks get-credentials -g rg-dev -n aks-prod --file ./kube.conf

# Confirm context
kubectl config current-context
kubectl get nodes

# For Entra clusters on a machine without kubelogin
az aks install-cli                      # installs kubectl + kubelogin
```

- The first `kubectl` command after Entra login triggers a device-code prompt for the cluster if you're not already authenticated via `kubelogin`.
- Admin creds disable Entra integration — reserve for break-glass. Audit log lines look anonymized.

## Node pools

AKS has a **system pool** (CriticalAddonsOnly) plus optional **user pools**.

```bash
# Add a user pool
az aks nodepool add \
  -g rg-dev --cluster-name aks-prod -n gpupool \
  --node-count 1 --node-vm-size Standard_NC6s_v3 \
  --mode User --zones 1 \
  --node-taints sku=gpu:NoSchedule \
  --labels workload=gpu

# Spot pool
az aks nodepool add \
  -g rg-dev --cluster-name aks-prod -n spotpool \
  --priority Spot --eviction-policy Delete --spot-max-price -1 \
  --node-vm-size Standard_D4s_v5 --enable-cluster-autoscaler \
  --min-count 0 --max-count 10

# Scale
az aks nodepool scale -g rg-dev --cluster-name aks-prod -n spotpool --node-count 5

# Autoscaler
az aks nodepool update -g rg-dev --cluster-name aks-prod -n userpool \
  --enable-cluster-autoscaler --min-count 2 --max-count 10

# Upgrade just this pool
az aks nodepool upgrade -g rg-dev --cluster-name aks-prod -n userpool \
  --kubernetes-version 1.30.5

az aks nodepool list -g rg-dev --cluster-name aks-prod -o table
az aks nodepool delete -g rg-dev --cluster-name aks-prod -n spotpool
```

- Spot pools evict when capacity is reclaimed; pair with `cluster-autoscaler` for resilience.
- `--mode System` pools must allow `CriticalAddonsOnly` taints; don't add user workloads there.

## Upgrade & maintenance

```bash
# Available versions
az aks get-upgrades -g rg-dev -n aks-prod -o table
az aks get-versions -l eastus -o table        # all versions in region

# Control-plane + all node pools (default)
az aks upgrade -g rg-dev -n aks-prod --kubernetes-version 1.30.5

# Control-plane only
az aks upgrade -g rg-dev -n aks-prod --kubernetes-version 1.30.5 --control-plane-only

# Maintenance window
az aks maintenanceconfiguration add \
  -g rg-dev --cluster-name aks-prod -n default \
  --weekday Sunday --start-hour 2 --duration 4

# Node image upgrades (weekly OS patches, independent of k8s version)
az aks nodepool upgrade -g rg-dev --cluster-name aks-prod -n nodepool1 --node-image-only
az aks upgrade -g rg-dev -n aks-prod --node-image-only         # all pools
```

- AKS **auto-upgrade channels**: `none`, `patch`, `stable`, `rapid`, `node-image`. Enable via `az aks update --auto-upgrade-channel stable`.
- [Kubernetes support matrix](https://learn.microsoft.com/en-us/azure/aks/supported-kubernetes-versions) — AKS supports N-2 minor versions.

## AKS networking & identity

```bash
# Add / rotate AKS-integrated ACR
az aks update -g rg-dev -n aks-prod --attach-acr mycr
az aks update -g rg-dev -n aks-prod --detach-acr oldcr

# Enable workload identity (if not at create)
az aks update -g rg-dev -n aks-prod \
  --enable-oidc-issuer --enable-workload-identity

# Get OIDC issuer URL (for federated cred on a user-assigned MI)
ISSUER=$(az aks show -g rg-dev -n aks-prod --query oidcIssuerProfile.issuerUrl -o tsv)

# Create federated credential on a user-assigned MI (pairs with a k8s ServiceAccount)
az identity federated-credential create \
  -g rg-mi --identity-name mi-app \
  --name fc-aks-app --issuer "$ISSUER" \
  --subject system:serviceaccount:prod:app-sa \
  --audiences api://AzureADTokenExchange

# Entra RBAC
az role assignment create \
  --role "Azure Kubernetes Service RBAC Admin" \
  --assignee alice@contoso.com \
  --scope $(az aks show -g rg-dev -n aks-prod --query id -o tsv)

# Change load-balancer outbound IPs / prefix
az aks update -g rg-dev -n aks-prod \
  --load-balancer-outbound-ip-prefixes <publicIpPrefixId>
```

## `az aks command invoke` (private clusters)

Run `kubectl` (or anything) against a private cluster without VPN / bastion:

```bash
az aks command invoke \
  -g rg-dev -n aks-prod \
  --command "kubectl get pods -A"

az aks command invoke \
  -g rg-dev -n aks-prod \
  --command "helm upgrade app ./chart" \
  --file ./chart
```

- Spins up a transient pod inside the cluster that runs your command and streams output.
- Requires the [`AKS Service`](https://learn.microsoft.com/en-us/azure/aks/access-private-cluster) built-in role on the cluster.
- Slower than native `kubectl` (pod startup ~30 s) — fine for one-shots, not for interactive debugging loops.

## Azure Container Apps

[Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/overview) = serverless containers with Dapr, scale-to-zero, KEDA autoscaling.

```bash
# Register provider (first use per sub)
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights

# Environment (shared networking / logging)
az containerapp env create \
  -g rg-dev -n cae-prod \
  --location eastus \
  --logs-workspace-id $(az monitor log-analytics workspace show -g rg-dev -n laws-prod --query customerId -o tsv) \
  --logs-workspace-key $(az monitor log-analytics workspace get-shared-keys -g rg-dev -n laws-prod --query primarySharedKey -o tsv)

# Internal-only environment with custom vnet
az containerapp env create \
  -g rg-dev -n cae-internal \
  --internal-only true \
  --infrastructure-subnet-resource-id <subnetId>

# Create a container app
az containerapp create \
  -g rg-dev -n app-web \
  --environment cae-prod \
  --image mycr.azurecr.io/myapp:v1 \
  --registry-server mycr.azurecr.io --registry-identity system \
  --ingress external --target-port 8080 \
  --cpu 0.5 --memory 1.0Gi \
  --min-replicas 0 --max-replicas 10 \
  --env-vars ENV=prod FEATURE_X=true

# Scale rules (KEDA-style)
az containerapp update \
  -g rg-dev -n app-web \
  --scale-rule-name http-rule --scale-rule-type http \
  --scale-rule-http-concurrency 100

az containerapp update \
  -g rg-dev -n app-queue \
  --scale-rule-name queue-rule --scale-rule-type azure-queue \
  --scale-rule-metadata queueName=jobs accountName=stgdev1 queueLength=5 \
  --scale-rule-auth "connection=connection-string"

# Revisions
az containerapp revision list -g rg-dev -n app-web -o table
az containerapp revision set-mode -g rg-dev -n app-web --mode multiple
az containerapp revision copy -g rg-dev -n app-web \
  --image mycr.azurecr.io/myapp:v2 --from-revision app-web--abc123

# Logs
az containerapp logs show -g rg-dev -n app-web --follow --tail 100
az containerapp exec -g rg-dev -n app-web --command "/bin/sh"
```

- **Revisions**: `single` mode retires the previous revision on each update; `multiple` keeps them side-by-side for blue/green + traffic splitting.
- **Scale to zero**: requires `--min-replicas 0`. App sleeps when idle; first request has a cold-start penalty.
- Registry pull via managed identity: `--registry-identity system` (or a user-assigned MI resource ID) — no username/password needed.

### Dapr sidecar

```bash
az containerapp create ... \
  --enable-dapr true \
  --dapr-app-id orders-svc \
  --dapr-app-port 8080 \
  --dapr-app-protocol http
```

Configure Dapr components (bindings, pub/sub, state) at the **environment** scope:

```bash
az containerapp env dapr-component set \
  -g rg-dev --name cae-prod --dapr-component-name statestore \
  --yaml ./statestore.yaml
```

## Gotchas

- **`az acr login` failure in non-interactive shells**: use `--expose-token` + `docker login -u 00000000-0000-0000-0000-000000000000 -p <token>` for CI agents without keychains.
- **`--attach-acr`** adds `AcrPull` to the **kubelet identity**, not the cluster identity. If you use a user-assigned kubelet MI, ensure the role lands on the right principal.
- **Spot pools + scale-to-zero** combine well, but node provisioning adds 3-5 min to cold starts. Keep a small on-demand base pool.
- **`az aks command invoke` token cache** lives inside the transient pod — it can't use your local `kubelogin` creds. Commands run as the **cluster's identity**; pass whatever RBAC you need on that principal.
- **AKS auto-upgrade** can surprise you — pin versions in prod, set a [maintenance window](https://learn.microsoft.com/en-us/azure/aks/planned-maintenance), and exclude weekends.
- **Container Apps revision names** are immutable: changing an env var creates a new revision, not a patch. Plan for rolling traffic splits.
- **ACR geo-replication**: writes go to the primary region, reads are local. Don't expect write-region failover — use separate registries or customer-managed DR.

## Sources

- [`az acr`](https://learn.microsoft.com/en-us/cli/azure/acr) · [`az acr task`](https://learn.microsoft.com/en-us/cli/azure/acr/task)
- [`az aks`](https://learn.microsoft.com/en-us/cli/azure/aks) · [`az aks nodepool`](https://learn.microsoft.com/en-us/cli/azure/aks/nodepool)
- [`az containerapp`](https://learn.microsoft.com/en-us/cli/azure/containerapp)
- [AKS best practices](https://learn.microsoft.com/en-us/azure/aks/best-practices)
- [AKS supported Kubernetes versions](https://learn.microsoft.com/en-us/azure/aks/supported-kubernetes-versions)
- [Workload identity on AKS](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)
- [Container Apps scaling](https://learn.microsoft.com/en-us/azure/container-apps/scale-app)
