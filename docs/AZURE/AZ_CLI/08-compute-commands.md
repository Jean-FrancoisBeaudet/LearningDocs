# Azure CLI — Compute Commands

> **VMs, scale sets, App Service, Functions, and ACI — command reference.**
> **Related:** [AZ-104 / 08 Virtual Machines](../AZ-104/08-virtual-machines.md) · [AZ-104 / 09 Containers](../AZ-104/09-containers.md) · [AZ-104 / 10 App Service](../AZ-104/10-app-service.md) · [11 AKS & ACR commands](11-aks-and-container-registry-commands.md)
> **Sources:** [`az vm`](https://learn.microsoft.com/en-us/cli/azure/vm) · [`az vmss`](https://learn.microsoft.com/en-us/cli/azure/vmss) · [`az webapp`](https://learn.microsoft.com/en-us/cli/azure/webapp) · [`az functionapp`](https://learn.microsoft.com/en-us/cli/azure/functionapp) · [`az container`](https://learn.microsoft.com/en-us/cli/azure/container)

---

## Table of contents

1. [Virtual machines (`az vm`)](#virtual-machines-az-vm)
2. [Disks, images, snapshots](#disks-images-snapshots)
3. [VM extensions & run-command](#vm-extensions--run-command)
4. [Scale sets (`az vmss`)](#scale-sets-az-vmss)
5. [App Service (`az webapp`)](#app-service-az-webapp)
6. [Deployment slots](#deployment-slots)
7. [Function Apps (`az functionapp`)](#function-apps-az-functionapp)
8. [Azure Container Instances (`az container`)](#azure-container-instances-az-container)

---

## Virtual machines (`az vm`)

### Create

```bash
# Minimal — Linux from Ubuntu LTS, auto-generated SSH key
az vm create \
  --resource-group rg-dev --name vm-web \
  --image Ubuntu2204 \
  --size Standard_B2ms \
  --admin-username azureuser \
  --generate-ssh-keys \
  --location eastus

# Windows with a specific image and existing vnet
az vm create \
  --resource-group rg-dev --name vm-win \
  --image Win2022Datacenter \
  --admin-username adminuser --admin-password 'P@ssw0rd123!' \
  --vnet-name vnet-dev --subnet subnet-vm \
  --public-ip-sku Standard \
  --public-ip-address-allocation Static

# With zones, boot diagnostics, MSI, tags
az vm create \
  -g rg-dev -n vm-app \
  --image Ubuntu2204 --size Standard_D4s_v5 \
  --zone 1 \
  --assign-identity \
  --boot-diagnostics-storage stgdiag1 \
  --tags env=dev costcenter=1234
```

- `--image` accepts a short alias (`Ubuntu2204`, `Win2022Datacenter`) or a full URN `Publisher:Offer:Sku:Version` or a custom image resource ID.
- `--zone` pins to an AZ; omit for regional placement.
- Generated SSH keys land in `~/.ssh/id_rsa` unless you pass `--ssh-key-values`.
- Use `--no-wait` and `az vm wait --created` for parallel multi-VM creates.

### Lifecycle

```bash
az vm list -o table
az vm list --resource-group rg-dev -o table
az vm show -g rg-dev -n vm-web --show-details       # + NIC, IP, power state
az vm list --show-details --query "[].{name:name, ip:publicIps, state:powerState}" -o table

az vm start       -g rg-dev -n vm-web
az vm stop        -g rg-dev -n vm-web                # still billed for compute
az vm deallocate  -g rg-dev -n vm-web                # release compute → stops billing
az vm restart     -g rg-dev -n vm-web
az vm redeploy    -g rg-dev -n vm-web                # force migration to another host
az vm reimage     -g rg-dev -n vm-web                # rebuild OS disk from image (ephemeral OS)
az vm resize      -g rg-dev -n vm-web --size Standard_D4s_v5

az vm delete -g rg-dev -n vm-web --yes --force-deletion true
```

- `stop` (without deallocate) keeps the compute reservation — you still pay for CPU/RAM. `deallocate` is what people usually mean by "stop to save money."
- `--force-deletion true` on delete skips graceful shutdown — useful for stuck VMs.

### Networking on an existing VM

```bash
az vm list-ip-addresses -g rg-dev -n vm-web -o table

# Add a data disk
az vm disk attach -g rg-dev --vm-name vm-web \
  --name datadisk1 --new --size-gb 128 --sku Premium_LRS

# Change NSG on primary NIC
az network nic update -g rg-dev -n vm-web-nic --network-security-group nsg-tight
```

### Availability options

```bash
# Availability set (fault + update domains in a single region)
az vm availability-set create -g rg-dev -n avset-web \
  --platform-fault-domain-count 2 --platform-update-domain-count 5

# VM in availability set
az vm create ... --availability-set avset-web --no-ppg

# Proximity placement group (low-latency co-location)
az ppg create -g rg-dev -n ppg-compute --type Standard
az vm create ... --ppg ppg-compute
```

- **AZ + Availability Set are mutually exclusive** on a VM.
- Flexible VMSS is usually preferable to availability sets for new workloads — see [`az vmss`](#scale-sets-az-vmss).

## Disks, images, snapshots

```bash
# Managed disks
az disk create -g rg-dev -n disk-data1 --size-gb 256 --sku Premium_ZRS --zone 1
az disk list -g rg-dev -o table
az disk update -g rg-dev -n disk-data1 --size-gb 512

# Snapshot
az snapshot create -g rg-dev -n snap-web-osdisk \
  --source /subscriptions/.../disks/vm-web_OsDisk_1_xyz

# Image (custom VM image from a deallocated generalized VM)
az vm deallocate -g rg-dev -n vm-golden
az vm generalize -g rg-dev -n vm-golden
az image create -g rg-dev -n img-golden --source vm-golden

# Shared Image Gallery
az sig create -g rg-dev --gallery-name sig-shared
az sig image-definition create --gallery-name sig-shared -g rg-dev \
  --gallery-image-definition imgdef-webapp \
  --publisher contoso --offer webapp --sku base \
  --os-type Linux --hyper-v-generation V2
az sig image-version create --gallery-name sig-shared -g rg-dev \
  --gallery-image-definition imgdef-webapp \
  --gallery-image-version 1.0.0 \
  --managed-image img-golden \
  --target-regions eastus=1 westus=1
```

- Shared Image Gallery replicates an image to target regions + zone-redundant storage — the recommended pattern for cross-region golden images.
- Disk bursting, performance tiers: `az disk update --tier P50`.

## VM extensions & run-command

```bash
# List available extensions on a given VM host OS
az vm extension image list --location eastus --query "[?contains(name,'CustomScript')]" -o table

# Install an extension
az vm extension set \
  -g rg-dev --vm-name vm-web \
  --publisher Microsoft.Azure.Extensions \
  --name CustomScript \
  --version 2.1 \
  --settings '{"fileUris":["https://example.com/init.sh"], "commandToExecute":"bash init.sh"}'

# Managed identity extension (legacy; system MI doesn't need it)
# DSC, Desired State Config
az vm extension set -g rg-dev --vm-name vm-win \
  --publisher Microsoft.Powershell --name DSC --version 2.83 \
  --settings '{"ModulesUrl":"https://example.com/dsc.zip","ConfigurationFunction":"config.ps1\\WebConfig"}'

# Remove
az vm extension delete -g rg-dev --vm-name vm-web --name CustomScript
```

### run-command (lightweight alternative to extensions)

```bash
# Inline command (Linux)
az vm run-command invoke \
  -g rg-dev -n vm-web \
  --command-id RunShellScript \
  --scripts "apt-get update && apt-get install -y nginx"

# Inline (Windows)
az vm run-command invoke \
  -g rg-dev -n vm-win \
  --command-id RunPowerShellScript \
  --scripts "Get-Service 'W3SVC' | Restart-Service"

# Persistent (v2 API, survives reboots and shows in portal)
az vm run-command create \
  -g rg-dev --vm-name vm-web --run-command-name rc-install-app \
  --script "bash install.sh" \
  --async-execution false
```

- `invoke` is one-shot; the **new** `az vm run-command create` (v2 API) is persistent and auditable.
- Output returns stdout/stderr and exit code in the JSON response.

## Scale sets (`az vmss`)

[VMSS](https://learn.microsoft.com/en-us/azure/virtual-machine-scale-sets/overview) comes in **Uniform** (classic) and **Flexible** (mix VMs across FDs/zones, default since 2022).

```bash
# Flexible (recommended for new)
az vmss create \
  -g rg-dev -n vmss-web \
  --orchestration-mode Flexible \
  --image Ubuntu2204 \
  --instance-count 3 \
  --vm-sku Standard_D2s_v5 \
  --zones 1 2 3 \
  --lb "" --app-gateway "" \
  --admin-username azureuser --generate-ssh-keys

# Scale operations
az vmss scale -g rg-dev -n vmss-web --new-capacity 10
az vmss list-instances -g rg-dev -n vmss-web -o table
az vmss restart -g rg-dev -n vmss-web --instance-ids 0 1 2

# Update — rolling upgrade
az vmss update -g rg-dev -n vmss-web \
  --set virtualMachineProfile.storageProfile.imageReference.version=latest
az vmss update-instances -g rg-dev -n vmss-web --instance-ids "*"   # or specific IDs

# Rolling upgrade policy (Flex)
az vmss update -g rg-dev -n vmss-web \
  --set upgradePolicy.rollingUpgradePolicy.maxBatchInstancePercent=20
```

### Autoscale

```bash
az monitor autoscale create \
  -g rg-dev --resource vmss-web --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name autoscale-vmss-web --min-count 2 --max-count 10 --count 3

az monitor autoscale rule create \
  --autoscale-name autoscale-vmss-web -g rg-dev \
  --condition "Percentage CPU > 75 avg 5m" \
  --scale out 1

az monitor autoscale rule create \
  --autoscale-name autoscale-vmss-web -g rg-dev \
  --condition "Percentage CPU < 25 avg 10m" \
  --scale in 1
```

See [13 Monitor & diagnostics commands](13-monitor-and-diagnostics-commands.md) for more autoscale details.

## App Service (`az webapp`)

```bash
# Plan + app (Linux)
az appservice plan create -g rg-dev -n plan-web --sku P1v3 --is-linux
az webapp create -g rg-dev -n app-web --plan plan-web \
  --runtime "DOTNETCORE:8.0"

# Runtimes list
az webapp list-runtimes --os linux -o table
az webapp list-runtimes --os windows -o table

# Deploy — ZIP
az webapp deploy -g rg-dev -n app-web --src-path ./build.zip --type zip

# Deploy — container
az webapp create -g rg-dev -n app-web --plan plan-web \
  --deployment-container-image-name mycr.azurecr.io/myapp:v1

# Deploy — from GitHub
az webapp deployment source config \
  -g rg-dev -n app-web \
  --repo-url https://github.com/org/repo \
  --branch main --manual-integration

# Inspect
az webapp show -g rg-dev -n app-web
az webapp browse -g rg-dev -n app-web                # opens default browser
az webapp log tail -g rg-dev -n app-web              # live log stream
az webapp log download -g rg-dev -n app-web
az webapp ssh -g rg-dev -n app-web                   # SSH into Linux container

# App settings & connection strings
az webapp config appsettings set -g rg-dev -n app-web \
  --settings DB_HOST=sqlprod01.database.windows.net FEATURE_X=true
az webapp config appsettings list -g rg-dev -n app-web -o table
az webapp config connection-string set -g rg-dev -n app-web \
  --connection-string-type SQLAzure \
  --settings Default="Server=tcp:sqlprod01..."

# Identity + Key Vault reference
az webapp identity assign -g rg-dev -n app-web
az webapp config appsettings set -g rg-dev -n app-web \
  --settings "SECRET=@Microsoft.KeyVault(VaultName=kv-prod;SecretName=app-secret)"

# Restart / stop / start
az webapp restart -g rg-dev -n app-web
az webapp stop    -g rg-dev -n app-web
az webapp start   -g rg-dev -n app-web
```

- `az webapp up` is an all-in-one "deploy from current directory" quick-start — great for demos, not for pipelines.
- Key Vault references in app settings: the app's MI must have `get` on the secret (RBAC: `Key Vault Secrets User`).

### Custom domain + TLS

```bash
az webapp config hostname add -g rg-dev --webapp-name app-web --hostname www.example.com
az webapp config ssl bind -g rg-dev --name app-web \
  --certificate-thumbprint <tp> --ssl-type SNI
# Managed certificate for the hostname (free App Service Managed Cert)
az webapp config ssl create -g rg-dev --name app-web --hostname www.example.com
```

## Deployment slots

Slots enable blue/green deploys within the same App Service plan (Standard tier+).

```bash
az webapp deployment slot create -g rg-dev -n app-web --slot staging
az webapp deployment slot list -g rg-dev -n app-web -o table

# Deploy to staging slot
az webapp deploy -g rg-dev -n app-web --slot staging --src-path build-v2.zip --type zip

# Swap prod ↔ staging
az webapp deployment slot swap -g rg-dev -n app-web --slot staging --target-slot production

# Auto-swap setting
az webapp deployment slot auto-swap -g rg-dev -n app-web \
  --slot staging --auto-swap-slot production

# Delete slot
az webapp deployment slot delete -g rg-dev -n app-web --slot staging
```

- Slot settings marked **"slot setting"** stay with the slot on swap (e.g., DB connection strings, environment tags). Mark via `--slot-settings` on `az webapp config appsettings set`.

## Function Apps (`az functionapp`)

```bash
# Storage + consumption plan
az storage account create -g rg-dev -n stgfuncs1 --sku Standard_LRS
az functionapp create \
  -g rg-dev -n func-etl \
  --consumption-plan-location eastus \
  --storage-account stgfuncs1 \
  --runtime dotnet-isolated --runtime-version 8 \
  --functions-version 4

# Premium/EP plan
az functionapp plan create -g rg-dev -n plan-func \
  --location eastus --sku EP1
az functionapp create -g rg-dev -n func-etl \
  --plan plan-func \
  --storage-account stgfuncs1 \
  --runtime node --runtime-version 20

# Deploy
az functionapp deployment source config-zip \
  -g rg-dev -n func-etl --src ./pub.zip

# App settings (incl. host-level Functions settings)
az functionapp config appsettings set -g rg-dev -n func-etl \
  --settings FUNCTIONS_WORKER_RUNTIME=dotnet-isolated AzureWebJobsStorage__accountName=stgfuncs1

# Identity
az functionapp identity assign -g rg-dev -n func-etl

# Logs
az functionapp log tail -g rg-dev -n func-etl
az functionapp log deployment show -g rg-dev -n func-etl
```

- For **Flex Consumption** (newer plan): `az functionapp create ... --flexconsumption-location eastus --runtime dotnet-isolated --instance-memory 2048 --maximum-instance-count 100`.
- Functions runs on the same App Service infra — `az webapp ...` and `az functionapp ...` overlap substantially but have runtime-specific flags.

## Azure Container Instances (`az container`)

Lightweight containers without managing AKS. Pay-per-second.

```bash
# Public image
az container create -g rg-dev -n aci-nginx \
  --image nginx:latest \
  --cpu 1 --memory 1.5 \
  --ports 80 --dns-name-label aci-nginx-demo \
  --ip-address Public

# Private image (ACR)
az container create -g rg-dev -n aci-app \
  --image mycr.azurecr.io/myapp:v1 \
  --registry-login-server mycr.azurecr.io \
  --registry-username <user> --registry-password <pw> \
  --cpu 2 --memory 4 \
  --vnet vnet-dev --subnet subnet-aci

# Inspect
az container list -g rg-dev -o table
az container show -g rg-dev -n aci-nginx
az container logs -g rg-dev -n aci-nginx
az container attach -g rg-dev -n aci-nginx           # live log + stdin
az container exec -g rg-dev -n aci-nginx --exec-command "/bin/sh"

# Lifecycle
az container restart -g rg-dev -n aci-nginx
az container stop    -g rg-dev -n aci-nginx
az container start   -g rg-dev -n aci-nginx
az container delete  -g rg-dev -n aci-nginx --yes

# Multi-container group (YAML)
az container create -g rg-dev --file aci-group.yaml
```

- ACI is the "run a container for ~an hour" tool. For sustained workloads, prefer [Container Apps](11-aks-and-container-registry-commands.md#container-apps).
- Private networking: ACI in a subnet can't have a public IP simultaneously — trade-off.

## Gotchas

- **`--size` SKUs are region-locked**: `az vm list-skus -l eastus -o table --query "[?resourceType=='virtualMachines']"` before picking.
- **Image aliases change silently** — `Ubuntu2204` used to be `UbuntuLTS`. Pin to a full URN in production templates.
- **`az vm create` generates an RG-scope NSG + public IP by default**; override with `--nsg ""` and `--public-ip-address ""` for locked-down environments.
- **`az webapp up`** auto-creates RG, plan, and app — convenient for demos, opaque for real ops. Use explicit commands in CI.
- **Slot swaps** warm up the target slot before cutting over — a swap can take 30-90 seconds. Use `az webapp config appsettings set --slot-settings` to prevent production DB connstrings from leaking into staging after a swap.
- **VMSS Flex** + `--orchestration-mode Uniform` are separate product tracks. You cannot convert between them in place.
- **App Service `az webapp ssh`** works on Linux only, and only if the container image has SSH enabled (built-in images do; custom images need an SSH daemon).
- **ACI with a subnet cannot use DNS name labels** — they're a public-IP feature.

## Sources

- [`az vm`](https://learn.microsoft.com/en-us/cli/azure/vm) · [`az vmss`](https://learn.microsoft.com/en-us/cli/azure/vmss) · [`az disk`](https://learn.microsoft.com/en-us/cli/azure/disk) · [`az image`](https://learn.microsoft.com/en-us/cli/azure/image) · [`az sig`](https://learn.microsoft.com/en-us/cli/azure/sig)
- [`az webapp`](https://learn.microsoft.com/en-us/cli/azure/webapp) · [`az appservice`](https://learn.microsoft.com/en-us/cli/azure/appservice)
- [`az functionapp`](https://learn.microsoft.com/en-us/cli/azure/functionapp)
- [`az container`](https://learn.microsoft.com/en-us/cli/azure/container)
- [VM sizes](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes) · [VMSS orchestration modes](https://learn.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-orchestration-modes)
- [App Service deployment slots](https://learn.microsoft.com/en-us/azure/app-service/deploy-staging-slots)
- [Functions hosting options](https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale)
