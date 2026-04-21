# Azure Virtual Machines

> **Exam mapping:** AZ-104 --> "Deploy and manage Azure compute resources" (20-25%)
> **One-liner:** Create, configure, encrypt, resize, and scale Azure VMs while choosing the right availability strategy (zones vs sets) and disk tier for cost and performance.
> **Related:** [07-arm-and-bicep](./07-arm-and-bicep.md) | [09-containers](./09-containers.md) | [10-app-service](./10-app-service.md)

---

## 1. Create a Virtual Machine

**Reference:** [Quickstart -- Create a Linux VM](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-cli) | [Quickstart -- Create a Windows VM](https://learn.microsoft.com/en-us/azure/virtual-machines/windows/quick-create-portal)

### 1.1 Required Settings

Every VM deployment requires:

| Setting | Notes |
|---------|-------|
| **Subscription** | Billing container; determines quotas and RBAC scope |
| **Resource group** | Logical container; all dependent resources (NIC, disk, NSG, public IP) land here by default |
| **VM name** | 1-64 chars; becomes the computer name (15-char Windows limit still applies inside the OS) |
| **Region** | Determines available sizes, images, and zone support |
| **Image** | OS + optional software; sourced from Azure Marketplace, [Azure Compute Gallery](https://learn.microsoft.com/en-us/azure/virtual-machines/azure-compute-gallery), or a custom VHD |
| **Size** | vCPU / RAM / disk count / NIC count -- see Section 4 |
| **Administrator account** | SSH public key (Linux) or username + password (Windows); password auth can be disabled on Linux |
| **Inbound port rules** | Quick allow for SSH (22), RDP (3389), HTTP (80), HTTPS (443); creates an NSG attached to the NIC |

Additional tabs in the portal cover: **Disks** (OS disk type, data disks), **Networking** (VNet, subnet, public IP, NSG, load balancer), **Management** (boot diagnostics, auto-shutdown, backup), **Advanced** (extensions, cloud-init / custom data, proximity placement groups), and **Tags**.

### 1.2 Marketplace Images

[Azure Marketplace](https://learn.microsoft.com/en-us/marketplace/azure-marketplace-overview) provides first-party (Windows Server, Ubuntu, RHEL) and third-party images. Images are referenced as `Publisher:Offer:SKU:Version` -- for example `Canonical:0001-com-ubuntu-server-jammy:22_04-lts-gen2:latest`.

```bash
# List popular images
az vm image list --output table

# Search for a specific publisher
az vm image list --publisher Canonical --offer 0001-com-ubuntu-server --all --output table
```

### 1.3 VM Generations -- Gen1 vs Gen2

| Feature | Gen1 | Gen2 |
|---------|------|------|
| **Boot firmware** | BIOS | UEFI |
| **OS disk size** | Up to 2 TiB | Up to 4 TiB |
| **Trusted launch** | Not supported (must upgrade to Gen2 first) | Supported |
| **Secure Boot / vTPM** | No | Yes (with trusted launch) |
| **Boot from large VHDX** | No | Yes |
| **Nested virtualisation** | Limited | Full support |

Gen2 is the default for new VMs. Gen1 VMs can be [upgraded to Gen2 + Trusted launch](https://learn.microsoft.com/en-us/azure/virtual-machines/trusted-launch-existing-vm-gen-1), but this is a one-way operation.

### 1.4 Trusted Launch

[Trusted launch](https://learn.microsoft.com/en-us/azure/virtual-machines/trusted-launch) protects against boot-kits and rootkits by combining:

- **Secure Boot** -- verifies that only signed OS loaders and drivers execute at boot.
- **vTPM** (virtual Trusted Platform Module 2.0) -- a dedicated secure vault for keys and measurements; enables attestation of the entire boot chain.
- **Boot integrity monitoring** -- reports boot health via [Microsoft Defender for Cloud](https://learn.microsoft.com/en-us/azure/defender-for-cloud/overview-page) integration.

Trusted launch is the **default security type** for new Gen2 VMs created via portal, CLI, PowerShell, Bicep, and ARM templates. It does not incur additional cost.

### 1.5 CLI and PowerShell

**Azure CLI:**

```bash
az vm create \
  --resource-group myRG \
  --name myVM \
  --image Ubuntu2204 \
  --size Standard_D2s_v5 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --security-type TrustedLaunch \
  --enable-secure-boot true \
  --enable-vtpm true \
  --public-ip-sku Standard
```

**Azure PowerShell:**

```powershell
New-AzVM `
  -ResourceGroupName "myRG" `
  -Name "myVM" `
  -Image "Ubuntu2204" `
  -Size "Standard_D2s_v5" `
  -Credential (Get-Credential) `
  -SecurityType "TrustedLaunch" `
  -EnableSecureBoot $true `
  -EnableVtpm $true
```

---

## 2. Encryption at Host

**Reference:** [Encryption at host](https://learn.microsoft.com/en-us/azure/virtual-machines/disk-encryption#encryption-at-host) | [Enable encryption at host](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-enable-host-based-encryption-portal)

### 2.1 What It Is

When **encryption at host** is enabled, encryption starts on the VM host server itself (the physical Azure server your VM is allocated to). Data on the temp disk and OS/data disk caches is encrypted at rest on the host, then flows encrypted to the Azure Storage service where [server-side encryption (SSE)](https://learn.microsoft.com/en-us/azure/virtual-machines/disk-encryption) persists it. This provides **end-to-end encryption** from the VM host through to Storage.

Key points:
- Covers **temp disk**, **OS disk cache**, and **data disk cache** -- these are the gaps SSE alone does not cover.
- Supports both **platform-managed keys (PMK)** and **customer-managed keys (CMK)** via Azure Key Vault.
- Does **not** consume VM CPU cycles (encryption happens on the host hardware).
- Does **not** impact VM performance.

### 2.2 Encryption at Host vs ADE vs SSE

| Aspect | SSE (Server-Side Encryption) | Encryption at Host | ADE (Azure Disk Encryption) |
|--------|------------------------------|--------------------|-----------------------------|
| **Where** | Storage service (data at rest on disk) | VM host (cache + temp disk + flow to Storage) | Inside the guest OS |
| **Technology** | AES-256 at storage layer | AES-256 at host layer | BitLocker (Windows) / dm-crypt (Linux) |
| **Temp disk** | Not encrypted | Encrypted | Encrypted |
| **OS/data disk cache** | Not encrypted | Encrypted | Not applicable (cache is at host) |
| **Persisted disk data** | Encrypted | Encrypted (flows through SSE) | Encrypted |
| **CPU impact** | None (storage-side) | None (host-side) | Uses guest VM CPU |
| **Key options** | PMK or CMK | PMK or CMK | CMK only (Key Vault) |
| **Enabled by default** | Yes (PMK) | No -- must opt in | No -- must opt in |
| **Mutual exclusivity** | Compatible with both | **Cannot coexist with ADE** | **Cannot coexist with encryption at host** |

> **ADE retirement notice:** Azure Disk Encryption is scheduled for retirement on **September 15, 2028**. Microsoft recommends migrating to encryption at host. After retirement, ADE-encrypted disks will fail to unlock after VM reboots.

### 2.3 Enabling Encryption at Host

**Step 1 -- Register the feature (one-time per subscription):**

```bash
az feature register --namespace Microsoft.Compute \
  --name EncryptionAtHost

# Check registration status
az feature show --namespace Microsoft.Compute \
  --name EncryptionAtHost --query properties.state

# Propagate registration
az provider register --namespace Microsoft.Compute
```

**Step 2 -- Create a VM with encryption at host:**

```bash
az vm create \
  --resource-group myRG \
  --name encVM \
  --image Win2022Datacenter \
  --size Standard_D4s_v5 \
  --admin-username azadmin \
  --admin-password 'P@ssw0rd1234!' \
  --encryption-at-host true
```

**Step 3 -- Enable on an existing VM (must be deallocated):**

```bash
az vm deallocate --resource-group myRG --name existingVM
az vm update --resource-group myRG --name existingVM \
  --set securityProfile.encryptionAtHost=true
az vm start --resource-group myRG --name existingVM
```

---

## 3. Move a Virtual Machine

**Reference:** [Move resources overview](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/move-resources-overview) | [VM move limitations](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/move-limitations/virtual-machines-move-limitations) | [Azure Resource Mover](https://learn.microsoft.com/en-us/azure/resource-mover/tutorial-move-region-virtual-machines)

### 3.1 Move to Another Resource Group (Same Subscription)

The simplest move. Use the portal (**Resource group --> Move resources**) or CLI:

```bash
az resource move \
  --destination-group newRG \
  --ids /subscriptions/{sub}/resourceGroups/oldRG/providers/Microsoft.Compute/virtualMachines/myVM
```

- All dependent resources (NIC, disk, NSG, public IP) must be moved together.
- The VM does **not** need to be stopped.
- The move is a metadata change -- the VM stays in the same region and on the same physical host.

### 3.2 Move to Another Subscription

Same as cross-RG move but with additional requirements:

- Both source and destination subscriptions must be in the same [Entra ID tenant](https://learn.microsoft.com/en-us/entra/fundamentals/whatis).
- The destination subscription must be registered for the resource provider (`Microsoft.Compute`).
- You need **write** permissions on both subscriptions.
- [Quota](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/azure-subscription-service-limits) in the target subscription must accommodate the VM size.
- A **validation** step runs before the actual move -- some resource types cannot be moved (e.g., VMs using [Azure Backup](https://learn.microsoft.com/en-us/azure/backup/backup-overview) with recovery points must have backups stopped first).

```bash
az resource move \
  --destination-group newRG \
  --destination-subscription-id {dest-sub-id} \
  --ids /subscriptions/{src-sub}/resourceGroups/oldRG/providers/Microsoft.Compute/virtualMachines/myVM
```

### 3.3 Move to Another Region

Cross-region moves are **not** a simple metadata change -- the VM must be re-created in the target region. Two approaches:

| Approach | How It Works | Best For |
|----------|-------------|----------|
| **[Azure Resource Mover](https://learn.microsoft.com/en-us/azure/resource-mover/overview)** | Dedicated hub for cross-region moves; handles dependency analysis, preparation, move, and commit steps | VMs, VNets, NSGs, public IPs, availability sets |
| **[Azure Site Recovery (ASR)](https://learn.microsoft.com/en-us/azure/site-recovery/azure-to-azure-tutorial-migrate)** | Replicates VM disks to the target region; you trigger a failover when ready | VMs where near-zero downtime is critical |

### 3.4 Move Constraints

| Constraint | Details |
|------------|---------|
| **VM extensions** | Extensions must be removed before cross-subscription move; re-install in the destination |
| **Managed disks** | Disks move with the VM for same-sub moves; for cross-region, Resource Mover creates new disks |
| **Availability sets** | Cannot move a VM into an AS after creation; the AS must be included in the move |
| **Load balancers** | Basic LBs move with the VM; Standard LBs may require manual re-creation |
| **Azure Backup** | Stop backup (retain data) before cross-sub move; re-enable afterward |

---

## 4. Manage Virtual Machine Sizes

**Reference:** [VM sizes overview](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview) | [Naming conventions](https://learn.microsoft.com/en-us/azure/virtual-machines/vm-naming-conventions)

### 4.1 Size Families

| Family | Category | Typical Use Cases |
|--------|----------|-------------------|
| **B-series** | General purpose (burstable) | Dev/test, low-traffic web servers, small databases; accumulates CPU credits during idle periods |
| **D-series** | General purpose | Balanced CPU-to-memory; enterprise apps, relational databases, mid-tier web servers |
| **E-series** | Memory optimized | In-memory analytics (SAP HANA), large caches, relational databases requiring high memory |
| **F-series** | Compute optimized | High CPU-to-memory ratio; batch processing, gaming servers, analytics, web servers |
| **N-series** (NC, ND, NV) | GPU optimized | ML training/inference (NC/ND), visualization/VDI (NV); subfamilies specify GPU type |
| **L-series** | Storage optimized | High disk throughput + low latency local NVMe; big data, NoSQL, data warehousing |
| **M-series** | Memory optimized (extreme) | Very large memory (up to 4 TiB); SAP HANA production workloads |
| **H-series** (HB, HC) | High-performance compute | MPI, finite element analysis, molecular dynamics, weather modelling |

### 4.2 Naming Convention

Format: `Standard_[Family][Subfamily][vCPUs][ConstrainedvCPUs][Features][Accelerator][Memory]_[Version]`

**Additive feature letters (lowercase):**

| Letter | Meaning |
|--------|---------|
| a | AMD-based processor |
| b | Remote storage bandwidth optimized |
| d | Local temp disk (ephemeral) |
| i | Isolated size (dedicated physical host) |
| l | Low memory |
| m | Memory intensive |
| n | Network optimized |
| p | ARM-based processor (Cobalt / Ampere Altra) |
| s | Compatible with Premium SSD storage |
| t | Tiny memory |

**Example breakdowns:**

| Size Name | Family | vCPUs | Features | Version |
|-----------|--------|-------|----------|---------|
| `Standard_D4s_v5` | D (general purpose) | 4 | s = Premium SSD capable | v5 |
| `Standard_E8ds_v5` | E (memory optimized) | 8 | d = local temp disk, s = Premium SSD | v5 |
| `Standard_B2ms` | B (burstable) | 2 | m = memory intensive, s = Premium SSD | (v1 implied) |
| `Standard_NC4as_T4_v3` | N (GPU), C subfamily | 4 | a = AMD, s = Premium SSD, T4 accelerator | v3 |

### 4.3 Resizing a VM

```bash
# List sizes available in the VM's current cluster (no deallocation needed)
az vm list-vm-resize-options \
  --resource-group myRG --name myVM --output table

# Resize (if the target size is in the current cluster)
az vm resize \
  --resource-group myRG --name myVM \
  --size Standard_D4s_v5

# If the new size is NOT available in the current cluster:
az vm deallocate --resource-group myRG --name myVM
az vm resize --resource-group myRG --name myVM --size Standard_E8s_v5
az vm start --resource-group myRG --name myVM
```

> **Key rule:** if the target size is available on the hardware cluster where the VM currently runs, you can resize without deallocation. Otherwise, the VM must be **deallocated** (stopped + released from the host), which changes its public IP (unless you use a static IP) and **loses temp disk data**.

**PowerShell:**

```powershell
$vm = Get-AzVM -ResourceGroupName "myRG" -Name "myVM"
$vm.HardwareProfile.VmSize = "Standard_D4s_v5"
Update-AzVM -VM $vm -ResourceGroupName "myRG"
```

---

## 5. Manage Virtual Machine Disks

**Reference:** [Managed disks overview](https://learn.microsoft.com/en-us/azure/virtual-machines/managed-disks-overview) | [Disk types](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-types)

### 5.1 Disk Types Comparison

| Disk Type | Media | Max Size | Max IOPS | Max Throughput | OS Disk? | Scenario |
|-----------|-------|----------|----------|----------------|----------|----------|
| **[Ultra Disk](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-types#ultra-disks)** | SSD | 64 TiB | 400,000 | 10,000 MB/s | No | IO-intensive: SAP HANA, top-tier databases |
| **[Premium SSD v2](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-types#premium-ssd-v2)** | SSD | 64 TiB | 80,000 | 1,200 MB/s | No | Production workloads needing tunable IOPS/throughput |
| **[Premium SSD](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-types#premium-ssds)** | SSD | 32 TiB | 20,000 | 900 MB/s | Yes | Production/performance-sensitive workloads |
| **[Standard SSD](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-types#standard-ssds)** | SSD | 32 TiB | 6,000 | 750 MB/s | Yes | Web servers, dev/test, light enterprise |
| **[Standard HDD](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-types#standard-hdds)** | HDD | 32 TiB | 2,000 (3,000*) | 500 MB/s | Yes (retiring Sept 2028) | Backup, non-critical, infrequent access |

\* 3,000 IOPS with performance plus enabled.

> **Ultra Disk and Premium SSD v2 cannot be used as OS disks.** Ultra Disks also cannot be used with [Azure Compute Gallery](https://learn.microsoft.com/en-us/azure/virtual-machines/azure-compute-gallery) or availability sets.

### 5.2 Disk Roles

| Role | Description | Delete Behaviour |
|------|-------------|------------------|
| **OS disk** | Contains the operating system; registered as SATA; max 4 TiB (Gen2) or 2 TiB (Gen1) | Deleted with VM by default (configurable) |
| **Data disk** | Stores application data; registered as SCSI; max count depends on VM size | Detached on VM delete by default |
| **Temp disk** | Ephemeral local storage (`/dev/sdb` on Linux, `D:\` on Windows); not a managed disk | **Data is lost** on stop/deallocate, resize, or host maintenance |

### 5.3 Snapshots

[Snapshots](https://learn.microsoft.com/en-us/azure/virtual-machines/snapshot-copy-managed-disk) are read-only, full point-in-time copies of a managed disk. Billed on actual data used (not provisioned size).

```bash
# Create a snapshot
az snapshot create \
  --resource-group myRG \
  --name mySnapshot \
  --source /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Compute/disks/myOsDisk

# Create a disk from a snapshot
az disk create \
  --resource-group myRG \
  --name newDisk \
  --source mySnapshot
```

### 5.4 Disk Operations with CLI

```bash
# Create a managed data disk (128 GiB Premium SSD)
az disk create \
  --resource-group myRG \
  --name myDataDisk \
  --size-gb 128 \
  --sku Premium_LRS

# Attach a data disk to a VM
az vm disk attach \
  --resource-group myRG \
  --vm-name myVM \
  --name myDataDisk

# Detach a data disk
az vm disk detach \
  --resource-group myRG \
  --vm-name myVM \
  --name myDataDisk
```

### 5.5 Disk Encryption Options Summary

| Option | Scope | Key Types | Default? | Notes |
|--------|-------|-----------|----------|-------|
| **SSE** | Persisted disk data | PMK (default), CMK | Yes | Automatic; all managed disks |
| **Encryption at host** | Temp disk + cache + persisted | PMK, CMK | No | See Section 2 |
| **ADE** | OS + data disks (guest-level) | CMK (Key Vault) | No | Retiring Sept 2028; mutually exclusive with encryption at host |
| **Confidential disk encryption** | OS disk via DM-Verity | PMK, CMK | No | For [confidential VMs](https://learn.microsoft.com/en-us/azure/confidential-computing/confidential-vm-overview) only |

---

## 6. Availability Zones and Availability Sets

**Reference:** [Availability options](https://learn.microsoft.com/en-us/azure/virtual-machines/availability) | [Availability zones](https://learn.microsoft.com/en-us/azure/reliability/availability-zones-overview) | [Availability sets](https://learn.microsoft.com/en-us/azure/virtual-machines/availability-set-overview)

### 6.1 Availability Zones (AZ)

[Availability zones](https://learn.microsoft.com/en-us/azure/reliability/availability-zones-overview) are **physically separate datacenter locations** within an Azure region, each with independent power, cooling, and networking.

- Each zone-enabled region has a **minimum of three zones**.
- Deploying VMs across 2+ zones provides **99.99% SLA** for VM connectivity.
- VMs are deployed as **zonal** (pinned to a specific zone, e.g., Zone 1) or, for some services, **zone-redundant** (platform spreads automatically).
- When you create a zonal VM you select the zone in the deployment -- you **cannot change the zone after creation**.

```bash
# Create a VM in availability zone 2
az vm create \
  --resource-group myRG \
  --name zoneVM \
  --image Ubuntu2204 \
  --size Standard_D2s_v5 \
  --zone 2 \
  --admin-username azureuser \
  --generate-ssh-keys
```

### 6.2 Availability Sets (AS)

An [availability set](https://learn.microsoft.com/en-us/azure/virtual-machines/availability-set-overview) is a logical grouping of VMs within a **single datacenter** that distributes VMs across:

- **Fault domains (FDs)** -- separate physical racks with independent power and network switches (max 3 FDs).
- **Update domains (UDs)** -- groups that reboot together during planned maintenance (max 20 UDs).

Deploying 2+ VMs in an AS provides **99.95% SLA**.

```bash
# Create an availability set
az vm availability-set create \
  --resource-group myRG \
  --name myAS \
  --platform-fault-domain-count 3 \
  --platform-update-domain-count 5

# Create a VM in the availability set
az vm create \
  --resource-group myRG \
  --name asVM \
  --image Win2022Datacenter \
  --size Standard_D2s_v5 \
  --availability-set myAS \
  --admin-username azadmin \
  --admin-password 'P@ssw0rd1234!'
```

### 6.3 Zones vs Sets -- When to Use Each

| Criteria | Availability Zones | Availability Sets |
|----------|--------------------|-------------------|
| **SLA** | 99.99% | 99.95% |
| **Protection level** | Datacenter-level failure | Rack-level failure (within one datacenter) |
| **Network latency** | Slightly higher (cross-datacenter) | Lower (same datacenter, close racks) |
| **Region support** | Only zone-enabled regions | All regions |
| **Can add VM after creation?** | No (zone is immutable) | **No** (AS must be specified at VM creation) |
| **Works with Ultra Disks?** | Yes | No |
| **Recommended for new deployments** | Yes (preferred) | Legacy -- use zones or VMSS Flexible instead |

### 6.4 Zone-Redundant vs Zonal Deployment

- **Zonal:** you explicitly assign a resource to Zone 1, 2, or 3. If that zone fails, the resource is unavailable. You deploy multiple VMs across zones for HA.
- **Zone-redundant:** the platform automatically replicates across all zones (e.g., ZRS storage, zone-redundant Standard Load Balancer). For VMs, you cannot get automatic zone-redundancy for individual VMs -- you achieve it via VMSS Flexible or manual multi-zone deployment behind a load balancer.

---

## 7. Virtual Machine Scale Sets (VMSS)

**Reference:** [VMSS overview](https://learn.microsoft.com/en-us/azure/virtual-machine-scale-sets/overview) | [Orchestration modes](https://learn.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-orchestration-modes)

### 7.1 Uniform vs Flexible Orchestration

| Feature | Uniform | Flexible (Recommended) |
|---------|---------|------------------------|
| **VM identity** | Scale set child VMs; not individually addressable via standard VM APIs | Standard Azure IaaS VMs; full VM API compatibility |
| **Instance management** | Managed via VMSS APIs only | Standard VM APIs (az vm, Get-AzVM) plus VMSS APIs |
| **Mix VM sizes** | No -- all instances identical | Yes -- mix VM types, Spot + on-demand |
| **Azure Backup / ASR** | Not supported on individual instances | Supported |
| **Max instances** | 1,000 (custom image) or 600 (Marketplace) | 1,000 |
| **Availability** | Spreads across FDs within a zone | Spreads across FDs in a region or within a zone |
| **Default (since Nov 2023)** | No | Yes (CLI/PowerShell default to Flexible) |

> **Uniform orchestration** is still supported but **Flexible is now the recommended mode** for all new deployments. Flexible gives you standard VM compatibility while retaining scale set features (autoscale, rolling upgrades, health probes).

### 7.2 Creating a Scale Set

```bash
# Create a Flexible VMSS with 2 initial instances across zones
az vmss create \
  --resource-group myRG \
  --name myVMSS \
  --image Ubuntu2204 \
  --vm-sku Standard_D2s_v5 \
  --instance-count 2 \
  --orchestration-mode Flexible \
  --zones 1 2 3 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --lb myLB \
  --lb-sku Standard
```

### 7.3 Autoscale Rules

[Autoscale](https://learn.microsoft.com/en-us/azure/azure-monitor/autoscale/autoscale-overview) adjusts instance count based on metrics or schedules.

**Metric-based autoscale:**

```bash
# Create autoscale setting: scale out when CPU > 75%, scale in when CPU < 25%
az monitor autoscale create \
  --resource-group myRG \
  --resource myVMSS \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name cpuAutoscale \
  --min-count 2 \
  --max-count 10 \
  --count 2

az monitor autoscale rule create \
  --resource-group myRG \
  --autoscale-name cpuAutoscale \
  --condition "Percentage CPU > 75 avg 10m" \
  --scale out 2

az monitor autoscale rule create \
  --resource-group myRG \
  --autoscale-name cpuAutoscale \
  --condition "Percentage CPU < 25 avg 10m" \
  --scale in 1
```

**Schedule-based autoscale:**

```bash
# Scale to 5 instances weekdays 08:00-18:00
az monitor autoscale profile create \
  --resource-group myRG \
  --autoscale-name cpuAutoscale \
  --name businessHours \
  --min-count 5 --max-count 10 --count 5 \
  --recurrence week Mon Tue Wed Thu Fri \
  --start 08:00 --end 18:00 --timezone "Eastern Standard Time"
```

### 7.4 Scale-In Policy

Controls which instances are removed during scale-in. Options:

| Policy | Behaviour |
|--------|-----------|
| **Default** | Balance across zones first, then remove the VM with the highest instance ID |
| **NewestVM** | Remove the newest VM (most recently created) |
| **OldestVM** | Remove the oldest VM (earliest created) |

### 7.5 Health Probes and Automatic Repairs

- **Application health extension** or **load balancer health probe** monitors instance health.
- [Automatic instance repairs](https://learn.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-automatic-instance-repairs) delete unhealthy instances and create new ones.
- A **grace period** (default 30 min) prevents repairs during initial provisioning.

### 7.6 Upgrade Policies

| Policy | Behaviour |
|--------|-----------|
| **Automatic** | All instances upgraded immediately when the model changes |
| **Rolling** | Instances upgraded in batches with configurable batch size, pause between batches, and max unhealthy percentage |
| **Manual** | No automatic upgrade; you trigger upgrades per instance |

For production workloads, **Rolling** upgrades with health probes provide zero-downtime deployments.

### 7.7 Extensions and Custom Images

- **Extensions** (Custom Script, DSC, Azure Monitor Agent) are defined in the VMSS model and applied to every new instance.
- **Custom images** from [Azure Compute Gallery](https://learn.microsoft.com/en-us/azure/virtual-machines/azure-compute-gallery) ensure consistent software across all instances. The gallery supports **image versioning** and **replication** to multiple regions.

---

## Exam Traps

| Trap | What to Remember |
|------|------------------|
| **ADE vs SSE vs Encryption at Host** | SSE is always on (persisted data only). Encryption at host adds temp disk + cache encryption. ADE is guest-level (BitLocker/dm-crypt). ADE and encryption at host are **mutually exclusive**. ADE is retiring Sept 2028. |
| **Availability Zone vs Set SLA** | AZ = **99.99%** (2+ VMs in 2+ zones). AS = **99.95%** (2+ VMs in an AS). Single VM with Premium SSD = 99.9%. |
| **Cannot add VM to AS after creation** | The availability set (or zone) must be specified at VM creation time. You cannot move an existing VM into an AS or change its zone. |
| **Temp disk data loss** | Temp disk data is **lost** on stop/deallocate, resize (if deallocation required), or host maintenance event. Never store persistent data on the temp disk. |
| **VMSS Uniform vs Flexible** | Flexible is the new default and recommended mode. Uniform VMs are not individually manageable via standard VM APIs. Exam may test whether Azure Backup works on VMSS instances (only with Flexible). |
| **Ultra Disk limitations** | Cannot be an OS disk. Cannot use with availability sets. Does not support disk caching. Only available in zones (or regional in specific regions). |
| **Premium SSD v2 limitations** | Cannot be an OS disk. Must be used with zonal VMs (if the region supports zones). Does not support host caching. |
| **Resizing requires deallocation?** | Only if the target size is not available on the current hardware cluster. Check `az vm list-vm-resize-options` first. |
| **Gen1 vs Gen2** | Gen2 supports Trusted launch, UEFI boot, and larger OS disks (4 TiB). Gen1-to-Gen2 upgrade is one-way. |
| **Moving VMs cross-subscription** | Must be same Entra ID tenant. Must stop backup first. Dependent resources (NIC, disk, NSG) must move together. |

---

## Sources

- [Create a Linux VM - Azure CLI](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-cli)
- [Create a Windows VM - Portal](https://learn.microsoft.com/en-us/azure/virtual-machines/windows/quick-create-portal)
- [Trusted launch for Azure VMs](https://learn.microsoft.com/en-us/azure/virtual-machines/trusted-launch)
- [Overview of managed disk encryption options](https://learn.microsoft.com/en-us/azure/virtual-machines/disk-encryption-overview)
- [Encryption at host -- enable via portal](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-enable-host-based-encryption-portal)
- [Server-side encryption of Azure Disk Storage](https://learn.microsoft.com/en-us/azure/virtual-machines/disk-encryption)
- [Move resources overview](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/move-resources-overview)
- [VM move limitations](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/move-limitations/virtual-machines-move-limitations)
- [Azure Resource Mover -- move VMs across regions](https://learn.microsoft.com/en-us/azure/resource-mover/tutorial-move-region-virtual-machines)
- [VM sizes overview](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/overview)
- [VM naming conventions](https://learn.microsoft.com/en-us/azure/virtual-machines/vm-naming-conventions)
- [Select a disk type for Azure IaaS VMs](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-types)
- [Managed disks overview](https://learn.microsoft.com/en-us/azure/virtual-machines/managed-disks-overview)
- [Availability options for Azure VMs](https://learn.microsoft.com/en-us/azure/virtual-machines/availability)
- [Availability zones overview](https://learn.microsoft.com/en-us/azure/reliability/availability-zones-overview)
- [Availability sets overview](https://learn.microsoft.com/en-us/azure/virtual-machines/availability-set-overview)
- [VMSS overview](https://learn.microsoft.com/en-us/azure/virtual-machine-scale-sets/overview)
- [VMSS orchestration modes](https://learn.microsoft.com/en-us/azure/virtual-machine-scale-sets/virtual-machine-scale-sets-orchestration-modes)
- [Autoscale overview](https://learn.microsoft.com/en-us/azure/azure-monitor/autoscale/autoscale-overview)
- [AZ-104 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104)
