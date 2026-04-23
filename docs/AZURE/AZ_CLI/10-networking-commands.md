# Azure CLI — Networking Commands

> **`az network` across vnets, NSGs, load balancers, application gateways, private endpoints, DNS, VPN, and firewall.**
> **Related:** [AZ-104 / 11 Virtual networks](../AZ-104/11-virtual-networks.md) · [AZ-104 / 12 Network security](../AZ-104/12-network-security.md) · [AZ-104 / 13 DNS & load balancing](../AZ-104/13-dns-and-load-balancing.md)
> **Sources:** [`az network`](https://learn.microsoft.com/en-us/cli/azure/network)

---

## Table of contents

1. [Virtual networks & subnets](#virtual-networks--subnets)
2. [Peering](#peering)
3. [NSG (network security groups)](#nsg-network-security-groups)
4. [Application security groups](#application-security-groups)
5. [Public IPs & NAT Gateway](#public-ips--nat-gateway)
6. [Load Balancer](#load-balancer)
7. [Application Gateway](#application-gateway)
8. [Private endpoints & Private Link](#private-endpoints--private-link)
9. [Private DNS](#private-dns)
10. [Public DNS (`az network dns`)](#public-dns-az-network-dns)
11. [VPN Gateway & ExpressRoute](#vpn-gateway--expressroute)
12. [Azure Firewall](#azure-firewall)
13. [Network Watcher](#network-watcher)

---

## Virtual networks & subnets

```bash
# Create vnet + initial subnet
az network vnet create \
  -g rg-dev -n vnet-dev \
  --address-prefixes 10.0.0.0/16 \
  --subnet-name subnet-app --subnet-prefixes 10.0.1.0/24 \
  --location eastus

# Additional subnets
az network vnet subnet create \
  -g rg-dev --vnet-name vnet-dev \
  --name subnet-db --address-prefixes 10.0.2.0/24 \
  --service-endpoints Microsoft.Sql Microsoft.Storage \
  --delegations Microsoft.DBforPostgreSQL/flexibleServers

# Inspect
az network vnet list -o table
az network vnet show -g rg-dev -n vnet-dev
az network vnet subnet list -g rg-dev --vnet-name vnet-dev -o table
az network vnet subnet show -g rg-dev --vnet-name vnet-dev -n subnet-app

# Update address space
az network vnet update -g rg-dev -n vnet-dev --address-prefixes 10.0.0.0/16 10.1.0.0/16

# Delete
az network vnet subnet delete -g rg-dev --vnet-name vnet-dev -n subnet-db
az network vnet delete -g rg-dev -n vnet-dev
```

- `--service-endpoints` = subnet-to-service routing over the Azure backbone (Storage, SQL, Key Vault, ...).
- `--delegations` are required for some managed services (Postgres Flexible, AKS node subnet for some CNI configs, Container Instances).
- Can't delete a vnet that still has NICs, private endpoints, or gateways attached.

## Peering

```bash
# Peer A → B
az network vnet peering create \
  -g rg-a -n peer-a-to-b \
  --vnet-name vnet-a --remote-vnet vnet-b-id-or-name \
  --allow-vnet-access --allow-forwarded-traffic

# Reverse direction (same cmd against the other vnet)
az network vnet peering create \
  -g rg-b -n peer-b-to-a \
  --vnet-name vnet-b --remote-vnet vnet-a-id-or-name \
  --allow-vnet-access --allow-forwarded-traffic

# With hub-spoke (gateway transit)
az network vnet peering create ... --use-remote-gateways
az network vnet peering create ... --allow-gateway-transit
```

- `--remote-vnet` accepts name (same subscription) or resource ID (cross-sub / cross-tenant).
- Peerings are **unidirectional records** — you need both ends. Status becomes `Connected` only when both sides exist.

## NSG (network security groups)

```bash
az network nsg create -g rg-dev -n nsg-app
az network nsg list -o table
az network nsg show -g rg-dev -n nsg-app

# Add a rule
az network nsg rule create \
  -g rg-dev --nsg-name nsg-app \
  --name allow-https --priority 100 --direction Inbound \
  --access Allow --protocol Tcp \
  --source-address-prefixes Internet --source-port-ranges '*' \
  --destination-address-prefixes VirtualNetwork --destination-port-ranges 443

# Deny rule (higher priority = evaluated first — lower number = higher priority)
az network nsg rule create \
  -g rg-dev --nsg-name nsg-app \
  --name deny-inbound-smb --priority 90 --direction Inbound \
  --access Deny --protocol Tcp \
  --source-address-prefixes '*' --destination-port-ranges 445

# Inspect ordered rules
az network nsg rule list -g rg-dev --nsg-name nsg-app \
  --query "sort_by([], &priority)[].{prio:priority, name:name, access:access, src:sourceAddressPrefix, dst:destinationAddressPrefix, port:destinationPortRange}" -o table

# Associate to subnet or NIC
az network vnet subnet update \
  -g rg-dev --vnet-name vnet-dev -n subnet-app \
  --network-security-group nsg-app

az network nic update \
  -g rg-dev -n vm-web-nic --network-security-group nsg-app
```

- Priority range: 100–4096. Default rules (65000–65500) allow intra-vnet and allow-outbound-internet.
- `VirtualNetwork`, `AzureLoadBalancer`, `Internet`, `Storage`, `Sql`, `AzureActiveDirectory` are [service tags](https://learn.microsoft.com/en-us/azure/virtual-network/service-tags-overview) — use them instead of IP ranges where available.

## Application security groups

ASGs let you tag NICs logically and use the tag in NSG rules.

```bash
az network asg create -g rg-dev -n asg-web
az network nic update -g rg-dev -n vm-web-nic \
  --ip-configurations ipconfig1 --application-security-groups asg-web

az network nsg rule create -g rg-dev --nsg-name nsg-app \
  --name allow-web-from-lb --priority 110 \
  --access Allow --protocol Tcp \
  --source-address-prefixes AzureLoadBalancer \
  --destination-asgs asg-web --destination-port-ranges 443
```

## Public IPs & NAT Gateway

```bash
# Public IP (Standard SKU = zonal/zone-redundant)
az network public-ip create \
  -g rg-dev -n pip-app \
  --allocation-method Static --sku Standard \
  --zone 1 2 3 --version IPv4 \
  --dns-name app-dev-eastus

# NAT Gateway (SNAT for outbound from subnets without public IPs on VMs)
az network nat gateway create \
  -g rg-dev -n ngw-app \
  --public-ip-addresses pip-ngw \
  --idle-timeout 10

az network vnet subnet update \
  -g rg-dev --vnet-name vnet-dev -n subnet-app \
  --nat-gateway ngw-app
```

- `--sku Standard` + explicit zones = zone-redundant public IP — required pairing for zone-redundant LBs / App Gateways.
- Basic SKU public IPs are [retiring](https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/public-ip-basic-upgrade-guidance) — use Standard in new work.

## Load Balancer

```bash
# Create Standard LB
az network lb create \
  -g rg-dev -n lb-web \
  --sku Standard \
  --frontend-ip-name feip-web --public-ip-address pip-app \
  --backend-pool-name bp-web

# Probe
az network lb probe create \
  -g rg-dev --lb-name lb-web -n probe-https \
  --protocol Https --port 443 --path /health --interval 5 --threshold 2

# Rule
az network lb rule create \
  -g rg-dev --lb-name lb-web -n rule-https \
  --protocol Tcp --frontend-port 443 --backend-port 443 \
  --frontend-ip-name feip-web --backend-pool-name bp-web \
  --probe-name probe-https \
  --idle-timeout 15 --floating-ip false

# Add backends
az network lb address-pool address add \
  -g rg-dev --lb-name lb-web --pool-name bp-web \
  --name vm-web-1 --vnet vnet-dev --ip-address 10.0.1.10

# Inbound NAT rule (for per-VM SSH/RDP via LB)
az network lb inbound-nat-rule create \
  -g rg-dev --lb-name lb-web -n ssh-vm1 \
  --protocol Tcp --frontend-port 50001 --backend-port 22 \
  --frontend-ip-name feip-web

# Outbound rule (explicit SNAT allocation)
az network lb outbound-rule create \
  -g rg-dev --lb-name lb-web -n ob-snat \
  --frontend-ip-configs feip-web --address-pool bp-web \
  --protocol Tcp --allocated-outbound-ports 10000 --idle-timeout 10
```

- Internal (private) LB: omit `--public-ip-address`, supply `--vnet-name` + `--subnet` + `--private-ip-address`.
- `az network lb list -o table` quickly reveals public vs private and SKU.

## Application Gateway

L7 (HTTP/S) load balancer with WAF and URL-based routing.

```bash
# Create with auto-generated backend pool
az network application-gateway create \
  -g rg-dev -n agw-web \
  --sku WAF_v2 --capacity 2 \
  --vnet-name vnet-dev --subnet subnet-agw \
  --public-ip-address pip-agw \
  --priority 100 \
  --servers 10.0.1.10 10.0.1.11 \
  --http-settings-protocol Http --http-settings-port 80

# WAF policy
az network application-gateway waf-policy create \
  -g rg-dev -n wafpol-web --type OWASP --version 3.2
az network application-gateway waf-policy managed-rule rule-set add \
  -g rg-dev --policy-name wafpol-web \
  --type OWASP --version 3.2

az network application-gateway update \
  -g rg-dev -n agw-web \
  --set firewallPolicy.id=$(az network application-gateway waf-policy show \
    -g rg-dev -n wafpol-web --query id -o tsv)

# Listeners, rules, redirects
az network application-gateway http-listener create \
  -g rg-dev --gateway-name agw-web -n lstn-https \
  --frontend-port 443 --ssl-cert cert-main

az network application-gateway rule create \
  -g rg-dev --gateway-name agw-web -n r-main \
  --http-listener lstn-https --address-pool bp-web \
  --http-settings bs-web --priority 100
```

- **Application Gateway v2 is required** for autoscale, zone redundancy, and WAF v2. V1 is effectively legacy.
- Changes take minutes to roll — `az network application-gateway wait` pairs with `--updated`.

## Private endpoints & Private Link

Private endpoints inject a PaaS service into your vnet via a NIC with a private IP.

```bash
# Storage example
STG_ID=$(az storage account show -g rg-dev -n stgdev1 --query id -o tsv)

az network private-endpoint create \
  -g rg-dev -n pe-stgdev1-blob \
  --vnet-name vnet-dev --subnet subnet-pe \
  --private-connection-resource-id "$STG_ID" \
  --group-id blob \
  --connection-name stgdev1-blob

# Private DNS zone for privatelink.blob.core.windows.net
az network private-dns zone create -g rg-dev -n privatelink.blob.core.windows.net
az network private-dns link vnet create \
  -g rg-dev -n link-vnet-dev \
  --zone-name privatelink.blob.core.windows.net \
  --virtual-network vnet-dev --registration-enabled false

# Wire the PE into the zone
az network private-endpoint dns-zone-group create \
  -g rg-dev --endpoint-name pe-stgdev1-blob --name dzg-pe \
  --private-dns-zone privatelink.blob.core.windows.net --zone-name blob

az network private-endpoint show -g rg-dev -n pe-stgdev1-blob \
  --query "{name:name, ip:customDnsConfigs[0].ipAddresses}"
```

- `--group-id` is service-specific: `blob`, `file`, `queue`, `table`, `dfs`, `web` for Storage; `sqlServer` for SQL; `vault` for Key Vault; `Sql` for Cosmos DB; ...
- Private endpoints break DNS for the public hostname inside the vnet only when you wire the Private DNS zone in. Forgetting step 3 is the #1 private-endpoint bug.

## Private DNS

```bash
az network private-dns zone create -g rg-dev -n internal.contoso.com
az network private-dns link vnet create \
  -g rg-dev -n link-vnet \
  --zone-name internal.contoso.com \
  --virtual-network vnet-dev --registration-enabled true

# Records
az network private-dns record-set a add-record \
  -g rg-dev -z internal.contoso.com -n db \
  --ipv4-address 10.0.2.10

az network private-dns record-set list -g rg-dev -z internal.contoso.com -o table
```

- `--registration-enabled true` auto-registers VMs' hostnames into the zone — handy but noisy.

## Public DNS (`az network dns`)

```bash
az network dns zone create -g rg-dev -n contoso.com
az network dns record-set a add-record \
  -g rg-dev -z contoso.com -n www --ipv4-address 203.0.113.5
az network dns record-set txt add-record \
  -g rg-dev -z contoso.com -n @ --value "v=spf1 include:_spf.example.com -all"
az network dns record-set cname set-record \
  -g rg-dev -z contoso.com -n api --cname api-backend.contoso.com

# Delegation: change registrar NS records to the zone's NS set
az network dns zone show -g rg-dev -n contoso.com --query nameServers
```

- NS records for the apex are auto-populated. Don't delete them.
- TTL default is 3600s; pass `--ttl` on record-set creation.

## VPN Gateway & ExpressRoute

### Site-to-site VPN

```bash
# Gateway subnet is mandatory and must be named exactly 'GatewaySubnet'
az network vnet subnet create -g rg-dev --vnet-name vnet-dev \
  --name GatewaySubnet --address-prefixes 10.0.255.0/27

az network public-ip create -g rg-dev -n pip-vpngw --allocation-method Static --sku Standard --zone 1 2 3

az network vnet-gateway create \
  -g rg-dev -n vpngw-dev \
  --vnet vnet-dev --public-ip-addresses pip-vpngw \
  --gateway-type Vpn --vpn-type RouteBased --sku VpnGw2AZ \
  --no-wait

# Local network gateway = on-prem endpoint
az network local-gateway create \
  -g rg-dev -n lng-hq \
  --gateway-ip-address 203.0.113.50 \
  --local-address-prefixes 192.168.0.0/16

# Connection
az network vpn-connection create \
  -g rg-dev -n conn-hq \
  --vnet-gateway1 vpngw-dev --local-gateway2 lng-hq \
  --shared-key 'sharedSecretHere' \
  --enable-bgp false
```

### Point-to-site (user VPN)

```bash
az network vnet-gateway update \
  -g rg-dev -n vpngw-dev \
  --address-prefixes 172.16.201.0/24 \
  --client-protocol OpenVPN

# Generate client config
az network vnet-gateway vpn-client generate \
  -g rg-dev -n vpngw-dev --processor-architecture Amd64
```

### ExpressRoute

```bash
az network express-route create \
  -g rg-dev -n er-circuit --bandwidth 1000 --peering-location "Washington DC" \
  --provider "Equinix" --sku-family MeteredData --sku-tier Standard

az network express-route peering create \
  -g rg-dev --circuit-name er-circuit --peering-type AzurePrivatePeering \
  --peer-asn 65001 --primary-peer-subnet 192.168.1.0/30 \
  --secondary-peer-subnet 192.168.2.0/30 --vlan-id 100
```

- ExpressRoute setup is **half CLI, half provisioning paperwork with your carrier** — CLI creates the circuit, the carrier provisions the link, then peering config matches.

## Azure Firewall

```bash
az network firewall create \
  -g rg-dev -n fw-hub \
  --vnet-name vnet-hub --sku AZFW_VNet --tier Standard \
  --location eastus

# Firewall needs a subnet named exactly 'AzureFirewallSubnet' (minimum /26)
az network vnet subnet create -g rg-dev --vnet-name vnet-hub \
  --name AzureFirewallSubnet --address-prefixes 10.10.1.0/26

# Public IP + wire it up
az network public-ip create -g rg-dev -n pip-fw --sku Standard --zone 1 2 3
az network firewall ip-config create \
  -g rg-dev --firewall-name fw-hub -n ipcfg-fw \
  --public-ip-address pip-fw --vnet-name vnet-hub

# Rule collection groups via firewall policy (Standard/Premium)
az network firewall policy create -g rg-dev -n fwpol-hub --sku Standard
az network firewall policy rule-collection-group create \
  -g rg-dev --policy-name fwpol-hub -n rcg-app --priority 200
az network firewall policy rule-collection-group collection add-filter-collection \
  -g rg-dev --policy-name fwpol-hub --rule-collection-group-name rcg-app \
  --name allow-out-web --collection-priority 100 --action Allow \
  --rule-name allow-http --rule-type NetworkRule \
  --source-addresses 10.0.0.0/16 --destination-addresses '*' --destination-ports 80 443 --ip-protocols TCP
```

## Network Watcher

```bash
# Enable per region (or use auto-enablement)
az network watcher configure -g NetworkWatcherRG -l eastus --enabled true

# Connectivity check (VM → target)
az network watcher test-connectivity \
  --source-resource vm-web -g NetworkWatcherRG \
  --dest-address 10.0.2.10 --dest-port 443

# Next-hop analysis (which NVA does traffic go through?)
az network watcher show-next-hop \
  --resource-group NetworkWatcherRG \
  --vm vm-web --source-ip 10.0.1.10 --dest-ip 10.0.2.10

# Packet capture
az network watcher packet-capture create \
  -g NetworkWatcherRG -n pc-vm-web \
  --vm vm-web --storage-account stgdev1 --time-limit 300

# NSG flow logs
az network watcher flow-log create \
  -l eastus --name fl-nsg-app \
  --nsg nsg-app --storage-account stgdev1 \
  --retention 30 --enabled true \
  --workspace laws-prod --interval 10 --traffic-analytics true
```

## Gotchas

- **Gateway subnets have name rules** — `GatewaySubnet` (VPN/ER) and `AzureFirewallSubnet` (Firewall). Misnaming = deploy fails silently.
- **NSG on gateway subnet**: not supported for VPN/ER gateway subnet; Azure enforces a minimum NSG behavior. Don't try to lock it down with an NSG.
- **Peering is non-transitive** — A↔B and B↔C does not give you A↔C. Use hub-and-spoke with a firewall/NVA, or [Virtual WAN](https://learn.microsoft.com/en-us/azure/virtual-wan/virtual-wan-about).
- **Private endpoint DNS** needs the zone + vnet link + zone-group wired — three separate commands. Forgetting any one leaves you resolving to the public IP and tripping over the firewall.
- **Application Gateway SKU change** (e.g., V1 → V2) requires a new gateway — no in-place upgrade.
- **Standard LB needs explicit outbound rules** for SNAT — without one, VMs in the backend pool lose internet egress after the first 64k ports exhaust.
- **`az network vnet subnet update --nat-gateway`** and `--network-security-group` accept `""` to detach — useful in cleanup scripts.
- **Private DNS auto-registration** collides with existing A records; set `--registration-enabled false` on zones used for private endpoints.
- **Service endpoints != Private endpoints**: service endpoints keep the PaaS public IP but route Azure-side; private endpoints give it a vnet private IP. Don't mix them for the same resource.

## Sources

- [`az network`](https://learn.microsoft.com/en-us/cli/azure/network)
- [`az network vnet`](https://learn.microsoft.com/en-us/cli/azure/network/vnet) · [`az network nsg`](https://learn.microsoft.com/en-us/cli/azure/network/nsg) · [`az network lb`](https://learn.microsoft.com/en-us/cli/azure/network/lb)
- [`az network application-gateway`](https://learn.microsoft.com/en-us/cli/azure/network/application-gateway)
- [`az network private-endpoint`](https://learn.microsoft.com/en-us/cli/azure/network/private-endpoint) · [`az network private-dns`](https://learn.microsoft.com/en-us/cli/azure/network/private-dns)
- [`az network vnet-gateway`](https://learn.microsoft.com/en-us/cli/azure/network/vnet-gateway) · [`az network firewall`](https://learn.microsoft.com/en-us/cli/azure/network/firewall) · [`az network watcher`](https://learn.microsoft.com/en-us/cli/azure/network/watcher)
- [Service tags](https://learn.microsoft.com/en-us/azure/virtual-network/service-tags-overview)
- [Virtual WAN](https://learn.microsoft.com/en-us/azure/virtual-wan/virtual-wan-about)
