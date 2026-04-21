# Azure Virtual Networks

> **Exam mapping:** AZ-104 → "Implement and manage virtual networking" (15–20%)
> **One-liner:** Virtual networks are the fundamental building block for private networking in Azure — they provide isolation, segmentation, and communication for Azure resources.
> **Related:** [12-network-security](12-network-security.md) | [13-dns-and-load-balancing](13-dns-and-load-balancing.md)

---

## 1. Virtual Networks and Subnets

### What is a Virtual Network (VNet)?

An Azure [Virtual Network (VNet)](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview) is a logically isolated network in Azure that enables resources to securely communicate with each other, the internet, and on-premises networks.

**Core purposes:**

| Purpose | Description |
|---|---|
| **Isolation** | Each VNet is isolated from other VNets. Resources in different VNets cannot communicate by default. |
| **Segmentation** | Subnets divide a VNet into smaller address ranges for organizing resources and applying security policies. |
| **Communication** | Resources within the same VNet communicate freely. Cross-VNet requires peering, VPN, or ExpressRoute. |
| **Filtering** | NSGs and Azure Firewall filter traffic between subnets and to/from the internet. |
| **Routing** | Azure automatically routes traffic between subnets, VNets, on-premises, and the internet. |

### Address Space (CIDR Notation)

A VNet requires one or more address spaces defined in [CIDR notation](https://learn.microsoft.com/en-us/azure/virtual-network/manage-virtual-network). Use **RFC 1918** private ranges:

| Range | CIDR | Addresses |
|---|---|---|
| 10.0.0.0 – 10.255.255.255 | 10.0.0.0/8 | ~16.7 million |
| 172.16.0.0 – 172.31.255.255 | 172.16.0.0/12 | ~1 million |
| 192.168.0.0 – 192.168.255.255 | 192.168.0.0/16 | ~65,000 |

You can add multiple address spaces to a single VNet and modify them after creation (as long as no subnets overlap with the removed range).

### Reserved Addresses per Subnet

Azure reserves **5 IP addresses** in every subnet — the first four and the last:

| Reserved address | Purpose |
|---|---|
| `x.x.x.0` | Network address |
| `x.x.x.1` | Default gateway |
| `x.x.x.2, x.x.x.3` | Azure DNS mapping |
| `x.x.x.255` (last) | Broadcast address |

**Example:** A `/29` subnet (8 addresses) has only **3 usable** host addresses. The smallest allowed subnet is `/29`.

> **Exam trap:** Questions will test whether you know the count is 5, not 2 or 3. A /28 (16 addresses) yields 11 usable.

### Creating VNets and Subnets

**Portal:** Virtual networks → Create → specify name, region, address space, and initial subnet.

**CLI:**

```bash
# Create a VNet
az network vnet create \
  --resource-group myRG \
  --name myVNet \
  --address-prefixes 10.0.0.0/16 \
  --subnet-name default \
  --subnet-prefixes 10.0.0.0/24 \
  --location eastus

# Add another subnet
az network vnet subnet create \
  --resource-group myRG \
  --vnet-name myVNet \
  --name backend \
  --address-prefixes 10.0.1.0/24
```

### Special / Default Subnets

Certain Azure services require dedicated subnets with **exact naming**:

| Subnet name | Required by | Notes |
|---|---|---|
| `GatewaySubnet` | VPN Gateway, ExpressRoute Gateway | Must be named exactly `GatewaySubnet`. Recommended /27 or larger. |
| `AzureBastionSubnet` | Azure Bastion | Must be named exactly `AzureBastionSubnet`. Minimum /26. |
| `AzureFirewallSubnet` | Azure Firewall | Must be named exactly `AzureFirewallSubnet`. Minimum /26. |
| `AzureFirewallManagementSubnet` | Azure Firewall (forced tunneling) | Dedicated management traffic subnet. |
| `RouteServerSubnet` | Azure Route Server | Minimum /27. |

### Subnet Delegation

[Subnet delegation](https://learn.microsoft.com/en-us/azure/virtual-network/subnet-delegation-overview) grants a specific Azure PaaS service permissions to create service-specific resources in the subnet (e.g., `Microsoft.Sql/managedInstances`, `Microsoft.Web/serverFarms`). A subnet can only be delegated to one service at a time.

---

## 2. Virtual Network Peering

### What is VNet Peering?

[VNet peering](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview) connects two Azure VNets so resources in either network can communicate with each other using private IP addresses. Traffic between peered VNets uses the Microsoft backbone network — **low latency, high bandwidth, no public internet**.

### Key Characteristics

- **Non-transitive:** If VNet-A peers with VNet-B and VNet-B peers with VNet-C, VNet-A **cannot** reach VNet-C through VNet-B unless you explicitly configure routing (e.g., via NVA or Azure Route Server).
- **Non-overlapping address spaces required** — peering fails if CIDRs overlap.
- **Bidirectional setup** — you must create a peering link on **both** VNets (the portal does this automatically; CLI requires two commands).

### Types of Peering

| Type | Scope | Latency |
|---|---|---|
| **Regional (VNet) peering** | Same Azure region | Lowest — same backbone fabric |
| **Global VNet peering** | Different Azure regions | Slightly higher — cross-region backbone |

> Global peering has some limitations: you cannot use a remote VNet gateway for transit through global peering in certain legacy configurations. Check current docs for your scenario.

### Peering Settings

| Setting | Effect |
|---|---|
| **Allow forwarded traffic** | Accept traffic that did not originate from the peered VNet (e.g., traffic forwarded by an NVA in the peer). |
| **Allow gateway transit** | Let the peered VNet use your VPN/ExpressRoute gateway. Set on the **hub** VNet (the one with the gateway). |
| **Use remote gateway** | Use the peered VNet's gateway instead of deploying your own. Set on the **spoke** VNet. Cannot be set if the spoke already has a gateway. |

**Hub-and-spoke pattern:** The hub VNet has the gateway and enables "Allow gateway transit". Each spoke enables "Use remote gateway". This avoids deploying a gateway in every spoke.

### Peering Status

| Status | Meaning |
|---|---|
| **Initiated** | Peering created on one side only — not yet functional. |
| **Connected** | Both sides configured — peering is active. |
| **Disconnected** | A previously connected peering was broken (e.g., one side deleted). |

### CLI

```bash
# Peer VNet-A → VNet-B
az network vnet peering create \
  --resource-group myRG \
  --name AtoB \
  --vnet-name VNetA \
  --remote-vnet /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/VNetB \
  --allow-vnet-access

# Peer VNet-B → VNet-A (must do both sides)
az network vnet peering create \
  --resource-group myRG \
  --name BtoA \
  --vnet-name VNetB \
  --remote-vnet /subscriptions/<sub>/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/VNetA \
  --allow-vnet-access
```

---

## 3. Public IP Addresses

### SKU Comparison: Basic vs Standard

> **Important:** Basic SKU public IPs were **retired on September 30, 2025**. New deployments use Standard SKU only. The exam may still test conceptual differences.

| Feature | Basic SKU (retired) | Standard SKU |
|---|---|---|
| **Allocation** | Static or Dynamic | **Static only** |
| **Availability zones** | Not supported | Zone-redundant or zonal |
| **Security** | Open by default | **Secure by default** — requires NSG to allow inbound |
| **Load balancer** | Basic LB only | Standard LB (required) |
| **Redundancy** | No zone redundancy | Zone-redundant by default |
| **Routing preference** | Not supported | Microsoft network or Internet routing |
| **Pricing** | Lower | Slightly higher, more features |
| **Supported resources** | NICs, VPN Gateways, App Gateways | NICs, Standard LB, VPN Gateways, App Gateways, Bastion, Firewall, NAT Gateway |

> **Exam trap:** Standard SKU public IPs are **static only** and **secure by default** (inbound traffic is denied until an NSG explicitly allows it). This is the most-tested difference.

### Allocation Method

| Method | Behavior |
|---|---|
| **Static** | IP assigned at creation and remains fixed until the resource is deleted. |
| **Dynamic** | IP assigned when associated with a running resource; may change on stop/deallocate. Standard SKU does NOT support dynamic. |

### Association Targets

A public IP can be associated with:

- **Network Interface (NIC)** — VM internet access
- **Standard Load Balancer** — frontend IP
- **VPN Gateway** — public endpoint for VPN
- **Application Gateway** — frontend listener
- **Azure Bastion** — management access
- **NAT Gateway** — outbound internet for a subnet
- **Azure Firewall** — inbound/outbound filtering

### Public IP Prefix

A [public IP prefix](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/public-ip-address-prefix) reserves a **contiguous range** of public IPs. Useful when you need predictable IPs for firewall allow-listing.

### CLI

```bash
# Create a Standard static public IP
az network public-ip create \
  --resource-group myRG \
  --name myPublicIP \
  --sku Standard \
  --allocation-method Static \
  --location eastus \
  --zone 1 2 3  # zone-redundant
```

---

## 4. User-Defined Routes (UDR)

### System Routes vs Custom Routes

Azure automatically creates [system routes](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview) to handle traffic within a VNet, between peered VNets, to the internet, and through gateways. You cannot delete system routes, but you can **override** them with user-defined routes (UDR).

Route selection follows **longest prefix match**, and when prefix lengths are equal, the priority is:

1. **User-defined routes (UDR)**
2. **BGP routes**
3. **System routes**

### Route Table

A [route table](https://learn.microsoft.com/en-us/azure/virtual-network/manage-route-table) is a collection of custom routes. You create a route table and then associate it to one or more subnets. Each subnet can have at most **one** route table, but a route table can be associated to multiple subnets.

### Next Hop Types

| Next hop type | Use case |
|---|---|
| **Virtual appliance** | Route traffic through an NVA (e.g., firewall). Requires a next hop IP address. |
| **Virtual network gateway** | Send traffic to a VPN or ExpressRoute gateway. |
| **Virtual network** | Override the default route within the VNet's address space. |
| **Internet** | Route traffic to the internet (can override a 0.0.0.0/0 route). |
| **None** | Drop (black-hole) the traffic. |

> **Service tags as prefix:** You can use [service tags](https://learn.microsoft.com/en-us/azure/virtual-network/service-tags-overview) (e.g., `AzureCloud`, `Storage`) as the address prefix in a UDR instead of an explicit CIDR. Azure manages the underlying IP ranges automatically.

### Forced Tunneling

Forced tunneling redirects **all** internet-bound traffic (0.0.0.0/0) to an on-premises appliance or NVA for inspection. Implemented by creating a UDR with:

- **Address prefix:** `0.0.0.0/0`
- **Next hop type:** Virtual network gateway (for on-prem) or Virtual appliance (for NVA)

### BGP Route Propagation

By default, a route table inherits BGP routes from on-premises via a VPN/ExpressRoute gateway. You can **disable BGP route propagation** on a route table to prevent gateway-learned routes from being injected into subnets.

### CLI

```bash
# Create a route table
az network route-table create \
  --resource-group myRG \
  --name myRouteTable \
  --location eastus \
  --disable-bgp-route-propagation false

# Add a route (force traffic through NVA)
az network route-table route create \
  --resource-group myRG \
  --route-table-name myRouteTable \
  --name toNVA \
  --address-prefix 10.1.0.0/16 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.0.1.4

# Associate route table to a subnet
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVNet \
  --name backend \
  --route-table myRouteTable
```

---

## 5. Troubleshoot Network Connectivity

### Azure Network Watcher

[Network Watcher](https://learn.microsoft.com/en-us/azure/network-watcher/network-watcher-overview) is a regional service that provides a suite of diagnostic and visualization tools. It is automatically enabled per region when you have a VNet.

### Key Diagnostic Tools

| Tool | What it does | When to use |
|---|---|---|
| **[IP flow verify](https://learn.microsoft.com/en-us/azure/network-watcher/ip-flow-verify-overview)** | Tests if a packet is allowed or denied to/from a VM; tells you **which NSG rule** matched. | "Why can't my VM receive traffic on port 443?" |
| **[Next hop](https://learn.microsoft.com/en-us/azure/network-watcher/next-hop-overview)** | Shows the next hop for a packet from a VM — reveals effective route. | "Where is my traffic being routed?" |
| **[Connection troubleshoot](https://learn.microsoft.com/en-us/azure/network-watcher/connection-troubleshoot-overview)** | Tests TCP/ICMP connectivity from a source to a destination; reports latency, reachability, and blocking rules. | "Can VM-A reach VM-B on port 3389?" |
| **NSG diagnostics** | Evaluates all NSG rules applied to a NIC/subnet and tells you the aggregate allow/deny result. | "What is the effective security posture for this NIC?" |
| **Packet capture** | Captures packets on a VM NIC for analysis (saved to storage account or local file). | Deep-dive protocol-level troubleshooting. |
| **Connection monitor** | Continuous monitoring of connectivity between endpoints; alerts on degradation. | Long-term reachability and latency monitoring. |
| **Topology** | Visual map of resources and their relationships in a VNet. | Understanding the network layout. |

### Effective Routes and Effective Security Rules

These views (available on a NIC resource) show the **merged result** of all system routes + UDRs + BGP routes (effective routes) and all NSG rules from NIC-level and subnet-level NSGs (effective security rules).

```bash
# Show effective routes for a NIC
az network nic show-effective-route-table \
  --resource-group myRG \
  --name myNIC \
  --output table

# Show effective NSG rules for a NIC
az network nic list-effective-nsg \
  --resource-group myRG \
  --name myNIC
```

### Systematic Troubleshooting Approach

1. **Identify the symptom** — Cannot connect? Intermittent? High latency?
2. **Check NSGs** — Use IP flow verify or effective security rules to confirm traffic is not blocked.
3. **Check routing** — Use next hop or effective routes to verify the packet reaches the correct destination.
4. **Check the destination** — Is the service running? Is the OS firewall (Windows Firewall / iptables) blocking?
5. **Check DNS** — Use `nslookup` or `dig` to verify name resolution. Check if custom DNS servers are configured on the VNet.
6. **Check peering** — Is the peering status "Connected"? Are the right settings enabled (allow forwarded traffic, gateway transit)?
7. **Capture packets** — If all else fails, use packet capture for protocol-level analysis.

### DNS Troubleshooting

- **Default Azure DNS** (168.63.129.16) resolves Azure-provided names and public DNS.
- **Custom DNS** configured on the VNet overrides Azure DNS — ensure the custom server is reachable and forwards correctly.
- Use `nslookup <hostname> 168.63.129.16` from inside the VNet to test Azure-provided resolution.

---

## Exam Traps

| Trap | Detail |
|---|---|
| **Peering is non-transitive** | A ↔ B and B ↔ C does NOT mean A can reach C. You need A ↔ C peering or NVA/Route Server for transit. |
| **5 reserved IPs per subnet** | First 4 + last 1 = 5 reserved. A /29 gives only 3 usable. A /28 gives 11 usable. |
| **Basic vs Standard IP SKU** | Standard is static-only, secure-by-default (needs NSG), zone-redundant. Basic was retired Sep 2025. |
| **UDR next hop types** | Five types: Virtual appliance, Virtual network gateway, Virtual network, Internet, None. Know them cold. |
| **Gateway transit requires a gateway** | "Allow gateway transit" is set on the hub (which must have a gateway). "Use remote gateway" is set on the spoke. Cannot enable both on the same VNet. |
| **Non-overlapping address spaces** | VNet peering fails if address spaces overlap. Plan CIDR ranges carefully. |
| **Forced tunneling** | A 0.0.0.0/0 UDR overrides the default internet route. All internet traffic goes to the specified next hop. |
| **Subnet delegation** | A subnet can be delegated to only one service at a time. |
| **Special subnet names** | `GatewaySubnet`, `AzureBastionSubnet`, `AzureFirewallSubnet` must be named exactly. |
| **Route priority** | UDR > BGP > System routes (when prefix length is equal). Longest prefix match wins first. |
| **Effective routes vs route table** | The route table shows your custom routes; effective routes on a NIC show the merged result of system + UDR + BGP. |

---

## Sources

- [Azure Virtual Network overview](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview)
- [Create, change, or delete a virtual network](https://learn.microsoft.com/en-us/azure/virtual-network/manage-virtual-network)
- [Manage subnets](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-manage-subnet)
- [Virtual network peering](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview)
- [Configure VPN gateway transit for peering](https://learn.microsoft.com/en-us/azure/vpn-gateway/vpn-gateway-peering-gateway-transit)
- [Public IP addresses](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/public-ip-addresses)
- [Upgrade Basic to Standard public IP](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/public-ip-basic-upgrade-guidance)
- [Virtual network traffic routing (UDR)](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview)
- [Manage route tables](https://learn.microsoft.com/en-us/azure/virtual-network/manage-route-table)
- [Network Watcher overview](https://learn.microsoft.com/en-us/azure/network-watcher/network-watcher-overview)
- [IP flow verify](https://learn.microsoft.com/en-us/azure/network-watcher/ip-flow-verify-overview)
- [Next hop](https://learn.microsoft.com/en-us/azure/network-watcher/next-hop-overview)
- [Connection troubleshoot](https://learn.microsoft.com/en-us/azure/network-watcher/connection-troubleshoot-overview)
- [AZ-104 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104)
- [AZ-104 training path: Configure and manage virtual networks](https://learn.microsoft.com/en-us/training/paths/az-104-manage-virtual-networks/)
