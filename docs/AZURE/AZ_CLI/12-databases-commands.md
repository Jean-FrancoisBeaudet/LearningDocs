# Azure CLI — Databases

> **`az sql`, `az cosmosdb`, `az postgres flexible-server`, `az mysql flexible-server`, `az redis`.**
> **Related:** [AZURE/COSMOSDB](../COSMOSDB/00-overview.md) · [14 Security & Key Vault](14-security-and-keyvault-commands.md)
> **Sources:** [`az sql`](https://learn.microsoft.com/en-us/cli/azure/sql) · [`az cosmosdb`](https://learn.microsoft.com/en-us/cli/azure/cosmosdb) · [`az postgres flexible-server`](https://learn.microsoft.com/en-us/cli/azure/postgres/flexible-server) · [`az mysql flexible-server`](https://learn.microsoft.com/en-us/cli/azure/mysql/flexible-server) · [`az redis`](https://learn.microsoft.com/en-us/cli/azure/redis)

---

## Table of contents

1. [Azure SQL Database](#azure-sql-database)
2. [SQL firewall & connectivity](#sql-firewall--connectivity)
3. [SQL Entra admin & auditing](#sql-entra-admin--auditing)
4. [SQL elastic pools & failover groups](#sql-elastic-pools--failover-groups)
5. [Cosmos DB](#cosmos-db)
6. [Cosmos DB data-plane via CLI](#cosmos-db-data-plane-via-cli)
7. [PostgreSQL Flexible Server](#postgresql-flexible-server)
8. [MySQL Flexible Server](#mysql-flexible-server)
9. [Azure Cache for Redis](#azure-cache-for-redis)

---

## Azure SQL Database

### Server + database

```bash
# Logical server
az sql server create \
  -g rg-dev -n sqlprod01 \
  --location eastus \
  --admin-user sqladmin \
  --admin-password 'S3cureP@ss!' \
  --enable-ad-only-auth false \
  --minimal-tls-version 1.2

# Single database (vCore serverless)
az sql db create \
  -g rg-dev --server sqlprod01 --name appdb \
  --edition GeneralPurpose --family Gen5 --capacity 2 \
  --compute-model Serverless --auto-pause-delay 60 \
  --backup-storage-redundancy Zone

# DTU (legacy, sometimes cheaper for small workloads)
az sql db create -g rg-dev -s sqlprod01 -n appdb --service-objective S1

# Hyperscale
az sql db create -g rg-dev -s sqlprod01 -n big \
  --edition Hyperscale --family Gen5 --capacity 4 --read-replicas 1

# List + show
az sql server list -o table
az sql db list -g rg-dev --server sqlprod01 -o table
az sql db show  -g rg-dev --server sqlprod01 --name appdb
```

### Scale, pause, delete

```bash
# Resize
az sql db update -g rg-dev -s sqlprod01 -n appdb --family Gen5 --capacity 4

# Serverless pause / resume is automatic; force-resume by connecting or:
az sql db update -g rg-dev -s sqlprod01 -n appdb --auto-pause-delay -1    # disable auto-pause

# Restore to point-in-time
az sql db restore \
  -g rg-dev -s sqlprod01 -n appdb-restored \
  --source-database appdb \
  --time '2026-04-20T12:00:00'

# Copy
az sql db copy -g rg-dev -s sqlprod01 -n appdb \
  --dest-name appdb-copy --dest-server sqlprod01

# Delete
az sql db delete -g rg-dev -s sqlprod01 -n appdb --yes
az sql server delete -g rg-dev -n sqlprod01 --yes
```

## SQL firewall & connectivity

```bash
# Allow an IP (public endpoint)
az sql server firewall-rule create \
  -g rg-dev -s sqlprod01 -n allow-office \
  --start-ip-address 203.0.113.10 --end-ip-address 203.0.113.10

# Allow all Azure services
az sql server firewall-rule create \
  -g rg-dev -s sqlprod01 -n AllowAllAzureIps \
  --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0

# Lock down — deny public
az sql server update -g rg-dev -n sqlprod01 --set publicNetworkAccess=Disabled

# Private endpoint for sqlprod01
az network private-endpoint create \
  -g rg-dev -n pe-sql-prod \
  --vnet-name vnet-dev --subnet subnet-pe \
  --private-connection-resource-id $(az sql server show -g rg-dev -n sqlprod01 --query id -o tsv) \
  --group-id sqlServer --connection-name sqlprod01
```

- Azure SQL firewall rules overlap but **private endpoint wins** when public access is disabled.
- VNet service endpoints (`az sql server vnet-rule create`) are an older alternative — prefer private endpoints for new work.

## SQL Entra admin & auditing

```bash
# Set an Entra admin group
az sql server ad-admin create \
  -g rg-dev --server-name sqlprod01 \
  --display-name "SQL Admins" \
  --object-id <groupObjectId>

# Entra-only authentication (disables SQL login)
az sql server ad-only-auth enable -g rg-dev -n sqlprod01

# Transparent Data Encryption (always on; customer-managed key)
az sql server key create \
  -g rg-dev --server sqlprod01 \
  --kid https://kv-prod.vault.azure.net/keys/sqltde/abc123
az sql server tde-key set \
  -g rg-dev --server sqlprod01 --server-key-type AzureKeyVault \
  --kid https://kv-prod.vault.azure.net/keys/sqltde/abc123

# Auditing → Log Analytics
az sql server audit-policy update \
  -g rg-dev --name sqlprod01 \
  --state Enabled --actions 'BATCH_COMPLETED_GROUP' \
  --log-analytics-target-state Enabled \
  --log-analytics-workspace-resource-id <lawsId>

# Advanced Threat Protection
az sql server threat-policy update \
  -g rg-dev -s sqlprod01 --state Enabled \
  --email-account-admins true
```

## SQL elastic pools & failover groups

```bash
# Elastic pool (shared DTU/vCore across multiple dbs)
az sql elastic-pool create \
  -g rg-dev -s sqlprod01 -n pool-prod \
  --edition GeneralPurpose --family Gen5 --capacity 4 \
  --db-max-capacity 2 --db-min-capacity 0

# Move an existing db into the pool
az sql db update -g rg-dev -s sqlprod01 -n appdb --elastic-pool pool-prod

# Failover group (geo-replication across regions)
az sql failover-group create \
  -g rg-dev -s sqlprod01 -n fg-appdb \
  --partner-server sqlprod01-dr \
  --failover-policy Automatic --grace-period 1

az sql failover-group list -g rg-dev -s sqlprod01 -o table
az sql failover-group set-primary -g rg-dev -s sqlprod01-dr -n fg-appdb   # manual failover
```

## Cosmos DB

```bash
# Account — pick the API at create time (SQL/Core, MongoDB, Cassandra, Table, Gremlin)
az cosmosdb create \
  -g rg-dev -n cosmosprod \
  --kind GlobalDocumentDB \
  --default-consistency-level Session \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=true \
  --locations regionName=westeurope failoverPriority=1 isZoneRedundant=true \
  --enable-automatic-failover true \
  --enable-multiple-write-locations false \
  --capabilities EnableServerless               # or omit for provisioned throughput

az cosmosdb list -o table
az cosmosdb show -g rg-dev -n cosmosprod
az cosmosdb keys list -g rg-dev -n cosmosprod --type connection-strings

# Update regions (add a read region)
az cosmosdb update -g rg-dev -n cosmosprod \
  --locations regionName=eastus failoverPriority=0 \
  --locations regionName=westeurope failoverPriority=1 \
  --locations regionName=southeastasia failoverPriority=2
```

### SQL-API database + container

```bash
# Database
az cosmosdb sql database create \
  -g rg-dev -a cosmosprod -n appdb

# Container (partition key required)
az cosmosdb sql container create \
  -g rg-dev -a cosmosprod -d appdb -n orders \
  --partition-key-path "/customerId" \
  --throughput 400 \
  --indexing-policy @indexing.json \
  --unique-key-policy @unique-keys.json \
  --default-ttl -1        # per-item TTL only, no container-wide expiry

# Autoscale
az cosmosdb sql container throughput update \
  -g rg-dev -a cosmosprod -d appdb -n orders --max-throughput 4000

# Show partitions / metrics
az cosmosdb sql container show -g rg-dev -a cosmosprod -d appdb -n orders
```

### Mongo / Cassandra / Table / Gremlin

```bash
# Mongo collection
az cosmosdb mongodb database create -g rg-dev -a cosmosmongo -n appdb
az cosmosdb mongodb collection create -g rg-dev -a cosmosmongo -d appdb -n orders \
  --shard "customerId" --throughput 400

# Cassandra keyspace + table
az cosmosdb cassandra keyspace create -g rg-dev -a cosmoscass -n appks
az cosmosdb cassandra table create -g rg-dev -a cosmoscass -k appks -n orders \
  --schema @schema.json --throughput 400

# Table API (distinct from storage-account Tables)
az cosmosdb table create -g rg-dev -a cosmostbl -n audit --throughput 400
```

### RBAC (SQL API data plane)

```bash
# Grant an identity data-plane access on a Cosmos account (replaces account-key auth)
az cosmosdb sql role definition list -g rg-dev -a cosmosprod -o table

az cosmosdb sql role assignment create \
  -g rg-dev -a cosmosprod \
  --role-definition-id 00000000-0000-0000-0000-000000000002   # built-in Data Contributor
  --principal-id <spOrMiObjectId> \
  --scope "/"
```

## Cosmos DB data-plane via CLI

`az cosmosdb ... show/list` targets ARM; actual CRUD of items lives in SDKs or `az rest`. Usable today:

```bash
# Query container metrics (ARM-reported, not your data)
az cosmosdb sql container show -g rg-dev -a cosmosprod -d appdb -n orders --query resource

# Change feed start/reset (via REST, preview CLI)
az cosmosdb sql container restore \
  -g rg-dev -a cosmosprod --database-name appdb --name orders \
  --restore-timestamp '2026-04-20T12:00:00Z'
```

Write ad-hoc queries via MongoDB shell / cqlsh / Cosmos Explorer in the portal — CLI is weak here.

## PostgreSQL Flexible Server

```bash
# Create with public access off
az postgres flexible-server create \
  -g rg-dev -n pg-prod \
  --location eastus --tier GeneralPurpose --sku-name Standard_D4s_v3 \
  --storage-size 128 --storage-auto-grow Enabled \
  --version 16 \
  --admin-user pgadmin --admin-password 'Pg@ssw0rd!' \
  --public-access Disabled \
  --vnet vnet-dev --subnet subnet-pg \
  --private-dns-zone privatelink.postgres.database.azure.com \
  --high-availability ZoneRedundant \
  --backup-retention 14 --geo-redundant-backup Enabled \
  --yes

# List / show
az postgres flexible-server list -o table
az postgres flexible-server show -g rg-dev -n pg-prod

# Databases
az postgres flexible-server db create -g rg-dev -s pg-prod -d appdb

# Firewall rules (public-access servers)
az postgres flexible-server firewall-rule create \
  -g rg-dev -n pg-prod --rule-name office \
  --start-ip-address 203.0.113.10 --end-ip-address 203.0.113.10

# Entra admin (replaces AAD Admin on legacy Single Server)
az postgres flexible-server ad-admin create \
  -g rg-dev -s pg-prod \
  --object-id <userOrGroupObjectId> --display-name "PG Admins" \
  --type Group

# Server parameters
az postgres flexible-server parameter set \
  -g rg-dev -s pg-prod --name log_connections --value on

# Restart / stop / start
az postgres flexible-server stop    -g rg-dev -n pg-prod
az postgres flexible-server start   -g rg-dev -n pg-prod
az postgres flexible-server restart -g rg-dev -n pg-prod

# Restore (PITR)
az postgres flexible-server restore \
  -g rg-dev -n pg-prod-restored --source-server pg-prod \
  --restore-time '2026-04-20T12:00:00Z'

# Read replica
az postgres flexible-server replica create \
  -g rg-dev -n pg-prod-ro --source-server pg-prod --zone 2
```

## MySQL Flexible Server

```bash
az mysql flexible-server create \
  -g rg-dev -n mysql-prod \
  --location eastus --sku-name Standard_D4ds_v4 --tier GeneralPurpose \
  --version 8.0.21 --storage-size 128 \
  --admin-user mysqladmin --admin-password 'MyS@ssw0rd!' \
  --public-access Disabled \
  --vnet vnet-dev --subnet subnet-mysql \
  --private-dns-zone privatelink.mysql.database.azure.com \
  --high-availability ZoneRedundant --standby-zone 2 \
  --yes

az mysql flexible-server db create -g rg-dev -s mysql-prod -d appdb
az mysql flexible-server firewall-rule create \
  -g rg-dev -n mysql-prod --rule-name office \
  --start-ip-address 203.0.113.10 --end-ip-address 203.0.113.10

az mysql flexible-server parameter set \
  -g rg-dev -s mysql-prod --name slow_query_log --value ON

az mysql flexible-server replica create \
  -g rg-dev --name mysql-prod-ro --source-server mysql-prod
```

- Same command shape as Postgres — choose per your app's needs. Flexible servers are the modern tier; **Single Server is being retired** in March 2025.

## Azure Cache for Redis

```bash
# Create (standard tier, 1 GB)
az redis create \
  -g rg-dev -n redis-prod \
  --location eastus \
  --sku Standard --vm-size C1 \
  --redis-version 6 \
  --enable-non-ssl-port false \
  --minimum-tls-version 1.2

# Premium with clustering + Entra + persistence
az redis create \
  -g rg-dev -n redis-prem \
  --sku Premium --vm-size P2 \
  --shard-count 2 \
  --enable-entraid-auth true \
  --redis-configuration '{"rdb-backup-enabled":"true","rdb-backup-frequency":"60","rdb-storage-connection-string":"<cs>"}'

# Get access keys
az redis list-keys -g rg-dev -n redis-prod

# Firewall
az redis firewall-rules create \
  -g rg-dev -n redis-prod --rule-name office \
  --start-ip 203.0.113.10 --end-ip 203.0.113.10

# Patch schedule
az redis patch-schedule create \
  -g rg-dev -n redis-prod \
  --schedule-entries '[{"dayOfWeek":"Sunday","startHourUtc":2,"maintenanceWindow":"PT5H"}]'

# Scale / upgrade SKU
az redis update -g rg-dev -n redis-prod --set sku.capacity=2

# Flush / reboot
az redis force-reboot -g rg-dev -n redis-prod --reboot-type AllNodes
```

- Premium tier adds: persistence (RDB/AOF), geo-replication, clustering, vnet injection (preview for newer "Azure Managed Redis").
- For new work consider [Azure Managed Redis](https://learn.microsoft.com/en-us/azure/redis/overview) — different ARM surface (`az redisenterprise` for Enterprise tier).

## Gotchas

- **SQL firewall rule `0.0.0.0`** opens to all Azure — not just your subs. Combined with TLS default and public endpoint, this is the most common auditor finding.
- **Cosmos `EnableServerless` is per-account**: provisioned vs serverless is set at create time and can't switch later. Choose based on traffic profile.
- **Cosmos SQL API RBAC does not cover the emulator or cross-account federated scenarios** — data-plane role assignments target one account at a time.
- **Postgres / MySQL Flexible Server vnet mode is immutable**: public vs private is fixed at create. For a change you re-create and migrate.
- **Flexible Server private DNS zones** need to be precreated or the CLI will fail with a confusing error — pass `--private-dns-zone <zoneName>` that already exists in the same resource group tree.
- **Redis Premium shard-count changes trigger a reboot** of all nodes; schedule maintenance windows accordingly.
- **Point-in-time restore** does not overwrite the source DB; it creates a new one. Plan the rename/swap yourself.
- **SQL elastic-pool moves** between dbs are online but brief — small latency spike while the ARM PATCH commits.

## Sources

- [`az sql`](https://learn.microsoft.com/en-us/cli/azure/sql) · [SQL Database purchasing models](https://learn.microsoft.com/en-us/azure/azure-sql/database/purchasing-models)
- [`az cosmosdb`](https://learn.microsoft.com/en-us/cli/azure/cosmosdb) · [COSMOSDB notes](../COSMOSDB/00-overview.md)
- [`az postgres flexible-server`](https://learn.microsoft.com/en-us/cli/azure/postgres/flexible-server) · [Postgres Flexible docs](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/)
- [`az mysql flexible-server`](https://learn.microsoft.com/en-us/cli/azure/mysql/flexible-server) · [MySQL retirement](https://learn.microsoft.com/en-us/azure/mysql/single-server/whats-happening-to-mysql-single-server)
- [`az redis`](https://learn.microsoft.com/en-us/cli/azure/redis) · [Azure Cache for Redis](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/)
