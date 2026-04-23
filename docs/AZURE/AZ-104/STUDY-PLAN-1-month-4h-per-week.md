# AZ-104 — 1-Month Study Plan (4 hours/week, ~16 hours total)

> **Exam:** Microsoft Certified: Azure Administrator Associate (AZ-104)
> **Format:** 40–60 questions, 100 minutes, **pass mark 700/1000**, case studies + lab-style + multiple choice.
> **Target date:** ~4 weeks from 2026-04-22 → exam in week of **2026-05-18 to 2026-05-24**.
> **Budget:** 4 focused hours/week × 4 weeks = **~16 hours**.

---

## Reality check first

**16 hours is lean for AZ-104.** Microsoft's own guidance and most pass reports sit around **40–80 hours** for someone without daily Azure admin experience. This plan works *if* you meet at least one of these:

- You already administer Azure day-to-day (portal + CLI/Bicep).
- You've passed AZ-900 recently and the concepts are fresh.
- You use the existing topical notes in `docs/AZURE/AZ-104/` as your primary source rather than re-reading raw docs.

If none apply, **extend to 6–8 weeks** (same 4 h/week cadence). Don't book the exam until you're hitting **80%+ on two different practice tests back-to-back** — book only after that bar is met.

---

## Exam domain weightings (2026 skills outline)

| # | Domain | Weight | Hours budgeted |
|---|---|---|---|
| 1 | Manage Azure identities & governance | 20–25% | 3.5 h |
| 2 | Implement and manage storage | 15–20% | 2.5 h |
| 3 | Deploy and manage Azure compute resources | 20–25% | 3.5 h |
| 4 | Implement and manage virtual networking | 15–20% | 2.5 h |
| 5 | Monitor and maintain Azure resources | 10–15% | 1.5 h |
| — | Practice tests + gap review | — | 2.5 h |

Total: **16 h**. The heaviest-weighted domains (1 and 3) get the most time.

---

## Existing notes in this repo (your primary source)

All files live under `docs/AZURE/AZ-104/`. This plan assumes you **read these as the main material**, not Microsoft Learn from scratch.

| # | File | Domain |
|---|---|---|
| 00 | `00-overview.md` | Map of the domain |
| 01 | `01-entra-users-and-groups.md` | D1 Identity |
| 02 | `02-access-to-azure-resources.md` | D1 RBAC |
| 03 | `03-subscriptions-and-governance.md` | D1 Governance |
| 04 | `04-storage-access-and-security.md` | D2 Storage |
| 05 | `05-storage-accounts.md` | D2 Storage |
| 06 | `06-azure-files-and-blob.md` | D2 Storage |
| 07 | `07-arm-and-bicep.md` | D3 Compute / IaC |
| 08 | `08-virtual-machines.md` | D3 Compute |
| 09 | `09-containers.md` | D3 Compute |
| 10 | `10-app-service.md` | D3 Compute |
| 11 | `11-virtual-networks.md` | D4 Networking |
| 12 | `12-network-security.md` | D4 Networking |
| 13 | `13-dns-and-load-balancing.md` | D4 Networking |
| 14 | `14-monitoring.md` | D5 Monitor |
| 15 | `15-backup-and-recovery.md` | D5 Maintain |
| 16 | `16-exam-cheatsheet.md` | Final review |

---

## Weekly plan

### Week 1 — Identity, Governance, Storage foundations (4 h)

Highest-weighted domain first, while energy is fresh.

| Block | Minutes | Material | Focus |
|---|---|---|---|
| Mon | 60 | `01-entra-users-and-groups.md` | Member vs Guest, group types, SSPR, licensing, dynamic groups (P1) |
| Wed | 60 | `02-access-to-azure-resources.md` | RBAC roles (Owner/Contributor/Reader/User Access Admin), scope inheritance, deny assignments |
| Fri | 60 | `03-subscriptions-and-governance.md` | Management groups, Policy vs Blueprint, resource locks, tags, cost |
| Sun | 60 | `05-storage-accounts.md` | Redundancy (LRS/ZRS/GRS/RA-GRS/GZRS), kinds, tiers |

**Hands-on (if time, 30 min):** Create a user, add to group, assign Reader at RG scope. Verify in portal + `az role assignment list`.

**Self-check (5 min):** Can you answer without notes — *"Dynamic group membership requires which license?"* (P1) *"Resource lock on RG — does it apply to child resources?"* (yes).

---

### Week 2 — Storage + Compute core (4 h)

| Block | Minutes | Material | Focus |
|---|---|---|---|
| Mon | 60 | `04-storage-access-and-security.md` + `06-azure-files-and-blob.md` | SAS (user delegation vs account vs service), stored access policies, blob tiers + lifecycle, soft delete/versioning, Azure Files auth |
| Wed | 75 | `08-virtual-machines.md` | Sizes, disk types (Standard HDD/SSD, Premium SSD, Ultra), availability zones vs sets, VMSS, encryption (SSE, ADE, encryption at host), Bastion |
| Fri | 45 | `10-app-service.md` | Plans (B/S/P/I), slots + swap, scaling rules, TLS, custom domains |
| Sun | 60 | `09-containers.md` + `07-arm-and-bicep.md` (skim) | ACI vs Container Apps vs AKS decision; Bicep syntax, parameters, modules, what-if deployment |

**Hands-on (if time, 30 min):** Deploy a VM via Bicep `what-if` then `deploy`. Stop/deallocate and note the cost difference.

---

### Week 3 — Networking (4 h)

Networking is where most people lose points on the exam — don't skimp.

| Block | Minutes | Material | Focus |
|---|---|---|---|
| Mon | 75 | `11-virtual-networks.md` | VNet/subnet sizing, peering (transitive? **no**), service endpoints vs private endpoints, VNet integration for App Service |
| Wed | 75 | `12-network-security.md` | NSG rule priority, default rules, ASG, effective rules, Azure Firewall vs NSG, Bastion vs JIT |
| Fri | 60 | `13-dns-and-load-balancing.md` | Public vs private DNS zones, **Load Balancer vs App Gateway vs Front Door vs Traffic Manager** (this *always* shows up) |
| Sun | 30 | Re-read `12-network-security.md` weak points | Focus on NSG rule evaluation order problems |

**Exam trap to internalize this week:** LB works at L4 (TCP/UDP). App Gateway L7 (HTTP/HTTPS, WAF, path-based). Front Door is global L7. Traffic Manager is DNS-only. If the question says "SSL offload" → App Gateway. If it says "route to nearest region" → Front Door or Traffic Manager.

---

### Week 4 — Monitor/Maintain + Practice + Gap Review (4 h)

| Block | Minutes | Material | Focus |
|---|---|---|---|
| Mon | 45 | `14-monitoring.md` | Metrics vs Logs, Log Analytics workspace, KQL basics (`where`, `summarize`, `project`), alert rules, action groups |
| Mon | 30 | `15-backup-and-recovery.md` | Recovery Services vault vs Backup vault, policies, ASR failover, cross-region restore |
| Wed | 60 | **Practice test #1** (timed, closed-book) | Use MeasureUp, Tutorials Dojo, or Microsoft's official practice assessment |
| Fri | 45 | **Review wrong answers** | For each miss: read the relevant note file section; add one line to `16-exam-cheatsheet.md` if the gap is real |
| Sun | 30 | `16-exam-cheatsheet.md` end-to-end | Speed-read; this is your day-before-exam artifact |
| Sun | 30 | **Practice test #2** (timed) | Different provider than test #1. Target **≥80%**. If below 75%, push the exam out by 1 week and add a Week 5 (see below). |

---

## If you need a Week 5 (exam not yet booked)

Trigger this if practice tests are < 75%, or if this month you missed a block.

- 1 h: Whatever domain scored lowest on practice tests (re-read + 20 extra questions in that area).
- 1 h: Bicep/ARM hands-on — author a template, deploy, modify, redeploy with `what-if`. This is heavily tested and easy to learn by doing.
- 1 h: KQL practice against a real Log Analytics workspace (create one, enable VM insights on a free-tier VM, query).
- 1 h: Final mixed practice test + cheatsheet review.

---

## Study mechanics

1. **Active recall over passive reading.** Close the file, explain the concept out loud, then check. Reading without recall wastes half these 16 hours.
2. **Portal + CLI for every major concept.** Even 2 minutes of clicking beats 20 minutes of reading. You're hitting your free Azure credit, not studying for a history exam.
3. **The cheatsheet is sacred.** Last 48 hours before the exam: only `16-exam-cheatsheet.md`. Nothing else.
4. **Flag, don't stall.** On practice tests, flag hard questions and move on. Time pressure *is* the test — 100 minutes for up to 60 items ≈ 100 seconds each, and case studies eat more.
5. **Don't buy a 40-hour video course with 16 hours to spend.** You don't have time. Videos are for Week 5+.

---

## Exam-day checklist

- Schedule 3–5 days before you intend to sit (Pearson VUE online or test center).
- Sleep > cram the night before.
- Online proctored: clear desk, no dual monitor, webcam + mic required, government photo ID.
- Read every question **twice**. Azure questions are often decided by one word (`dynamic`, `private`, `premium`, `zone-redundant`).
- Flag, finish, then return to flagged.

---

## Common high-yield exam traps (memorize verbatim)

- **Dynamic groups** → require **Entra ID P1**.
- **Conditional Access** → requires **P1**. MFA-only-for-admins is free via Security Defaults.
- **Peering is not transitive.** Hub-and-spoke needs a firewall/NVA or Virtual WAN for spoke-to-spoke.
- **NSG priority:** lower number = higher priority. Default deny-all-inbound exists at 65500.
- **Blob access tiers:** Hot / Cool / Cold / Archive. **Archive cannot be read directly** — rehydrate first (hours).
- **GRS vs ZRS:** GRS = 2 regions (async). ZRS = 3 zones in 1 region. GZRS = both.
- **User-delegated SAS** uses Entra ID (preferred). **Account SAS** uses storage key (riskier).
- **Lock types:** `CanNotDelete` vs `ReadOnly`. Inheritance flows down (RG lock → all child resources).
- **Soft delete** applies to containers and blobs *separately*. Enable both.
- **Availability Zone** ≠ **Availability Set**. Zones = datacenters (higher SLA). Sets = racks within one DC.
- **Load Balancer SKUs:** Basic retired. Standard is zone-redundant, supports AZ.
- **Private Endpoint** creates a NIC in your VNet; **Service Endpoint** just adds the VNet to the PaaS firewall. Private Endpoint is preferred.

---

## Sources

- [Microsoft Learn — AZ-104 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104)
- [Microsoft Certified: Azure Administrator Associate](https://learn.microsoft.com/en-us/credentials/certifications/azure-administrator/)
- [AZ-104 Exam Readiness Zone (video series)](https://learn.microsoft.com/en-us/shows/exam-readiness-zone/preparing-for-az-104-manage-azure-identities-and-governance-1-of-5)
- [AZ-104 MicrosoftLearning labs (GitHub)](https://github.com/MicrosoftLearning/AZ-104-MicrosoftAzureAdministrator)
- [Microsoft Learn training path — AZ-104](https://learn.microsoft.com/en-us/training/courses/az-104t00)
