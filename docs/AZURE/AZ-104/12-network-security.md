# Network Security (NSGs, Bastion, Endpoints)

> **Exam mapping:** AZ-104 -- "Implement and manage virtual networking" (15--20 %)
> **One-liner:** Azure provides layered network security through stateful packet filtering (NSGs), managed jump-box access (Bastion), and private connectivity to PaaS services (service endpoints and private endpoints) -- the exam tests rule evaluation order, Bastion SKU constraints, and the endpoint comparison heavily.
> **Related:** [11-virtual-networks](11-virtual-networks.md) | [13-dns-and-load-balancing](13-dns-and-load-balancing.md)

---

## 1. Network Security Groups (NSGs)

A [network security group](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview) (NSG) is a **stateful Layer 3/Layer 4 packet filter** that controls inbound and outbound traffic to Azure resources in a virtual network. "Stateful" means that if you allow an inbound request, the return traffic is automatically permitted without a separate outbound rule (and vice versa).

NSGs contain **security rules** evaluated by a five-tuple match: source, source port, destination, destination port, and protocol.

### 1.1 Default security rules

Azure creates the following immutable default rules in every NSG. You **cannot delete** them, but you can **override** them by creating custom rules with a higher priority (lower number).

#### Inbound defaults

| Rule name | Priority | Source | Destination | Port | Protocol | Action |
|-----------|----------|--------|-------------|------|----------|--------|
| **AllowVNetInBound** | 65000 | VirtualNetwork | VirtualNetwork | 0-65535 | Any | Allow |
| **AllowAzureLoadBalancerInBound** | 65001 | AzureLoadBalancer | 0.0.0.0/0 | 0-65535 | Any | Allow |
| **DenyAllInBound** | 65500 | 0.0.0.0/0 | 0.0.0.0/0 | 0-65535 | Any | **Deny** |

#### Outbound defaults

| Rule name | Priority | Source | Destination | Port | Protocol | Action |
|-----------|----------|--------|-------------|------|----------|--------|
| **AllowVnetOutBound** | 65000 | VirtualNetwork | VirtualNetwork | 0-65535 | Any | Allow |
| **AllowInternetOutBound** | 65001 | 0.0.0.0/0 | Internet | 0-65535 | Any | Allow |
| **DenyAllOutBound** | 65500 | 0.0.0.0/0 | 0.0.0.0/0 | 0-65535 | Any | **Deny** |

> `VirtualNetwork`, `AzureLoadBalancer`, and `Internet` in the table above are [service tags](https://learn.microsoft.com/en-us/azure/virtual-network/service-tags-overview), not literal IP addresses.

### 1.2 Custom rules

| Property | Details |
|----------|---------|
| **Priority** | 100--4096. Lower number = higher priority. Processing stops at the first match. |
| **Source / Destination** | Any, individual IP, CIDR block, [service tag](https://learn.microsoft.com/en-us/azure/virtual-network/service-tags-overview), or [application security group](https://learn.microsoft.com/en-us/azure/virtual-network/application-security-groups) (ASG). |
| **Port** | Single port, range (e.g. `10000-10005`), or comma-separated mix (`80, 443, 8000-8080`). |
| **Protocol** | TCP, UDP, ICMP, ESP, AH, or Any. (ESP and AH are ARM-template only, not portal.) |
| **Action** | Allow or Deny. |

Augmented security rules let you specify **multiple IPs, ranges, and ports** in a single rule (Resource Manager deployments only).

### 1.3 Association: subnet vs. NIC

An NSG can be associated to:

- **A subnet** -- applies to every resource in that subnet.
- **A NIC** -- applies to the specific VM's network interface.

You can associate **both** at the same time. Both must allow the traffic for it to flow.

### 1.4 Processing order (critical for the exam)

```
INBOUND traffic flow:
  Internet / other source
    --> Subnet NSG (evaluated first)
      --> NIC NSG (evaluated second)
        --> VM

OUTBOUND traffic flow:
  VM
    --> NIC NSG (evaluated first)
      --> Subnet NSG (evaluated second)
        --> Internet / destination
```

Within each NSG, rules are evaluated **lowest priority number first**. The first matching rule wins; no further rules are processed.

> **Key point:** if the subnet NSG denies traffic, the NIC NSG never sees it. Both NSGs must independently permit the traffic.

### 1.5 CLI reference

```bash
# Create an NSG
az network nsg create \
  --resource-group myRG \
  --name myNSG

# Create a custom rule (allow HTTPS inbound)
az network nsg rule create \
  --resource-group myRG \
  --nsg-name myNSG \
  --name AllowHTTPS \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes '*' \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 443

# Associate NSG to a subnet
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVNet \
  --name mySubnet \
  --network-security-group myNSG

# Associate NSG to a NIC
az network nic update \
  --resource-group myRG \
  --name myNIC \
  --network-security-group myNSG
```

---

## 2. Application Security Groups (ASGs)

An [application security group](https://learn.microsoft.com/en-us/azure/virtual-network/application-security-groups) (ASG) is a **logical grouping of NICs** that lets you write NSG rules referencing application roles instead of explicit IP addresses.

### 2.1 Why ASGs matter

Without ASGs, adding a new web server means updating every NSG rule that references web-tier IPs. With ASGs you assign the NIC to the `WebServers` ASG once, and all existing rules automatically apply.

### 2.2 Example workflow

```
1. Create ASGs:   WebServers, AppServers, DbServers
2. Assign NICs to ASGs (via NIC IP configuration)
3. Write NSG rules using ASGs as source/destination:
     Allow Inbound TCP/443 from Internet to WebServers
     Allow Inbound TCP/8080 from WebServers to AppServers
     Allow Inbound TCP/1433 from AppServers to DbServers
     Deny  Inbound Any   from Any to DbServers
```

### 2.3 Constraints (exam favorites)

| Constraint | Detail |
|------------|--------|
| All NICs in an ASG must be in the **same VNet**. | You cannot span ASGs across VNets. |
| Source ASG and destination ASG in the same rule must be in the **same VNet**. | Cross-VNet ASG references are not allowed. |
| You cannot mix an ASG with an IP range in the **same rule field** (source or destination). | Use one or the other per field. |
| An ASG can be associated with **multiple NICs**. | A NIC can belong to **multiple ASGs** (one per IP configuration). |

### 2.4 CLI reference

```bash
# Create an ASG
az network asg create \
  --resource-group myRG \
  --name WebServers

# Assign a NIC's IP config to the ASG
az network nic ip-config update \
  --resource-group myRG \
  --nic-name myNIC \
  --name ipconfig1 \
  --application-security-groups WebServers

# Use ASG in an NSG rule
az network nsg rule create \
  --resource-group myRG \
  --nsg-name myNSG \
  --name AllowWebToApp \
  --priority 200 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-asgs WebServers \
  --destination-asgs AppServers \
  --destination-port-ranges 8080
```

---

## 3. Evaluate Effective Security Rules

When NSGs are associated at both subnet and NIC levels, the **effective** rules are an aggregation of both. Troubleshooting requires inspecting the merged result.

### 3.1 Methods to check effective rules

| Method | How |
|--------|-----|
| **Portal** | VM --> Networking --> NIC --> **Effective security rules** tab |
| **Azure CLI** | `az network nic list-effective-nsg --resource-group myRG --name myNIC` |
| **PowerShell** | `Get-AzEffectiveNetworkSecurityGroup -NetworkInterfaceName myNIC -ResourceGroupName myRG` |

### 3.2 Network Watcher tools

[Azure Network Watcher](https://learn.microsoft.com/en-us/azure/network-watcher/overview) provides two key diagnostic capabilities:

| Tool | Purpose |
|------|---------|
| [**IP flow verify**](https://learn.microsoft.com/en-us/azure/network-watcher/ip-flow-verify-overview) | Tests whether a specific packet (source IP/port, destination IP/port, protocol) is allowed or denied. Returns the **name of the NSG rule** that matched. |
| [**NSG diagnostics**](https://learn.microsoft.com/en-us/azure/network-watcher/diagnose-network-security-rules) | Provides a detailed view of all rules evaluated for a traffic flow, showing the aggregated result and which specific rule allowed or denied the traffic. |

### 3.3 CLI: IP flow verify

```bash
az network watcher test-ip-flow \
  --resource-group myRG \
  --vm myVM \
  --direction Inbound \
  --protocol Tcp \
  --local 10.0.1.4:80 \
  --remote 203.0.113.5:60000

# Output: Access: Allow | Deny, RuleName: <name-of-matching-rule>
```

> **Exam tip:** when a question says "a VM cannot be reached," the answer is often: check effective security rules or run IP flow verify in Network Watcher.

---

## 4. Azure Bastion

[Azure Bastion](https://learn.microsoft.com/en-us/azure/bastion/bastion-overview) is a **fully managed PaaS jump-box service** that provides secure RDP and SSH connectivity to VMs **over TLS** through the Azure portal or native client -- without exposing a public IP on the target VM.

### 4.1 How it works

1. You deploy Bastion into a dedicated subnet (**AzureBastionSubnet**).
2. You connect via the Azure portal (or native client on Standard+) -- traffic flows over port 443 (TLS) to the Bastion host.
3. Bastion initiates the RDP/SSH session to the target VM's **private IP** on the backend.
4. The target VM **never needs a public IP address**.

### 4.2 SKU comparison table

| Feature | Developer | Basic | Standard | Premium |
|---------|-----------|-------|----------|---------|
| **AzureBastionSubnet required** | No | Yes (/26+) | Yes (/26+) | Yes (/26+) |
| **Public IP required** | No | Yes | Yes | No (private-only option) |
| **Dedicated host** | No (shared) | Yes | Yes | Yes |
| **VNet peering support** | No | Yes | Yes | Yes |
| **Concurrent connections** | 1 VM only | Multiple | Multiple | Multiple |
| **Host scaling (instances)** | N/A | 2 (fixed) | 2--50 | 2--50 |
| **Connect via portal** | Yes | Yes | Yes | Yes |
| **Native client (CLI)** | No | No | Yes | Yes |
| **File transfer** | No | No | Yes | Yes |
| **Shareable link** | No | No | Yes | Yes |
| **IP-based connect** | No | No | Yes | Yes |
| **Custom inbound port** | No | No | Yes | Yes |
| **Kerberos authentication** | Yes | Yes | Yes | Yes |
| **Session recording** | No | No | No | Yes |
| **Private-only deployment** | No | No | No | Yes |
| **Cost** | Free | Paid (hourly) | Paid (hourly) | Paid (hourly) |

### 4.3 Deployment requirements

- The dedicated subnet **must** be named exactly `AzureBastionSubnet`.
- Minimum subnet size is **/26** (64 addresses) for Basic/Standard/Premium. Larger is recommended for scaling (/25 or /24).
- Basic and Standard require a **Standard SKU, static-allocation public IP**.
- Developer SKU does **not** require a dedicated subnet or public IP (shared infrastructure, 1 VM at a time, dev/test only).
- SKU upgrades are supported (Developer --> Basic --> Standard --> Premium), but **downgrades are not** -- you must delete and recreate.

### 4.4 Capacity

| Metric | Developer | Basic | Standard / Premium |
|--------|-----------|-------|--------------------|
| Max concurrent RDP | 1 | 40 (2 x 20) | 1,000 (50 x 20) |
| Max concurrent SSH | 1 | 80 (2 x 40) | 2,000 (50 x 40) |

### 4.5 CLI reference

```bash
# Create the AzureBastionSubnet
az network vnet subnet create \
  --resource-group myRG \
  --vnet-name myVNet \
  --name AzureBastionSubnet \
  --address-prefixes 10.0.1.0/26

# Create a public IP for Bastion
az network public-ip create \
  --resource-group myRG \
  --name myBastionIP \
  --sku Standard \
  --allocation-method Static

# Create the Bastion host
az network bastion create \
  --resource-group myRG \
  --name myBastion \
  --public-ip-address myBastionIP \
  --vnet-name myVNet \
  --sku Standard \
  --scale-units 2
```

---

## 5. Service Endpoints

A [virtual network service endpoint](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-service-endpoints-overview) extends your **VNet identity** to Azure PaaS services over the Azure backbone network. Traffic no longer traverses the public internet.

### 5.1 How they work

1. You enable a service endpoint for a specific service on a subnet.
2. The subnet's **source IP switches from public to private** for traffic to that service.
3. You configure the PaaS resource's firewall to **allow traffic only from that VNet/subnet**.
4. Traffic flows over the **Azure backbone** -- never touches the public internet.

### 5.2 Supported services (generally available)

| Service | Provider namespace |
|---------|--------------------|
| [Azure Storage](https://learn.microsoft.com/en-us/azure/storage/common/storage-network-security) | `Microsoft.Storage` |
| Azure Storage (cross-region) | `Microsoft.Storage.Global` |
| [Azure SQL Database](https://learn.microsoft.com/en-us/azure/azure-sql/database/vnet-service-endpoint-rule-overview) / Azure Synapse | `Microsoft.Sql` |
| [Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/how-to-configure-vnet-service-endpoint) | `Microsoft.AzureCosmosDB` |
| [Azure Key Vault](https://learn.microsoft.com/en-us/azure/key-vault/general/overview-vnet-service-endpoints) | `Microsoft.KeyVault` |
| [Azure Service Bus](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-service-endpoints) | `Microsoft.ServiceBus` |
| [Azure Event Hubs](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-service-endpoints) | `Microsoft.EventHub` |
| [Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/app-service-ip-restrictions) | `Microsoft.Web` |
| Azure AI Services | `Microsoft.CognitiveServices` |
| [Azure Container Registry](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-vnet) | `Microsoft.ContainerRegistry` |

### 5.3 Limitations (exam tested)

| Limitation | Detail |
|------------|--------|
| **PaaS still uses a public IP** | The endpoint secures the route, but the PaaS resource retains its public endpoint. |
| **No on-premises access** | Service endpoints only work from VNet subnets. On-prem traffic must use the PaaS public IP (or use ExpressRoute Microsoft peering with NAT). |
| **Source IP switch** | Enabling an endpoint switches the source IP from public to private. Existing firewall rules using public IPs break -- update them **before** enabling the endpoint. |
| **No granular resource targeting** | A service endpoint is per-service, not per-resource instance (use [service endpoint policies](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-service-endpoint-policies-overview) to restrict to specific Storage accounts). |
| **DNS unchanged** | DNS entries still resolve to public IP addresses. |

### 5.4 CLI reference

```bash
# Enable service endpoint for Storage and SQL on a subnet
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVNet \
  --name mySubnet \
  --service-endpoints Microsoft.Storage Microsoft.Sql

# Add VNet rule to a storage account firewall
az storage account network-rule add \
  --resource-group myRG \
  --account-name mystorageacct \
  --vnet-name myVNet \
  --subnet mySubnet
```

---

## 6. Private Endpoints

A [private endpoint](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview) is a **network interface with a private IP address in your VNet** that connects to an Azure PaaS service (or your own service) via [Azure Private Link](https://learn.microsoft.com/en-us/azure/private-link/private-link-overview). Traffic never leaves the Microsoft network and uses a private IP end-to-end.

### 6.1 How they work

1. You create a private endpoint in a subnet -- Azure provisions a **NIC with a private IP** from that subnet.
2. The NIC maps to a specific PaaS resource instance (e.g., one particular storage account).
3. All traffic to that resource flows through the private IP -- the public endpoint can optionally be disabled entirely.
4. **DNS resolution** must be configured so the PaaS FQDN resolves to the private IP instead of the public IP.

### 6.2 Service endpoint vs. private endpoint (comparison table)

| Aspect | Service Endpoint | Private Endpoint |
|--------|-----------------|-----------------|
| **IP address used** | PaaS keeps its public IP; VNet source IP switches to private | Private IP from your VNet assigned to the PaaS resource |
| **Traffic path** | Azure backbone (optimized route) | Azure backbone via Private Link |
| **Granularity** | Per-service (all Storage, all SQL, etc.) | Per-resource instance (one specific storage account) |
| **On-premises access** | Not supported | Supported (over VPN/ExpressRoute -- traffic reaches private IP) |
| **DNS changes** | None (still resolves to public IP) | Required (FQDN must resolve to private IP) |
| **Cost** | Free | Charged (per hour + per GB data processed) |
| **Subnet config** | Enable endpoint on subnet | Deploy NIC into subnet |
| **Public endpoint** | Remains active | Can be disabled entirely |
| **Complexity** | Simple (one toggle per subnet) | More complex (NIC, DNS zone, approval) |
| **Microsoft recommendation** | Legacy approach | **Preferred** for secure, private access |

### 6.3 DNS integration (critical for the exam)

When you create a private endpoint, the PaaS FQDN (e.g., `mystorageacct.blob.core.windows.net`) must resolve to the **private IP**, not the public one. If DNS is wrong, traffic bypasses the private endpoint.

**Approaches:**

| Method | How it works |
|--------|-------------|
| [**Azure Private DNS Zone**](https://learn.microsoft.com/en-us/azure/dns/private-dns-overview) (recommended) | Create a zone (e.g., `privatelink.blob.core.windows.net`), link it to your VNet. Azure auto-creates an A record for the private endpoint's IP. The FQDN's CNAME chain resolves through the privatelink zone. |
| **Custom DNS server** | Configure conditional forwarding for the `privatelink.*` zone to Azure DNS (168.63.129.16). |
| **Hosts file** | Manual override on individual machines (not scalable, testing only). |

### 6.4 Approval workflow

| Mode | Behavior |
|------|----------|
| **Auto-approve** | If you are the owner/contributor of the target resource, the connection is approved automatically. |
| **Manual approve** | If the private endpoint is in a different tenant or you lack permissions, the resource owner must approve via portal / CLI. The endpoint stays in a `Pending` state until approved. |

### 6.5 Network policies on the private endpoint subnet

By default, [network policies](https://learn.microsoft.com/en-us/azure/private-link/disable-private-endpoint-network-policy) (NSGs, UDRs) are **disabled** on private endpoints. You can explicitly enable them:

```bash
# Enable NSG support on a private-endpoint subnet
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVNet \
  --name peSubnet \
  --private-endpoint-network-policies Enabled
```

Once enabled, you can apply NSG rules to control traffic to/from private endpoints.

### 6.6 CLI reference

```bash
# Create a private endpoint for a storage account (blob)
az network private-endpoint create \
  --resource-group myRG \
  --name myStoragePE \
  --vnet-name myVNet \
  --subnet peSubnet \
  --private-connection-resource-id "/subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageacct" \
  --group-id blob \
  --connection-name myStorageConnection

# Create the private DNS zone
az network private-dns zone create \
  --resource-group myRG \
  --name privatelink.blob.core.windows.net

# Link DNS zone to VNet
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name privatelink.blob.core.windows.net \
  --name myDNSLink \
  --virtual-network myVNet \
  --registration-enabled false

# Create DNS record for the private endpoint
az network private-endpoint dns-zone-group create \
  --resource-group myRG \
  --endpoint-name myStoragePE \
  --name myZoneGroup \
  --private-dns-zone privatelink.blob.core.windows.net \
  --zone-name blob
```

---

## Exam Traps

1. **NSG processing order:** inbound = subnet NSG first, then NIC NSG. Outbound = NIC NSG first, then subnet NSG. Both must allow the traffic.
2. **Default rules cannot be deleted**, only overridden with higher-priority custom rules (lower priority number).
3. **Default rules use priority 65000--65500.** Custom rules use 100--4096. There is no overlap, so custom rules always evaluate first.
4. **NSGs are stateful.** If outbound traffic is allowed, the return inbound traffic is automatically permitted (no explicit inbound rule needed for the response).
5. **ASG constraints:** all NICs in an ASG and all ASGs referenced in a single rule must be in the **same VNet**.
6. **Bastion subnet must be named `AzureBastionSubnet`** and be **/26 or larger** (for Basic/Standard/Premium).
7. **Bastion Developer SKU** is free but limited to 1 VM connection, no VNet peering, no dedicated host -- **not for production**.
8. **Bastion SKU downgrades are not supported** -- you must delete and recreate.
9. **Service endpoints still use the PaaS public IP** -- they secure the route, not the endpoint address. Private endpoints provide a true private IP.
10. **Service endpoints do not work from on-premises.** Private endpoints do (via VPN/ExpressRoute).
11. **Private endpoint DNS is critical.** If you do not configure a Private DNS Zone (or equivalent), the FQDN resolves to the public IP and traffic bypasses the private endpoint.
12. **Network policies (NSGs/UDRs) are disabled by default on private endpoint subnets** -- you must explicitly enable them if you want NSG filtering on private endpoint traffic.
13. **Service endpoint = per-service; private endpoint = per-resource instance.** A service endpoint for `Microsoft.Storage` covers all storage accounts, whereas a private endpoint maps to one specific account.

---

## Sources

- [Network security groups overview](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview)
- [How network security groups filter network traffic](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-group-how-it-works)
- [Application security groups](https://learn.microsoft.com/en-us/azure/virtual-network/application-security-groups)
- [Create, change, or delete a network security group](https://learn.microsoft.com/en-us/azure/virtual-network/manage-network-security-group)
- [NSG diagnostics -- Network Watcher](https://learn.microsoft.com/en-us/azure/network-watcher/diagnose-network-security-rules)
- [IP flow verify -- Network Watcher](https://learn.microsoft.com/en-us/azure/network-watcher/ip-flow-verify-overview)
- [What is Azure Bastion?](https://learn.microsoft.com/en-us/azure/bastion/bastion-overview)
- [Azure Bastion SKU comparison](https://learn.microsoft.com/en-us/azure/bastion/bastion-sku-comparison)
- [Azure Bastion configuration settings](https://learn.microsoft.com/en-us/azure/bastion/configuration-settings)
- [Virtual network service endpoints](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-service-endpoints-overview)
- [What is a private endpoint?](https://learn.microsoft.com/en-us/azure/private-link/private-endpoint-overview)
- [Azure Private Link overview](https://learn.microsoft.com/en-us/azure/private-link/private-link-overview)
- [Azure Private DNS zones](https://learn.microsoft.com/en-us/azure/dns/private-dns-overview)
- [Manage network policies for private endpoints](https://learn.microsoft.com/en-us/azure/private-link/disable-private-endpoint-network-policy)
- [Service tags overview](https://learn.microsoft.com/en-us/azure/virtual-network/service-tags-overview)
- [az network nsg CLI reference](https://learn.microsoft.com/en-us/cli/azure/network/nsg?view=azure-cli-latest)
