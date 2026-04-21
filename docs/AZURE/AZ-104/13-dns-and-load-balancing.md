# Azure DNS and Load Balancing

> **Exam mapping:** AZ-104 -- "Implement and manage virtual networking" (15--20 %)
> **One-liner:** Host DNS records (public and private), resolve names inside VNets, and distribute L4 traffic with Azure Load Balancer -- all three are bread-and-butter admin tasks on the exam.
> **Related:** [11-virtual-networks](11-virtual-networks.md) | [12-network-security](12-network-security.md)

---

## 1 Azure DNS

Azure DNS lets you host DNS zones in Azure's global anycast network -- the same infrastructure that backs `168.63.129.16`. There are two flavours: **public** (internet-facing) and **private** (VNet-internal).

### 1.1 Public DNS Zones

A [public DNS zone](https://learn.microsoft.com/en-us/azure/dns/dns-zones-records) hosts records for a domain that is resolvable from the internet.

**Creating a zone and adding records (CLI)**

```bash
# Create the zone
az network dns zone create \
  --resource-group rg-dns \
  --name contoso.com

# Add an A record
az network dns record-set a add-record \
  --resource-group rg-dns \
  --zone-name contoso.com \
  --record-set-name www \
  --ipv4-address 10.0.0.4

# Add an MX record
az network dns record-set mx add-record \
  --resource-group rg-dns \
  --zone-name contoso.com \
  --record-set-name @ \
  --exchange mail.contoso.com \
  --preference 10
```

#### Supported record types

| Type | Purpose |
|------|---------|
| **A** | Maps name to IPv4 address |
| **AAAA** | Maps name to IPv6 address |
| **CNAME** | Canonical-name alias (cannot coexist with other records at the same name) |
| **MX** | Mail exchange |
| **NS** | Delegates a sub-zone to another name server |
| **PTR** | Reverse-DNS lookup |
| **SOA** | Start of Authority -- one per zone, auto-created |
| **SRV** | Service locator (protocol, port, priority, weight) |
| **TXT** | Arbitrary text (SPF, DKIM, domain verification) |
| **CAA** | Certification Authority Authorization -- controls which CAs may issue certs |

> With [DNSSEC](https://learn.microsoft.com/en-us/azure/dns/dnssec) signing enabled, **DS** and **TLSA** record types are also available.

#### Record sets and TTL

- Records of the same type at the same name form a **record set** (e.g. two A records for round-robin).
- Each record set has a **TTL** (default 3600 s). Lowering TTL before a migration speeds up propagation but increases query volume.

#### Alias records

[Alias records](https://learn.microsoft.com/en-us/azure/dns/dns-alias) are a special capability that makes a DNS record **point directly to an Azure resource** instead of a static IP. When the resource's IP changes, the DNS record auto-updates -- no stale records, no dangling references.

Supported alias targets:
- Azure **Public IP** address resource
- Azure **Traffic Manager** profile
- Azure **Front Door** profile
- Azure **Load Balancer** frontend (public)
- Another **record set in the same zone**

Alias records are supported on **A**, **AAAA**, and **CNAME** record types.

> **Exam trap -- alias vs CNAME:** A CNAME is a static pointer to another DNS name and cannot sit at the zone apex (`@`). An alias record *can* sit at the apex and auto-resolves to the resource's current IP. Alias records also skip the extra CNAME-resolution hop, so they are faster.

#### Delegating a domain to Azure DNS

1. Create the public DNS zone in Azure -- Azure assigns four NS records (e.g. `ns1-01.azure-dns.com`).
2. At your **domain registrar**, update the NS records for the domain to point to those four Azure DNS name servers.
3. Propagation depends on the registrar TTL (can take minutes to 48 hours).

### 1.2 Private DNS Zones

[Azure Private DNS](https://learn.microsoft.com/en-us/azure/dns/private-dns-overview) provides name resolution **within and across VNets** without deploying custom DNS servers or exposing records to the internet.

```bash
# Create a private DNS zone
az network private-dns zone create \
  --resource-group rg-dns \
  --name private.contoso.com

# Link the zone to a VNet with auto-registration enabled
az network private-dns link vnet create \
  --resource-group rg-dns \
  --zone-name private.contoso.com \
  --name link-vnet-hub \
  --virtual-network vnet-hub \
  --registration-enabled true
```

#### Virtual network links

A private DNS zone must be **linked** to one or more VNets to be usable. There are two link modes:

| Mode | What it does | Limit |
|------|-------------|-------|
| **Resolution link** | VMs in the linked VNet can resolve records in the zone | Up to **1 000** VNets per zone |
| **Registration link** (auto-registration) | VMs in the linked VNet automatically get A records created/removed when they start/stop | Up to **1** registration-enabled VNet per zone (but a VNet can be registration-linked to only **1** private zone) |

> **Exam trap -- auto-registration limit:** A private DNS zone can have auto-registration enabled for **only one** virtual network link. Many resolution-only links are fine, but the 1:1 auto-registration constraint is heavily tested.

**What auto-registration creates:** When a VM is deployed in the registration VNet, Azure automatically creates an A record (`vmname.private.contoso.com`) and a PTR record. When the VM is deallocated/deleted, the records are removed.

#### Split-horizon DNS

You can host the *same* domain name as both a public zone and a private zone. VMs inside linked VNets resolve the private zone; external clients resolve the public zone. This is called **split-horizon** (or split-brain) DNS.

### 1.3 Azure-provided DNS vs Custom DNS

| Feature | Azure-provided DNS (`168.63.129.16`) | Custom DNS server |
|---------|--------------------------------------|-------------------|
| Config effort | Zero -- works out of the box | Must deploy and manage VM(s) |
| VNet-to-VNet resolution | No (single-VNet only, unless private DNS zone linked) | Yes (forwarders / conditional forwarding) |
| Custom domain names | No (uses `*.internal.cloudapp.net`) | Yes |
| Record types | A only (auto) | Full |
| Private DNS zones | Works with private DNS zone links | Can forward to `168.63.129.16` for zone resolution |

You can configure custom DNS servers per VNet (Settings > DNS servers) or per NIC. The IP `168.63.129.16` is a virtual IP that routes to the Azure platform -- it also serves as the health-probe source for Load Balancer and the DHCP relay.

---

## 2 Azure Load Balancer

[Azure Load Balancer](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-overview) is a **Layer 4 (TCP/UDP)** load balancer that distributes inbound flows to backend pool instances using a 5-tuple hash (source IP, source port, destination IP, destination port, protocol). It is zone-redundant in the Standard SKU.

### 2.1 SKU Comparison

| Feature | Basic (retired Sept 30 2025) | Standard |
|---------|------------------------------|----------|
| **SLA** | None | **99.99 %** (2+ healthy instances) |
| **Backend pool size** | 300 instances | **1 000** instances |
| **Backend pool type** | NIC-based only | NIC-based **or** IP-based |
| **Health probes** | TCP, HTTP | TCP, HTTP, **HTTPS** |
| **Availability Zones** | Not supported | **Zone-redundant** and zonal frontends |
| **Secure by default** | No -- open to internet | **Yes** -- closed unless NSG allows traffic |
| **Multiple frontends** | Yes | Yes |
| **Outbound rules** | Not available | Available (explicit SNAT config) |
| **HA Ports** | Not available | Available (internal LB only) |
| **Global VNet peering backends** | No | **Yes** |
| **Diagnostics / metrics** | Not supported | Full Azure Monitor integration |

> **Exam trap -- Basic LB retired:** Basic Load Balancer was **retired on September 30, 2025**. No new Basic LBs could be created after March 31, 2025. Existing ones continue to function but are unsupported and carry no SLA. The exam will still reference Basic for comparison, but all new deployments must use Standard. See [upgrade guidance](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-basic-upgrade-guidance).

### 2.2 Public Load Balancer

A [public load balancer](https://learn.microsoft.com/en-us/azure/load-balancer/components) maps the **public IP and port** of incoming traffic to the **private IP and port** of a backend VM. It provides outbound internet connectivity for backend VMs via SNAT.

```
Internet --> [Public IP : port] --> LB rule --> [Backend pool VM : port]
```

### 2.3 Internal (Private) Load Balancer

An [internal load balancer](https://learn.microsoft.com/en-us/azure/load-balancer/components) uses a **private IP** as its frontend. Traffic is distributed within a VNet or from on-premises via VPN/ExpressRoute. No public IP is involved.

Use cases: multi-tier apps (web tier LB --> app tier LB --> DB tier), line-of-business apps, hybrid connectivity.

### 2.4 Components Deep-Dive

#### Frontend IP configuration

- **Public LB:** one or more public IP addresses (or public IP prefix).
- **Internal LB:** a private IP from a subnet (static or dynamic allocation).
- A single LB can have **multiple** frontend IPs (multi-VIP).

#### Backend pools

- Collection of VMs or VMSS instances that serve traffic.
- **NIC-based:** VMs are added by their NIC resource -- classic approach.
- **IP-based:** backends specified by IP address -- allows cross-VNet and cross-region (Standard only).

#### Health probes

[Health probes](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-custom-probe-overview) determine which backend instances receive traffic.

| Setting | Description | Default |
|---------|-------------|---------|
| **Protocol** | TCP, HTTP, or HTTPS (Standard only) | TCP |
| **Port** | The port the probe hits on the backend | -- |
| **Path** | URL path for HTTP/HTTPS probes (e.g. `/health`) | `/` |
| **Interval** | Seconds between probes | 5 s (portal), 15 s (API) |
| **Unhealthy threshold** | Consecutive failures before marking instance down | 2 |

- Probe **source IP** is always `168.63.129.16` -- NSGs must allow this.
- HTTP probes expect a **200 OK** response; anything else = unhealthy.
- HTTPS probes do **not** validate the TLS certificate.

#### Load-balancing rules

A rule ties a **frontend IP + port** to a **backend pool + port**.

| Setting | Options |
|---------|---------|
| **Protocol** | TCP or UDP |
| **Frontend port** | The port clients connect to |
| **Backend port** | The port on the backend VM |
| **Session persistence** | **None** (5-tuple hash, default), **Client IP** (2-tuple: src IP + dest IP), **Client IP and Protocol** (3-tuple: src IP + dest IP + protocol) |
| **Idle timeout** | 4--30 minutes (default 4) |
| **Floating IP** | When enabled, the frontend IP is exposed directly on the VM NIC (used for SQL AlwaysOn, HA NVAs) |
| **HA Ports** | Protocol = All, Port = 0 -- load-balances **every** TCP/UDP port on an internal Standard LB. Essential for NVAs (firewalls, WAN optimizers). |

> **Exam trap -- HA Ports:** HA Ports rules are available **only on internal Standard Load Balancer**. They cannot be configured on public LBs or Basic LBs.

#### Inbound NAT rules

Map a specific **frontend IP + port** to a specific **backend VM + port**. Used for direct SSH/RDP access to individual VMs behind the LB (e.g. `publicIP:50001 --> VM1:22`, `publicIP:50002 --> VM2:22`).

#### Outbound rules (SNAT)

[Outbound rules](https://learn.microsoft.com/en-us/azure/load-balancer/outbound-rules) configure Source Network Address Translation (SNAT) for backend pool members to reach the internet.

- Available on **Standard SKU only**.
- Each frontend public IP provides **64 512** SNAT ports to distribute across the backend pool.
- You can configure **manual port allocation** (explicit ports per instance) or **default port allocation**.
- Adding more frontend IPs increases the total SNAT port budget.

### 2.5 CLI Quick-Reference

```bash
# Create an internal Standard LB
az network lb create \
  --resource-group rg-lb \
  --name lb-internal \
  --sku Standard \
  --vnet-name vnet-app \
  --subnet snet-backend \
  --frontend-ip-name fe-private \
  --backend-pool-name pool-app

# Create a health probe
az network lb probe create \
  --resource-group rg-lb \
  --lb-name lb-internal \
  --name probe-http \
  --protocol Http \
  --port 80 \
  --path "/health"

# Create a load-balancing rule
az network lb rule create \
  --resource-group rg-lb \
  --lb-name lb-internal \
  --name rule-http \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name fe-private \
  --backend-pool-name pool-app \
  --probe-name probe-http

# Add a backend address pool (for IP-based backends)
az network lb address-pool create \
  --resource-group rg-lb \
  --lb-name lb-internal \
  --name pool-cross-vnet \
  --vnet vnet-remote
```

---

## 3 Troubleshoot Load Balancing

### 3.1 Common Issues

| Problem | Root cause | Fix |
|---------|-----------|-----|
| **All VMs showing unhealthy** | NSG blocking probe source `168.63.129.16`, wrong probe port/path, app not listening | Allow AzureLoadBalancer service tag in NSG; verify app binds to correct port |
| **Some VMs unhealthy** | App crashed on specific instances, OS firewall blocking probe | Restart app, check Windows Firewall / iptables |
| **Intermittent connectivity** | SNAT port exhaustion on outbound | Add public IPs, use NAT Gateway, configure outbound rules with manual port allocation |
| **Asymmetric routing** | Multiple LBs or UDRs causing return traffic to bypass the LB | Ensure UDR routes return traffic back through the LB frontend |
| **Backend pool misconfiguration** | VM NIC not in the correct backend pool, IP address mismatch | Verify backend pool membership, check NIC association |
| **Connection timeouts** | Idle timeout too short, backend app slow to respond | Increase idle timeout (max 30 min), enable TCP keepalives |

### 3.2 Diagnostic Tools

| Tool | What it shows |
|------|---------------|
| [**Load Balancer Insights**](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-insights) (Azure Monitor) | Pre-built workbook: topology, health probe status, data-path availability, SNAT metrics |
| **Health Probe Status** metric | Per-backend-instance probe state (up/down) over time |
| **Data Path Availability** metric | End-to-end availability (frontend to backend) -- the LB's "heartbeat" |
| **SNAT Connection Count** metric | Successful vs failed outbound connections -- split by Connection State to spot exhaustion |
| [**Network Watcher**](https://learn.microsoft.com/en-us/azure/network-watcher/network-watcher-overview) | IP flow verify, NSG diagnostics, connection troubleshoot, packet capture |
| **Effective Security Rules** | Shows the computed NSG rules on a VM NIC -- confirms whether `168.63.129.16` is allowed |

### 3.3 Systematic Troubleshooting Approach

```
1. Check Health Probe Status metric
   --> Are backends healthy?
       |
       No --> 2. Check NSG / effective security rules
              --> Is 168.63.129.16 allowed on probe port?
                  |
                  No --> Fix NSG rule (allow AzureLoadBalancer service tag)
                  Yes --> 3. Check backend application
                          --> Is the app listening on the probe port/path?
                          --> Is the OS firewall blocking?
       |
       Yes --> 4. Check LB rules
               --> Correct frontend port --> backend port mapping?
               --> Session persistence misconfigured?
               |
               Rules OK --> 5. Check SNAT (outbound issues only)
                            --> SNAT Connection Count Failed > 0?
                            --> Add public IPs or use NAT Gateway
```

---

## Exam Traps

1. **Basic LB has no SLA and is retired** -- Basic LB was retired September 30, 2025. It has no SLA and no AZ support. All exam "best practice" answers use Standard. Know the differences for comparison questions.

2. **Alias records auto-update; CNAMEs do not** -- An alias record tracks the Azure resource and updates automatically when its IP changes. A CNAME is static. Alias records can also sit at the zone apex (`@`), while CNAME records cannot.

3. **Private DNS auto-registration limit** -- A private DNS zone supports auto-registration from **only one** virtual network link. You can have up to 1 000 resolution-only links, but the registration link is 1:1. A virtual network can also only be registration-linked to one private zone.

4. **Standard LB is secure by default** -- Standard LB blocks all traffic unless an **NSG** explicitly allows it. This is the opposite of Basic LB (which was open by default). Forgetting the NSG is the number-one cause of "LB is configured but nothing works."

5. **Health probe source IP is `168.63.129.16`** -- NSGs on backend VMs must allow inbound traffic from this IP on the probe port. The service tag `AzureLoadBalancer` covers this. If you block it, all backends show unhealthy.

6. **HA Ports only on internal Standard LB** -- HA Ports (protocol=All, port=0) are exclusively available on **internal** Standard Load Balancer. Not available on public LBs or the retired Basic SKU. Common scenario: NVA (firewall) behind an internal LB.

---

## Sources

- [Overview of DNS zones and records](https://learn.microsoft.com/en-us/azure/dns/dns-zones-records)
- [Alias records overview](https://learn.microsoft.com/en-us/azure/dns/dns-alias)
- [Azure Private DNS overview](https://learn.microsoft.com/en-us/azure/dns/private-dns-overview)
- [What is auto-registration in Azure DNS private zones](https://learn.microsoft.com/en-us/azure/dns/private-dns-autoregistration)
- [Virtual network links for private DNS zones](https://learn.microsoft.com/en-us/azure/dns/private-dns-virtual-network-links)
- [What is Azure Load Balancer?](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-overview)
- [Azure Load Balancer SKUs](https://learn.microsoft.com/en-us/azure/load-balancer/skus)
- [Azure Load Balancer components](https://learn.microsoft.com/en-us/azure/load-balancer/components)
- [Health probes](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-custom-probe-overview)
- [Outbound rules](https://learn.microsoft.com/en-us/azure/load-balancer/outbound-rules)
- [Troubleshoot health probe status](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-troubleshoot-health-probe-status)
- [Troubleshoot common Azure Load Balancer problems](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-troubleshoot)
- [Diagnostics with metrics, alerts, and resource health](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-standard-diagnostics)
- [Upgrading from Basic Load Balancer -- guidance](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-basic-upgrade-guidance)
- [Azure Private DNS FAQ](https://learn.microsoft.com/en-us/azure/dns/dns-faq-private)
