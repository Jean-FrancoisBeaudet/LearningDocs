# Azure App Service

> **Exam mapping:** AZ-104 -- "Deploy and manage Azure compute resources" (20--25 %)
> **One-liner:** Azure App Service is the fully managed HTTP-based hosting platform for web apps, REST APIs, and mobile back ends -- you choose a runtime, push code (or a container), and Azure handles OS patching, load balancing, and scaling.
> **Related:** [07-arm-and-bicep](07-arm-and-bicep.md) | [08-virtual-machines](08-virtual-machines.md) | [09-containers](09-containers.md)

---

## 1. App Service Plans

An [App Service plan](https://learn.microsoft.com/en-us/azure/app-service/overview-hosting-plans) defines the **compute resources** (VM size, instance count, OS, region) that your web apps run on. Multiple apps can share the same plan -- they share the same VM instances. Deployment slots also consume resources from the same plan.

### Pricing model

- **Per-plan, not per-app.** You pay for every VM instance in the plan regardless of how many apps are deployed to it.
- Shared tier charges per-app CPU quota; dedicated tiers charge per VM instance.
- Free tier has no charge. SNI SSL connections are free; IP-based SSL connections carry an hourly charge (one free IP-SSL binding at Standard+).

### Tier comparison table

| Tier | SKUs | Shared / Dedicated | Custom domains | SSL/TLS | Deployment slots | Max scale-out | Autoscale | VNet integration | ASE |
|------|------|---------------------|----------------|---------|-----------------|---------------|-----------|-----------------|-----|
| **Free** | F1 | Shared | No | No | 0 | 1 (shared) | No | No | No |
| **Shared** | D1 | Shared | Yes | No (no SSL binding) | 0 | 1 (shared) | No | No | No |
| **Basic** | B1, B2, B3 | Dedicated | Yes | Yes | 0 | 3 | No | Yes | No |
| **Standard** | S1, S2, S3 | Dedicated | Yes | Yes | 5 | 10 | Yes | Yes | No |
| **Premium v3** | P0v3, P1v3, P2v3, P3v3 | Dedicated | Yes | Yes | 20 | 30 | Yes | Yes | No |
| **Premium v4** | P0v4, P1v4 -- P5mv4 | Dedicated | Yes | Yes | 20 | 30 | Yes | Yes | No |
| **Isolated v2** | I1v2 -- I6v2 | Dedicated (ASE) | Yes | Yes | 20 | 100 | Yes | Built-in | Yes |

> **Key point:** Apps in the same plan share the same VM instances. If you need to isolate compute per-app, create a separate App Service plan.

### CLI -- create a plan

```bash
az appservice plan create \
  --name myPlan \
  --resource-group myRG \
  --sku S1 \
  --location eastus \
  --is-linux          # omit for Windows
```

---

## 2. Scaling

### Scale up (vertical) vs Scale out (horizontal)

| Action | What changes | How |
|--------|-------------|-----|
| **Scale up** | Pricing tier / VM size (more CPU, RAM, disk, features) | Change the plan's SKU |
| **Scale out** | Number of VM instances running the plan | Manual count or autoscale rules |

### Manual scale out

Available on Basic+. Set a fixed instance count via the portal or CLI:

```bash
az appservice plan update --name myPlan --resource-group myRG --number-of-workers 3
```

### Autoscale (metric-based & schedule-based)

[Autoscale](https://learn.microsoft.com/en-us/azure/azure-monitor/autoscale/autoscale-overview) is an **Azure Monitor** feature. It requires **Standard tier or higher**.

**Metric-based rules** -- scale out/in when a metric crosses a threshold:
- CPU percentage, Memory percentage, HTTP queue length, Disk queue length, Data In/Out, custom metrics.
- Each rule has: metric source, operator, threshold, direction (increase/decrease), instance count change, **cooldown period** (default 5 min -- prevents flapping).

**Schedule-based rules** -- scale to a fixed instance count on a schedule (e.g., 10 instances every weekday 08:00--18:00).

**Autoscale profiles** -- combine metric rules and schedule rules:
- **Default profile** -- always active unless overridden.
- **Recurring profile** -- day-of-week / time-of-day pattern.
- **Fixed-date profile** -- specific date range (e.g., Black Friday).

> **Evaluation logic:** If multiple scale-out rules trigger, autoscale picks the **highest** resulting instance count. For scale-in, **all** rules must agree.

**Cooldown period:** After a scale action, autoscale waits (default 5 min) before evaluating again. This prevents rapid oscillation.

### Automatic scaling (HTTP-based, Premium plans)

[Automatic scaling](https://learn.microsoft.com/en-us/azure/app-service/manage-automatic-scaling) is a separate feature (Premium v2/v3+) that handles scaling based on HTTP traffic without user-defined rules. Set a **maximum burst** instance count and App Service manages the rest.

### CLI -- create autoscale

```bash
# Create an autoscale setting with a CPU-based rule
az monitor autoscale create \
  --resource-group myRG \
  --resource myPlan \
  --resource-type Microsoft.Web/serverfarms \
  --name myAutoscale \
  --min-count 1 --max-count 10 --count 2

az monitor autoscale rule create \
  --resource-group myRG \
  --autoscale-name myAutoscale \
  --condition "CpuPercentage > 70 avg 5m" \
  --scale out 2

az monitor autoscale rule create \
  --resource-group myRG \
  --autoscale-name myAutoscale \
  --condition "CpuPercentage < 30 avg 5m" \
  --scale in 1
```

---

## 3. Create an App Service

An [Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/overview) web app runs inside an App Service plan. You choose:

- **Runtime stack:** .NET, Java, Node.js, Python, PHP, Ruby, Go, or custom container.
- **Operating system:** Windows or Linux (some runtimes are Linux-only).
- **Region:** must match the App Service plan.

### CLI -- create a web app

```bash
az webapp create \
  --name myWebApp \
  --resource-group myRG \
  --plan myPlan \
  --runtime "DOTNET:8.0"    # see `az webapp list-runtimes`
```

### Deployment options

| Method | Description |
|--------|-------------|
| **Local Git** | Push from a local repo to App Service's built-in Git remote |
| **GitHub Actions** | CI/CD pipeline triggered on push; YAML workflow auto-generated |
| **Azure DevOps** | Azure Pipelines integration |
| **ZIP deploy** | Upload a ZIP package via CLI (`az webapp deploy`) or REST API |
| **Docker container** | Deploy a custom container from Docker Hub, ACR, or private registry |
| **FTP/FTPS** | Legacy file-based deployment |

### Configuration settings

- **App settings** = environment variables exposed to the app at runtime. On Windows they are injected as env vars; on Linux they are set as `--env` flags. They override values in `appsettings.json` (for .NET).
- **Connection strings** = same concept, specifically for database connections. Prefixed by type (e.g., `SQLAZURECONNSTR_`).
- Both app settings and connection strings can be marked as **slot settings** (sticky to the slot).

### Managed identity

[Managed identities](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview) eliminate the need to store credentials. App Service supports:

- **System-assigned** -- tied to the app lifecycle; deleted when the app is deleted.
- **User-assigned** -- independent lifecycle; can be shared across resources.

Use managed identity to authenticate to Key Vault, Storage, SQL Database, and other Azure services without secrets.

---

## 4. Certificates and TLS

Reference: [Add and manage TLS/SSL certificates](https://learn.microsoft.com/en-us/azure/app-service/configure-ssl-certificate)

### Certificate options

| Option | Cost | Requirements | Notes |
|--------|------|-------------|-------|
| **Free App Service managed certificate** | Free | Basic+ tier, custom domain mapped | SNI SSL only. No wildcards. No naked/root domains with Traffic Manager. No private DNS. Not exportable. Auto-renewed by DigiCert. |
| **App Service certificate** | Paid | Purchased through Azure | Azure-managed, stored in Key Vault, auto-renewable |
| **Import from Key Vault** | BYOC | PKCS12 (PFX) cert in Key Vault | App Service syncs every 24 h; auto-updates bindings |
| **Upload private certificate** | BYOC | PFX file, password-protected | Must include full chain (intermediates + root) |

### TLS/SSL bindings

| Binding type | How it works | When to use |
|-------------|-------------|-------------|
| **SNI SSL** | Uses Server Name Indication header to select cert | Default; works with shared IPs; no extra cost |
| **IP-based SSL** | Dedicates a static IP to the binding | Needed for clients that don't support SNI; carries hourly charge |

### Enforce HTTPS and minimum TLS version

- **HTTPS Only** setting (`az webapp update --https-only true`) redirects all HTTP traffic to HTTPS.
- **Minimum TLS version:** set to **1.2** (recommended) or 1.3 via portal or CLI. Older clients on TLS 1.0/1.1 will be rejected.

```bash
az webapp config set \
  --name myWebApp --resource-group myRG \
  --min-tls-version 1.2 \
  --ftps-state Disabled       # also disable unencrypted FTP
```

---

## 5. Custom DNS Names

Reference: [Map a custom DNS name](https://learn.microsoft.com/en-us/azure/app-service/app-service-web-tutorial-custom-domain)

### DNS record types

| Scenario | Record type | Name | Value |
|----------|-------------|------|-------|
| **Subdomain** (www.contoso.com) | CNAME | `www` | `myapp.azurewebsites.net` |
| **Root / apex domain** (contoso.com) | A | `@` | App's inbound IP address |
| **Domain verification** (always) | TXT | `asuid` or `asuid.www` | App's custom domain verification ID |

### Steps

1. **Verify ownership** -- create a TXT record named `asuid.<subdomain>` with the verification ID shown in the portal (or `asuid` for root).
2. **Create the DNS record** -- CNAME for subdomain, A record for root.
3. **Add the hostname in App Service:**

```bash
az webapp config hostname add \
  --webapp-name myWebApp \
  --resource-group myRG \
  --hostname www.contoso.com
```

4. **Bind a certificate** -- see section 4.

### Traffic Manager integration

When using [Azure Traffic Manager](https://learn.microsoft.com/en-us/azure/traffic-manager/traffic-manager-overview) for multi-region routing, CNAME the custom domain to the Traffic Manager profile (`contoso.trafficmanager.net`). The managed certificate for a subdomain supports Traffic Manager; the managed certificate for a root domain does **not** support Traffic Manager.

---

## 6. Backup

Reference: [Back up and restore your app](https://learn.microsoft.com/en-us/azure/app-service/manage-backup)

### What is backed up

- App configuration (app settings, connection strings)
- File content (wwwroot)
- Connected database (SQL Database, MySQL -- up to 4 GB)

Total backup size limit: **10 GB** (app + database combined).

### Requirements

| Requirement | Detail |
|-------------|--------|
| Tier | **Standard, Premium, or Isolated** |
| Storage account | A general-purpose Azure Storage account with a Blob container |
| Max backup size | 10 GB (app + DB) |

### Manual vs scheduled backup

- **Manual:** on-demand via portal or CLI.
- **Scheduled:** configure a retention policy (how many backups to keep) and a schedule (e.g., every 12 h or every day).

### Partial backups

You can exclude specific folders/files from the backup by specifying exclusion rules (e.g., old log files).

### CLI

```bash
# Configure backup
az webapp config backup create \
  --resource-group myRG \
  --webapp-name myWebApp \
  --container-url "https://mystorage.blob.core.windows.net/backups?sv=..." \
  --backup-name myBackup

# Schedule recurring backups
az webapp config backup update \
  --resource-group myRG \
  --webapp-name myWebApp \
  --container-url "https://mystorage.blob.core.windows.net/backups?sv=..." \
  --frequency 1d \
  --retain-one true \
  --retention 30
```

---

## 7. Networking

Reference: [App Service networking features](https://learn.microsoft.com/en-us/azure/app-service/networking-features)

### Inbound vs outbound networking features

| Direction | Feature | Purpose | Tier required |
|-----------|---------|---------|---------------|
| **Inbound** | [Access restrictions](https://learn.microsoft.com/en-us/azure/app-service/overview-access-restrictions) | Allow/deny by IP, service tag, service endpoint, VNet subnet | All tiers |
| **Inbound** | [Private Endpoints](https://learn.microsoft.com/en-us/azure/app-service/networking/private-endpoint) | Expose the app on a private IP inside a VNet | Premium v2+ or Isolated |
| **Inbound** | Service endpoints | Restrict traffic to specific subnets | Basic+ |
| **Outbound** | [VNet integration](https://learn.microsoft.com/en-us/azure/app-service/overview-vnet-integration) | Route outbound traffic through a VNet | Basic+ (Regional VNet Integration) |
| **Outbound** | [Hybrid Connections](https://learn.microsoft.com/en-us/azure/app-service/app-service-hybrid-connections) | TCP tunnel to on-premises without VPN | Standard+ |
| **Outbound** | NAT Gateway | Dedicated outbound IP via NAT gateway on the integration subnet | Requires VNet integration |

### VNet integration (outbound)

- **What it does:** mounts virtual interfaces on worker VMs so that outbound traffic flows through a delegated subnet in your VNet. This lets the app reach private resources (VMs, private endpoints, on-premises via ExpressRoute/VPN).
- **It does NOT provide inbound access.** For inbound, use Private Endpoints or access restrictions.
- **Subnet requirements:** a dedicated, delegated subnet (`Microsoft.Web/serverFarms`). Recommended /26 (64 addresses). Minimum /28.
- **Routing:** by default only RFC1918 (private) traffic goes through the VNet. Enable "Route All" / outbound internet traffic to send all traffic through the VNet (useful for forced tunneling via firewall/NVA).
- **Two VNet integrations per plan** (two subnets max).

### Private Endpoints (inbound)

- Places the app behind a private IP inside a VNet.
- External internet traffic is blocked; only clients with network access to the private endpoint can reach the app.
- Access restrictions **do not apply** to traffic coming through a private endpoint.

### Access restrictions

- Rule-based allow/deny list evaluated in priority order.
- Source types: **IP address ranges**, **service tags** (e.g., AzureFrontDoor.Backend), **service endpoints**, **VNet subnets**.
- Separate rule sets for the main site and the SCM/Kudu site.

### Outbound IPs

Every app has a set of **outbound IP addresses** (shown in app Properties). These can change if you scale up/down tiers. For a stable outbound IP, use VNet integration + NAT gateway.

---

## 8. Deployment Slots

Reference: [Set up staging environments](https://learn.microsoft.com/en-us/azure/app-service/deploy-staging-slots)

### Overview

Deployment slots are **live apps with their own hostnames** (`<app>-<slot>.azurewebsites.net`). They enable:

- **Staging and validation** before production.
- **Blue-green deployment** -- swap a warmed-up staging slot into production with zero downtime.
- **Instant rollback** -- swap back if something goes wrong.

### Tier requirements

| Tier | Max deployment slots |
|------|---------------------|
| Free / Shared / Basic | 0 (not supported) |
| **Standard** | **5** |
| **Premium** | **20** |
| **Isolated** | **20** |

### What swaps vs what stays

| Swapped (follows the code) | NOT swapped (stays with the slot) |
|---|---|
| App settings (unless marked slot-sticky) | Custom domain names |
| Connection strings (unless marked slot-sticky) | TLS/SSL certificates & bindings |
| Language framework versions | Scale settings |
| 32/64-bit platform, WebSockets | Always On, diagnostic log settings |
| Handler mappings, public certs | IP restrictions, CORS |
| WebJobs content | Publishing endpoints |
| Hybrid connections, service endpoints, CDN | Managed identities |
| Mounted storage (unless marked slot-sticky) | Virtual network integration |

> **Slot-sticky (deployment slot setting):** mark an app setting or connection string as a "slot setting" and it will NOT swap -- it stays with the slot. This is essential for settings like database connection strings that differ between staging and production.

> Settings marked with \* (Protocol settings/HTTPS-only, TLS version, client certs, IP restrictions, Always On, diagnostics, CORS) became slot-sticky by default. You can revert to legacy behavior with the app setting `WEBSITE_OVERRIDE_PRESERVE_DEFAULT_STICKY_SLOT_SETTINGS=0` on every slot.

### Auto-swap

When enabled on a slot, every deployment to that slot triggers an automatic swap to the target slot (typically production) after warm-up completes. Supported on Windows only (not Linux/containers).

### Traffic routing (percentage-based)

Route a percentage of production traffic to a non-production slot for A/B testing or canary releases:

```bash
az webapp traffic-routing set \
  --resource-group myRG \
  --name myWebApp \
  --distribution staging=15    # 15% of traffic to the staging slot
```

The client is pinned to a slot for the session via the `x-ms-routing-name` cookie.

### CLI

```bash
# Create a slot
az webapp deployment slot create \
  --name myWebApp \
  --resource-group myRG \
  --slot staging

# Deploy to the slot (example: ZIP deploy)
az webapp deploy \
  --name myWebApp \
  --resource-group myRG \
  --slot staging \
  --src-path app.zip

# Swap staging into production
az webapp deployment slot swap \
  --name myWebApp \
  --resource-group myRG \
  --slot staging \
  --target-slot production
```

---

## Exam Traps

1. **Free and Shared tiers do not support custom SSL/TLS bindings.** You need at least **Basic** for SSL; you need at least **Standard** for deployment slots, autoscale, and backup.
2. **Autoscale requires Standard+.** Basic supports manual scale-out only (up to 3 instances). Automatic scaling (HTTP-based) requires Premium v2/v3+.
3. **Slot swap swaps most settings -- except slot-sticky ones.** App settings and connection strings marked as "deployment slot setting" stay with the slot, not the code. TLS settings, IP restrictions, Always On, diagnostics, and CORS are now slot-sticky by default.
4. **Backup requires Standard+ AND a storage account with a blob container.** Max 10 GB (app + DB). Partial backups are possible.
5. **VNet integration is outbound only.** It lets your app reach resources inside a VNet but does NOT provide inbound private access. For inbound private access, use **Private Endpoints**.
6. **Private Endpoints are inbound only.** They expose the app on a private IP. Access restrictions do not apply to traffic through a private endpoint.
7. **Free managed certificate limitations:** does not support wildcard domains, root/apex domains with Traffic Manager, or private DNS. Not exportable. Not available in ASE.
8. **CNAME for subdomains, A record for root domains.** Both require a TXT `asuid` verification record.
9. **Apps in the same plan share resources.** Scaling the plan (up or out) affects all apps in it. Deployment slots also consume plan resources.
10. **Auto-swap is not supported on Linux / Web App for Containers.**

---

## Sources

- [App Service plans overview](https://learn.microsoft.com/en-us/azure/app-service/overview-hosting-plans)
- [App Service overview](https://learn.microsoft.com/en-us/azure/app-service/overview)
- [Scale up an app](https://learn.microsoft.com/en-us/azure/app-service/manage-scale-up)
- [Get started with autoscale](https://learn.microsoft.com/en-us/azure/azure-monitor/autoscale/autoscale-get-started)
- [Automatic scaling](https://learn.microsoft.com/en-us/azure/app-service/manage-automatic-scaling)
- [Add and manage TLS/SSL certificates](https://learn.microsoft.com/en-us/azure/app-service/configure-ssl-certificate)
- [Secure a custom DNS name with TLS/SSL binding](https://learn.microsoft.com/en-us/azure/app-service/configure-ssl-bindings)
- [Map a custom DNS name](https://learn.microsoft.com/en-us/azure/app-service/app-service-web-tutorial-custom-domain)
- [Back up and restore your app](https://learn.microsoft.com/en-us/azure/app-service/manage-backup)
- [App Service networking features](https://learn.microsoft.com/en-us/azure/app-service/networking-features)
- [VNet integration overview](https://learn.microsoft.com/en-us/azure/app-service/overview-vnet-integration)
- [Private Endpoints for App Service](https://learn.microsoft.com/en-us/azure/app-service/networking/private-endpoint)
- [Set up staging environments (deployment slots)](https://learn.microsoft.com/en-us/azure/app-service/deploy-staging-slots)
- [App Service Managed Certificate changes (July 2025)](https://learn.microsoft.com/en-us/azure/app-service/app-service-managed-certificate-changes-july-2025)
- [AZ-104 study guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104)
