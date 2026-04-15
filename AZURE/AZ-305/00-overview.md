# AZ-305 — Designing Microsoft Azure Infrastructure Solutions

**Advanced study overview and deep-dive launchpad**

> Target exam: **AZ-305** (Microsoft Certified: Azure Solutions Architect Expert)
> Companion exam required for the Expert badge: **AZ-104** (Azure Administrator Associate)
> Passing score: **700 / 1000**
> Current English exam update: **April 17, 2026** (localized versions follow ~8 weeks later)
> Sources: [Exam page](https://learn.microsoft.com/en-us/credentials/certifications/exams/az-305/) · [Official study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-305) · [Certification path](https://learn.microsoft.com/en-us/credentials/certifications/azure-solutions-architect/)

---

## Exam shape

- ~40–60 items: multiple choice, drag-and-drop, hot-area, build-list, and **case studies** (multi-question scenarios with fixed scenario text).
- Case studies dominate — expect 2–3 scenarios weighting a large portion of scored items. Read requirements *before* the architecture diagram.
- Exam is **design-focused**, not how-to-click. Nearly every question is "given these constraints, pick the best service/SKU/topology."
- **Cost, SLA composition, RTO/RPO, compliance boundaries, and identity blast radius** are the recurring decision axes.

## Skill areas & weightings (April 2026 update)

| # | Functional Group | Weight |
|---|---|---|
| 1 | Design identity, governance, and monitoring solutions | **25–30%** |
| 2 | Design data storage solutions | **20–25%** |
| 3 | Design business continuity solutions | **15–20%** |
| 4 | Design infrastructure solutions | **25–30%** |

Earlier 2025 iterations weighted these as 26 / 24 / 17 / 33. Expect infra to remain the largest single area; identity/governance is climbing.

## How to use this overview

Each section below contains:
1. **Scope** — what the objective domain covers.
2. **Core services & concepts** — the must-know decision space.
3. **Exam traps** — distinctions the exam loves to test.
4. **Deep-dive prompt** — paste into a new Claude session with `/azure-expert` to generate a full topical note in this folder.

> **Style convention for all deep-dive notes:** embed inline `learn.microsoft.com` hyperlinks on the first mention of every Azure service, feature, tier, and limit (matching the density of `01-identity-governance-monitoring.md`) — not only a trailing Sources list. Each prompt below already carries this instruction.

---

## 1. Design identity, governance, and monitoring solutions (25–30%)

### Scope
Tenant and subscription topology, identity lifecycle, authorization model, policy/cost/compliance guardrails, and observability strategy.

### Core services & concepts
- **Microsoft Entra ID** (tenants, multi-tenant vs single-tenant apps, B2B guests, External ID / former B2C, Entra Domain Services).
- **Hybrid identity**: Entra Connect Sync, Cloud Sync, Pass-through Auth, Password Hash Sync, Federation — pick based on outage tolerance and on-prem password policy needs.
- **Authentication & access**: Conditional Access, MFA, PIM (just-in-time + access reviews), managed identities (system vs user-assigned), workload identities, federated credentials (OIDC from GitHub/other clouds).
- **Authorization**: RBAC vs ABAC vs Entra roles vs Azure roles, custom roles, deny assignments, management-group scope inheritance.
- **Governance**: Management groups, Azure Policy (initiatives, `deployIfNotExists`, `modify`), Blueprints → deprecated in favor of **Template Specs + Deployment Stacks**, resource locks, tag policies, landing zones (ALZ / CAF).
- **Cost management**: Cost Management + Billing, budgets with action groups, reservations vs savings plans vs spot, chargeback tagging.
- **Monitoring**: Azure Monitor, Log Analytics workspaces (design: single vs per-region vs per-BU), Application Insights (workspace-based), Diagnostic Settings routing, Metrics vs Logs, alert rules + action groups, Workbooks, Network Watcher, Defender for Cloud signals.

### Exam traps
- **RBAC vs Entra role** scope: managing Azure resources ≠ managing the directory.
- **Management groups** inherit policy and RBAC *down*; you cannot break inheritance, only add deny at lower scope.
- **Log Analytics** per-GB ingestion vs Commitment Tiers — a design question often hinges on predictable ingest.
- **PIM** is an Entra ID P2 feature.
- **Managed identity** cannot be used cross-tenant; use workload identity federation or service principal.
- **Diagnostic settings** → separate destinations (LA, Storage, Event Hub) — don't confuse with Activity Log export.

### Deep-dive prompt
```
/azure-expert Deep-dive AZ-305 "Design identity, governance, and monitoring solutions" (25–30%).
Produce AZURE/AZ-305/01-identity-governance-monitoring.md covering:
Entra ID tenant/subscription/management-group topology, hybrid identity options and
decision matrix (PHS/PTA/Federation/Cloud Sync), Conditional Access + MFA + PIM design,
RBAC vs ABAC vs Entra roles, managed identities & workload identity federation, Azure
Policy + initiatives + remediation + Deployment Stacks, cost governance (budgets,
reservations, savings plans), Log Analytics workspace design (centralized vs federated),
Application Insights workspace-based mode, diagnostic settings routing, alerting &
action groups, Defender for Cloud posture. Include decision tables, Bicep/CLI snippets
for policy assignments and managed identities, and an "Exam traps" section with
distractor-style gotchas. Embed inline Microsoft Learn hyperlinks on the first mention
of every Azure service, feature, tier, and limit (matching the density of
`01-identity-governance-monitoring.md`) — not only a trailing Sources list.
```

---

## 2. Design data storage solutions (20–25%)

### Scope
Pick the right data service and configuration for a given workload: relational, NoSQL, object/file/disk storage, caching, and analytics stores.

### Core services & concepts
- **Relational**: Azure SQL Database (DTU vs vCore, Hyperscale, Serverless, Business Critical vs General Purpose, Elastic Pools, read replicas, failover groups, Always Encrypted), SQL Managed Instance, PostgreSQL Flexible Server (incl. Citus), MySQL Flexible Server.
- **NoSQL / multi-model**: Cosmos DB (API choice: NoSQL / MongoDB / Cassandra / Gremlin / Table), partition key design, RU economics + autoscale, **5 consistency levels**, multi-region writes, change feed, TTL, backups (continuous vs periodic).
- **Object / file**: Storage Accounts (GPv2, Premium block blob/file/page), access tiers Hot/Cool/Cold/Archive, lifecycle management, immutable blobs / WORM, SFTP & NFS 3.0 on blob, **Azure Files** (SMB/NFS, AD auth, Entra Kerberos), **Azure NetApp Files** for HPC/HANA, **Data Lake Storage Gen2** (hierarchical namespace).
- **Caching & search**: Azure Cache for Redis tiers (Basic/Standard/Premium/Enterprise/Enterprise Flash), persistence, geo-replication, **Azure AI Search** vector + hybrid + semantic ranker (when it replaces a vector DB).
- **Analytics**: Synapse Analytics, Microsoft Fabric, Data Factory, Event Hubs capture, Databricks as a design option.
- **Security posture**: Private Endpoints vs Service Endpoints, customer-managed keys (CMK) with Key Vault, double encryption, data exfiltration protections.

### Exam traps
- **Cosmos consistency** order: Strong → Bounded Staleness → Session → Consistent Prefix → Eventual. "Multi-region reads" ≠ "multi-region writes."
- **Azure SQL** geo-restore vs failover group vs auto-failover group vs active geo-replication — different RTO/RPO.
- **Hot/Cool/Cold/Archive**: Archive is *offline* — rehydrate hours, not minutes. Minimum storage durations trigger early-deletion fees.
- **ZRS vs GRS vs RA-GRS vs GZRS**: ZRS is zonal redundancy within a region; GRS is secondary region but not readable unless RA-GRS/GZRS.
- **Managed Instance vs SQL DB**: MI supports cross-database queries, CLR, SQL Agent — questions lean on feature gaps.
- **Storage account firewall** + Private Endpoint interaction (deny public, private only).

### Deep-dive prompt
```
/azure-expert Deep-dive AZ-305 "Design data storage solutions" (20–25%).
Produce AZURE/AZ-305/02-data-storage.md covering:
Azure SQL DB (tiers, Hyperscale, Serverless, BC vs GP, Elastic Pools, Always Encrypted,
failover groups) vs SQL MI vs PostgreSQL/MySQL Flexible Server decision matrix;
Cosmos DB (API selection, partition key design, RU/s vs autoscale, 5 consistency levels
with concrete trade-offs, multi-region writes, change feed, backups); Storage Accounts
(redundancy LRS/ZRS/GRS/RA-GRS/GZRS/RA-GZRS, access tiers + lifecycle, immutable blobs,
SFTP/NFS, ADLS Gen2); Azure Files + Azure NetApp Files; Redis tier selection;
encryption (SSE, CMK, double, Always Encrypted), Private Endpoints, data exfiltration.
Include RTO/RPO tables, Bicep snippets for failover groups & CMK, and "Exam traps".
Embed inline Microsoft Learn hyperlinks on the first mention of every Azure service,
feature, tier, and limit (matching the density of `01-identity-governance-monitoring.md`)
— not only a trailing Sources list.
```

---

## 3. Design business continuity solutions (15–20%)

### Scope
Backup, disaster recovery, and high availability aligned to **RPO / RTO / SLA** business requirements — across compute, data, and entire regions.

### Core services & concepts
- **Azure Backup**: Recovery Services vault vs Backup vault, policies, soft delete + immutability, VM / SQL in VM / Azure Files / Blob / disk / PostgreSQL backups, cross-region restore, Backup Center.
- **Azure Site Recovery (ASR)**: Azure-to-Azure DR, on-prem-to-Azure (VMware/Hyper-V/physical), replication policies, recovery plans, test failover vs failover vs commit.
- **Database DR patterns**: SQL auto-failover groups, Cosmos multi-region + automatic/manual failover, Storage object replication, geo-redundant Redis.
- **HA within a region**: Availability Zones vs Availability Sets vs VM Scale Sets, zone-redundant services (App Gateway v2, Load Balancer Standard, SQL DB BC zone-redundant, AKS).
- **Multi-region topologies**: active-active (Front Door + dual writes), active-passive (warm standby), pilot light, backup-and-restore. Understand **SLA composition** (multiplying SLAs in series vs parallel) and how AZ/region redundancy changes the number.
- **Networking failover**: Traffic Manager (DNS) vs Front Door (L7 anycast) vs Cross-Region Load Balancer (L4) — latency, health probes, session affinity.

### Exam traps
- **RPO vs RTO**: RPO = data loss tolerance; RTO = downtime tolerance.
- **Azure Backup cannot** protect everything natively — unsupported PaaS must use service-native export or continuous backup.
- **ASR RPO** is seconds for Azure-to-Azure but **minutes to hours** for some source workloads — don't assume "near-zero."
- **Traffic Manager** is DNS-based: TTL governs failover time. Not a substitute for Front Door when you need instant failover.
- **Zone-redundant** ≠ **zonal** (zonal pins to one zone — it's *less* resilient, not more).
- **SLA math**: services in series multiply (99.9% × 99.9% ≈ 99.8%); redundant parallel paths complement (1 − (1 − 0.999)²).

### Deep-dive prompt
```
/azure-expert Deep-dive AZ-305 "Design business continuity solutions" (15–20%).
Produce AZURE/AZ-305/03-business-continuity.md covering:
Defining & meeting RTO/RPO targets; Azure Backup (RSV vs Backup vault, policies,
soft delete + immutable, cross-region restore, per-workload support matrix); ASR
(A2A and on-prem scenarios, recovery plans, test failover); database-native DR
(SQL auto-failover groups, Cosmos multi-region, Storage object replication,
GZRS); HA primitives (AZ vs AS vs VMSS, zone-redundant services); multi-region
topologies (active-active vs active-passive vs pilot light) and SLA composition math;
traffic steering (Front Door vs Traffic Manager vs Cross-Region LB). Include
decision tree, RTO/RPO comparison table, Bicep for zone-redundant SQL + Front Door
origin group, and "Exam traps". Embed inline Microsoft Learn hyperlinks on the first
mention of every Azure service, feature, tier, and limit (matching the density of
`01-identity-governance-monitoring.md`) — not only a trailing Sources list.
```

---

## 4. Design infrastructure solutions (25–30%)

### Scope
The largest domain: compute platform, application architecture, networking, integration/messaging, and migration — i.e. the actual building blocks of the workload.

### Core services & concepts
- **Compute selection**: VMs (sizes, spot, dedicated hosts), VM Scale Sets (Uniform vs Flexible), App Service (plans, slots, VNet integration, Private Endpoint), Azure Functions (Consumption, Premium, Flex Consumption, Dedicated — and triggers/bindings), **Azure Container Apps** (Dapr, KEDA, revisions) vs **AKS** (node pools, workload identity, network plugins: Azure CNI / Overlay / kubenet) vs ACI vs App Service containers vs Service Fabric.
- **Application architecture patterns**: N-tier, microservices, event-driven, CQRS, saga, retry + circuit breaker, idempotency, sharding, cache-aside, queue-based load leveling, strangler-fig for migration.
- **Messaging & integration**: **Service Bus** (queues/topics, sessions, dead-lettering, transactions, premium + Private Endpoint) vs **Event Grid** (discrete events, push, retry+DLQ) vs **Event Hubs** (streaming, partitions, capture, Kafka protocol). **API Management** (tiers, Consumption vs Developer vs Standard v2 / Premium v2, VNet modes, policies, revisions/versions), Logic Apps (Consumption vs Standard), Azure Front Door (Standard/Premium, WAF, private origins), Application Gateway v2 (WAF v2, autoscaling).
- **Networking**: Hub-and-spoke topology, Virtual WAN, VNet peering (transit via NVA/Azure Firewall/Route Server), ExpressRoute (circuits, peerings, FastPath, Global Reach) vs Site-to-Site VPN, Azure Bastion, Private Link + Private Endpoints vs Service Endpoints, NSG vs ASG vs Azure Firewall (Standard vs Premium with IDPS/TLS inspection) vs third-party NVAs, DDoS Protection Standard/IP.
- **Migration**: Azure Migrate (Discovery, Server Assessment, Server Migration, Data Migration Assistant/Database Migration Service), rehost vs refactor vs rearchitect vs rebuild, database migration (online vs offline), landing zones before migration waves.
- **DevOps / IaC**: Bicep, ARM, Terraform, **Deployment Stacks** (replacing Blueprints for lifecycle-managed deployments), GitHub Actions + Azure DevOps pipelines with OIDC federated credentials.

### Exam traps
- **Functions Consumption vs Premium vs Flex Consumption**: cold start, VNet integration, max duration. Flex Consumption now supports VNet + scale-to-zero with faster scale.
- **Container Apps vs AKS**: ACA wins on operational simplicity and event-driven scale; AKS wins when you need raw Kubernetes primitives, custom CNI, or GPU/Windows node pool control.
- **APIM tiers**: only Developer/Premium (v1) and Premium v2 support full VNet injection; Standard v2 supports *outbound* VNet. Consumption has no VNet.
- **Event Grid vs Event Hubs vs Service Bus**: "discrete event" vs "stream of telemetry" vs "transactional message." Exam writes the prompt in business language — map keywords.
- **VNet peering is non-transitive** — spoke-to-spoke needs a hub NVA, Azure Firewall, Route Server, or Virtual WAN.
- **ExpressRoute** private peering doesn't reach PaaS public endpoints without Private Link; Microsoft peering is separate.
- **App Gateway** is regional (L7); **Front Door** is global (L7 anycast). Combining them is a valid pattern but watch for double-WAF cost.

### Deep-dive prompt
```
/azure-expert Deep-dive AZ-305 "Design infrastructure solutions" (25–30%).
Produce AZURE/AZ-305/04-infrastructure.md covering:
Compute decision matrix (VM/VMSS vs App Service vs Functions plans vs Container Apps
vs AKS vs ACI vs Service Fabric) with trade-offs and exam-style scenarios;
application architecture patterns (N-tier, microservices, event-driven, CQRS, saga,
idempotency, cache-aside, queue-based load leveling, strangler-fig); messaging
(Service Bus vs Event Grid vs Event Hubs — full decision guide with gotchas),
API Management tier + VNet modes, Logic Apps Standard vs Consumption, Front Door vs
App Gateway; networking (hub-and-spoke vs Virtual WAN, peering transitivity,
ExpressRoute vs VPN, Private Link/Endpoints vs Service Endpoints, Azure Firewall
Premium features, Bastion, DDoS); migration strategy (Azure Migrate tooling, 6 Rs,
DMS online vs offline); IaC with Bicep + Deployment Stacks + OIDC federated deploys.
Include decision tables, Bicep snippets (hub-spoke, Private Endpoint, APIM, ACA),
and a long "Exam traps" section. Embed inline Microsoft Learn hyperlinks on the first
mention of every Azure service, feature, tier, and limit (matching the density of
`01-identity-governance-monitoring.md`) — not only a trailing Sources list.
```

---

## Cross-cutting foundations (study these before tackling any domain)

These don't map to a single objective but show up *everywhere*:

- **Azure Well-Architected Framework** — the 5 pillars (Reliability, Security, Cost Optimization, Operational Excellence, Performance Efficiency) are the lens the exam grades "best" answers through.
- **Cloud Adoption Framework + Azure Landing Zones** — enterprise-scale reference architecture, management-group hierarchy, platform vs application landing zones.
- **Zero Trust** principles — verify explicitly, least privilege, assume breach. Framed into identity, device, network, application, data pillars.
- **Region & AZ model** — paired vs non-paired regions, zone count per region, zonal vs zone-redundant resources.

### Deep-dive prompt (foundations)
```
/azure-expert Deep-dive AZ-305 cross-cutting foundations.
Produce AZURE/AZ-305/05-foundations-waf-caf-landingzones.md covering:
Well-Architected Framework 5 pillars with tradeoff tensions, Cloud Adoption Framework
methodologies, Azure Landing Zones (management group hierarchy, platform vs app
landing zones, ALZ accelerators), Zero Trust pillars mapped to Azure controls, region
and AZ model (paired regions, zone-redundant services list), and how each lens shows
up in AZ-305 scenario questions. Include a cheat-sheet mapping "exam phrase" →
"WAF pillar that's being tested". Embed inline Microsoft Learn hyperlinks on the first
mention of every Azure service, feature, tier, and limit (matching the density of
`01-identity-governance-monitoring.md`) — not only a trailing Sources list.
```

---

## Suggested study path

1. Read this overview end-to-end.
2. Generate the **Foundations** deep-dive first (section 5 prompt) — it frames everything else.
3. Generate deep-dives in exam-weight order: **Infrastructure → Identity/Gov/Monitoring → Data → BCDR**.
4. Drill case studies from MeasureUp or equivalent practice vendor; re-read the exam traps in each deep-dive after every wrong answer.
5. Re-check the [official study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-305) shortly before exam day — Microsoft publishes change logs inside it.

---

## Sources

- [Microsoft Learn — Exam AZ-305](https://learn.microsoft.com/en-us/credentials/certifications/exams/az-305/)
- [Microsoft Learn — AZ-305 official study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-305)
- [Microsoft Certified: Azure Solutions Architect Expert](https://learn.microsoft.com/en-us/credentials/certifications/azure-solutions-architect/)
- [Exam readiness: Identity, governance, monitoring (Part 1/4)](https://learn.microsoft.com/en-us/shows/exam-readiness-zone/preparing-for-az-305-01-fy25)
- [Exam readiness: Infrastructure (Part 4/4)](https://learn.microsoft.com/en-us/shows/exam-readiness-zone/preparing-for-az-305-04-fy25)
- [AZ-305T00 instructor-led course](https://learn.microsoft.com/en-us/training/courses/az-305t00)
- [Skills-measured PDF](https://arch-center.azureedge.net/Learning/Credentials/exam-az-305-designing-microsoft-azure-infrastructure-solutions-skills-measured.pdf)
